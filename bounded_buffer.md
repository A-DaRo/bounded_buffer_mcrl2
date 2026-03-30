# The Bounded Buffer from a CCS perspective

**Observable Behavior:** We first introduce the observable behavior of the bounded buffer. At a high level, the bounded buffer prints items sequentially until NROF_ITEMS -1 is reached, the process then terminates. Formally, let $N$ denote NROF_ITEMS. Let the observable action of printing an item to stdout be denoted by the output action $out(k)$. Let $0$ denote the inactive (nil) process representing termination.

We define a parameterized process $P(k)$ representing the system state when it is ready to print item $k$:
$P(k) \stackrel{\text{def}}{=} [k < N] \, out(k) . P(k + 1) + [k = N] \, 0$

The overall visible behavior of the system is initialized at $k = 0$: $System \stackrel{\text{def}}{=} P(0)$

**Internal logic and components of the Bounded Buffer:** Now that we have defined the visible behavior, we proceed with dissecting each system component in isolation, finally we derive the overall CCS formula describing the entire system internal logic.

### Next Item Function:

The CCS expression presented below, models the internal state of ‘get_next_item()‘ using a request counter $c$ and a set of already issued jobs $S$. Upon receiving a request (‘req‘), it increments $c$ and evaluates four mutually exclusive branches: if all $N$ jobs are assigned, it returns the termination flag $N$; if $c \leq P$, it non-deterministically ($\tau$) selects and returns any unissued job. To guarantee deadlock avoidance for subsequent requests ($c > P$), it evaluates the required item $c - P$—if it has not yet been issued, the function deterministically returns it; if it is already in $S$, it falls back to a non-deterministic choice among the remaining available items. After outputting the selected value via $\overline{rep}(x)$, it recursively loops back to its initial state with the newly updated set $S$.

$GetNextItem(c, S) \stackrel{\text{def}}{=} req . Eval(c + 1, S)$

$Eval(c, S) \stackrel{\text{def}}{=} [c > N] \, \overline{rep}(N) . GetNextItem(c, S)$
$+ [c \leq P] \sum_{x \in V \setminus S} \tau . \overline{rep}(x) . GetNextItem(c, S \cup \{x\})$
$+ [P < c \leq N \land (c - P) \notin S] \, \overline{rep}(c - P) . GetNextItem(c, S \cup \{c - P\})$
$+ [P < c \leq N \land (c - P) \in S] \sum_{x \in V \setminus S} \tau . \overline{rep}(x) . GetNextItem(c, S \cup \{x\})$

**Initial State:** $GetNextItem(0, \emptyset)$

**Legend:**
* $N = \text{NROF\_ITEMS}$
* $P = \text{NROF\_PRODUCERS}$
* $V = \{0, 1, \dots, N - 1\}$ (the set of all valid items)
* $c = \text{counter of jobs requested}$
* $S = \text{set of already issued jobs}$
* $req = \text{input action (function call)}$
* $\overline{rep}(v) = \text{output action (function return value)}$
* $\tau = \text{internal unobservable action (random selection)}$

---

### Buffer and Consumer:

The Buffer and Consumer are presented together, the two processes synchronize on the variable(s) $get(x), \overline{get}(x)$. The ‘Buffer‘ handles inputs from external producers (‘put‘) as long as its capacity $B$ is not exceeded, and serves items to the consumer strictly in FIFO order using $head(Q)$ and $tail(Q)$. The ‘Consumer‘ fetches items from the buffer, performs an unobservable random sleep ($\tau$), emits the observable output ($out(x)$), and increments its counter $c$ until all $N$ items are processed, at which point it terminates gracefully ($0$).

$Buffer(Q) \stackrel{\text{def}}{=} [|Q| < B] \, put(x) . Buffer(Q \cdot \langle x \rangle)$
$+ [|Q| > 0] \, get(head(Q)) . Buffer(tail(Q))$

$Consumer(c) \stackrel{\text{def}}{=} [c < N] \, get(x) . \tau . out(x) . Consumer(c + 1)$
$+ [c = N] \, 0$

