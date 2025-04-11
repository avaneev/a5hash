# A5HASH - Fast Hash Function (in C/C++)

## Introduction

The `a5hash()` function available in the `a5hash.h` file implements a fast
64-bit hash function, designed for hash-table, hash-map, and bloom-filter
uses. Function's code is portable, cross-platform, scalar, zero-allocation,
is header-only, inlineable C (C++ compatible). Compatible with 32-bit
platforms, but the use there is not recommended due to a lacking performance.
The `a5hash32()` function provides a native 32-bit compatibility, for small
hash-tables or hash-maps.

This function features a very high hashing throughput for small
strings/messages (about 11 cycles/hash for 0-64-byte strings, hashed
repeatedly). The bulk throughput is, however, only moderately fast
(10-15 GB/s), and that is for a purpose... All newest competing "fast" hash
functions try to be fast both in common keyed string hash-maps, and in large
data hashing. In most cases, this is done for the sake of better looks in
benchmarks as such hash functions rarely offer streamed hashing required for
large data or file hashing...

`a5hash` was designed to be "ultimatively" fast only for common string/small
key data hash-maps and hash-tables, by utilizing "forced inlining" feature
present in most modern C/C++ compilers: this is easily achievable since in
compiled binary form, `a5hash` is very small - about 300-400 bytes, depending
on compiler and architecture. Moreover, if the default seed (0) is used, or if
a constant-size data is being hashed, this further reduces the code size and
increases the hashing throughput.

`a5hash` produces different hashes on big- and little-endian systems. This is
a deliberate design choice, to narrow down the scope of uses to run-time
hashing. If you need a reliable and fast hash function for files, with
portable hashes, the [komihash](https://github.com/avaneev/komihash) is a
great choice.

In overall, `a5hash` achieves three goals: ultimate speed at run-time hashing,
very small code size, and use of a novel mathematical construct. Compared to
most, if not all, existing hash functions, `a5hash` does not use accumulators
nor "compression" of variables: the 128-bit result of multiplication is used
directly as input on the next iteration. It is most definite that mathematics
does not offer any simpler way to perform high-quality hashing than that.
Also, compared to fast "unprotected" variants of `wyhash` and `rapidhash`,
`a5hash` has no issue if the "blinding multiplication" happens - the function
immediately recovers the zeroed-out "seeds".

This function passes all [SMHasher](https://github.com/rurban/smhasher) and
[SMHasher3](https://gitlab.com/fwojcik/smhasher3) tests. The function was
also tested with the [xxHash collision tester](https://github.com/Cyan4973/xxHash/tree/dev/tests/collisions)
at various settings, with the collision statistics satisfying the
expectations.

This function and its source code (which is
[ISO C99](https://en.wikipedia.org/wiki/C99)) were quality-tested on:
Clang, GCC, MSVC, Intel C++ compilers; x86, x86-64 (Intel, AMD), AArch64
(Apple Silicon) architectures; Windows 11, AlmaLinux 9.3, macOS 15.3.2.
Full C++ compliance is enabled conditionally and automatically, when the
source code is compiled with a C++ compiler.

## Usage

```c
#include <stdio.h>
#include "a5hash.h"

int main()
{
    const char s1[] = "This is a test of a5hash.";
    const char s2[] = "7 chars";

    printf( "%llx\n", a5hash( s1, strlen( s1 ), 0 )); // b163640b41959e6b
    printf( "%llx\n", a5hash( s2, strlen( s2 ), 0 )); // e49a0cc72256bbac
}
```

As a bonus, the `a5hash.h` file provides the `a5hash_umul128()`
general-purpose inline function which implements a portable unsigned 64x64 to
128-bit multiplication.

## A5HASH128

The `a5hash128()` function produces 128-bit hashes, and features a significant
performance for large data hashing - 25-30 GB/s. It is also fairly fast for
hash-map uses, but a bit slower than the `a5hash()` function.

```c
#include <stdio.h>
#include "a5hash.h"

int main()
{
    const char s1[] = "This is a test of a5hash128.";
	uint64_t h[ 2 ];

	a5hash128( s1, strlen( s1 ), 0, h );

    printf( "%llx%llx\n", h[ 0 ], h[ 1 ]); // d608834ffd24ffcc26eb486ffc018bbb
}
```

## Comparisons

The benchmark was performed using [SMHasher3](https://gitlab.com/fwojcik/smhasher3),
on Xeon E-2386G (RocketLake) running AlmaLinux 9.3. This benchmark includes
only the fastest hash functions that pass all state-of-the-art tests.
`XXH3-64` here does not, but it is too popular to not include it. `rapidhash`
is a replacement to `wyhash`. The hash functions, except `a5hash` at the
moment, are a part of the testing package.

Small key speed values are in cycles/hash, other values are in cycles/op.
`std init` and `std run` are `std::unordered_map` init and running tests,
`par init` and `par run` are `greg7mdp/parallel-hashmap` init and running
tests. All values are averages over 10 runs.

|Hash function|Small key speed|std init|std run|par init|par run|
|----         |----           |----    |----   |----    |----   |
|a5hash       |**17.41**      |**519** |**404**|294     |**277**|
|rapidhash    |18.10          |526     |430    |308     |292    |
|rust-ahash-fb|19.33          |533     |429    |286     |304    |
|XXH3-64      |21.31          |533     |428    |292     |290    |
|komihash     |22.54          |539     |434    |**285** |296    |
|polymurhash  |28.44          |537     |458    |335     |335    |

## Customizing C++ namespace

If for some reason, in C++ environment, it is undesired to export `a5hash`
symbols into the global namespace, the `A5HASH_NS_CUSTOM` macro can be defined
externally:

```c++
#define A5HASH_NS_CUSTOM a5hash
#include "a5hash.h"
```

Similarly, `a5hash` symbols can be placed into any other custom namespace
(e.g., a namespace with hash functions):

```c++
#define A5HASH_NS_CUSTOM my_hashes
#include "a5hash.h"
```

This way, `a5hash` functions can be referenced like `my_hashes::a5hash(...)`.
Note that since all `a5hash` functions have a `static inline` specifier, there
can be no ABI conflicts, even if the `a5hash.h` header is included in
unrelated, mixed C/C++, compilation units.

## A5RAND

The `a5rand()` function available in the `a5hash.h` file implements a
simple, but reliable, self-starting, and fast (`0.50` cycles/byte) 64-bit
pseudo-random number generator (PRNG) with `2^64` period. It is based on the
same mathematical construct as the `a5hash` hash function. `a5rand` passes
`PractRand` tests.

```c
#include <stdio.h>
#include "a5hash.h"

int main()
{
    uint64_t Seed1 = 0, Seed2 = 0;
    int i;

    for( i = 0; i < 8; i++ )
    {
        printf( "%llx\n", a5rand( &Seed1, &Seed2 ));
    }
}
```

Output:

```
2492492492492491
83958cf072b19e08
1ae643aae6b8922e
f463902672f2a1a0
f7a47a8942e378b5
778d796d5f66470f
966ed0e1a9317374
aea26585979bf755
```

## Thanks

Thanks to [Alisa Sireneva](https://github.com/purplesyringa) for discovering
a potential issue (however, requiring inputs longer than 1000 bytes) with the
original `a5hash v1`. The issue was fully resolved in `a5hash v5`.

Thanks to Frank J. T. Wojcik and prior authors for
[SMHasher3](https://gitlab.com/fwojcik/smhasher3).

Thanks to Chris Doty-Humphrey for
[PractRand](https://pracrand.sourceforge.net/).
