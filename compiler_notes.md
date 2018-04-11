Whenever I try to compile recc in Linux runlevel 5, the GUI gets stuck (old Laptop orz).
Thus I work in runlevel 3, without GUI and Chinese IM.
That's why this article is in English.

# Notes on the build system
## Basic workflow
See the [one page explaination](http://recc.robertelder.org/details/).

Basically, it's like `*.[ch] -> *.i -> *.l2 -> l1 -> machine`,
but the machine instructions is not like a kernel image but rather a big C array.
Anyway they are equivalent.

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
       map<uchar*, struct entity> targets; // registered entities
       auto relationships;                 // just maintains a set of relationships <ent_a, ent_b, ent_rel>
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






## structs as translated

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




















# Initial commit

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
