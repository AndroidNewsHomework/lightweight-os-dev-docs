Whenever I try to compile recc in Linux runlevel 5, the GUI gets stuck (old Laptop orz).
Thus I work in runlevel 3, without GUI and Chinese IM.
That's why this article is in English.

# Notes on the build system
## Basic workflow
See the [one page explaination](http://recc.robertelder.org/details/).

Basically, it's like `*.[ch] -> *.i -> *.l2 -> l1 -> machine`,
but the machine instructions is not like a kernel image but rather a big C array.
Anyway they are equivalent.

Citing a graph in op-cpu-programmer-reference-manual,
```
                     main.c  code.c  foo.c              C source code
                       |       |       |
                       v       v       v
                     main.i  code.i  foo.i              Preprocessed
                       |       |       |
                       v       v       v
                     main.l2 code.l2 foo.l2             Assembly source -> *.s
                          \    |    /
                           v   v   v
      Object file -> *.o   program.l1--+---------------+---------------+
                                       |               |               |
                                       |               |               |
                                       v               v               v
    Object emulator-readable     program.l0.js    program.l0.c    program.l0.py
```

Be noted that .l2 and .l1 files contain not only instructions and data but also
directives, most frequently such as SW and DW.

## On the register-dependency mechanism
This seems to be the Makefile of recc, but it is written in C.
For example, the kernel is not made by executing `make`.
Rather, `build_kernel.c` containing all the build information
(dependencies, file types etc) will be compiled into executable
`build_kernel`.
This `build_kernel` is our make program, kernel image is built
by executing it.

Specially, the dependencies of a .l2 file should either be a .i file (asm generation)
or multiple .l2 files (linking them together).
You may get a copy of the structured dependency map by making a request to
[Hoblovski](https://github.com/Hoblovski).

Furthermore, you might encounter concepts such as phase1, phase2 and phase3 while
reading recc-implementation. Note that when building the kernel, only functions in phase3
are used.
Phase1 and phase2 simply create necessary data structure to 'bootstrap' recc.

# How the kernel image is build
```sh
recc/:$ make kernel/build_kernel
recc/:$ kernel/build_kernel
```

The executable `build_kernel` makes the kernel image.

## The build\_kernel file
The following code generates kernel image:
```
void build_tests(void){
    struct build_state * state = create_build_state();
    register_libc_objects(state);
    register_builtin_objects(state);
    register_compiler_objects(state);
    register_kernel_objects(state);
    new_register_data_structures_objects(state);
    construct_generated_c_entities(state);

    construct_entity(state, "test/kernel.l0.js");

    destroy_build_state(state);
}
```

### The build\_state variable
```
struct build_state{
       auto memory_pool_collection;        // custom allocator, for performance reasons only
       map<uchar*, struct entity*> targets; // registered entities
       auto relationships;                 // a series of relationships <ent_a, ent_b, ent_rel>
};
```

### register\_libc\_objects(state)
Contains a series of

* `register_entity(state, ent_name, entity_type);`
add a new entity `ent_name` to `state->targets`.

* `add_entity_attribute(state, ent_name, attr_key_str, attr_val_str);`
    add a attribute `<attr_key_str, attr_val_str>` to ent's attributes.
    (lookup `state->targets` to find ent corresponding to `ent_name`)

* `register_dependency(state, ent_name_a, ent_name_b);`
    just add the relationship: a DEPENDS_ON_AT_BUILD_TIME b, b IS_DEPENDED_ON_AT_BUILD_TIME_BY b

libc objects includes `libc/*.c`

### register\_builtin\_objects
Also a series of commands like above.

builtin objects include `builtin/*.c` and `builtin/l2/*.l2`.

### register\_compiler\_objects
Registers the filesystem impl and data structures impl.

Indirectly but equivalently (with a wrapper `register_ri_object`), also contains a series of commands above.

compiler objects include the following files in recc-implementation
* io
* preprocessor
* code\_generator
* parser
* linker
* lexer
* memory\_pool\_collection
* regex\_engine
* type\_engine
* replace\_tool
* binary\_exponential\_buffer

### register\_kernel\_objects
Includes entity definition and dependency of the kernel.


### new\_register\_data\_structures\_objects
Do a bunch of `make_map`, `make_list`, `make_merge_sort` etc,
generating data structures and algorithms.

Do a big bunch of `register_generated_type`.
Seems to copy `types/**/*.h` into `generated/*.h` just adding proper include's.

Files in `recc/` and also `kernel/filesystem.c` include some of these generated headers.

### construct\_generated\_c\_entities
```
    for e in state->entity
        if(e->registered && e->type == ENTITY_TYPE_GENERATED_C_CODE)
            construct_entity(state, (const char *)e->name);
```
This generates the C codes in `generated/`.

### construct_entity(state, "test/kernel.l0.js")
This not only builds `test/kernel.l0.js` but also all of its dependencies,
including the kernel image `kernel/kernel.l1`.

### The construct\_entity(state, ent\_name) function
Basically this function builds the dependencies of `ent`,
and then build if necessary `ent` using `make_target(state, ent)`.

`make_target` simply does 2 things:

* `make_header_target(state, target)`

* `make_c_compiler_target(state, target)`
Depending on the type of `target`:

* .l1 file: only 1 case: linking (.l2's->.l1).
This is done by `do_link`, with (possibly) page aligned and (possibly) metadata only.

* .l2 file: only 2 cases: asm translating (.i->.l2),  or linking (.l2's->.l2).
The former delegate to `do_code_generation`, the latter to `do_link`.

* .l0 file: simply translating .l1 file into .l0 file. No need to look at.
Delegate to `l0_generator_state_create`.

* filesystem impl file: in fact wtf is this a single entity type?
_TODO WTF_

* .i file: dependencies should be a single .c file.
Delegate to `do_preprocess`.

* .c file: `assert(0)`, not supported.

### The do\_code\_generation function
This serve as the core of the compiling system.
Source code (.i) to assembly code (.l2) is done here.

_TODO_

### The do\_link function
Links multiple assembly / object files together.

This function first lex and parse the input file,
does things like resolving symbols. Then it transfers control
to either `do_link_to_l1` or `do_link_to_l2`.

* do\_link\_to\_l1
    1. Reorder input files by inserting relocatables into the gaps of non-relocatables.
        Then determine exact position of symbols.
    2. Write offset information (simply `OFFSET 0xXXXXXXXX`)
    3. Write region information. (`REGION ... START ... END ... PERMISSION ...`)
        A region is a contigous area of memory that contains objects of the same category and correspond to some permission,
        either variable (rw), constant (r) or function (rx).
    4. Export symbol information. (`EXTERNAL ... STRING ...`)
        Contains symbol id and symbol name (represented in null-terminated string).
    5. Some stupid checks on object file overlaps and empty object files.
    6. Write program instructions.
    7. _TODO_  `output_symbols`

* do\_link\_to\_l2
    1. Write offset information.
    2. Export symbols (`FUNCTION VARIABLE CONSTANT ...`)
    3. Requirements of symbols (`REQUIRES IMPLEMENTS ... INTERNAL EXTERNAL`)
    4. Write program instructions.
    5. _TODO_  `output_symbols` ?

With the above description, it's easy to see that the kernel image structure

* `OFFSET 0x0`
	kernel image starts from memory address 0

* `REGION 0x0; START 0xXXXXXXXX; END 0xXXXXXXXX; PERMISSION 0xX;`
	memory area specification.

* `EXTERNAL 0x2; STRING 0x646E655F ...`
	symbols this object exports to the external environment, so other programs can use it,
	such as `_end_unblock_tasks_for_event`.<br>
	Disabling this section does not prevent the kernel from running correctly,
	while reducing 10% of kernel image size.

* `add SP PC ZR; beq ZR ZR 1; DW 0x109400; add SP SP WR; ...`
	the instructions. Of course they take up more than 90% of the kernel image.

* `DW 0xXXXXXXXX; DW 0xXXXXXXXX ...`
	data area. _TODO_ probably `output_symbols`?


















------------------------------------------------------------------------------

# Pseudo codes for important functions

## do\_code\_generation
```
do_code_generation(in_file, out_file, offset) -> err0
/*                  .i       .l2     RELOCATABLE    */
    struct parser_state parser_state;
    struct code_gen_state state;
    struct c_lexer_state c_lexer_state;

    unsigned_char_list_create(&lexer_output);
    unsigned_char_list_create(&preprocessed_input);
    unsigned_char_list_create(&generated_code);
    unsigned_char_list_create(&buffered_symbol_table);

    add_file_to_buffer(&preprocessed_input, (char*)in_file);


    /* LEX */
    create_c_lexer_state(&c_lexer_state, &lexer_output, memory_pool_collection, in_file, unsigned_char_list_data(&preprocessed_input), unsigned_char_list_size(&preprocessed_input));
    lex_c(&c_lexer_state);


    /* PARSE */
    struct type_engine_state type_engine;
    create_type_engine_state(&type_engine, memory_pool_collection);
    create_parser_state(&parser_state, memory_pool_collection, &c_lexer_state, &generated_code, unsigned_char_list_data(&preprocessed_input), &type_engine);
    parse(&parser_state);


    /* CODEGEN */
    create_code_gen_state(&state, &parser_state, &generated_code, &buffered_symbol_table);
    generate_code(&state);


    /* SYMBOL INFORMATION */
    /* OFFSET */
    buffered_printf(state.buffered_symbol_table, "OFFSET %s;\n", offset);
    /* FUNC/VAR/CONST */
    for (linker_object* obj : state.object_declarations) {
        switch(obj->type){
            case L2_FUNCTION:{
                buffered_printf(state.buffered_symbol_table, "FUNCTION %s %s;\n", obj->start_label, obj->end_label);
                break;
            }case L2_VARIABLE:{
                buffered_printf(state.buffered_symbol_table, "VARIABLE %s %s;\n", obj->start_label, obj->end_label);
                break;
            }case L2_CONSTANT:{
                buffered_printf(state.buffered_symbol_table, "CONSTANT %s %s;\n", obj->start_label, obj->end_label);
                break;
            }default:{
                assert(0 && "Unknown type.");
            }
        }
    }
    /* REQ/IMPL INT/EXT */
    for (linker_symbol* symbol : state.symbols) {
        if(symbol->is_required && symbol->is_implemented){
            buffered_puts(state.buffered_symbol_table, "REQUIRES, IMPLEMENTS");
        }else if(symbol->is_required){
            buffered_puts(state.buffered_symbol_table, "REQUIRES");
        }else if(symbol->is_implemented){
            buffered_puts(state.buffered_symbol_table, "IMPLEMENTS");
        }else{
            assert(0 && "Symbol is not required or implemented?");
        }

        if(symbol->is_external){
            buffered_puts(state.buffered_symbol_table, " EXTERNAL ");
        }else{
            buffered_puts(state.buffered_symbol_table, " INTERNAL ");
        }

        buffered_printf(state.buffered_symbol_table, "%s\n", key);
    }


    /* writing to output .l2 file */
    unsigned_char_ptr_list_destroy(&keys);
    unsigned_char_list_add_all_end(&buffered_symbol_table, &generated_code);
    output_buffer_to_file(&buffered_symbol_table, (char *)out_file);


    /* cleanup */
    destroy_code_gen_state(&state);
    destroy_parser_state(&parser_state);
    destroy_type_engine_state(&type_engine);
    destroy_c_lexer_state(&c_lexer_state);
    unsigned_char_list_destroy(&buffered_symbol_table);
    unsigned_char_list_destroy(&lexer_output);
    unsigned_char_list_destroy(&preprocessed_input);
    unsigned_char_list_destroy(&generated_code);
```

















------------------------------------------------------------------------------

# structs as translated

```
struct entity
    map<uchar*, uchar*> attributes;
    uchar* name;
    enum type = {
        GENERATED_C_CODE,
        SOURCE,
        L2_FILE,
        L1_FILE,
        L0_FILE,
        C_FILE,
        OBJECT_FILE,
        PREPROCESSED_FILE,
        FILESYSTEM_IMPLEMENTATION,
        SYMBOL_FILE
    }
    bool satisfied == isbuilt?
    bool registered

struct entity_type_relationship
    struct entity* entity;
    enum type == {
        DEPENDS_ON_AT_BUILD_TIME,
        IS_DEPENDED_ON_AT_BUILD_TIME_BY,

        CONTAINS,
        IS_CONTAINED_BY,

        INCLUDES,
        IS_INCLUDED_BY
    }
```



















------------------------------------------------------------------------------

# Reading Initial commit

__TODO: maybe this is useless? delete later__

struct memory_pooler
    相当于自定义的 allocator, 使用类似 vector 的指数倍增算法加速分配
        struct memory_pooler
            一个 allocator 的配置
            object_size:
                这个 allocator 分配的内存块只有这么大

    void create(struct memory_pooler *, unsigned object_size);
    void destroy(struct memory_pooler *);
    void* malloc(struct memory_pooler *);
    void free(struct memory_pooler *, void *);

struct memory_pooler_collection
    void create(struct memory_pooler_collection *);
    void destroy(struct memory_pooler_collection *);
    struct memory_pooler* get_pool(struct memory_pooler_collection *, unsigned int object_size);
        如果已经有这样的 pooler 返回, 否则创建一个


lexer.c:
    unsigned t_space(struct common_lexer_state * common_lexer_state, unsigned int tentative_position)
        @common_lexer_state:
            contains the file and till where we've parsed, holds buffer
        @returns:
            number of bytes advanced. The region gone through are all spaces.
            could be 0 (tentative_position failed), or greater than 0

    int lex_c(struct c_lexer_state * c_lexer_state, unsigned char * filename, unsigned char * buffer, unsigned int buffer_size){
        @buffer, buffer_size:
            used as buffer of c_lexer_state
        @filename:
            used also to initialize c_lexer_state
        @returns:
            0 if successful
            1 if failed
        @c_lexer_state:
            put the tokens into it
                `struct_c_lexer_token_ptr_list_add(&c_lexer_state->tokens, new_token);`
