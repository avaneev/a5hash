# A5HASH - Fast Hash Function (in C/C++)

## Introduction

The `a5hash()` function available in the `a5hash.h` file implements a fast
64-bit hash function, designed for hash-table, hash-map, and bloom-filter
uses. Function's code is portable, cross-platform, scalar, zero-allocation,
is header-only, inlineable C (C++ compatible). Provided 64-bit and 128-bit
hash functions of the `a5hash` family are compatible with 32-bit platforms,
but the use of them there is not recommended due to a lacking performance.
On 32-bit platforms it is recommended to use the available `a5hash32()`
function which provides a native 32-bit compatibility, if 32-bit hash values
are enough.

This function features a very high hashing throughput for small
strings/messages (about 11 cycles/hash for 0-64-byte strings, hashed
repeatedly on Zen5). The bulk throughput is, however, only moderately fast
(15 GB/s), and that is for a purpose... All newest competing "fast" hash
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

Note that this function is not cryptographically-secure: in open systems, and
within any server-side internal structures, it should only be used with a
secret seed, to minimize the chance of a collision attack (hash flooding).
Compared to fast "unprotected" variants of `wyhash` and `rapidhash`, `a5hash`
has no issue if the "blinding multiplication" happens - the function
immediately recovers the zeroed-out "seeds" (see below).

