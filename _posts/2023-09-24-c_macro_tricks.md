---
title: "c macro tricks"
---

These are some useful C macro tricks that I've never seen before that
help me write more typesafe code/apis whithout having to resort to C++'s templates:

## MEM macros

Say you have your memory functions like `mem_zero`, `mem_copy`, `mem_move`, `mem_eq`, etc
which usually take two void pointers and a size as parameters.
When dealing with dynamic slices (ptr + len) I've had a few times the arguments passed wrong
because i was not taking into account the sizeof the element!

```c
// like this (which is wrong):
mem_copy(dst_array, src_array, src_array_len);

// instead of this (which correctly copies all of the array memory):
mem_copy(dst_array, src_array, src_array_len * sizeof(src_array[0]));
```

To have it be a little more typesafe, i've changed all `src` params to memory functions to take `char*` instead of `void*`
and made these macros that enhance them:

```c
#define MEM(...) (char*)ptr_mut(&(__VA_ARGS__)), sizeof(__VA_ARGS__)
#define SLICE_MEM(PTR, LEN) (char*)(PTR), ((LEN) * sizeof((PTR)[0]))
#define RANGE_MEM(START, END) (char*)(START), ((u64)((END) - (START)) * sizeof((START)[0]))
```

Which now lets you use them like so:

```c
mem_copy(dst_array, SLICE_MEM(src_array, src_array_len));

// sometimes I use ranges instead of slices as they're usually more ergonomic for code that does parsing
mem_copy(dst_array, RANGE_MEM(src_array_start, src_array_end));

// using the MEM macro also lets you pass a block of memory from an array or struct
// without having to remember to also pass the correct size
u32 my_buf[128];
mem_zero(MEM(my_buf));
struct MyStruct* my_struct_instance = alloc(...);
mem_zero(MEM(*my_struct_instance));
struct MyStruct my_struct_instance = ...;
mem_zero(MEM(my_struct_instance));
mem_zero(MEM(big_struct->zero_this));
```

If you now forget to use these macros, the compiler will raise a warning
since there would be a pointer conversion (`your_type*` to `char*`) (at least if you have a high enough warning level).
It's also ok passing `char*` even if you forget the macros as `len` is be the same as `len * sizeof(char)`.

## Typesafe variadic params

This next one is the best typesafe variadic params interface I could do in C.

In my string formatting lib, I want to be able to call it like so: `fmt(arena, "format % %", arg0, arg1);`
and have the `fmt` function be defined as `str fmt(struct Arena* arena, str format, struct FmtArg args[], u32 args_len);`.

The macro I came up with which best approximates that is:

```c
// expands into both the array of __VA_ARGS__ and its length
#define VARIADIC(TYPE, ...) ((TYPE[]){(TYPE){0}, __VA_ARGS__} + 1), \
	(sizeof((TYPE[]){(TYPE){0}, __VA_ARGS__}) / sizeof(TYPE) - 1)

// STR just converts a cstring literal to struct str {const char* ptr; u64 len;};
#define FMT(ARENA, FORMAT, ...) fmt(ARENA, STR(FORMAT), VARIADIC(struct FmtArg, __VA_ARGS__))
```

With that I can call `fmt` like
`FMT(arena, "format % %", (struct FmtArg){FMT_ARG_U64, .as_u64=37}, (struct FmtArg){FMT_ARG_F64, .as_f64=3.14})`.
Which then with some helper functions becomes: `FMT(arena, "format % %", fmt_num(37), fmt_num(3.14))`

And the why it's different is because it also works calling it without args (`FMT(arena, "just format no args")`)
even without compiler support for `##__VA_ARGS__` or `__VA_OPT__`.

That is because the array declaration inside `FMT` always has at least one element (the first is always empty)
and we do the `+ 1` before passing it to `fmt` to make it point to the first real arg.
Then we also have to compensate for it in the fmt `args_len` param by doing `- 1`
since it would be counting the empty arg otherwise.

I also use this trick to make a better api for:
```c
struct Arena* get_scratch_arena(const struct Arena* conflict_arenas[], u32 conflict_arenas_len);
```

```c
// the actual function is now `get_scratch_arena_raw`
// we can reuse the same `VARIADIC` macro here
#define get_scratch_arena(...) get_scratch_arena_raw(VARIADIC(const struct Arena*, __VA_ARGS__))
```

