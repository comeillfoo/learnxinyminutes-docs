---
name: Applied Pi calculus (ProVerif)
contributors:
    - ["Lenar Khannanov (Assisted-by: LLM)"]
---

Applied Pi Calculus extends the &pi;-calculus with equational theories (e.g.,
encryption, hashing) to model cryptographic protocols. ProVerif is a symbolic
protocol verifier built on this calculus — it checks secrecy, authentication,
and observational equivalence using type systems and equational logic.

```proverif
// Comments start with //
(* Multi-line
   comment *)
// ———————————————————————————————————————————————
// 1. NAMES, CONSTANTS & TYPES
// ———————————————————————————————————————————————

// Free names (non-duplicable, fresh per execution) — e.g., nonces, keys
new n: Name.  // "n" is a fresh, private name (used for keys/nonces)

// Public names (known to attacker) — e.g., server IDs, protocol names
let alice = "Alice".  // alias (syntactic sugar)
let bob = "Bob".

// Function symbols (with equational theory)
// Declare with `fun` and specify equational properties (see §4)
fun enc/2.          // enc(x, k) — symmetric encryption
fun dec/2.          // dec(y, k) — decryption
fun hash/1.         // hash(x) — one-way hash
fun pair/2.         // pair(x, y) — tuple

// Constant symbols (e.g., protocol identifiers)
const protocol_id : Bitstring.

// Types (optional but recommended; inferred if omitted)
// Type signatures help ProVerif’s type system:
//   - `bitstring` — uninterpreted data
//   - `privkey` / `pubkey` — asymmetric key types
//   - `symkey` — symmetric key
//   - `msg` — generic message
// Types are *not* enforced at runtime — only for static analysis.

// ———————————————————————————————————————————————
// 2. PROCESSES (The Core Language)
// ———————————————————————————————————————————————

// *Nil* — process that does nothing
0

// *Parallel composition* — runs processes concurrently
P | Q

// *Input* — wait for message on channel `c`
!c(x).P    // Replicated input (accepts infinitely many messages)
c(x).P     // Non-replicated input (single use)

// *Output* — send message `t` on channel `c`
c<t>.P     // Sends term `t`, then continues as `P`

// *Restriction* — fresh name with scope limited to `P`
new x: Type. P

// *Conditional* — if term `t1` equals `t2`
if t1 = t2 then P else Q
// (ProVerif supports `if` with `=` and `!=`)

// *Let-binding* — pattern-match and reuse
let x = t in P

// *Prefixing* — sequential composition (implicit in all above)
c<t>.0  // send `t`, then stop

// ———————————————————————————————————————————————
// 3. KEY PROTOCOL EXAMPLE: Needham-Schroeder Public Key
// ———————————————————————————————————————————————

// Keys: alice_pk (public), alice_sk (private), etc.
new alice_sk: PrivKey.
new bob_sk: PrivKey.
fun pk/1. // pk(sk) → public key
eq pk(alice_sk) = alice_pk.
eq pk(bob_sk) = bob_pk.

// Channel for communication (public, attacker-controlled)
channel ch.

// Alice’s process (initiator)
process
  new na: Bitstring.                      // Fresh nonce
  ch<pk(alice_sk), na>.                  // 1. Send her pub key + nonce
  in(ch, (pubkey_bob, enc(nb, pk(alice_sk)))).  // 2. Receive Bob’s key + nb encrypted
  out(ch, enc(na, pk(bob_sk))).          // 3. Send back na encrypted
  in(ch, enc(nb, pk(alice_sk))).         // 4. Receive nb encrypted
  // Alice authenticates Bob if nb matches what she sent
  if ... then ... else 0
|>
// Bob’s process (responder)
process
  in(ch, (alice_pk, na)).                // 1. Receive A’s key + nonce
  new nb: Bitstring.                     // Fresh nonce
  out(ch, (pk(bob_sk), enc(nb, alice_pk))).      // 2. Send B’s key + nb encrypted
  in(ch, enc(na, pk(bob_sk))).           // 3. Receive na encrypted
  out(ch, enc(nb, alice_pk)).            // 4. Send nb encrypted
|>

// Simplified (common in ProVerif tutorials):
process
  new na: Bitstring.
  out(ch, (alice_pk, na)).               // 1
  in(ch, (bob_pk, enc(nb, alice_pk))).   // 2
  out(ch, enc(na, bob_pk)).              // 3
  in(ch, enc(nb, alice_pk)).             // 4
  0
|>
|>
// (Full protocol requires freshness & replay checks — see ProVerif docs)

// ———————————————————————————————————————————————
// 4. EQUATIONAL THEORIES (Crucial for Crypto!)
// ———————————————————————————————————————————————

// Declare functions with axioms (ProVerif supports built-in and custom theories)

// Built-in theories:
// - `deduced`: attacker can derive terms using axioms
// - `destructible`: terms can be broken down (e.g., decryption)

// Common built-ins (use `open` or defaults):
// `xor` theory for symmetric crypto:
fun xor/2.
axiom xor(x, xor(x, y)) = y.
axiom xor(x, y) = xor(y, x).
axiom xor(x, xor(y, z)) = xor(xor(x, y), z).

// Symmetric encryption (e.g., AES):
fun enc/2.
fun dec/2.
axiom dec(enc(x, k), k) = x.          // Correct decryption
// Note: No axiom for enc(dec(y, k'), k) — attacker *cannot* decrypt w/ wrong key!

// Asymmetric encryption:
fun enc_pub/2.  // enc_pub(m, pk)
fun dec_pub/2.  // dec_pub(c, sk)
eq pk(sk) = pk_sk.   // map sk ↔ pk (via equality)
axiom dec_pub(enc_pub(x, pk(sk)), sk) = x.

// Hash functions (one-way — no inversion axiom):
fun hash/1.
// ProVerif *cannot* invert hash, but can use:
//   - hash(x) = hash(y) → x = y?  NO! (collision resistance not assumed)
//   - Instead, assume hash is injective *for known inputs* via `injective` declaration

// Declare injectivity (proves no collisions):
injective hash.

// Declare *destructible* (attacker can break into parts):
//   (default for pairs, tuples, lists)
destructible pair.   // pair(x,y) can be split via pattern-matching

// ———————————————————————————————————————————————
// 5. GOALS: Secrecy, Authentication, Equivalence
// ———————————————————————————————————————————————

// secrecy goal: Is `t` secret?
query t: Bitstring.
// ProVerif checks: Is `t` *always* derivable by attacker?  
// → If `false` in result: ✅ secret (attacker cannot learn `t`)

// Authentication goal (e.g., "if Bob receives `na`, Alice sent it"):
// Use `obs equivalence` or `event-based` reasoning.

// Observational equivalence (e.g., voter privacy):
process
  if b then
    out(ch, vote0)
  else
    out(ch, vote1)
|> ≈ 
process
  out(ch, vote_random)
|>
// ProVerif checks: Are the two processes *indistinguishable*?

// Event-based (using `event` declarations):
event voted(vote: Bitstring).
process
  if b = 0 then
    event voted(0); out(ch, vote0)
  else
    event voted(1); out(ch, vote1)
|>

// Then verify: 
//   - `inj-event voted` (injective event: each vote is unique)
//   - `forall v. queried(voted(v)) => queried(broadcast(v))` (auth)

// ———————————————————————————————————————————————
// 6. BUILT-IN HELPERS & UTILITIES
// ———————————————————————————————————————————————

// Lists (with `open util/list`):
//   nil, cons(x, xs), head, tail, length, append...

// Destructible tuples:
//   pair(x,y), proj1(t), proj2(t)
//   (ProVerif assumes pair is injective and destructible by default)

// Default equational theory:
//   - `pair`: destructible, injective
//   - `xor`: abelian group theory (built-in)
//   - `rsa`: asymmetric encryption axioms (if declared)

// Attacker model (Dolev-Yon):
// - Can see all public channels
// - Can generate *any* term from:
//   • Public names & constants
//   • Functions (even if not invertible, via equational logic)
//   • His own fresh names
// - Cannot invert one-way functions (e.g., hash) unless you axiomatize it

// ———————————————————————————————————————————————
// 7. RUNNING PROVERIF
// ———————————————————————————————————————————————

// Save as `protocol.apl`
// Run in shell:
$ proverif protocol.apl

// Common command-line flags:
proverif -inline protocol.apl
proverif -query-type secrecy protocol.apl

// Output:
//   - "result: secret" ✅
//   - "result: attacker(secret)" ❌ (counterexample trace given)
//   - "obs-equivalent" / "not obs-equivalent" for equivalence queries

// To get counterexample trace: use `-trace` flag.
```