`a5hash` produces different hashes on big- and little-endian systems. This is
a deliberate design choice, to narrow down the scope of uses to run-time
hashing and embedded storage since endianness-correction usually imposes a
20% performance penalty. If you need a reliable and fast hash function for
files, with portable hashes, [komihash](https://github.com/avaneev/komihash)
is a great choice.

In overall, `a5hash` achieves three goals: ultimate speed at run-time hashing,
very small code size, and use of a novel mathematical construct. Compared to
most, if not all, existing hash functions, `a5hash` does not use accumulators
nor "compression" of state variables: the 128-bit result of multiplication is
used directly as input on the next iteration. It is most definite that
mathematics does not offer any simpler way to perform high-quality hashing
than that.

This function passes all [SMHasher](https://github.com/rurban/smhasher) and
[SMHasher3](https://gitlab.com/fwojcik/smhasher3/-/tree/main/results?ref_type=heads)
tests. The function was also tested with the
[xxHash collision tester](https://github.com/Cyan4973/xxHash/tree/dev/tests/collisions)
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

int main(void)
{
    const char s1[] = "This is a test of a5hash.";
    const char s2[] = "7 chars";

    printf( "%016llx\n", a5hash( s1, strlen( s1 ), 0 )); // b163640b41959e6b
    printf( "%016llx\n", a5hash( s2, strlen( s2 ), 0 )); // e49a0cc72256bbac
}
```

As a bonus, the `a5hash.h` file provides the `a5hash_umul128()`
general-purpose inline function which implements a portable unsigned 64x64 to
128-bit multiplication.

## A5HASH-128

The `a5hash128()` function produces 128-bit hashes, and, compared to 64-bit
`a5hash()` function, features a significant performance for large data
hashing - 50 GB/s on Zen5. It is also fairly fast for hash-map uses, but a bit
slower than the `a5hash()` function. Among 128-bit hashes that pass the
state-of-the-art tests, it's likely the fastest 128-bit hash function for
hash-maps.

By setting function's `rh` pointer argument to 0, it is also possible to use
the function as a 64-bit hash function.

```c
#include <stdio.h>
#include "a5hash.h"

int main(void)
{
    const char s1[] = "This is a test of a5hash128.";
    uint64_t h[ 2 ];

    h[ 0 ] = a5hash128( s1, strlen( s1 ), 0, h + 1 );

    printf( "%016llx%016llx\n", h[ 0 ], h[ 1 ]); // 98c5ac0564ade8309501bd1fb4535b32
}
```

## Blinding Multiplication

The "blinding multiplication" (BM) is a common issue of almost all fast hash
functions based on NxN bit multiplication involving 2\*N bit input per
iteration. BM happens when one of the input values coincide with hash
function's current state or constants, and yields zero result.

While, statistically speaking, hash function's state may transitively become
zero, and this is a probable outcome, it becomes problematic, if hash function
can't recover from it quickly, and retain the seeded state. `a5hash` instantly
recovers from BM (via addition of `val01` and `val10` constants), with its
further state being unknown, if the `UseSeed` is unknown. On the contrary,
the "unprotected" `rapidhash` does not recover the seed - it becomes zero,
and thus the state becomes known.

Another issue is that NxN bit multiplication that yields zero discards a half
of 2\*N bit input completely, potentially leading to collisions. The
"protected" `rapidhash` tries to tackle this issue, but doing so, it creates
another issue: combining of inputs on adjacent iterations without
transformation, which also yields collisions.

However, one should consider the probability of BM happening on practical
inputs. `a5hash` state (`Seed1` and `Seed2`) is close to uniformly-random at
all times, which means only purely random input can trigger BM with an
expected probability. Textual, sparse, or otherwise structured inputs have a
negligible chance of BM happening: they form an implicit sieve which makes
many states completely resistant to BM. In this case, the expected linear
probability of arbitrary input match to a seed should be multiplied by
independent probability of not matching due to sieve.

When the `UseSeed` is unknown, and when the output of the hash function is
unknown (as in the case of server-side structures), for an attaker it is
simply impossible to know immediately, whether or not BM was ever triggered on
any hash function's iteration (side-channel timing attack requires hash
collisions to accumulate quickly to be measurable).

To sum up, `a5hash` is probabilistically resistant to blinding multiplication
in general case, when hash function's `UseSeed` is unknown and hash function's
output is not exposed.

## Comparisons

The benchmark was performed using [SMHasher3](https://gitlab.com/fwojcik/smhasher3),
on Xeon E-2386G (RocketLake) running AlmaLinux 9.3. This benchmark includes
only the fastest hash functions that pass all state-of-the-art tests.
`XXH3-64` here does not, but it is too popular to not include it. `rapidhash`
is a replacement to `wyhash`.

Small key speed values are in cycles/hash, other values are in cycles/op.
`std init` and `std run` are `std::unordered_map` init and running tests,
`par init` and `par run` are `greg7mdp/parallel-hashmap` init and running
tests. All values are averages over 10 runs.

|Hash function|Small key speed|std init|std run|par init|par run|
|----         |----           |----    |----   |----    |----   |
|a5hash       |**17.20**      |**523** |**404**|294     |**277**|
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

int main(void)
{
    uint64_t Seed1 = 0, Seed2 = 0;
    int i;

    for( i = 0; i < 8; i++ )
    {
        printf( "%016llx\n", a5rand( &Seed1, &Seed2 ));
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

## Why A5?

The constants `0xAAAA...` and `0x5555...` used in `a5hash` and `a5rand`
represent repetitions of `10` and `01` bit-pairs. While they do not look
special as adders in PRNGs, or even look controversial due to absense of
spectral information, they have an important property of effectively restoring
uniformity of any number.

Consider this code, which calculates sum of successive bit independences of a
set of numbers, and same of numbers XORed with `0xAAAA` constant, if a
number's bit independence is low:

```c
#include <stdint.h>
#include <stdio.h>

int main(void)
{
    int c1[ 6 ] = { 0 };
    int c2[ 6 ] = { 0 };
    for( int i = 0; i < ( 1 << 16 ); i++ )
    {
        for( int j = 0; j < 6; j++ )
        {
            int c = __popcnt(( i ^ ( i << ( 1 + j ))) >> j );
            if( c < 8 - j )
            {
                int i2 = i ^ 0xAAAA;
                c1[ j ] += c;
                c2[ j ] += __popcnt(( i2 ^ ( i2 << ( 1 + j ))) >> j );
            }
        }
    }
    for( int j = 0; j < 6; j++ )
    {
        printf( "%i %i\n", c1[ j ], c2[ j ]);
    }
}
```

Output:

```
84048 164128
58344 66338
21331 54049
5824 8607
1273 5533
202 574
```

## Thanks

Thanks to [Alisa Sireneva](https://github.com/purplesyringa) for discovering
a potential issue (however, requiring inputs longer than 1000 bytes) with the
original `a5hash v1`. The issue was fully resolved in `a5hash v5`.

Thanks to Frank J. T. Wojcik and prior authors for
[SMHasher3](https://gitlab.com/fwojcik/smhasher3).

Thanks to Chris Doty-Humphrey for
[PractRand](https://pracrand.sourceforge.net/).

## No Thanks

Reddit for discouragement.
