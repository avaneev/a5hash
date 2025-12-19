# A5HASH - Fast Hash Functions (in C/C++)

## Introduction

The `a5hash` family of hash functions, available in the `a5hash.h` header
file, provides high-performance hash functions, designed for hash-table,
hash-map, and bloom-filter use cases. The code is portable, cross-platform,
scalar, zero-allocation, header-only, inlinable C (compatible with C++).
Released in open-source form under the MIT license.

The provided 64-bit and 128-bit hash functions of the `a5hash` family are
compatible with 32-bit platforms, but their use there is not recommended due
to reduced performance of a 64-bit multiplication. On 32-bit platforms it is
recommended to use the available `a5hash32()` function which provides native
32-bit compatibility, if 32-bit hash values are sufficient.

The `a5hash()` 64-bit function achieves very high hashing throughput for small
strings/messages (about 11 cycles/hash for 0-64-byte strings, hashed
repeatedly on Zen 5). The bulk throughput is intentionally moderately fast
(15 GB/s on Ryzen 9950X). Most recent competing "fast" hash functions aim to
perform well both for common keyed string hash-maps and large data hashing.
In most cases, this is done for the sake of better benchmark results, even
though such hash functions rarely provide the streamed hashing required for
large data or file hashing.

`a5hash()` was designed to be ultimately fast only for common string and
small-key hash-maps, and hash-tables, by utilizing the "forced inlining"
feature present in most modern C and C++ compilers: this is easily achievable,
since in its compiled binary form, `a5hash()` is very small - about 270 to
330 bytes on x86-64 and arm64. Moreover, if the default seed (0) is used, or
if data of a compile-time constant size is hashed, these conditions further
reduce the code size and increase the hashing throughput.

Note that `a5hash` functions are not cryptographically secure. To minimize
the risk of a successful collision attack (hash flooding) in open systems and
in server-side internal structures, they should only be used with a secret
seed, and their output should not be exposed. This is especially important if
a malicious party is likely to try to influence the hash outputs. However, if
"blinding multiplication" (BM, state cancellation) occurs at a random
iteration, these functions immediately recover from the zeroed-out state
(see below). `a5hash` functions should not be used for MACs or other strictly
cryptographic purposes.