**Legend:**
* $N = \text{NROF\_ITEMS}$
* $B = \text{BUFFER\_SIZE}$
* $Q = \text{sequence (queue) of items currently in the buffer}$
* $|Q| = \text{current number of items in the buffer}$
* $\epsilon = \text{empty queue}$
* $Q \cdot \langle x \rangle = \text{queue Q with item x appended to the tail}$
* $head(Q) = \text{the first item at the front of queue Q}$
* $tail(Q) = \text{queue Q with the first item removed}$
* $c = \text{counter of items consumed so far}$
* $put(x) = \text{input action (from producers) placing item x into the buffer}$
* $get(x), \overline{get}(x) = \text{internal synchronization channel between the Buffer and Consumer}$
* $out(x) = \text{observable output action (printing item x to stdout)}$
* $\tau = \text{unobservable internal action (representing the Consumer’s random sleep/work time)}$

### Producers and Turn Coordination, Broadcast and Signal solutions:

The Producer(s) behavior, combined with the ‘get_next_item()‘ function, is the central core of the enhancements to the classical bounded buffer problem. We distinguish two possible solutions, one based on broadcast calls and one implementing private signal interactions.

**Broadcast Solution:** The CCS expression models the independent producer threads and their strict requirement to insert items in ascending order. Each $Prod_i$ fetches a job via $req$ and $\overline{rep}(x)$. If $x = N$, it means all jobs are exhausted, and the producer gracefully terminates ($0$). For a valid job ($x < N$), the producer undergoes an unobservable random sleep ($\tau$). To satisfy the requirement that items enter the buffer in strictly ascending order ($0, 1, 2, \dots, N - 1$), the producers coordinate with the $Turn(e)$ process. The producer waits to synchronize on the restricted channel $pass(x)$, which the $Turn$ process only enables when $x$ matches the current expected sequence number $e$. Once synchronized, the producer places its item in the buffer ($put(x)$) and loops back for its next job, while $Turn$ increments its counter to allow the next sequential item.

$Turn(e) \stackrel{\text{def}}{=} [e < N] \, pass(e) . Turn(e + 1) + [e = N] \, 0$
$Prod_i \stackrel{\text{def}}{=} req . \overline{rep}(x) . ([x = N] \, 0 + [x < N] \, \tau . \overline{pass}(x) . put(x) . Prod_i)$
$Producers \stackrel{\text{def}}{=} \left( \prod_{i=0}^{P-1} Prod_i \right) \mid Turn(0) \setminus \{pass\}$

---

### Legend:
* $N = \text{NROF\_ITEMS}$
* $P = \text{NROF\_PRODUCERS}$
* $e = \text{the expected next item to be placed into the buffer (turn counter)}$
* $x = \text{the job number (item) fetched by a producer}$
* $i = \text{unique identifier for a producer thread (0 \leq i < P)}$
* $req, \overline{rep}(x) = \text{actions to call and receive the return value from get\_next\_item()}$
* $\tau = \text{unobservable internal action representing the random sleep (simulating work/packaging)}$
* $pass(x), \overline{pass}(x) = \text{internal synchronization channel enforcing the ascending order turn}$
* $put(x) = \text{output action placing the item x into the buffer}$
* $\prod_{i=0}^{P-1} = \text{parallel composition over all P producers}$

**Signal Solution:** In order to model direct signaling between independent processes, we introduce a coordinator subroutine $Coord(e, O)$ process that maintains a mapping ($O$) of fetched items to their respective producer threads. When a $Prod_i$ fetches an item $x$, it registers its ownership via $reg(x, i)$ and strictly waits on its private synchronization channel $turn_i(x)$. The coordinator constantly monitors the expected item $e$; as soon as $e$ is registered by some thread $j$, it explicitly signals only that specific thread via $\overline{turn_j}(e)$. After the producer inserts the item and acknowledges ($ack$), the coordinator increments $e$ and immediately evaluates the next target, entirely eliminating broadcasting and spurious wakeups.

$Coord(e, O) \stackrel{\text{def}}{=} [e < N] (\sum_{x,i} \overline{reg}(x, i) . Coord(e, O \cup \{(x, i)\}) + [\exists j : (e, j) \in O] \, \overline{turn_j}(e) . ack . Coord(e + 1, O \setminus \{(e, j)\})) + [e = N] \, 0$

