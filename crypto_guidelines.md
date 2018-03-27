# Mitigating Speculative Attacks in Crypto

Author: [chandlerc@google.com](mailto:chandlerc@google.com)

The goal of this document is to provide guidelines to authors of cryptographic
software on how to avoid speculative execution attacks on their code leaking
secret data such as private keys.


## Background

Google's Project Zero and a number of external researchers have published
attacks that use speculative execution to leak secret information:
* https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html
* https://spectreattack.com/spectre.pdf

We are primarily concerned with variants #1 and #2 from the Project Zero report
which are the same as variants #1 and #2 in the Spectre paper. The Meltdown
attack is independent and not relevant for securely authoring cryptographic
software.

This document will not attempt to summarize or explain these attacks. Please
refer to the above material or existing public write-ups for that information.
We assume the reader is reasonably familiar with these variants and how they
work.


## Scope and Limitations

We are merely presenting guidelines here, and only guidelines specific to the
cryptographic library itself. The larger application using it may well need its
own mitigation techniques, and a failure to mitigate the application may render
moot anything done within the library. However the crypto code itself must also
take steps, and in some cases unusual steps, that we try to outline here.

Speculative execution is also a relatively new and unstudied attack surface. As
a consequence, we expect these guidelines to grow and evolve over time as we
learn more about both the attacks and how to mitigate them most effectively.

Last but not least, we present many of these techniques in the spirit of defense
in depth rather than due to any particular known exploit techniques.


## Guidelines


### 1) Do not conditionally choose between constant and non-constant time

It may be natural in crypto code to do the equivalent of:
```
if (Key.isPublic()) {
  NonConstantTimeAlgo(Key.data());
} else {
  ConstantTimeAlgo(Key.data());
}
```
However, speculative execution can bypass this check and execute very
significant portions of the non-constant time algorithm in the speculative
domain. This may well leak enough bits to compromise key data.

This is probably the most important immediate shift in design of crypto code.
Our strong suggestion is to remove these kinds of alternatives and simply always
use constant time algorithms.

If there is no option, one of the various mitigations for variant 1 attacks that
are being considered should be used here, but the exact mechanisms of those have
not been clearly defined yet.


### 2) Avoid indirect branches in constant-time code

By indirect branches, we mean either switches that could be implemented with
a jump-table, calls through a function pointer, or higher level primitives like
virtual calls.

Even if these indirect branches aren't in any way influenced by the secret data,
they may be redirected through misprediction in a speculative domain to a gadget
that leaks data currently in registers.

Eventually, there should be compiler features to help facilitate this, but
currently they are too coarse grained (whole program, not one code region) and
use slow implementations rather than simply avoiding the constructs or producing
an error. For now, avoiding known language constructs is no more risky or
brittle than the rest of constant time code. And in many cases, the constant
time code is written in a low-level assembly language that makes avoiding these
easy.


### 3) Do not _conditionally_ scrub registers or local buffers of key data

If your cryptographic code tries to scrub private key data from registers or
local buffers used during a constant-time algorithm, that scrubbing needs to be
unconditional to be effective. For example, don't do things like this (somewhat
silly) example:
```
bool ConstantTimeAlgo(uint64_t *data, size_t size) {
  uint64_t first_chunk = data[0];
  uint64_t second_chunk = data[1];
  ...
  if (error_indicating_data_was_never_valid) {
    // We never had valid data, just bail.
    return false; // BAD BAD BAD!!!!
  }
  ...
  scrub_data(first_chunk);
  scrub_data(second_chunk);
  return true;
}
```
If the attacker can cause the condition to be mispredicted and some secret data
was already loaded, it may persist in registers in speculative execution far
enough to be leaked. Instead, just scrub registers unconditionally.

Speculative execution attacks may also make it somewhat easier to leak private
key data from both registers and local buffers on the stack by directing
speculative execution to information leak gadgets that don't occur during normal
execution. If this is a concern, you should scrub both registers and local
buffers as close to the end of their use as possible. Avoid any branches and
especially indirect branches or returns prior to scrubbing. Note that C code is
hard to make correct for this scrubbing, prefer compiler-provided primitives or
assembly constructs. For example, see http://llvm.org/PR36913 which requests
explicit support in Clang and LLVM for scrubbing registers.


### 4) Do not store private key data in global buffers

Several mitigation techniques for speculative execution techniques either don't
work or are substantially less efficient when mitigating accesses to global
variables and buffers. Avoid storing private key data or other secrets in them.
Instead, store private key data in heap allocated buffers when they need
long-term storage.


## More Extreme Security vs. Performance Tradeoffs

We believe that the above, combined with application mitigations, are
sufficient. However, it is possible to add more defense-in-depth layers of
protection at the expense of **significant** performance. However, the
performance hit for these techniques is likely to be much more significant, and
the security provided is through more depth, not covering any other specific
vectors.


### 5) Block speculation into constant-time algorithms

Constant-time algorithms are particularly tempting to attack as we *know* they
load secret data from memory and operate on it. To make it fundamentally harder
to do this, you can use an instruction at the start of the algorithm that blocks
or serializes all speculation. For x86, both Intel and AMD are suggesting an
`lfence` instruction.


### 6) Put key data into UC (UnCacheable) memory regions.

This will make speculative accesses significantly more challenging if possible
at all. However, this may come with the highest performance hit of all.

The idea of uncacheable memory is what it says on the tin - it won't be cached
and will always come from main memory. This doesn't mean it has constant time
(sadly). Perhaps that could be added in future architectures / OSes (nothing
seems infeasible about it), but today it is just uncacheable. However, due to
never staying in the cache, it isn't available for speculative execution to read
out of the cache. This can prevent a very wide array of speculative execution
attacks, including things we've never thought of, because it is a very
fundamental and broad mitigation. That doesn't mean it is _guaranteed_ to
protect against speculative execution attacks, and so other more targeted
mitigations may still be necessary.


## Future work

We are hoping to add more sections in the future as others contribute or as
things settle and become more widely available. First, we'd love to have example
assembly for some of the suggested patterns above targeting various
architectures. Second, we'd like to add a summary of compiler- and
application-level mitigations that are relevant to keep in mind or use in
conjunction with a crypto library. However, the above should get maintainers
started making the (hopefully few) changes necessary in their implementations.