`a5hash` hash functions produce different hashes on big- and little-endian
systems. This was a deliberate design choice to narrow down the scope of use
cases to run-time hashing and embedded storage, as endianness correction
usually imposes a significant performance penalty. If a reliable and fast hash
function for files, with portable hashes, is needed,
[komihash](https://github.com/avaneev/komihash) is a great choice.

Overall, `a5hash` achieves three goals: attains an ultimate speed for run-time
hashing of small keys, has very small code size, and uses a novel mathematical
construct. Compared to most, if not all, existing hash functions, `a5hash`
hash functions do not use explicit accumulators or folding of state variables:
the full 128-bit result of multiplication is used directly as input on the
next iteration. This construction appears to be near the minimal complexity
required for high-quality hashing (in the `SMHasher` testing sense). Such
a conclusion stems from the fact that the `a5hash()` function is structurally
simple, and further reduction of the number of operations seems impossible,
while other existing similarly simple constructs like `xorshift` do not
provide high-quality hashing.

`a5hash` hash functions pass all [SMHasher](https://github.com/rurban/smhasher)
and [SMHasher3](https://gitlab.com/fwojcik/smhasher3/-/tree/main/results?ref_type=heads)
tests. The functions were also tested using the
[xxHash collision tester](https://github.com/Cyan4973/xxHash/tree/dev/tests/collisions)
at various settings, with collision statistics meeting theoretical
expectations.

These functions and the source code (which conforms to
[ISO C99](https://en.wikipedia.org/wiki/C99)) were quality-tested on:
Clang, GCC, MSVC, and Intel C++ compilers; x86, x86-64 (Intel, AMD), and
AArch64 (Apple Silicon) architectures; Windows 11, AlmaLinux 9.3, and macOS
15.3.2. Full C++ compliance is enabled conditionally and automatically when
the source code is compiled with a C++ compiler.

## Usage

Simply copy `a5hash.h` into a project. It is a header-only library.

```c
#include <stdio.h>
#include "a5hash.h"

int main(void)
{
    const char s1[] = "This is a test of a5hash.";
    const char s2[] = "7 chars";

    printf( "0x%016llx\n", a5hash( s1, strlen( s1 ), 0 )); // 0xa04d5b1d10d1f246
    printf( "0x%016llx\n", a5hash( s2, strlen( s2 ), 0 )); // 0xe49a0cc72256bbac
}
```

As a bonus, the `a5hash.h` file provides the `a5hash_umul128()`
general-purpose inline function which implements a portable unsigned 64x64 to
128-bit multiplication.

## A5HASH-128

The `a5hash128()` function produces 128-bit hashes, and, compared to 64-bit
`a5hash()` function, features a significantly higher performance for large
data hashing - 48 GB/s on Ryzen 9950X. It is also fairly fast for use in
hash-maps, but a bit slower than the `a5hash()` function. Among 128-bit hashes
that pass the state-of-the-art tests, it's likely the fastest 128-bit hash
function for hash-maps.

By setting the function's `rh` pointer argument to `0`, `NULL`, or `nullptr`
(in C++), it can also be used as a 64-bit hash function with high bulk
throughput.

```c
#include <stdio.h>
#include "a5hash.h"

int main(void)
{
    const char s1[] = "This is a test of a5hash128.";
    uint64_t h[ 2 ];

    h[ 0 ] = a5hash128( s1, strlen( s1 ), 0, h + 1 );

    printf( "0x%016llx%016llx\n", h[ 0 ], h[ 1 ]); // 0xdd2f427007acd1b23609225842ec8020
}
```

## Comparisons

The benchmark was performed using [SMHasher3](https://gitlab.com/fwojcik/smhasher3)
(commit `7be7e9e820724779926c052db41c15bf88d5863b`) on a Xeon E-2386G
(Rocket Lake) running AlmaLinux 9.3. This benchmark includes only the fastest
hash functions that pass all state-of-the-art tests. `XXH3-64` here does not,
but it is too popular to exclude it. `rapidhash` source code explicitly notes
it is based on `wyhash`.

Small-key speed values are in cycles per hash, while other values are in
cycles per operation. `std init` and `std run` are `std::unordered_map` init
and running tests, `par init` and `par run` are `greg7mdp/parallel-hashmap`
init and running tests. All values are averages over 10 runs.

| Hash function | Small-key speed | std init | std run | par init | par run |
|---------------|-----------------|----------|---------|----------|---------|
| a5hash        | **11.50**       | **523**  | **404** | 294      | **277** |
| rapidhash     | 18.10           | 526      | 430     | 308      | 292     |
| rust-ahash-fb | 19.33           | 533      | 429     | 286      | 304     |
| XXH3-64       | 21.31           | 533      | 428     | 292      | 290     |
| komihash      | 22.54           | 539      | 434     | **285**  | 296     |
| polymurhash   | 28.44           | 537      | 458     | 335      | 335     |

## Customizing C++ namespace

In C++ environments where it is undesirable to export `a5hash` symbols into
the global namespace, the `A5HASH_NS_CUSTOM` macro can be defined externally:

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
Note that since all `a5hash` functions have the `static inline` specifier,
there can be no ABI conflicts, even if the `a5hash.h` header is included in
unrelated, mixed C/C++, compilation units.

## Seeding

All `a5hash` functions can be seeded with values of any statistical quality.
However, a resistance against hash flooding requires the seeds to be
maximum-entropy, uniformly random values.

## A5RAND

The `a5rand()` function available in the `a5hash.h` file implements
a simple, but reliable, self-starting, and fast (`0.50` cycles/byte) 64-bit
pseudo-random number generator (PRNG) with a `2^64` period. It is based on
the same mathematical construct as the `a5hash()` hash function. `a5rand()`
passes `PractRand` tests (at least up to 1 TB length, at the default
settings) and `SmokeRand` tests (full setting).

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

## Design Analysis

### Why A5?

The constants `0xAAAA...` and `0x5555...` used in `a5hash()` and `a5rand()`
represent repetitions of `10` and `01` bit-pairs over register. While they do
not look special as adders in PRNGs, or even look controversial due to the
absence of bit-wise spectral information, they have the important property of
substantially reducing internal biases of any given number.

Consider this code, which calculates the sum of internal successive bit
independence estimates of a set of numbers, and the same for set's numbers
XORed and summed with the `0xAAAA` constant, if a set number's successive bit
independence is low. The output shows that the estimates are higher for
numbers that have been XORed or added with the constant. Other statistically
uniform constants also increase the estimates, but to a lesser extent, making
the `0xAAAA...` and `0x5555...` constants special.

This example empirically explains why both `a5hash()` and `a5rand()` generate
high-quality random numbers despite their reliance on multiplication,
an operation with imperfect dispersion. The quantification method is
reasonable because it aligns with uniformity assessments; for numbers to be
uniformly random, correlations between successive output bits must be minimal,
and such estimate should not be low.

Additional empirical evidence suggests that while the XOR operation normalizes
successive bit independence, the addition operation maximizes it. Both
operations are used in `a5hash` hash functions in accordance with this
observation. This finding can be evaluated by adjusting the `c < 70` condition
in the code (112 is expected for 16-bit uniformly random numbers).

```c
#include <stdio.h>

int count_indep( int v )
{
    int c = 0;
    for( int i = 0; i < 14; i++ )
    {
        c += __builtin_popcount(( v ^ v << ( i + 1 )) >> ( i + 1 ));
    }
    return( c );
}

int main(void)
{
    int c1 = 0, c2 = 0, c3 = 0, c4 = 0, c5 = 0;
    for( int i = 0; i < ( 1 << 16 ); i++ )
    {
        int c = count_indep( i );
        if( c < 70 )
        {
            c1 += c;
            c2 += count_indep( i ^ 0xAAAA );
            c3 += count_indep( i + 0xAAAA );
            c4 += count_indep( i ^ 0x6A88 );
            c5 += count_indep( i + 0x6A88 );
        }
    }
    printf( "%i %i %i %i %i\n", c1, c2, c3, c4, c5 );
}
```

Output:

```
76090 149759 160004 145679 155002
```

The standard "bit independence criterion" quantification applied to
the numbers themselves demonstrates a similar improvement, although less
variable:

```c
#include <stdio.h>

int count_bic( int v )
{
    int c = 0;
    for( int j = 0; j < 15; j++ )
    {
        for( int i = 0; i < 15; i++ )
        {
            c += (( v >> i ) ^ ( v >> j )) & 1;
        }
    }
    return( c );
}

int main(void)
{
    int c1 = 0, c2 = 0, c3 = 0, c4 = 0, c5 = 0;
    for( int i = 0; i < ( 1 << 16 ); i++ )
    {
        int c = count_bic( i );
        if( c < 65 ) // 104 is expected for 16-bit uniformly random numbers.
        {
            c1 += c;
            c2 += count_bic( i ^ 0xAAAA );
            c3 += count_bic( i + 0xAAAA );
            c4 += count_bic( i ^ 0x6A88 );
            c5 += count_bic( i + 0x6A88 );
        }
    }
    printf( "%i %i %i %i %i\n", c1, c2, c3, c4, c5 );
}
```

Output:

```
23520 52640 53056 51600 52280
```

### Uniformity Analysis

At the time when this hash function was developed empirically, cryptography
had no precise and straightforward techniques for differential cryptanalysis
of the multiplication of large independent variables. This complicates
a formal proof that the hash function's output and state are uniform.

During the era when many classical hash functions were designed, the most
advanced attack method was differential cryptanalysis. While powerful against
substitution-permutation networks (like in block ciphers), it is difficult to
apply to multiplication of two independent variables as inputs. Predicting
the differential propagation (i.e., how a difference in the input affects
the difference in the output) through a multiplication gate is very complex.
The carry propagation affects many bits in an unpredictable way.

While the linear behavior of XOR and the limited carry of addition can be
modeled with some success, the complex, high-level non-linearity of
multiplication makes this nearly impossible with the mathematical tools
available.

At the same time, the empirical evidence is usually sufficient for
non-cryptographic contexts, even though it does not constitute hard proof.
The following example provides evidence that the `a5hash()` function -
specifically, its core operation, the `a5rand()` function - retains
practically a uniformly random output on every iteration. It models the entire
hashing process, from the initial input to the final output, which uses
equivalent operations.

In this example, the `out` value is provided to and analyzed by `PractRand`,
which passes statistical tests for uniform randomness. Here, the `ent` value
acts as an input message with a sparse structure.

```c
uint64_t Seed1, Seed2, ent, out, Ctr = 0;

loop:

out = a5rand( &Seed1, &Seed2 );
ent = Ctr ^ Ctr << 15 ^ Ctr << 31 ^ Ctr << 47;
Ctr++;
Seed1 ^= ent;
Seed2 ^= ent;
```

Furthermore, and critically for the initial hashing iteration, successive
multiplications maintain output uniformity for at least one iteration -
and typically several - even without the addition of the `val01` and `val10`
constants to the state variables.

### State Uniformity

However, the above written does not touch on the statistics of internal state
variables. It should be noted that they cannot be used directly as
high-quality uniformly random values. But in the context of the "blinding
multiplication" claim below, the following code is sufficient evidence of the
absence of bias in the hash function's state:

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

As expected, this code detects about `8\*2^12` matches (the `ent` is not added
to the seeds here, but its addition decreases collisions due to
auto-correlation effects). Admittedly, by itself this is not a strong evidence
for the hash function state's uniformity. But it is adequate to demonstrate
the resistance against the "blinding multiplication" occurring at some
iteration.

This test demonstrates that the `a5rand()` construct continuously maintains
an approximately unbiased state and collision statistics close to a uniform
distribution, for structured inputs with any initial seed. Additionally,
the state variables follow the same multi-lag auto-correlation statistics as
uniformly random numbers (see the `count_indep()` function above).

### State Uniformity Margin

Although the state variables are approximately uniform, a bit-independence
test between the hash function's randomly chosen `UseSeed` parameter and
the subsequent `Seed1` and `Seed2` (after multiplication) reveals that up to
6 bits in either state variable can be predicted by the `UseSeed`. At the same
time, a differential analysis indicates that the average bit difference
(avalanche effect) between the `UseSeed`/`Seed1` and `UseSeed`/`Seed2` pairs
is approximately 50% (+/- 0.3%). However, since the `UseSeed` is unknown, this
imperfect bit independence only reveals an imperfect dispersion resulting from
the multiplication of independent variables, which is to be expected.

Another observation, unrelated to the `UseSeed` biases, is a perfect
correlation in the 63rd bits of both `Seed1` and `Seed2` after the initial
multiplication involving the hash function's known constants. This correlation
reduces the effective entropy of both state variables at the initial
iteration - and of `val01` and `val10` constants throughout - by 1 bit
relative to the theoretical 64-bit entropy. While the significance of the
0.5-level correlations related to the known constants and between both state
variables in three other bits is debatable, accounting for them would reduce
the entropy by only up to 3 bits.

### Blinding Multiplication

"Blinding multiplication" (BM), or more broadly, "state cancellation," is
a common issue in almost all fast hash functions based on NxN bit
multiplication involving 2\*N-bit input per iteration. BM happens during
hashing iterations when one of the input values coincides (via XOR or
addition) with a hash function's current state or constants, resulting in
a zero value - the only outcome that poses a problem. While each such hash
function attempts to handle BM, it is generally impossible to eliminate it
completely without reducing a hash function's performance.

BM should not be confused with the Birthday Paradox problem. Statistically,
BM is a problem concerning the equality of an arbitrary input and an
independent, uniformly distributed value, which has a theoretical probability
of `2^-64` for 64-bit variables.

Statistically speaking, a hash function's state may transiently become zero,
and it is a probable outcome. However, this becomes problematic if the hash
function can't recover from it rapidly and retain the seeded state. `a5hash`
instantly recovers from occasional BM within iteration via the subsequent
addition of `val01` and `val10` constants. This is not an ad-hoc fix, but a
part of the hashing iterations. These constants are derived from the unknown
initial seed, and the further state of the function remains similarly unknown.
In contrast, the "unprotected" `rapidhash` does not recover the seed - it
becomes zero, and thus the state becomes known (this and the next claim can be
verified by carefully examining the `rapid_mum()` function in the context of
the source code of `rapidhash`).

Another issue is that NxN bit multiplication which yields zero, discards
half of 2\*N bit input completely, potentially leading to collisions.
The "protected" `rapidhash` tries to tackle this issue, but in doing so, it
creates another issue: combining inputs on adjacent iterations without
transformation, which also yields collisions, but in a veiled manner. Also,
if BM happens at the last input, this input is passed to the final
`rapid_mix()` call, yielding a hash value with poor uniformity.

However, one should consider the probability of a BM event occurring with
crafted inputs during a brute-force attack. Such an attack would continuously
scan a set of inputs without knowledge of a secret seed. This scenario is
similar to the code example in the "Uniformity Analysis" chapter that
evaluates collisions with structured inputs.

Given that the `a5hash` state is unbiased and approximates an expected random
collision statistics for two of its variables (`Seed1` and `Seed2`) at all
times (minus a 1- to 4-bit entropy margin, as detailed in the "State
Uniformity Margin" chapter), BM is triggered in either variable with
a theoretical probability of `2^-62` (optimistic) or `2^-59` (pessimistic) for
a 64-bit secret seed. In either case, this probability can be considered
negligible for non-cryptographic use cases.

When the `UseSeed` is unknown, and when the output of the hash function is
unknown (as in the case of server-side structures), it is impossible for
an attacker to know immediately whether BM was triggered in any of the hash
function iterations: collision confirmation requires
a statistically-significant timing information, thus needing hash collisions
to accumulate quickly to be measurable.

This requires an attacker to not only trigger a BM in one of the state
variables but also to validate that a series of additional inputs produces
multiple collisions. The "cost" of such an attack increases exponentially with
the timing noise level. For instance, if an attacker can measure a response
with 100-microsecond precision over a public network, but the hash-map
collision mitigation completes in 2 microseconds (or 5,000 cycles -
a reasonable estimate for modern systems), the attacker would need at least 64
additional inputs per BM assessment. This represents a `2^6` (or a 64-fold)
increase in the number of required inputs.

In summary, when used with a secret 64-bit seed and non-exposed outputs,
`a5hash()` and `a5hash128()` are practically resistant to blinding
multiplication attacks. But this resistance is specific and should not be
construed as a complete cryptographic security evaluation.

This conclusion applies only to hash functions that use a 64-bit seed.
The 32-bit `a5hash32()` function should not be used in open systems due to
32-bit seeding.

## Thanks

The author acknowledges [Alisa Sireneva](https://github.com/purplesyringa),
although their rhetoric was predominantly negative, for discovering a
potential issue in the original `a5hash v1`. This issue, which manifests with
certain inputs longer than 1000 bytes, was fully resolved in `a5hash v5`.

Thanks to Frank J. T. Wojcik and previous authors for
[SMHasher3](https://gitlab.com/fwojcik/smhasher3).

Thanks to Chris Doty-Humphrey for
[PractRand](https://pracrand.sourceforge.net/).

Thanks to the team developing [DeepSeek chat](https://www.deepseek.com/) that
helps with grammar and consistency evaluation.
