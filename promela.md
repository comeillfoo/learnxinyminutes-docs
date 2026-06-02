---
name: Promela
contributors:
    - ["Lenar Khannanov (Assisted-by: LLM)"]
---

PROMELA (Process or Protocol Meta Language) is a verification modeling language.
The language allows for the dynamic creation of concurrent processes to model,
for example, distributed systems.

```promela
// Single-line comment starts with //

/*
   Multi-line
   comments
*/

// ———————————————————————————————
// BASIC TYPES & VARIABLES
// ———————————————————————————————

bool flag;                  // Boolean (0 or 1)
byte b = 42;                // 8-bit unsigned integer (0..255)
short int si = -10;         // Signed 16-bit integer (−32768..32767)
mtype = { REQUEST, ACK, DONE };  // Symbolic type (enum of strings)

int x = 5;
chan ch = [1] of { int, mtype };  // Buffered channel: size 1, holds int or mtype values

// Global variables visible to all processes (shared memory)
bool shared = false;

// Local to a process (each process has its own copy)
// Declared in a proctype (see below)


// ———————————————————————————————
// PROCESSES (PROTOCOLS/AGENTS)
// ———————————————————————————————

// A process is defined by a proctype. You can have multiple instances.
proctype Sender(byte id) {
    byte seq = 0;

    // Initialization (optional, but often used)
    do
    :: seq < 5 ->
        // Send a message
        ch!id;           // Non-blocking send (if buffer not full)
        // ch!seq, REQUEST;  // Send multiple items: tuple
        seq++;
    :: else ->           // Fallback (optional, like `break`)
        break;
    od
}

// Another process (e.g., receiver)
proctype Receiver(byte id) {
    int msg;

    // Receive from channel
    ch?msg;              // Blocking receive (waits until message available)

    // Conditional check
    if
    :: msg > 0 ->
        printf("Received %d from %d\n", msg, id)
    :: else ->
        printf("Empty or error\n")
    fi
}


// ———————————————————————————————
// CONTROL FLOW
// ———————————————————————————————

// Conditional: if/-fi (not "if/end")
if
:: b == 42 ->
    printf("B is 42\n");
:: b != 42 ->
    printf("B is not 42\n");
fi

// Loops:
do
:: flag == false ->
    flag = true;
:: flag ->
    break;
od

// assert(condition); — used to verify invariants (fails verification if false)
assert(x >= 0);

// skip; — no-op (like NOP)
skip;

// atomic { ... } — executes block atomically (no interleaving)
atomic {
    x = x + 1;
    y = x;
}


// ———————————————————————————————
// CHANNELS & COMMUNICATION
// ———————————————————————————————

// Channels are FIFO, typed, and optionally bounded
chan q = [M] of { int };     // M = max messages in flight (0 = unbuffered / rendez-vous)

// Send (non-blocking if M > 0; blocking if M = 0)
q!42;                        // Send value 42
q!a, b;                      // Send tuple (a and b)

// Receive (blocking)
q?x;                         // Receive into local variable x
q?a, b;                      // Receive tuple

// Non-deterministic receive with guard
q?val;                       // Any process can receive — model checker explores all interleavings
if
:: (val > 0) -> ...
fi


// ———————————————————————————————
// INITIALIZATION & EXECUTION
// ———————————————————————————————

// One or more processes are started in init block
init {
    // Spawn multiple instances of processes
    run Sender(1);
    run Sender(2);
    run Receiver(10);
    run Receiver(11);

    // Or inline processes (like procedures)
    // (But only one `init` block allowed)
}

// ———————————————————————————————
// USEFUL BUILT-INS
// ———————————————————————————————

// printf(format, args...) — prints to stderr at runtime
printf("Hello, world\n");

// atomic — ensures block runs without preemption
// (use sparingly; defeats concurrency)

// prefer(cond1, cond2) — prioritize transition when cond1 && !cond2
// (advanced: helps avoid starvation)

// _nr_end — marks non-deterministic choice point
do
::_nr_end -> break;
od

// LTL properties (for SPIN verification):
// (written outside model, but often included in .pml as comments or in .pml header)
// ltl p1 { []<>(p && !q) }   // "Eventually p and not q always holds"
```
