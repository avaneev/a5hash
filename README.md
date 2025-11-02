# A5HASH - Fast Hash Function (in C/C++)

## Introduction

The `a5hash()` function available in the `a5hash.h` file implements a fast
64-bit hash function, designed for hash-table, hash-map, and bloom-filter
use cases. The function's code is portable, cross-platform, scalar,
zero-allocation, header-only, inlinable C (compatible with C++). The provided
64-bit and 128-bit hash functions of the `a5hash` family are compatible with
32-bit platforms, but their use there is not recommended due to a reduced
performance. On 32-bit platforms it is recommended to use the available
`a5hash32()` function which provides a native 32-bit compatibility, if 32-bit
hash values are sufficient.

This function achieves very high hashing throughput for small strings/messages
(about 11 cycles/hash for 0-64-byte strings, hashed repeatedly on Zen5).
The bulk throughput is, however, only moderately fast (15 GB/s on Ryzen
9950X), and that is intentional. Most recent competing "fast" hash functions
aim to perform well both for common keyed string hash-maps and large data
hashing. In most cases, this is done for the sake of better benchmark results,
even though such hash functions rarely offer the streamed hashing required for
large data or file hashing...

`a5hash()` was designed to be ultimately fast only for common string/small key
data hash-maps and hash-tables, by utilizing the "forced inlining" feature
present in most modern C/C++ compilers: this is easily achievable since in
compiled binary form, `a5hash()` is very small - about 300 bytes on x86-64 and
arm64. Moreover, if the default seed (0) is used, or if data of compile-time
constant size is hashed, this further reduces the code size and increases
hashing throughput.

Note that this function is not cryptographically-secure: in open systems and
within server-side internal structures, it should only be used with a secret
seed, to minimize the chance of a successful collision attack (hash flooding).
If an occasional "blinding multiplication" occurs, `a5hash` hash functions
immediately recover from the zeroed-out state (see below).

