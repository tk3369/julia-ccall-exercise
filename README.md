# julia-ccall-exercise

Here's how to access complex struct.  Couple notes:

* julia does not suppot union hence workaround below
* c compiler pad the struct for memory alignment reasons
* anything union types may be represented as NTuple{N, Cuchar} for later reinterpretation
* any padding may be represented as NTuple{N, Cuchar}
* deference Ptr{T} using `unsafe_load` function
* convert Tuple{UInt8} to Vector{UInt8} using `collect` function
* deference Cstring using `unsafe_string` function

```julia

julia> const TEST = "/tmp/libtest.so"
"/tmp/libtest.so"

julia> struct DEFC
         mess1::NTuple{8, Cuchar}  # double, char*, or struct def*
         a::UInt16
         b::UInt16
         pad2::NTuple{4, Cuchar}
         v::Int16
         pad3::NTuple{6, Cuchar}
       end

julia> z = ccall((:make4, TEST), DEFC, ())
DEFC((0xe0, 0x8f, 0x73, 0xd9, 0xf5, 0x7f, 0x00, 0x00), 0x0002, 0x0003, (0xf5, 0x7f, 0x00, 0x00), 1, (0x00, 0x00, 0x00, 0x00, 0x00, 0x00))

julia> p = reinterpret(Ptr{DEFC}, collect(z.mess1))[1]
Ptr{DEFC} @0x00007ff5d9738fe0

julia> pz = unsafe_load(p, 2)
DEFC((0x37, 0x1f, 0x98, 0x0d, 0x01, 0x00, 0x00, 0x00), 0x6f74, 0x0072, (0x00, 0x00, 0x00, 0x00), 7, (0x00, 0x00, 0x01, 0x00, 0x03, 0x00))

julia> cp = reinterpret(Cstring, collect(pz.mess1))[1]
Cstring(0x000000010d981f37)

julia> unsafe_string(cp)
"abc"
```

C program:
```c
#include <stdlib.h>
#include <stdio.h>

typedef struct __attribute__((__packed__)) abc
{
		union {
				unsigned int a;
				long b;
		} ab;
		int c;
} ABC, *ABCP;

ABC make1()
{
		ABC x;
		x.c = 1;
		x.ab.b = 2;
		return x;
}

ABC make2()
{
		ABC x;
		x.c = 1;
		x.ab.a = 2;
		return x;
}

ABCP make3()
{
		ABCP x = malloc(2 * sizeof(ABC));
		x->c = 1;
		x->ab.a = 2;
		printf("x=%p\n", x);
		ABCP y = x + 1;
		y->c = 1;
		y->ab.b = 2;
		printf("y=%p\n", y);
		printf("size of ABC = %lu\n", sizeof(ABC));
		return x;
}

typedef struct def
{
		union  {
				double x;   /* 8 bytes */
				char *y;    /* 8 bytes as for 64-bit system */
				struct  {    
						struct def *array; /* 8 bytes */
						unsigned short a;  /* 2 bytes */
						unsigned short b;  /* 2 bytes + padded 4 bytes */
				} z;
		} w;
		unsigned short v;  /* 2 bytes -> padded to 8 bytes (64-bit system) */
} DEF, *DEFP;

DEF make4()
{
		DEF x;
		x.v = 1;
		x.w.z.a = 2;
		x.w.z.b = 3;
		x.w.z.array = malloc(2 * sizeof(DEF));
		x.w.z.array[0].v = 5;
		x.w.z.array[0].w.x = 6;
		x.w.z.array[1].v = 7;
		x.w.z.array[1].w.y = "abc";
		return x;
}

DEF make5()
{
		DEF x;
		x.v = 1;
		x.w.x = 3.0;
		return x;
}

int main(int argc, char **argv) {
		printf("size of DEF = %lu\n", sizeof(DEF));
		printf("size of unsigned short = %lu\n", sizeof(unsigned short));
		printf("size of struct def * = %lu\n", sizeof(struct def *));
		printf("size of char * = %lu\n", sizeof(char *));
		printf("size of double = %lu\n", sizeof(double));
}

```
