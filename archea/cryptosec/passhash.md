# Password Hashing
Good password storage, like any core security algorithm, is hard. 
Obvious first steps are a salted hash, but what hashing algorithm,
and how to deal with the result?

Repeated hashing (stretching) is a basic way to deal with brute-force
attacks on a stolen hash. 

Goals:
- Verifyable
- Non-invertible
- Tunable
- Resists common threat models
- For various reasons, we avoid parallellizable approaches.

Threats:
- MitM&replay
- Side channel attacks
- GPUs
- ASICs

Ideas:
We assume the client remembers nothing except the passphrase, so salts must be stored server-side and
sent as needed.

Sponges are flexible. We'll use one of those, but will look into using one with nice properties later.

Password setup and password verification are different processes: setup is allowed to take
longer, and requires more effort to protect against MitM since no shared secret exists.

Against MitM, assuming a compromised channel: Challenge-response. The client never transmits 
everything it knows. 
Since the server must be able to dynamically generate challenges, it either needs 
an asymmetric property, or must at some point know more about the client's state than
a MitM. 

### Verification
The server provides a salt, which the client uses for a first run of stretching.
Attacks prevented: MitM eavesdropping on plaintext or rainbow. 

Next concern: malicious server. This is basically a worse case of the offline attack,
with any stretching the server does neutralized on log-in. So this can only be countered
by making the stretching until the first synced state strong, and preventing reversibility
of the sponge state.

The stretching fills a sponge, of `r+k+c` bits. `r` is the in/out size, `k+c` is the hidden storage.
The client, after stretching, extracts bits from the sponge (directly or by some scheme. How much do
we trust the strength of the sponge?) to send to the server. 
The server applies the second run of stretching, until it has a near (unsalted) pre-image of the stored
hash. The (large) pre-image is then salted two different ways: the first produces the stored hash, the second
reveals a code that, XORed with a saved array, produces a verifier for the extra information the client has.
Always run verification, before even looking at the hash to check against, to make sure the client can't conclude 
that it had a partial success in an online brute-force attempt. 


Verifier: several options, easiest (TODO: see if improvements are possible):
Take the top `r+k` bits of the client's state, the last
`c` remaining hidden to protect against inversions of the sponge step function (in case of a malicious server, 
or MitM with access to a leak, which is similar). Client and server send each other random numbers, which are 
mutual challenges, inserting them in an agreed order into the sponge. Then they squeeze for several steps, taking
turns in telling each other their outcomes. 
Attack prevented: replay.

Stretching:
This is where the real work is: we need to prevent dedicated hardware or massive parallel computers from computing
this any more easily than the legitimate user. 

Techniques: We kill parallelism, introduce branching, jump wildly through a large dataset, and add a bit of Halting uncertainty.

First, we obfuscate from side-channel attacks by choice of hash. This can be any known good hash. 
However, we will not attempt to prevent a side-channel attack on internal values once we start our
branching/Halting passes. We will, however, mess with GPUs before we break sidechannels.

There are two actionable concerns here: side channels, and GPUs. GPUs don't like branching, while side channels do.
Therefore, we have two approaches: be as unpredictable as possible, including dependencies on the key, or
be completely predictable, keeping the key out of side channels. A possible compromise is to be dependent
on a salt.


# Predictable, no side channels:
We define a three-way merge of blocks: to each block, append halves of each of the other blocks, such that 
each half-block is appended to exactly one other block. then hash the result, yielding a new block for each 
input block, by which we then replace it. This introduces sequential dependencies killing parallelism.

Now we take steps: first, we generate a stream of cryptopseudorandom bits, first filling a large buffer, 
then a small buffer. We now iterate over the small buffer back to front, starting again at the back if 
we run out: perform a three-way merge between the current block in the small buffer, the current block 
in the large buffer, and the previous block in the large buffer. The "previous block" for the first block 
is a tuning salt: it is chosen at password setup time to ensure termination at a later stage. 

The sequence by which we go through blocks in the large buffer is the bit reversal sequence, which hits 
blocks up and down the buffer nicely.

Alternatively to three-way merging, one can use a single sponge, initialized with the tuning salt, and run
each block through the sponge in the sequence described above, ignoring the "previous block" part, starting
in the small buffer and alternating between small and large.

This approach can be repeated, with different fixed sequences to make access patterns on the "history" of
the small buffer as nasty as possible.

Optionally, for non-halting behaviour in this step: keep hashing blocks in some sequence until one block
has a hash output containing enough leading 0's, for some value of "enough". After that, terminate the 
side-channel-resistant phase.

# Side channels, unpredictable

First, take the scrypt approach: swap some numbers around in the large buffer, using the small buffer as 
a random input stream for choosing locations where to swap to and from. Additionally, one can add conditional
instructions mandating a hash and possibly rehash before or after swapping. These should rarely occur, but
often enough to make betting on it not occurring a losing proposition on SIMD and SPMD systems.

Then, add some potentially non-halting behaviour: run the small buffer as the description of a longjumping Turing 
machine, with the large buffer as tape, and semantics defined such that any random sequence is a valid machine, but 
many programs have many non-terminating inputs. 

Alternatively, do some callstack abuse, with several functions with different numbers and types of arguments, calling each other conditionally and with middle recursion and several local variables, requiring stack management. 