`a5hash` hash functions produce different hashes on big- and little-endian
systems. This is a deliberate design choice to narrow down the scope of use
cases to run-time hashing and embedded storage, as endianness-correction
usually imposes a 20% performance penalty. If you need a reliable and fast
hash function for files, with portable hashes, [komihash](https://github.com/avaneev/komihash)
is a great choice.

Overall, `a5hash` achieves three goals: ultimate speed at run-time hashing,
very small code size, and use of a novel mathematical construct. Compared to
most, if not all, existing hash functions, `a5hash` hash functions do not use
accumulators nor "compression" of state variables: the full 128-bit result of
multiplication is used directly as input on the next iteration. It appears
unlikely that mathematics offers a simpler way to perform high-quality hashing
(in the `SMHasher` sense) than this. Such conclusion stems from the fact that
the hash function is structurally simple and a further reduction of the number
of operations seems impossible, while other existing similarly simple
constructs like `xorshift` do not provide high-quality hashing.

This function passes all [SMHasher](https://github.com/rurban/smhasher) and
[SMHasher3](https://gitlab.com/fwojcik/smhasher3/-/tree/main/results?ref_type=heads)
tests. The function was also tested using the
[xxHash collision tester](https://github.com/Cyan4973/xxHash/tree/dev/tests/collisions)
at various settings, with collision statistics meeting theoretical
expectations.

This function and its source code (which conforms to
[ISO C99](https://en.wikipedia.org/wiki/C99)) were quality-tested on:
Clang, GCC, MSVC, Intel C++ compilers; x86, x86-64 (Intel, AMD), AArch64
(Apple Silicon) architectures; Windows 11, AlmaLinux 9.3, macOS 15.3.2.
Full C++ compliance is enabled conditionally and automatically when the
source code is compiled with a C++ compiler.

## Usage

```c
#include <stdio.h>
#include "a5hash.h"

int main(void)
{
    const char s1[] = "This is a test of a5hash.";
    const char s2[] = "7 chars";

    printf( "%016llx\n", a5hash( s1, strlen( s1 ), 0 )); // 263ac96450b128bc
    printf( "%016llx\n", a5hash( s2, strlen( s2 ), 0 )); // e49a0cc72256bbac
}
```

As a bonus, the `a5hash.h` file provides the `a5hash_umul128()`
general-purpose inline function which implements a portable unsigned 64x64 to
128-bit multiplication.

## A5HASH-128

The `a5hash128()` function produces 128-bit hashes, and, compared to 64-bit
`a5hash()` function, features significant performance for large data hashing -
50 GB/s on Ryzen 9950X. It is also fairly fast for hash-map use cases, but
a bit slower than the `a5hash()` function. Among 128-bit hashes that pass the
state-of-the-art tests, it's likely the fastest 128-bit hash function for
hash-maps.

By setting the function's `rh` pointer argument to 0, it can also be used
as a 64-bit hash function with high bulk throughput.

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

## Comparisons

The benchmark was performed using [SMHasher3](https://gitlab.com/fwojcik/smhasher3)
on a Xeon E-2386G (RocketLake) running AlmaLinux 9.3. This benchmark includes
only the fastest hash functions that pass all state-of-the-art tests.
`XXH3-64` here does not, but it is too popular to exclude it. `rapidhash` is a
replacement for `wyhash`.

Small key speed values are in cycles per hash, while other values are in
cycles per operation. `std init` and `std run` are `std::unordered_map` init
and running tests, `par init` and `par run` are `greg7mdp/parallel-hashmap`
init and running tests. All values are averages over 10 runs.

|Hash function|Small key speed|std init|std run|par init|par run|
|----         |----           |----    |----   |----    |----   |
|a5hash       |**17.20**      |**523** |**404**|294     |**277**|
|rapidhash    |18.10          |526     |430    |308     |292    |
|rust-ahash-fb|19.33          |533     |429    |286     |304    |
|XXH3-64      |21.31          |533     |428    |292     |290    |
|komihash     |22.54          |539     |434    |**285** |296    |
|polymurhash  |28.44          |537     |458    |335     |335    |

## Customizing C++ namespace

If, for some reason, in a C++ environment, it is undesirable to export
`a5hash` symbols into the global namespace, the `A5HASH_NS_CUSTOM` macro can
be defined externally:

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
same mathematical construct as the `a5hash()` hash function. `a5rand()` passes
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

## Uniformity Analysis

At the time when this hash function is developed empirically, cryptography
has no straightforward techniques for differential cryptanalysis of the
multiplication of independent variables. This fact complicates formally
proving the uniformity of the hash function's output and state.

During the era when many classical hash functions were designed, the most
advanced attack method was differential cryptanalysis. While powerful against
substitution-permutation networks (like in block ciphers), it is difficult to
apply to multiplication of two independent variable inputs. Predicting
the differential propagation (i.e., how a difference in the input affects the
difference in the output) through a multiplication gate is very complex.
The carry propagation affects many bits in an unpredictable way.

While the linear behavior of XOR and the limited carry of addition can be
modeled with some success, the complex, high-level non-linearity of
multiplication makes this nearly impossible with the mathematical tools
available.

The empirical evidence that the `a5hash()` hash function and its core
operation represented by the `a5rand()` function practically retains
a uniformly random output on every iteration is provided by the following
example. Its `out` is analyzed by `PractRand`, which passes the statistical
tests for uniform randomness. The `ent` value here acts as an input message
with some sparse structure.

```c
uint64_t Seed1, Seed2, ent, out, Ctr = 0;

loop:

out = a5rand( &Seed1, &Seed2 );
ent = Ctr ^ Ctr << 15 ^ Ctr << 31 ^ Ctr << 47;
Ctr++;
Seed1 ^= ent;
Seed2 ^= ent;
```

Secondly, and this is important for the hash function's structure, successive
multiplications maintain uniformity of random output for at least one
iteration (actually, a few), without requiring the addition of `val01` and
`val10` constants to the state variables.

However, this does not touch on the statistics of internal state variables.
Admittedly, they cannot be used directly as high-quality uniformly random
values. But in the context of the "blinding multiplication" claim below, the
following code is sufficient evidence of the absence of bias in the hash
function's state:

```c
#include <stdio.h>
#include "a5hash.h"

int main(void)
{
    uint64_t Seed1 = 0, Seed2 = 0, ent, Ctr = 0;
    int c = 0;
    int j, i;

    for( j = 0; j < ( 1 << 8 ); j++ )
    {
        Seed1 = a5rand( &Seed1, &Seed2 );
        Seed2 = a5rand( &Seed1, &Seed2 );
        
        for( i = 0; i < ( 1 << 20 ); i++ )
        {
            a5rand( &Seed1, &Seed2 );

            ent = Ctr ^ Ctr << 15 ^ Ctr << 31 ^ Ctr << 47;
            Ctr++;

            if(( Seed1 & 0xFFFF ) == ( ent & 0xFFFF ) ||
                ( Seed1 & 0xFFFF0000 ) == ( ent & 0xFFFF0000 ) ||
                ( Seed1 & 0xFFFF00000000 ) == ( ent & 0xFFFF00000000 ) ||
                ( Seed1 & 0xFFFF000000000000 ) == ( ent & 0xFFFF000000000000 ) ||
                ( Seed2 & 0xFFFF ) == ( ent & 0xFFFF ) ||
                ( Seed2 & 0xFFFF0000 ) == ( ent & 0xFFFF0000 ) ||
                ( Seed2 & 0xFFFF00000000 ) == ( ent & 0xFFFF00000000 ) ||
                ( Seed2 & 0xFFFF000000000000 ) == ( ent & 0xFFFF000000000000 ))
            {
                c++;
            }
        }
    }

    printf( "%i\n", c );
}
```

As expected, this code detects about 8\*2<sup>12</sup> matches (the `ent` is
not added to the seeds here, but its addition decreases collisions due to
auto-correlation effects). Admittedly, by itself this is not an overall strong
evidence for the hash function's state uniformity. But it is adequate for the
resistance against the "blinding multiplication" occurring at some step.

This test demonstrates that the `a5rand()` construct maintains approximately
unbiased state and collision statistics close to that of uniform distribution,
continuously, and for any initial seed. Additionally, the state variables
follow same multi-lag auto-correlation statistics as uniformly random numbers
(see the `count_indep()` function below).

## State Uniformity Margin

Although the state variables are approximately uniform, a bit independence
test between the randomly chosen `UseSeed` and subsequent `Seed1` and `Seed2`
(after multiplication) reveals that up to 6 bits of either state variable can
be predicted by the `UseSeed`, though not simultaneously. At the same time,
a differential analysis indicates that average bit difference (avalanche)
between the `UseSeed`-`Seed1` and `UseSeed`-`Seed2` pairs is 50.0%. However,
since the `UseSeed` is unknown, this imperfect bit independence only reveals
an imperfect dispersion from the multiplication of independent variables,
which is to be expected.

A perfect correlation was identified in the 63rd bit of `Seed1` and `Seed2`
after the initial multiplication, related to the hash function's constants,
which reduces their effective entropy by one bit. While the significance of
correlations in other bits is debatable, accounting for them would reduce the
entropy by no more than an additional 3 bits.

## Blinding Multiplication

"Blinding multiplication" (BM) is a common issue in almost all fast hash
functions based on NxN bit multiplication involving 2\*N bit input per
iteration. BM happens when one of the input values coincides with the hash
function's current state or constants, and yields a zero result. While each
such hash function tries to handle BM, in general it is impossible to
completely "fix" BM without reducing hash function's performance.

While statistically speaking, a hash function's state may transiently become
zero, and it is a probable outcome, this becomes problematic if hash function
can't recover from it rapidly, and retain the seeded state. `a5hash` instantly
recovers from occasional BM (via the addition of `val01` and `val10`
constants), with its further state being unknown, if the `UseSeed` is unknown.
On the contrary, the "unprotected" `rapidhash` does not recover the seed - it
becomes zero, and thus the state becomes known.

Another issue is that NxN bit multiplication which yields zero, discards
half of 2\*N bit input completely, potentially leading to collisions. The
"protected" `rapidhash` tries to tackle this issue, but doing so, it creates
another issue: combining inputs on adjacent iterations without transformation,
which also yields collisions, but in a veiled manner. Also, if BM happens at
the last input, this input is passed to the final `rapid_mix` call, yielding a
hash value with poor avalanche properties.

However, one should consider the probability of BM happening on practical
inputs, in the context of brute-force attack that continuously scans a set of
inputs, not knowing a fixed seed. `a5hash` state (`Seed1` and `Seed2`) is
unbiased and meets usual random collision statistics at all times, with 1-bit
margin, which means BM is triggered with a theoretical probability of
2<sup>-63</sup> for a 64-bit seed: this can be considered negligible for
non-cryptographic use cases.

When the `UseSeed` is unknown, and when the output of the hash function is
unknown (as in the case of server-side structures), it is impossible for an
attacker to know immediately whether BM was triggered in any of hash
function's iterations: side-channel timing information requires hash
collisions to accumulate quickly to be measurable. This requires not only
triggering BM in one of the state variables, but also validating that a series
of additional inputs yields multiple collisions: this additionally increases
the "cost" of attack exponentially, depending on timing noise level.

To sum up, `a5hash` is practically resistant to blinding multiplication in
the general case, when hash function's `UseSeed` is unknown and hash
function's output is not exposed.

## Why A5?

The constants `0xAAAA...` and `0x5555...` used in `a5hash` and `a5rand`
represent repetitions of `10` and `01` bit-pairs. While they do not look
special as adders in PRNGs, or even look controversial due to absence of
spectral information, they have the important property of substantially
reducing internal biases of any number.

Consider this code, which calculates the sum of internal successive bit
independence estimates of a set of numbers, and same of numbers XORed and
summed with the `0xAAAA` constant, if a number's bit independence is low:

```c
#include <stdio.h>

int count_indep( int v )
{
    int c = 0;
    for( int i = 0; i < 14; i++ )
    {
        c += __builtin_popcount(( v ^ v << ( i + 1 )) >> i );
    }
    return( c );
}

int main(void)
{
    int c1 = 0, c2 = 0, c3 = 0;
    for( int i = 0; i < ( 1 << 16 ); i++ )
    {
        int c = count_indep( i );
        if( c < 70 )
        {
            c1 += c;
            c2 += count_indep( i ^ 0xAAAA );
            c3 += count_indep( i + 0xAAAA );
        }
    }
    printf( "%i %i %i\n", c1, c2, c3 );
}
```

Output:

```
55106 115862 123898
```

## Thanks

The author acknowledges [Alisa Sireneva](https://github.com/purplesyringa),
although their rhetoric was preeminently aggressive, for discovering a
potential issue in the original `a5hash v1`. This issue, which required inputs
longer than 1000 bytes, was fully resolved in `a5hash v5`.

Thanks to Frank J. T. Wojcik and prior authors for
[SMHasher3](https://gitlab.com/fwojcik/smhasher3).

Thanks to Chris Doty-Humphrey for
[PractRand](https://pracrand.sourceforge.net/).

## No Thanks

Reddit for discouragement.