$Prod_i \stackrel{\text{def}}{=} req . \overline{rep}(x) . ([x = N] \, 0 + [x < N] \, \tau . reg(x, i) . turn_i(x) . put(x) . ack . Prod_i)$

$Producers \stackrel{\text{def}}{=} \left( \prod_{i=0}^{P-1} Prod_i \right) \mid Coord(0, \emptyset) \setminus \{reg, turn_k, ack \mid k \in Z_P \}$

**Legend:**
* $O = \text{Set of pairs (x, i) mapping an item x to its owning thread i}$
* $reg(x, i) = \text{Channel to register that thread i owns item x}$
* $turn_i(x) = \text{Private channel (thread-local CV) to explicitly wake up thread i}$
* $ack = \text{Channel to acknowledge the item was placed, advancing the turn counter}$

---

### Complete Bounded Buffer System

To conclude the formalization of the bounded buffer problem, we unify the distinct subsystems—the job generator, the producers, the buffer, and the consumer—into a single, comprehensive CCS process. By running these components in parallel and restricting all internal communication channels, we isolate the system’s observable behavior.

$BoundedBuffer \stackrel{\text{def}}{=} (GetNextItem(0, \emptyset) \mid Producers \mid Buffer(\epsilon) \mid Consumer(0)) \setminus \{get, req, rep, put\}$

**Legend:**
* $BoundedBuffer = \text{the overall system process encompassing all threads and resources}$
* $| = \text{parallel composition operator allowing independent processes to interact}$
* $\setminus \{...\} = \text{restriction operator hiding internal synchronization channels from the environment}$
* $req, rep = \text{internal channels linking the producers to the get\_next\_item() function}$
* $put = \text{internal channel linking the producers to the buffer}$
* $get = \text{internal channel linking the consumer to the buffer}$

**Pseudocode Implementations of the CCS expressions:**

**Consumer - Buffer Synchronization**
```c
1 void consumer () {
2 int consumed_items = 0;
3 int out = 0; // Buffer extraction index
4
5 while ( consumed_items < NROF_ITEMS ) {
6 pthread_mutex_lock (& buffer_mutex );
7
8 // Wait until there is at least one item in the buffer
9 while ( count == 0) {
10 pthread_cond_wait (& not_empty , & buffer_mutex );
11 }
12
13 // Extract the item from the buffer
14 int item = buffer [ out ];
15 out = ( out + 1) % BUFFER_SIZE ;
16 count --;
17
18 // Wake up a blocked producer waiting for buffer space
19 pthread_cond_signal (& not_full ) ;
20
21 pthread_mutex_unlock (& buffer_mutex );
22
23 sleep_random () ; // Simulate processing work outside critical section
24 printf ("%d\n", item ); // Output observable behavior
25
26 consumed_items ++;
27 }
28 }
```
The consumer thread continuously fetches items from the buffer until all NROF_ITEMS have been processed. It locks the `buffer_mutex` and waits on the `not_empty` condition variable to ensure it does not read from an empty array. Because the producers have already guaranteed that items are placed in absolute ascending order, the consumer simply acts as a standard FIFO reader. After removing an item and decrementing the buffer count, it signals the `not_full` condition variable to awaken any producer waiting for buffer capacity. Finally, the consumer releases the lock, simulates its random processing delay, and prints the item to standard output, producing the required strict sequential behavior.

---

