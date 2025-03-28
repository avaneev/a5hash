# A5HASH - Fast Hash Function (in C/C++)

## Introduction

The `a5hash()` function available in the `a5hash.h` file implements a fast
64-bit hash function, designed for hash-table, hash-map, and bloom-filter
uses. Function's code is portable, cross-platform, scalar, zero-allocation,
is header-only, inlineable C (C++ compatible). Compatible with 32-bit
platforms, but the use there is not recommended due to a lacking performance.

This function features a very high hashing throughput for small
strings/messages (about 11 cycles/hash for 0-64-byte strings). The bulk
throughput is, however, only moderately fast (10-15 GB/s), and that is for a
purpose... All newest competing "fast" hash functions try to be fast both in
common keyed string hash-maps, and in large data hashing. In most cases,
this is done for the sake of better looks in benchmarks as such hash functions
rarely offer streamed hashing required for large data or file hashing...

`a5hash` was designed to be "ultimatively" fast only for common string/small
key data hash-maps and hash-tables, by utilizing "forced inlining" feature
present in most modern C/C++ compilers: this is easily achievable since in
compiled binary form, `a5hash` is very small - about 400 bytes, depending on
compiler. Moreover, if the default seed (0) is used, or if a constant-size
data is being hashed, this further reduces the code size and increases the
hashing throughput.

`a5hash` produces different hashes on big- and little-endian systems. This is
a deliberate design choice, to narrow down the scope of uses to run-time
hashing. If you need a reliable and fast hash function for files, with
portable hashes, the [komihash](https://github.com/avaneev/komihash) is a
great choice.

In overall, `a5hash` achieves three goals: ultimate speed at run-time hashing,
very small code size, and use of a novel mathematical construct. Compared to
most, if not all, existing hash functions, `a5hash` does not use accumulators:
the 128-bit result of multiplication is used directly as input on the next
iteration. It is most definite that mathematics does not offer any simpler way
to perform hashing than that. Also, compared to fast "unprotected" variants of
`wyhash` and `rapidhash`, `a5hash` has no issue if the "blinding
multiplication" happens - the function immediately recovers zeroed-out
`seeds`.

This function passes all [SMHasher](https://github.com/rurban/smhasher) and
[SMHasher3](https://gitlab.com/fwojcik/smhasher3) tests. The function was
also tested with the [xxHash collision tester](https://github.com/Cyan4973/xxHash/tree/dev/tests/collisions)
at various settings, with the collision statistics meeting the expectations.

This function and its source code (which is
[ISO C99](https://en.wikipedia.org/wiki/C99)) were quality-tested on:
Clang, GCC, MSVC, Intel C++ compilers; x86, x86-64 (Intel, AMD), AArch64
(Apple Silicon) architectures; Windows 11, AlmaLinux 9.3, macOS 15.3.2.

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
an issue with the original `a5hash v1`.

Thanks to Frank J. T. Wojcik and prior authors for
[SMHasher3](https://gitlab.com/fwojcik/smhasher3).
