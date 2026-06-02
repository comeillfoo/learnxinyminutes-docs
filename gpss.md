---
name: GPSS
contributors:
    - ["Lenar Khannanov (Assisted-by: LLM)"]
---

GPSS is one of the oldest simulation languages (1960s, IBM), designed for
modeling discrete-event systems &mdash; especially manufacturing, logistics, and
telecommunications. Though largely superseded by modern tools (e.g., AnyLogic,
SimPy), GPSS remains elegant in its block-diagram-as-code paradigm and is still
used in niche legacy/industrial systems.

```gpss
* GPSS comment starts with *
/ GPSS also supports /* */ style

/* ———————————————————————————————
   1. BASIC CONCEPTS
   ———————————————————————————————

   In GPSS, simulation is modeled as *entities* (trucks, patients, packets)
   moving through a network of *stations* (blocks), each performing actions:
   • Arrival, processing, waiting, service, departure.

   The *transporter* is the **transaction** (also called *entity* or *unit*).
   It flows through *blocks*, which modify its state or delay it.

   ——————————————————————————————— */

* ———————————————————————————————
   2. TRANSACTION GENERATION & TERMINATION
   ———————————————————————————————

GENERATE 5,2    * Create 1st transaction at time 5, inter-arrival = 5 + EXPONENTIAL(2)
                * If only one number: deterministic inter-arrival (e.g., GENERATE 5)

TERMINATE       * Destroy the transaction (end of life)

START 10        * Run simulation for up to 10 time units (or until all terminate)
                * Without START, GPSS defaults to 1 transaction; START is required.

* Example: Generate 100 customers, then stop when all processed
GENERATE 3      * first at time 3, then every 3 time units
TERMINATE       * but we need to control *when* to stop generating

* Better: Use a counter
GENERATE 1
    ADD 1, @1    * @1 = attribute 1 (initialize to 0 by default)
    TEST L @1,100  * if @1 <= 100, continue; else skip next block
    TERMINATE
    GENERATE 1   * next customer

* ———————————————————————————————
   3. SERVICE & DELAY BLOCKS
   ———————————————————————————————

ENTER   QUEUE1    * Enter queue named "QUEUE1" (increment queue count)
ADVANCE 2,0.5   * Wait 2 ± 0.5 (uniform distribution: 1.5 to 2.5)
LEAVE   QUEUE1    * Leave queue (decrement count)
ADVANCE 3         * Service time = constant 3

* Alternative: SEIZE, ADVANCE, RELEASE (for resources)
SEIZE   SERVER    * Seize 1 unit of resource "SERVER"
ADVANCE 4         * Service time
RELEASE SERVER    * Release the resource

* Multi-unit seizure:
SEIZE   3         * Seize 3 units of the default resource (often "CPU")

* ———————————————————————————————
   4. QUEUEING & STATS
   ———————————————————————————————

QUEUE   QUE1      * Increment queue length & wait time stats
DEPART  QUE1      * Decrement queue length & record departure

* Stats are automatically collected:
   - Queue length (average, max, total time)
   - Server utilization
   - Total delay

* To display stats:
OUTPUT

* To save to file:
OUTPUT FILE('stats.out')

* ———————————————————————————————
   5. ATTRIBUTES & VARIABLES
   ———————————————————————————————

* Attributes: `@1` to `@9` (per-transaction, mutable)
SET   @1,5       * @1 ← 5
ADD   @1,1       * @1 ← @1 + 1
MULT  @1,2       * @1 ← @1 × 2

* Global variables: `V1` to `V99` (system-wide, shared)
SET   V1,100     * V1 ← 100
ADD   V1,@1      * V1 ← V1 + transaction's @1

* Special attributes:
   @0 = current simulation time
   @10 = queue delay (from ENTER to LEAVE)
   @11 = service time (from SEIZE to RELEASE)

* ———————————————————————————————
   6. BRANCHING & LOGIC
   ———————————————————————————————

* Conditional branching (skip if false):
TEST  L @1,10    * if @1 < 10 → continue; else skip next block
JUMP  BLOCK_A    * always jumps to label "BLOCK_A"

* Example: Rejection if overloaded
QUEUE   Q1
TEST  G @0,50   * if current time > 50, skip LEAVE (i.e., reject)
LEAVE   Q1
JUMP    DONE
REJECT           * terminates transaction (like TERMINATE, but counts as loss)

DONE    TERMINATE

* Logical ops (on attributes/variables):
TEST E @1,V1    * if @1 = V1
TEST N @1,0     * if @1 ≠ 0
TEST G V1,100   * if V1 > 100

* ———————————————————————————————
   7. ARRAYS & TABLES (Advanced)
   ———————————————————————————————

* Table: static lookup table (like a constant map)
TABLE   T1,10,20,0.5
* Creates table T1: values 10, 11, ..., 20 (step 0.5)

* Use in ADVANCE:
ADVANCE T1,@2    * sample from table based on @2 (rounded index)

* Array: dynamic indexed storage
ARRAY   A1,100     * A1[1..100]
SET     @1,5
SET     A1(@1),100   * A1[5] ← 100
FETCH   A1(@1),@2    * @2 ← A1[5] (now 100)

* ———————————————————————————————
   8. REALISTIC EXAMPLE: Simple Bank
   ———————————————————————————————

* Model: Customers arrive, wait in line, get served by 1 teller, leave.

GENERATE 3          * customers arrive every 3 min (avg)
QUEUE   LINE        * join queue
SEIZE   TELLER      * seize teller (1 unit)
ADVANCE 5,1         * service time: uniform [4,6] minutes
RELEASE TELLER      * release teller
DEPART  LINE        * leave queue (stats updated)
TERMINATE           * exit

START 100           * simulate 100 minutes

* Result: Probes utilization, average wait in LINE, throughput.

* ———————————————————————————————
   9. MODERN GPSS H (IBM’s evolved dialect)
   ———————————————————————————————

* Uses *block diagrams* (not text) in IDE, but text equivalent exists:
   BLOCKS:   GEN → QUE → SEIZE → ADV → REL → TERM
   Arrows = flow

* Key modern additions:
   - `SCHEDULE` — for scheduled events (e.g., maintenance)
   - `TRANSFER` — conditional flow (like JUMP, but conditional)
   - `FUNCTION` — user-defined functions
   - `STORAGE` — for bulk resources (e.g., warehouse capacity)

* Example with STORAGE (warehouse):
STORAGE WAREHOUSE,100,100  * capacity=100, current=100

GENERATE 2
   ENTER   WAREHOUSE,1     * take 1 unit
   ADVANCE 5
   LEAVE   WAREHOUSE,1     * release 1 unit
TERMINATE

* ———————————————————————————————
   10. KEY SYNTAX NOTES
   ———————————————————————————————

* Case-insensitive (GENERATE = generate)
* Blocks are *sequential* (flow top-to-bottom)
* Labels (e.g., `BLOCK_A:`) are optional but help readability
* Comments start with `*` (column 1) or `/`
* No explicit semicolons — blocks are line-delimited
* Default time unit: arbitrary (but usually minutes/hours)

* Common blocks summary:

BLOCK         ACTION
──────────────────────────────────────────────────────
GENERATE      create new transaction (with optional delay)
TERMINATE     destroy transaction (exit)
QUEUE         increment queue count, record wait start
DEPART        decrement queue count, record wait end
SEIZE         claim resource unit(s)
RELEASE       release resource unit(s)
ADVANCE       delay (deterministic or random)
TEST          conditional skip (skip if false)
JUMP          unconditional branch
SET           assign to attribute/variable
FETCH         read array value
ARRAY         declare indexed storage

* ———————————————————————————————
   11. RUNNING GPSS
   ———————————————————————————————

1. Save as `model.gpss` (or `.gpss` / `.gpl`)
2. Run in GPSS/H interpreter (IBM, LINDA, or modern forks like *OpenGPSS*)
   $ gpss model.gpss
3. Output appears in console or `output.txt`

* To generate block diagram (in IBM GPSS/H):
   → Use GUI or `DRAW` command.

* To inspect stats interactively:
   → Type `STATS` at prompt during simulation.
```