**Producer (Broadcast) - Buffer Synchronization**
```c
1 // Shared variables
2 int expected_item = 0; // The absolute sequence number required next
3 int count = 0; // Number of items currently in the buffer
4 int in = 0; // Buffer insertion index
5 int buffer [ BUFFER_SIZE ];
6
7 pthread_mutex_t buffer_mutex = PTHREAD_MUTEX_INITIALIZER ;
8 pthread_cond_t not_full = PTHREAD_COND_INITIALIZER ;
9 pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER ;
10 pthread_cond_t turn_cv = PTHREAD_COND_INITIALIZER ;
11
12 void producer () {
13 while ( true ) {
14 int item = get_next_item () ;
15 if ( item == NROF_ITEMS ) break ; // All jobs distributed
16
17 sleep_random () ; // Simulate packaging / work outside critical section
18
19 pthread_mutex_lock (& buffer_mutex );
20
21 // Wait until it is this item ’s absolute turn to be placed in the buffer
22 while ( item != expected_item ) {
23 pthread_cond_wait (& turn_cv , & buffer_mutex ) ;
24 }
25
26 // Wait until there is space in the buffer
27 while ( count == BUFFER_SIZE ) {
28 pthread_cond_wait (& not_full , & buffer_mutex );
29 }
30
31 // Insert item into the bounded buffer
32 buffer [ in ] = item ;
33 in = ( in + 1) % BUFFER_SIZE ;
34 count ++;
35 expected_item ++; // Advance the turn for the next sequential item
36
37 // Wake up the consumer and any waiting producers
38 pthread_cond_signal (& not_empty );
39 pthread_cond_broadcast (& turn_cv );
40
41 pthread_mutex_unlock (& buffer_mutex );
42 }
43 }
```
The producer threads must enforce the strict absolute ascending order before attempting to place an item into the buffer. By locking the `buffer_mutex` and waiting on the `turn_cv` condition variable, a producer halts its execution until its fetched item exactly matches the `expected_item` global counter. Once it is the producer’s turn, it ensures the buffer is not full by waiting on the `not_full` condition variable. After safely inserting the item, the producer increments the `expected_item` counter and broadcasts to `turn_cv` to awaken any sibling producers that might be holding the next sequential item, while also signaling `not_empty` to alert the consumer.

**Producer (Signal) - Buffer Synchronization**
```c
1 // Shared variables
2 int expected_item = 0;
3 int count = 0, in = 0;
4 int buffer [ BUFFER_SIZE ];
5 int item_owner [ NROF_ITEMS ]; // Initialize all elements to -1
6
7 pthread_mutex_t buffer_mutex = PTHREAD_MUTEX_INITIALIZER ;
8 pthread_cond_t not_full = PTHREAD_COND_INITIALIZER ;
9 pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER ;
10 pthread_cond_t coord_cv [ NROF_PRODUCERS ]; // Initialize array of CVs
11
12 void producer ( int thread_id ) { // thread_id is 0 to NROF_PRODUCERS - 1
13 while ( true ) {
14 int item = get_next_item () ;
15 if ( item == NROF_ITEMS ) break ;
16
17 sleep_random () ;
```
(Code continued from page 6)
```c
18
19 pthread_mutex_lock (& buffer_mutex );
20
21 // Register ownership of this specific item
22 item_owner [ item ] = thread_id ;
23
24 // Wait on the producer ’s strictly private condition variable
25 while ( item != expected_item ) {
26 pthread_cond_wait (& priv_cv [ thread_id ], & buffer_mutex );
27 }
28
29 while ( count == BUFFER_SIZE ) {
30 pthread_cond_wait (& not_full , & buffer_mutex );
31 }
32
33 buffer [ in ] = item ;
34 in = ( in + 1) % BUFFER_SIZE ;
35 count ++;
36 expected_item ++;
37
38 pthread_cond_signal (& not_empty );
39
40 // O (1) targeted wakeup : check if the next expected item is already claimed
41 if ( expected_item < NROF_ITEMS && item_owner [ expected_item ] != -1) {
42 int target_thread = item_owner [ expected_item ];
43 pthread_cond_signal (& priv_cv [ target_thread ]) ;
44 }
45
46 pthread_mutex_unlock (& buffer_mutex );
47 }
48 }
```
Each producer thread is mapped to a dedicated condition variable inside `coord_cv`. The `item_owner` array acts as the lookup table (the set $O$ in CCS), mapping an item index to the `thread_id` that fetched it. When a thread successfully inserts its item into the buffer, it simply looks up the `thread_id` that owns the newly incremented `expected_item`. If the item has already been fetched and registered by a peer thread, it selectively triggers `pthread_cond_signal` on that peer’s private condition variable.