Which then lets us just call it like:

```c
get_scratch_arena(); // <- also works whith zero args!
get_scratch_arena(some_arena);
get_scratch_arena(some_arena, some_other_arena);
get_scratch_arena(some_arena, some_other_arena, many_many_arenas, why_do_you_need_so_much_arenas, boy_do_i_like_arenas);
```

As far as I know, though, it will not work in C++ as it does not support rvalue arrays in function param position.

## BONUS: how I do generics in C

This is not that much of a macro trick but more of one way to do generics/templates within C's limitations.
By taking advantage of the fact that C compilers always try to inline functions, we can implement them in a
"type erased" form and then reassemble the type information at call sites.

For example, here's the code for iterating a tree like data struct in a pre order fashion:

```c
const void* ptr_add(const void* ptr, u64 offset) { return (const char*)ptr + offset; }

const void*
tree_pre_next_raw(struct TreeLayout layout, const void* ptr) {
	// utility accessor macro
	#define MEMBER_PTR(PTR, OFFSET) (*(const void* const*)ptr_add(PTR, OFFSET))

	const void* next = MEMBER_PTR(ptr, layout.first_offset);
	if (!next) {
		for (const void* p = ptr; p; p = MEMBER_PTR(p, layout.parent_offset)) {
			next = MEMBER_PTR(p, layout.next_offset);
			if (next) {
				break;
			}
		}
	}
	return next;

	// cleanup defined macro
	#undef MEMBER_PTR
}

#define tree_pre_next(TYPE, PTR) ((const TYPE*)tree_pre_next_raw(TREE_LAYOUT(TYPE), (const TYPE*)(PTR)))
```

For it to actually work, we define `struct TreeLayout`.
It serves as a way to encode as data the layout of the concrete tree node struct we're iterating.

```c
struct TreeLayout {
	u32 parent_offset;
	u32 first_offset;
	u32 next_offset;
};
#define TREE_LAYOUT_EX(TYPE, PARENT, FIRST, NEXT) ((struct TreeLayout){ \
	.parent_offset = OFFSET(TYPE, PARENT), \
	.first_offset = OFFSET(TYPE, FIRST), \
	.next_offset = OFFSET(TYPE, NEXT), \
	})
#define TREE_LAYOUT(TYPE) TREE_LAYOUT_EX(TYPE, parent, first, next)
```

The `TREE_LAYOUT` is responsible for creating a `struct TreeLayout` from a type that resembles a tree.
It's also possible to use `TREE_LAYOUT_EX` if you have different member names than `parent`, `first` or `next`.
Since the `tree_pre_next` macro has access to the type being used, it has some casts in the expanded code
just to be sure the usage code is passing a pointer of the expected type.

Then we can finally use it like so:

```c
// the concrecte tree node type we will iterate
struct MyTestTreeNode {
	char some_data;
	char pad0[7];

	// NOTE: the following members do not need to be in the same order as `struct TreeLayout`
	// NOTE: nor do they need to be in the top of the struct
	struct MyTestTreeNode* parent;
	struct MyTestTreeNode* next;
	struct MyTestTreeNode* first;
};

for (const struct TestTreeNode* node = nodes; node; node = tree_pre_next(struct TestTreeNode, node)) {
	// ...
}
```

What we find is that if the compiler can see the implementation for `tree_pre_next_raw` when generating code
at the callsite, it will probably generate code as if it was a concrete implementation (like template instantiation).

I've found that the best way to help the compiler is to create a wrapper function that deals with concrete types
and then invoke the type erased version inside (which is where you usually would also insert any further custom logic).

The nice thing of this approach is that it's still okay to debug since you can always cast those `void*`
to the correct types and step through the statements as usual.
Also nice, compared to the technique of `#define T     #include "collection.h"`, is that you don't need to
isolate every piece of generic code into several files and of course you don't need to predeclare all instantiations.

## Conclusion

These are macros that were born out of the need to have better type safety within the limitations of C.
They are some way to sprinkle a little bit of static analysis while also enabling reasonable usage code.

All that said, I still think it's best to limit the use of macros to only
these "type safety language enhancements" and try to not go crazy.
Like, we don't try to hide the wrapped function, we never do control flow inside macros, and try our best
to keep the macro bodies to one liners.

You can find all of these examples in their "real" form in my C foundation lib:
https://github.com/vamolessa/foundation/.
