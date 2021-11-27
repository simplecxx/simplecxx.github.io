---
layout: post
title:  "Debunking all myths about MT19937 random number generator"
date:   2021-11-27 12:00:00 +0000
tags: c++ random-number-generator RNG PRNG
---

In this article I will debunk some popular myths about the Mersenne twister in
C++ that have accumulated over the years. The source for some of the myths is
this article named
[C++ Seeding Surprises](https://www.pcg-random.org/posts/cpp-seeding-surprises.html).
Let's start with few definitions.

**PRNG, pseudo-random number generator**
: This is a deterministic algorithm that takes certain state as input and as
  output it gives a pseudo-random number and the modified state that should
  be used as the next input. PRNGs are almost always predicable. If you know enough
  consecutive output random numbers, you can infer the internal state. They
  should not be used for cryptography, only for things like simulations or
  games.

**CSPRNG, cryptographically secure pseudo-random number generator**
: These algorithms are also deterministic, but they are not predicable. If you
  see some outputs you should not be able to guess its internal state, future
  or previous outputs.

**TRNG, True random number generator**
: TRNG is based on physical phenomena. When we observe certain physical
  phenomena that is nondeterministic and measure it with number, that number is
  considered a truly random number. Such TRNGs are connected to a computer as
  IO devices.

**Internal state of PRNG**
: The internal state usually consists of few numbers of certain size
  (32-bit, 64-bit, etc).

**Initial internal state**
: The first numbers that go in the state, before getting the first pseudo-random
  number, are called initial state.

**Seed**
: A number or sequence of numbers that are used to generate the initial internal
  state. Often the seed and the initial internal state are the same, but not
  always. E.g. in Mersenne twister (MT19937) the initial internal state is
  always different than the seed.

I will now continue with the myths.

## Myth 1: MT19937 can never generate 7 or 13 as the first output when seeded with single integer

This is wrong. Here is sample code that debunks this myth.

```
#include <random>
#include <cassert>

int main()
{
	auto rnd = std::mt19937(1080100664);
	assert(rnd() == 7);

	rnd.seed(736640520);
	assert(rnd() == 7);

	rnd.seed(1292535796);
	assert(rnd() == 13);
}
```

How did I found these seeds? I'll leave that as an exercise to the reader.

## Myth 2: MT19937 has bad internal states with lot of zeroes

This was true until 2002. After the first publication in 1998, some flaws were
discovered with initial internal states that contained lot of zeroes. This was
[fixed in 2002](http://www.math.sci.hiroshima-u.ac.jp/m-mat/MT/MT2002/emt19937ar.html)
by the original authors. They introduced two initialization functions that take
the seed, do some fiddling on the bits and only after that they fill the state.
There is one algorithm that takes one integer and appropriately fills the state
and another algorithm take takes sequence of arbitrary length and then fills the
state.

If you look at the [constructors of the standard MT19937](https://en.cppreference.com/w/cpp/numeric/random/mersenne_twister_engine/mersenne_twister_engine)
you should notice similar algorithms for seeding. Even if the seed sequence
generates lot of consecutive zeroes, the additional bit fiddling will fix that.
With those algorithms in place, there is explicit distinction between the seed
and the initial state. You should not try to directly fill the internal state
for the purposes of seeding.

## Myth 3: You should seed MT19937 with sequence of 624 integers

No, you shouldn't. As I stated previously, the seed and the state are different.
If the state is huge, the seed does not need to be. Consider the state
as implementation detail. The seed matters mostly for the randomness of the few
initial outputs. The more seeds you have, the more initial states you have,
the more uniformity you have in the first few outputs.

If 32-bit seed is not enough for you, use 64-bit seed, i.e. `std::seed_seq`
constructed with two integers. Do you know how big is 2^64? Just one for-loop
that iterates all 64-bit numbers and does nothing else may take maybe 200 years
to execute. Is that not enough for you? Use 128-bit seed, like my example in my
[previous article]({% post_url 2018-11-03-seed-mt19937 %}). With 128-bit seed you have
gazzilion different initial states. Generating all of them would take bazzilion years.

I have two more arguments related to `std::random_device` on why using 624
integers is too much.

`std::random_device` is very much platform specific facility, but most likely it
is CSPRNG. There are some things that are common. The OS collects randomness
from various IO like mouse movement, keyboard typing, temporary files and fills
up the randomness, the entropy gets higher. As you request random numbers, the
randomness pool is spent. If you overuse it, it will deplete. Modern operating
systems try very hard this not to happen. Still, you should not fight with the
OS. You should treat `std::random_device` as finite resource that should be
spent wisely. Therefore, using just few values from `std::random_device` is the
right way to seed. Using 624 values from `std::random_device` is wrong.

Secondly, calls to `std::random_device` are slow. Internally, it does a system
call which is more expensive than normal function call (in user space). The system
call may further do some IO, which is, again, a slow operation. When you request
more unnecessary data from `std::random_device`, you unnecessarily slow your program.

## Myth 4: std::seed_seq is not bijection

This is probably the only correct statement in that bad article, but also it is
completely useless. Is it a big problem if there are few collisions? They tested the case
where the input is two 32-bit integers and the output is also two 32-bit integers.
Why would one use `seed_seq` in the first place in such case? If the internal
state of some PRNG is 64 bits, I would directly seed it with 64-bit number. Can
someone test for bijection where the input is 2 or 4 integers and the output is
624 integers? 

## Conclusion

The current design of the library for random numbers is actually good.
There can be some improvements, like ease of use. For starters, I would improve the
documentation in the standard by giving an example that shows the little dance between `random_device`,
`seed_seq` and `mt19937`. I will use just 4 integers from `random_device` in that example.
Future proposals about random numbers in C++ must not make `random_device` to conform to
the named requirement SeedSequence because that would make it very easy to use
`random_device` the wrong way.
