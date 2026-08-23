# Interview Q&A — Rubik's Cube Solver

---

## 1. Question

**Walk me through the problem your project solves and why the approach isn't trivial. What makes a Rubik's Cube hard for a computer?**

### What the Interviewer Is Testing
Product understanding; ability to frame a computational problem; awareness of state-space complexity and why brute force fails.

### Strong Answer

The 3×3 Rubik's Cube has approximately 43 quintillion possible states — 4.3×10¹⁹, to be exact. Every one of those states is solvable in at most 20 moves, which is called God's Number. The challenge isn't that solutions don't exist — it's that finding one efficiently is a graph search problem with a branching factor of 18, since each move has 6 faces × 3 types (clockwise, counter-clockwise, 180°).

Brute-force BFS works for shallow scrambles — maybe up to 7 moves — but at depth 7 you're looking at 18⁷ ≈ 612 million states just for that depth level. Memory explodes. DFS has the opposite problem: it's memory-efficient but doesn't guarantee an optimal solution and can explore incredibly deep branches that never lead to a solution.

The approach I took was to implement IDA* — Iterative Deepening A* — which combines DFS's memory efficiency with a heuristic that prunes the search tree. The heuristic comes from a pre-computed corner pattern database: I run BFS from the solved state to record the minimum moves needed to solve every possible corner configuration (about 88 million of them). During search, this heuristic tells IDA* "you need at least X more moves from here," and any path that exceeds the current bound gets pruned. This reduces the effective branching factor from 18 down to roughly 3–5.

The key insight is that the heuristic is admissible — it never overestimates the true distance — so IDA* is guaranteed to find an optimal solution.

### Likely Follow-Up
"Why can't you just use A* with that heuristic instead of IDA*?"

### Strong Follow-Up Direction
Explain that standard A* stores the entire open set and closed set in memory. For the Rubik's Cube, even with pruning, the number of explored states can be millions. IDA* restarts with increasing bounds, trading repeated work for bounded memory. Mention that your implementation actually uses a priority queue within each bound iteration (more like bounded A*), which is a hybrid approach.

### Red Flags to Avoid
- Saying "it's just a BFS problem" without acknowledging the exponential state space.
- Claiming the project finds God's Number solutions — it finds optimal solutions from a given scramble, which is different from proving God's Number.
- Confusing the branching factor (18) with the number of faces (6).

---

## 2. Question

**You implemented three different internal representations for the Rubik's Cube. Why? Wouldn't one be enough?**

### What the Interviewer Is Testing
Design judgment; understanding of data structure trade-offs; ability to justify complexity versus simplicity.

### Strong Answer

I implemented three representations — a 3D array `char[6][3][3]`, a 1D flat array `char[54]`, and a bitboard `uint64_t[6]` — behind a shared abstract base class. The honest reason is partly pedagogical: this is a portfolio project and I wanted to demonstrate OOP polymorphism and explore how data layout affects performance in a computationally intensive context.

But there's a genuine engineering reason too. The 3D array maps directly to the physical cube — face 0, row 1, column 2 is exactly where you'd expect. It's easy to debug and verify move correctness. The 1D array is more cache-friendly because it's contiguous memory. The bitboard is where the real performance comes — face rotations become bit-shift operations, and hashing the entire cube state is just XOR of 6 integers.

The trade-off is maintenance: three implementations means three sets of move logic to keep correct. In a production system, I'd keep only the bitboard for solving and the 3D array for debugging and testing. The 1D array doesn't offer enough of an advantage over either to justify its existence.

The abstract interface approach has a real benefit, though: all four solvers are C++ templates parameterised on the representation type and hash function. So any solver works with any representation with zero code duplication and zero runtime dispatch overhead.

### Likely Follow-Up
"How much faster is the bitboard compared to the 3D array in practice?"

### Strong Follow-Up Direction
Be honest: the repository doesn't contain benchmarks. Explain that the theoretical advantage is O(1) bit-shift face rotation vs. O(9) element swaps, and O(1) XOR hashing vs. O(54) string concatenation. Suggest how you'd benchmark: same scramble, same algorithm, measure wall-clock time and cache misses. Mention that the hash function quality matters more than move speed for BFS/IDA* because hash table operations dominate.

### Red Flags to Avoid
- Claiming large speedup numbers without having measured them.
- Not knowing which representation is used in the active code path (bitboard).
- Saying the virtual functions in the base class cause performance problems — the solvers use templates to avoid virtual dispatch.

---

## 3. Question

**Explain the bitboard representation in detail. How does one-hot encoding work for colors, and how does a face rotation become a bit shift?**

### What the Interviewer Is Testing
Deep technical understanding of bit manipulation; ability to explain low-level optimisations clearly.

### Strong Answer

Each face of the cube has 9 stickers, but the center never moves — it defines the face's color. So I only encode 8 stickers per face. Each sticker gets 8 bits, and the color is stored as a one-hot encoding: WHITE = bit 0 set (value 1), GREEN = bit 1 (value 2), RED = bit 2 (value 4), and so on for 6 colors. That's 8 stickers × 8 bits = 64 bits = one `uint64_t` per face.

The stickers are arranged clockwise starting from the top-left: positions 0 through 7. When you rotate a face 90° clockwise, each sticker moves to the position two indices later (0→2, 1→3, ..., 6→0, 7→1). In the bitboard, that means shifting the color data by 2 × 8 = 16 bits. The rotation becomes: `bitboard[face] = (bitboard[face] << 16) | (bitboard[face] >> 48)`. That's one or two CPU instructions.

Side rotations — the stickers on adjacent faces that move when you rotate a face — are more complex. Those use bitmask extraction and insertion: extract 3 sticker values (24 bits) from one face, insert them into the correct positions on another face. The `rotateSide` helper function handles this with masks and shifts.

Hashing is also elegant: XOR all 6 `uint64_t` values together. It's O(1) and produces a reasonable hash, though I'd note the collision rate is higher than ideal because XOR is symmetric — swapping two faces gives the same hash.

The trade-off: `getColor()` has to count bits to recover the color index from the one-hot encoding, which is a small loop. And the bit manipulation makes the move code harder to verify visually compared to array swaps.

### Likely Follow-Up
"Why one-hot encoding instead of just storing a 3-bit color index?"

### Strong Follow-Up Direction
Explain that 3-bit encoding would pack more data per word but face rotation would require extracting, rearranging, and repacking individual 3-bit fields — which is significantly more complex than a uniform bit shift. One-hot uses more space per sticker (8 bits vs. 3) but the uniform sticker size enables the elegant shift-based rotation.

### Red Flags to Avoid
- Not knowing that the center sticker is implicit (not stored).
- Confusing one-hot encoding with binary encoding.
- Claiming the bitboard uses "zero memory" or similar exaggerations.

---

## 4. Question

**How does the corner pattern database work? Walk me through the indexing scheme — how do you map a corner configuration to a database index?**

### What the Interviewer Is Testing
Understanding of combinatorics, admissible heuristics, and practical data structure design.

### Strong Answer

The corner pattern database stores the minimum number of moves to solve just the 8 corners of the cube, ignoring edges. Each corner has two properties: its position (which of the 8 corner slots it's in) and its orientation (which of 3 rotations it's in). The total number of distinct corner configurations is 8! × 3⁷ = 88,179,840. The 8th corner's orientation is determined by the other 7, which is why it's 3⁷ and not 3⁸.

The index has two parts. First, I convert the corner permutation (which corner is in which slot) to a lexicographic rank using a Lehmer code. The `PermutationIndexer<8>` template class does this in O(8) time using precomputed ones-count lookup tables and factorial values. The Lehmer code works by counting, for each position, how many unseen elements are smaller than the current element — this gives a mixed-radix number that converts to a unique rank in 0..40,319.

Second, I compute an orientation number. Each of the 7 independent corners has orientation 0, 1, or 2. I treat these as digits in base 3: `o[0]×729 + o[1]×243 + o[2]×81 + ... + o[6]×1`. This gives a number in 0..2,186.

The final index is `rank × 2187 + orientationNumber`. This maps every corner configuration to a unique position in the database array, where I store the pre-computed minimum move count.

To generate the database, I run BFS from the solved state. At each step, I apply all 18 moves to each state, compute the new index, and if it hasn't been seen at a smaller depth, I record the depth and enqueue. The generator runs to depth 8 — by that point, most of the 88 million states have been reached.

### Likely Follow-Up
"Why does the database size in the code say 100,179,840 instead of 88,179,840?"

### Strong Follow-Up Direction
Acknowledge this discrepancy. The theoretical number of corner states is 8! × 3⁷ = 88,179,840, but the code uses 100,179,840. This likely includes padding or an earlier calculation. The candidate should say they'd need to verify this — it might be `8! × 3⁷` rounded up for alignment, or `8^8 × 3⁷` from a different encoding scheme. Being honest about not knowing is better than guessing.

### Red Flags to Avoid
- Not understanding why 3⁷ instead of 3⁸.
- Saying "it's just a hash table" — it's a direct-address array, not a hash table.
- Claiming the database covers all possible cube states (it covers corner states only).

---

## 5. Question

**Your IDA\* implementation uses a priority queue and a visited set. That's unusual — standard IDA\* is recursive with O(d) memory. Why did you make this choice, and what's the trade-off?**

### What the Interviewer Is Testing
Understanding of algorithm variants; ability to reason about memory vs. computation trade-offs; honesty about design decisions.

### Strong Answer

You're right — textbook IDA* is a simple recursive DFS with a threshold cutoff, and its key advantage is O(d) memory where d is the solution depth. My implementation is more of a hybrid between A* and IDA*: within each bound iteration, I use a priority queue ordered by `f(n) = g(n) + h(n)`, plus `unordered_map`s for visited states and back-pointers.

The benefit is that within each iteration, I don't re-visit states — the visited set prevents redundant exploration. Standard recursive IDA* re-explores the same state many times within a single iteration if it's reachable via multiple paths. For the Rubik's Cube, where the same state can be reached by many different move sequences, this can be significant.

The trade-off is that I've lost IDA*'s primary advantage: O(d) memory. My visited set and priority queue can grow to hold millions of states, each consuming ~50+ bytes in the hash map. For deep scrambles, this could become a memory bottleneck.

If I were to redesign this, I'd probably implement pure recursive IDA* for better memory characteristics and only add transposition tables (limited-size caches of visited states) if profiling showed that redundant exploration was the bottleneck. The current approach works for the 13-move scrambles I've tested, but it wouldn't scale well to 18+ move scrambles.

### Likely Follow-Up
"How would you decide between the two approaches?"

### Strong Follow-Up Direction
Benchmark both. Measure: (1) wall-clock time for scrambles of depth 8, 10, 12, 14; (2) peak memory usage; (3) states explored. If the hybrid is faster but uses 10× more memory, consider a bounded transposition table (LRU cache of visited states with a size limit) as a middle ground.

### Red Flags to Avoid
- Not recognising that the implementation deviates from standard IDA*.
- Claiming the priority queue approach is strictly better.
- Not understanding that the visited set destroys IDA*'s O(d) memory guarantee.

---

## 6. Question

**The BFS solver uses `unordered_map<T, bool, H>` for visited states and `unordered_map<T, MOVE, H>` for back-pointers. What's the memory cost, and how would you reduce it?**

### What the Interviewer Is Testing
Understanding of hash map internals; memory-aware engineering; knowledge of alternative data structures.

### Strong Answer

Each `unordered_map` entry for the bitboard representation stores: the key (6 × 8 bytes = 48 bytes for the `uint64_t[6]` bitboard), the value (1 byte for `bool` or 4 bytes for the `MOVE` enum), plus hash table metadata (next pointer, hash cache — typically 16–24 bytes per bucket on 64-bit systems). So each state costs roughly 70–90 bytes across both maps.

At BFS depth 7, we're looking at up to 18⁷ ≈ 612 million states at the frontier alone, plus all states at depths 0–6. Even with pruning via the visited set, the practical state count at depth 7 can be tens of millions, consuming several gigabytes.

To reduce memory, several approaches:

1. **Merge the two maps into one**: Instead of `unordered_map<T, bool>` and `unordered_map<T, MOVE>`, use a single `unordered_map<T, MOVE>` where the presence of a key implies visited=true.

2. **Use a more compact hash set**: Replace `unordered_map` with `unordered_set` and store move information differently. Or use a flat hash map (like `robin_map` or Abseil's `flat_hash_map`) which has much lower per-entry overhead.

3. **Compress the cube state**: Instead of storing 48 bytes per key, compute a unique 8-byte hash or use the corner encoding (4 bytes) + edge encoding as the key.

4. **Use bidirectional BFS**: Search from both the scrambled state and the solved state, meeting in the middle. This reduces the maximum search depth by half, exponentially reducing memory.

In practice, BFS is only used for shallow scrambles in this project — IDA* handles deeper ones.

### Likely Follow-Up
"Tell me about the hash function. What's the collision rate?"

### Strong Follow-Up Direction
The `HashBitboard` uses XOR of 6 `uint64_t` values. XOR is symmetric, so `bitboard[0]=A, bitboard[1]=B` hashes the same as `bitboard[0]=B, bitboard[1]=A` when combined via XOR — that's a lot of collisions. A better approach would be to use boost-style hash_combine: `hash ^= value + 0x9e3779b9 + (hash << 6) + (hash >> 2)`, which breaks symmetry.

### Red Flags to Avoid
- Not knowing the approximate per-entry cost of `unordered_map`.
- Suggesting a `std::map` (tree-based) as a "more efficient" alternative — it's slower and uses more memory per entry.
- Ignoring the hash function quality issue.

---

## 7. Question

**How does the webcam scanner classify colors? What are the failure modes, and how would you make it more robust?**

### What the Interviewer Is Testing
Computer vision fundamentals; understanding of real-world sensing challenges; practical improvement thinking.

### Strong Answer

The scanner captures each face from the webcam, samples a 5×5 pixel region at the center of each grid cell, applies a channel-wise median filter (sort each BGR channel independently, take the median), converts the median BGR pixel to HSV color space, and classifies using hardcoded hue thresholds: Red is 160–190°, Orange is 3–19°, Yellow is 20–30°, Green is 60–90°, Blue is 100–120°. White is detected separately using an RGB similarity check — all three channels above 200 and pairwise differences below 30.

The failure modes are:

1. **Lighting variations**: Under warm lighting, white looks yellow; under fluorescent lighting, colors shift. Fixed thresholds break immediately.
2. **Shadows and reflections**: Cube faces aren't uniformly lit — edges have shadows, centers may have glare. The 5×5 median helps but doesn't solve this.
3. **Hue gaps**: Some hue ranges aren't covered (31–59, 91–99, 121–159). A sticker in these ranges gets classified as White by the default fallback — silently wrong.
4. **No validity check**: After scanning all 6 faces, the system doesn't verify that exactly 9 stickers of each color were detected, or that the resulting configuration is a valid Rubik's Cube permutation. An invalid scan goes straight to the solver, which may run forever.

To improve:

- **Calibration mode**: Have the user show each face color once; record HSV ranges adaptively.
- **K-means clustering**: Cluster all 54 sampled colors into 6 groups and assign based on cluster centers.
- **ML classifier**: Train a small CNN or SVM on labelled sticker crops.
- **Validity check**: After scanning, verify the cube state is a valid permutation (correct parity, correct orientation sum).

### Likely Follow-Up
"Why HSV instead of RGB for classification?"

### Strong Follow-Up Direction
RGB mixes brightness with color identity — a dark red and a bright red have very different RGB values. HSV separates Hue (color identity), Saturation (color intensity), and Value (brightness). This means you can classify color by hue alone, regardless of lighting intensity. It's not perfect (hue wraps around at 0/180 for red, and low-saturation colors lose hue information), but it's the standard first approach for color classification in CV.

### Red Flags to Avoid
- Saying "HSV is always better than RGB" — it has limitations (e.g., white/grey have undefined hue).
- Not mentioning the missing cube validity check as a critical failure mode.
- Suggesting deep learning as the first solution without mentioning simpler alternatives.

---

## 8. Question

**Your `RubiksCube` base class has 18 pure virtual functions for moves. Is that a good design, or would you do it differently?**

### What the Interviewer Is Testing
OOP design judgment; understanding of interface design trade-offs; ability to critique own code.

### Strong Answer

Having 18 pure virtual functions — `f()`, `fPrime()`, `f2()`, `u()`, `uPrime()`, `u2()`, and so on for 6 faces — is honest about the interface but arguably too granular. The `move(MOVE ind)` function in the base class already dispatches to the appropriate virtual function via a switch statement, and `invert(MOVE ind)` does the same for inverses. The solvers only ever call `move()` and `invert()`, never the individual move functions directly.

The alternative would be to have a single `virtual RubiksCube& applyMove(MOVE) = 0` function and let each implementation handle the dispatch internally. This would reduce the interface from 18 virtual functions to 1, making it easier to implement new representations.

However, the current design has a benefit: each move function can be independently optimised. In the bitboard, `u()` uses a very different bit manipulation pattern than `l()`, so having them as separate functions is natural for the implementation. If they were all behind a single `applyMove()`, the implementation would just have an internal switch — functionally equivalent, but arguably less clear.

For a production codebase, I'd probably go with a single `applyMove(MOVE)` in the interface and let implementations handle dispatch. 18 virtual functions is a lot of boilerplate to maintain across 3 implementations. But for this project, where I also use the individual functions in test code (e.g., `cube.u()` directly), the current design works.

One thing I'd definitely fix: `uPrime()` is implemented as `u(); u(); u()` — three clockwise rotations to achieve one counter-clockwise rotation. That works correctly but does 3× the work. A direct counter-clockwise implementation would be faster, though the simplicity of the current approach has value for correctness verification.

### Likely Follow-Up
"What about the `u'=u³` pattern — does it impact solver performance?"

### Strong Follow-Up Direction
In the hot search loop, the solver calls `move()` and `invert()` millions of times. Each `LPRIME` call triggers 3 `l()` calls instead of 1 optimised call. For the bitboard, each `l()` involves several bitmask operations. This is a 3× overhead on counter-clockwise and a 2× overhead on 180° moves. Whether it matters in practice depends on whether the bottleneck is move execution or hash map operations. It should be profiled.

### Red Flags to Avoid
- Saying the design is perfect without acknowledging the boilerplate.
- Not noticing the `uPrime() = u(); u(); u()` pattern.
- Suggesting runtime polymorphism is the wrong tool here — it's fine for the non-performance-critical parts.

---

## 9. Question

**The NibbleArray stores two 4-bit values per byte. Walk me through the get and set operations. What bugs could hide in this kind of code?**

### What the Interviewer Is Testing
Bit manipulation fluency; attention to detail in low-level code; understanding of common bugs in compact data structures.

### Strong Answer

The `NibbleArray` uses a `vector<uint8_t>` where each byte stores two nibbles. For an even index, the value is in the upper 4 bits; for an odd index, it's in the lower 4 bits.

**Get** for position `pos`: compute `i = pos / 2`. If `pos` is odd, return `arr[i] & 0x0F` (mask lower nibble). If even, return `arr[i] >> 4` (shift upper nibble down).

**Set** for position `pos` with value `val`: compute `i = pos / 2`. If odd, set lower nibble: `arr[i] = (arr[i] & 0xF0) | (val & 0x0F)` — keep upper nibble, replace lower. If even, set upper nibble: `arr[i] = (arr[i] & 0x0F) | (val << 4)` — keep lower nibble, replace upper.

Potential bugs:

1. **Off-by-one in size calculation**: The storage size is `size/2 + 1`. If `size` is even, the last byte's lower nibble is wasted but allocated. If `size` is odd, both nibbles of the last byte are used. The `+1` is a safety margin — but it means the last element of an even-sized array accesses a valid byte, which is correct.

2. **Boundary check uses `<=` instead of `<`**: The `assert(pos <= this->size)` should probably be `assert(pos < this->size)`. With `<=`, you can access one past the end — which would read from the safety `+1` byte. It works but it's technically a bug.

3. **Value overflow**: If `val > 15`, the `& 0x0F` in the odd case masks it correctly, but the even case shifts `val << 4` — if `val` has bits above bit 3, they'd overflow into bits above the byte. In practice, values are always 0–15 (move counts), so this doesn't trigger, but there's no assertion on the value parameter.

4. **Thread safety**: None. Concurrent reads are safe, but concurrent writes to adjacent positions that share a byte would cause data races.

### Likely Follow-Up
"How much memory does this actually save compared to a byte array?"

### Strong Follow-Up Direction
For ~88 million entries, a byte array uses ~84 MB. The NibbleArray uses ~44 MB (half) plus a small overhead for the vector metadata. The actual file is ~50 MB, which includes the `+1` padding and possibly filesystem block alignment. The savings are meaningful because this database is loaded entirely into RAM for every solver invocation.

### Red Flags to Avoid
- Not catching the `<=` vs. `<` boundary check issue.
- Saying "it saves 50% memory" without explaining why that matters for this specific project.
- Not understanding the even/odd convention.

---

## 10. Question

**How does the Lehmer code work in your `PermutationIndexer`? Why not just use a hash map to map permutations to indices?**

### What the Interviewer Is Testing
Combinatorial mathematics understanding; ability to justify algorithmic decisions; awareness of space-time trade-offs.

### Strong Answer

The Lehmer code converts a permutation of N elements into its lexicographic rank — a unique integer from 0 to N!-1. For 8 corners, that's 0 to 40,319.

The algorithm works by computing, for each position, how many elements after it are smaller (and haven't been "used" yet). This gives a mixed-radix number called the Lehmer code. Then you convert from factorial number system to base-10 by multiplying each digit by the appropriate factorial.

My `PermutationIndexer<8>` optimises this using two precomputed tables:
1. **Ones-count lookup**: A 256-entry table (2⁸-1) mapping each byte to its popcount. This lets me quickly count "seen" elements using a bitset.
2. **Factorial table**: Precomputed `pick(N-1-i, K-1-i)` values for the conversion.

For each element in the permutation, I mark it as "seen" in a bitset, then count how many "seen" elements are to its left by looking up the ones count of the appropriate bit range. The Lehmer digit is `perm[i] - numOnes` — the element's value minus the number of smaller elements already placed.

Why not a hash map? For 8! = 40,320 entries, a hash map would work — you'd map each permutation (as a string or array) to an integer. But:

1. **The database has 88 million entries** — each needs its permutation rank. Computing it in O(8) with the Lehmer code is faster than a hash map lookup with ~40K entries.
2. **The Lehmer code is a perfect hash** — no collisions, no wasted space. A hash map has overhead (load factor, chaining/probing, key storage).
3. **Direct addressing**: The rank is used as a direct array index. No indirection, no cache misses from pointer chasing.

### Likely Follow-Up
"What's the time complexity of the rank function?"

### Strong Follow-Up Direction
O(K) where K is the number of elements in the permutation (K=8 here). For each element, there's a bitset operation (O(1) with the lookup table) and a multiplication. The precomputed factorial table avoids recomputation. Total: 8 iterations with constant-time operations each.

### Red Flags to Avoid
- Not understanding that the Lehmer code gives a *rank* (position in sorted order), not a hash.
- Confusing lexicographic rank with a hash function — ranks are unique and dense (0 to N!-1), hashes are not.
- Saying "just use a map" without acknowledging the 88M lookups that make per-lookup cost matter.

---

## 11. Question

**Your `IDAstarSolver` constructor loads the pattern database from a file but doesn't check whether `fromFile()` succeeded. What happens if the file is missing?**

### What the Interviewer Is Testing
Error handling awareness; ability to trace failure propagation; reliability engineering mindset.

### Strong Answer

This is a real bug. In `IDAstarSolver`, the constructor calls `cornerDB.fromFile(fileName)`, which returns `false` if the file doesn't exist. But the return value is never checked.

If the file is missing, `fromFile()` returns `false`, and the database remains in its initialised state — the `NibbleArray` is filled with `0xFF`, which means every `getNumMoves()` call returns `0xF = 15`. The solver then starts IDA* with every heuristic estimate being 15.

Since the initial bound starts at 1 and the heuristic says "15 moves needed" for every state, `f(n) = g(n) + h(n)` will exceed the bound for every state at depth < 15. The solver will keep increasing the bound: 1, 15, 15, 15... (since `next_bound` starts at 100 and gets minimised to 15 for every state). Once the bound reaches 15, the solver essentially becomes exhaustive DFS to depth 15 without meaningful pruning.

For a 5-move scramble, this might still produce a solution — but it would take enormously longer than with a proper heuristic. For deeper scrambles, it might run for hours or run out of memory.

The fix is straightforward: check the return value and either throw an exception or log an error message.

```cpp
if (!cornerDB.fromFile(fileName)) {
    throw std::runtime_error("Failed to load pattern database: " + fileName);
}
```

There's also a related issue: if the file exists but has the wrong size, `fromFile()` throws a raw `const char*` string (`"Database corrupt!"`), which can't be caught by `catch(std::exception&)` blocks.

### Likely Follow-Up
"How would you design the error handling for a production version?"

### Strong Follow-Up Direction
Use a proper exception hierarchy (`std::runtime_error` for recoverable errors, `std::logic_error` for programming errors). Validate the database after loading: check that `getNumMoves(solvedCube) == 0`. Consider making database loading a separate step with a progress indicator, since loading 50 MB takes noticeable time.

### Red Flags to Avoid
- Not recognising this as a bug.
- Saying "it would crash" — it doesn't crash, it silently degrades, which is worse.
- Not knowing what value `0xFF` maps to in the nibble array (15 for each nibble).

---

## 12. Question

**What would happen if someone scanned a physically impossible cube state — say, two white centers? How would the solver behave?**

### What the Interviewer Is Testing
Edge case reasoning; system robustness; understanding of search algorithm behavior on invalid input.

### Strong Answer

The system has no cube state validation. The `CubeScanner` classifies each sticker independently and writes colors directly to the cube object via `setColor()`. If the scanned state is invalid — duplicate colors, wrong parity, impossible corner/edge orientation — it goes straight to the solver without any check.

What happens depends on the type of invalidity:

1. **Duplicate colors on one face**: The cube state is technically representable, but no sequence of moves leads to the solved state. The IDA* solver would keep increasing its bound indefinitely. The heuristic would return non-zero values, and the search would never find a solved state. The program would run forever or until memory is exhausted.

2. **Wrong corner permutation parity**: In a valid Rubik's Cube, the corner permutation and edge permutation have the same parity. If they don't match, the cube is unsolvable. Same outcome — infinite search.

3. **Wrong corner orientation sum**: The sum of all 8 corner orientations must be divisible by 3. If not, the cube is unsolvable.

This is a critical missing feature. The fix is a `isValidCubeState()` function that checks:
- Exactly 9 stickers of each of the 6 colors.
- Corner permutation parity matches edge permutation parity.
- Corner orientation sum is divisible by 3.
- Edge orientation sum is divisible by 2.

These checks are well-known in the cubing community and can be implemented in O(1) with the existing corner extraction methods. The BFS solver would at least terminate (when the queue empties), but IDA* could run indefinitely.

### Likely Follow-Up
"Why didn't you add this validation?"

### Strong Follow-Up Direction
Be honest — acknowledge it's an oversight. The project focused on the algorithmic core, and the scanner was added later. The validation logic requires understanding of the Rubik's Cube group structure (permutation parity, orientation constraints), which is an additional piece of mathematics beyond the solving algorithms. It should be prioritised as a fix.

### Red Flags to Avoid
- Saying "it would throw an error" — it doesn't.
- Not knowing the mathematical constraints of valid cube states.
- Dismissing the issue as unimportant.

---

## 13. Question

**The `CornerDBMaker` BFS stops at depth 8. What does that mean for states at depth 9 or deeper, and how does it affect solver performance?**

### What the Interviewer Is Testing
Understanding of heuristic completeness; ability to reason about the impact of approximations on algorithm behavior.

### Strong Answer

The BFS generator in `CornerDBMaker` processes states level by level and breaks at `curr_depth == 9`, meaning it records move counts for states reachable in 0 through 8 moves from the solved position. Any corner configuration that requires 9 or more moves to solve its corners is left at its initialised value of `0xFF`, which the nibble array returns as `0xF = 15`.

The impact on the solver depends on how common these unreached states are. The maximum number of moves to solve corners alone is about 11 (from published Rubik's Cube mathematics). So states at depth 9, 10, and 11 are missing from the database.

For the IDA* solver, a missing state returns `h(n) = 15` as its heuristic estimate. Since the true distance is at most 11 for corners, this overestimates — which means the heuristic is **no longer admissible** for these states. An overestimating heuristic means IDA* might not find the optimal solution.

In practice, the impact is limited because:
1. Most corner configurations (maybe 95%+) are reachable within 8 moves.
2. For the remaining states, the heuristic returns 15, which is higher than any true corner distance. This causes IDA* to defer exploring these states to later (higher) bounds — it doesn't skip them entirely, it just explores them last.
3. The bound increases iteratively, so eventually these states will be explored.

But the theoretical guarantee of optimality is weakened. To fix this, the BFS should run to completion (all ~88M states reached). It would take longer to generate but would produce a complete, admissible heuristic.

### Likely Follow-Up
"How long would a complete BFS take?"

### Strong Follow-Up Direction
At depth 8, BFS has already explored the vast majority of the 88M states. The remaining states at depth 9–11 are a small fraction. The total computation is bounded by O(88M × 18) ≈ 1.6 billion move operations. On a modern machine, this might take 10–30 minutes, which is a one-time offline cost. The file name `cornerDepth5V1.txt` suggests an even earlier version went to depth 5 — the depth limit may have been a debugging convenience that was never removed.

### Red Flags to Avoid
- Saying the heuristic is still admissible with 0xF for unreached states — it overestimates, so it's not.
- Not realising that "depth 5 V1" in the filename might indicate the original limit was even shallower.
- Claiming the missing states "never occur in practice" without data.

---

## 14. Question

**How would you add edge pattern databases to strengthen the heuristic? What's the engineering challenge?**

### What the Interviewer Is Testing
Scalability thinking; understanding of heuristic design in AI; practical memory engineering.

### Strong Answer

The current heuristic uses only corners. Adding edge databases would provide a stronger lower bound — you'd take the maximum of the corner heuristic and the edge heuristic(s), which is still admissible because the max of admissible heuristics is admissible.

The engineering challenge is primarily memory. A Rubik's Cube has 12 edges, each with 2 possible orientations. The full edge state space is 12! × 2¹¹ ≈ 980 billion entries — far too large to store directly. The standard approach, following Korf's 1997 paper, is to split edges into two or three subgroups:

- **Two groups of 6 edges**: Each group has 12!/(12-6)! × 2⁶ ≈ 42.6 million entries. Two databases = ~85M entries total, about ~43 MB each with nibble packing.
- **Three groups (6-6-6 with overlap)**: Increases memory but gives stronger heuristic bounds.

For each group, you run BFS from the solved state, ignoring the positions and orientations of edges outside the group. The indexing uses a partial permutation variant of the Lehmer code — the `PermutationIndexer<12, 6>` template already supports this (K < N).

The practical challenges:
1. **Memory**: Two edge databases + corner database ≈ 150 MB. Manageable, but three databases means more disk I/O and startup time.
2. **Generation time**: Each edge database BFS explores ~42M states × 18 moves × ~11 depth levels. On a single machine, this might take 30–60 minutes per database.
3. **Index computation**: Edge extraction from the cube state requires more complex code than corner extraction — edges span pairs of faces.
4. **Multiple heuristic lookups**: Each search node now requires 2–3 database lookups instead of 1, slightly increasing per-node cost.

The payoff: the combined heuristic would reduce the effective branching factor further, enabling solution of 15+ move scrambles.

### Likely Follow-Up
"Would you use additive or max-based combination of the heuristics?"

### Strong Follow-Up Direction
Max-based is always safe (admissible). Additive is only admissible if the pattern databases are for disjoint move sets — which isn't the case here (every face turn affects both corners and edges). Using additive combination with non-disjoint databases would break admissibility and could produce suboptimal solutions. Korf uses max in his original paper.

### Red Flags to Avoid
- Suggesting additive combination without checking disjointness.
- Proposing to store the full 12! × 2¹¹ ≈ 980 billion entries.
- Not knowing the Korf reference.

---

## 15. Question

**The project has no automated tests. If you were to add a test suite, what would be your priority list?**

### What the Interviewer Is Testing
Testing strategy; ability to prioritise based on risk; practical engineering judgment.

### Strong Answer

I'd start with the highest-risk, highest-value tests and work outward:

**Priority 1 — Move Correctness (Unit Tests)**:
For each of the 3 representations and all 18 moves: apply a move, then apply its inverse, assert the cube returns to the original state. This verifies `move()` and `invert()` are consistent. Also: apply a move 4 times and verify the cube returns to the original state (every Rubik's Cube move has order 4 or 2). This is the foundation — if moves are wrong, everything else breaks.

**Priority 2 — Cross-Representation Consistency (Unit Tests)**:
Apply the same sequence of moves to all 3 representations. Verify they produce the same color at every position. This catches representation-specific bugs.

**Priority 3 — Solver Correctness (Integration Tests)**:
For scrambles of known depth (1–8 moves): scramble → solve → assert `isSolved()` returns true. For BFS and IDDFS: assert solution length equals scramble depth (optimality). Use deterministic scrambles (fixed seed) for reproducibility.

**Priority 4 — Pattern Database Correctness (Integration Tests)**:
Generate a fresh database, verify `getNumMoves(solvedCube) == 0`. Apply known scrambles, verify the returned heuristic value is ≤ the optimal solution length (admissibility check).

**Priority 5 — NibbleArray and PermutationIndexer (Unit Tests)**:
Set/get all values in a NibbleArray, verify correctness. Verify `PermutationIndexer::rank()` for known permutations (e.g., identity = 0, reverse = N!-1).

**Priority 6 — Edge Cases**:
Already-solved cube → solver returns empty solution. Single-move scramble. Maximum practical scramble.

I'd use Google Test or Catch2 for the framework. The tests should be runnable without OpenCV (scanner tests would be separate and require mocking or integration with a known image).

### Likely Follow-Up
"How would you test the scanner?"

### Strong Follow-Up Direction
For unit testing: provide pre-captured images with known colors and verify `classifyColor()` returns the correct color. For integration: use a set of test images (different lighting conditions, different cube colors) and measure classification accuracy. The scanner is the hardest part to test because it depends on camera and lighting — this is where property-based testing or golden-image comparison helps.

### Red Flags to Avoid
- Starting with scanner tests instead of move correctness.
- Suggesting end-to-end tests only without unit tests.
- Not mentioning the need for deterministic scrambles in solver tests.

---

## 16. Question

**How would you deploy this project as a usable product — say, as a web application or mobile app? What architecture changes would you need?**

### What the Interviewer Is Testing
System design thinking; ability to translate a local tool into a distributed system; understanding of web/mobile architecture patterns.

### Strong Answer

The current project is a monolithic C++ desktop application. Moving it to a web or mobile product requires several architectural changes:

**Backend Service**: The C++ solver becomes a backend service exposed via a REST API. The endpoint might be `POST /solve` accepting a cube state (54 sticker colors as JSON) and returning the solution moves. The solver runs on the server, which has the pattern database pre-loaded in memory.

**Architecture**:
```
Mobile/Web Client → API Gateway → Solver Service (C++) → Response
                                      ↑
                              Pattern Database (in-memory, ~50MB)
```

**Key Changes**:

1. **Separate scanner from solver**: The webcam scanning would move to the client (mobile camera + on-device color classification, or browser-based using WebRTC + TensorFlow.js for color detection). The client sends the detected cube state to the backend.

2. **Add a timeout and job queue**: IDA* for deep scrambles can take seconds or more. For a web service, add a timeout (e.g., 30 seconds) and return a "try simpler scramble" error if it exceeds. For longer jobs, use a job queue (Redis/RabbitMQ) and return results via WebSocket or polling.

3. **Solver as a stateless service**: Each request is independent. The pattern database is loaded once at startup and shared across requests (read-only, thread-safe). The cube state is request-scoped.

4. **Input validation**: The API must validate the incoming cube state (exactly 9 of each color, valid permutation parity) before passing it to the solver.

5. **Scalability**: The solver is CPU-bound. Horizontal scaling means multiple solver instances behind a load balancer. Each instance needs ~50 MB for the database — manageable.

6. **Alternative**: Compile the C++ solver to WebAssembly and run it entirely client-side. This avoids the need for a backend but requires shipping the 50 MB database to the browser (or generating it on-demand, which is slow).

### Likely Follow-Up
"What about latency? How fast does the solver need to respond?"

### Strong Follow-Up Direction
For a user-facing web app, target sub-5-second response for most scrambles. The IDA* solver handles 5-move scrambles in milliseconds and 10-move scrambles in seconds. For scrambles approaching 15+ moves, you'd either need a stronger heuristic or a switch to Kociemba's two-phase algorithm, which solves any cube in < 1 second.

### Red Flags to Avoid
- Suggesting the desktop application can be "just deployed to the web" without changes.
- Not considering the 50 MB database loading cost.
- Ignoring input validation in the API.

---

## 17. Question

**What observability would you add to this system if you were monitoring it in production?**

### What the Interviewer Is Testing
Operational thinking; understanding of observability pillars (logging, metrics, tracing); practical monitoring design.

### Strong Answer

Currently, the project has zero observability — no logging, no metrics, no tracing. Output goes directly to `cout`.

For a production version, I'd add three layers:

**Structured Logging**:
- Log every solve request: scramble, solver type, representation type, timestamp.
- Log the solution: move count, solver time, states explored, peak memory.
- Log errors: database load failures, invalid cube states, timeouts.
- Use a structured logging library (spdlog for C++) with JSON output for log aggregation.

**Metrics**:
- `solve_duration_seconds` (histogram) — with labels for solver type and scramble depth.
- `solve_states_explored` (histogram) — correlate with duration.
- `solve_success_rate` (counter) — successful solves vs. timeouts/errors.
- `scanner_classification_confidence` (histogram) — if we add confidence scores.
- `database_load_time_seconds` (gauge) — one-time startup metric.
- `memory_usage_bytes` (gauge) — track for memory leak detection.
- Expose via Prometheus-compatible endpoint.

**Alerting**:
- Alert on solve success rate dropping below 95%.
- Alert on p99 solve latency exceeding 10 seconds.
- Alert on database load failure.

**Tracing** (if distributed):
- Trace ID through scanner → validator → solver → response.
- Span for each IDA* iteration (bound increase).

The most immediately valuable metric is `solve_duration_seconds` bucketed by scramble depth — this would be the first thing I'd add to understand real-world performance characteristics.

### Likely Follow-Up
"What's the most important metric if you could only pick one?"

### Strong Follow-Up Direction
Solve duration by scramble depth. It reveals: (1) whether the heuristic is effective (should see exponential growth, but slower than theoretical 18^d), (2) the practical depth limit, (3) outliers that indicate pathological cases. It's also directly user-facing — users care about how long they wait.

### Red Flags to Avoid
- Saying "just add print statements."
- Not distinguishing between logging (events), metrics (aggregatable numbers), and tracing (request flow).
- Suggesting overly complex observability for what might remain a single-machine tool.

---

## 18. Question

**What technical debt would you address first if you had two weeks to improve the project? Justify your prioritisation.**

### What the Interviewer Is Testing
Prioritisation skills; ability to distinguish high-impact from low-impact improvements; pragmatic engineering judgment.

### Strong Answer

With two weeks, I'd focus on four items, ordered by impact and risk reduction:

**Week 1:**

1. **Check `fromFile()` return value + fix exception types** (1 day): This is a silent-failure bug that makes the solver produce wrong results or run forever. Fix: add a return-value check in the IDA* constructor, convert string-literal throws to `std::runtime_error`. This is the highest-risk bug in the codebase.

2. **Add cube state validation** (2 days): After scanning, validate the cube state is a legal permutation. This prevents the solver from running forever on invalid input. Implement checks: exactly 9 of each color, corner permutation parity, corner orientation sum mod 3 = 0. This is critical for any user-facing mode.

3. **Add a basic test suite** (2 days): Set up Google Test, write: move correctness tests (apply + invert = identity), cross-representation consistency, solver correctness for scrambles of depth 1–6. This gives confidence for future changes.

**Week 2:**

4. **CLI interface** (2 days): Add argument parsing: `--solver=bfs|dfs|iddfs|ida*`, `--representation=3d|1d|bitboard`, `--scramble-depth=N`, `--no-webcam`, `--database-path=PATH`. This replaces the commented-out test blocks and makes the project actually usable without modifying source code.

5. **Complete the corner database BFS** (1 day): Remove the depth-8 limit in `CornerDBMaker` and regenerate the database. This restores admissibility of the heuristic for all corner states.

6. **Clean up `main.cpp`** (1 day): Extract the commented-out code into the test suite. Reduce `main.cpp` to a clean CLI entry point.

**Why this order**: Items 1–2 fix bugs that produce silently wrong results. Item 3 prevents regressions. Items 4–6 improve usability and maintainability. I wouldn't spend time on performance optimisation (hash function, direct counter-clockwise moves) until the correctness and testing foundation is solid.

### Likely Follow-Up
"Why not start with performance improvements?"

### Strong Follow-Up Direction
Performance doesn't matter if the results are wrong. The `fromFile()` bug means the solver can silently run without a heuristic. The missing validation means invalid input causes infinite loops. These are correctness issues, and correctness always comes before performance. Once we can verify correctness with tests, then we can safely optimise and measure the impact.

### Red Flags to Avoid
- Starting with "add edge databases" or "implement Kociemba" — those are feature additions, not debt reduction.
- Saying "rewrite the whole thing."
- Not mentioning the `fromFile()` bug.

---

## 19. Question

**If you could redesign this project from scratch with the benefit of hindsight, what would you do differently?**

### What the Interviewer Is Testing
Self-awareness; design maturity; ability to learn from implementation experience.

### Strong Answer

Several things I'd change:

**1. Start with Kociemba's two-phase algorithm instead of single-phase IDA*.**
Kociemba's algorithm solves any cube in under 30 moves (typically ~20) and runs in under a second. It works by first reducing the cube to a subgroup using one set of moves, then solving from there. It's the standard for fast solvers. I'd still implement IDA* for educational value, but Kociemba would be the production solver.

**2. Keep only the bitboard representation.**
The 3D and 1D arrays were useful for learning, but maintaining three representations is unnecessary maintenance. I'd keep the bitboard for solving and add a `toString()` method for human-readable output.

**3. Design for testability from the start.**
I'd structure the code so the scanner, solver, and model are independently testable. This means: no commented-out test blocks in `main.cpp`, proper dependency injection (pass the database to the solver rather than loading inside the constructor), and a test harness from day one.

**4. Use proper error handling.**
Replace `throw "string"` with typed exceptions. Check all return values. Add input validation at every boundary.

**5. Implement direct inverse moves.**
Instead of `uPrime() = u(); u(); u()`, implement each inverse as its own optimised rotation. This is a 3× performance improvement for counter-clockwise moves.

**6. Use `std::mt19937` instead of `srand/rand`.**
Better randomness, thread-safe, reproducible with a seed.

**7. Separate the `main()` function from test code.**
Use a proper CLI parser and test framework from the start.

**What I wouldn't change**: The abstract base class design pattern, the template-based solvers, the Lehmer code indexing, and the NibbleArray approach. These are solid design decisions that worked well.

### Likely Follow-Up
"How long would Kociemba's algorithm take to implement?"

### Strong Follow-Up Direction
Kociemba's algorithm is significantly more complex than single-phase IDA*. It requires: (1) understanding the H group (G1 = ⟨U, D, R2, L2, F2, B2⟩), (2) building move tables for phase 1 and phase 2 transitions, (3) computing pruning tables for both phases, (4) implementing a two-phase search. It's a 2–4 week implementation effort for someone familiar with the mathematics. Libraries like `kociemba` (Python) exist, but reimplementing in C++ for full understanding and performance is worthwhile.

### Red Flags to Avoid
- Saying "I wouldn't change anything."
- Proposing a complete rewrite in a different language without justification.
- Not knowing what Kociemba's algorithm is (it's the standard for cube solvers).

---

## 20. Question

**How would you measure whether your solver is actually producing optimal solutions? How would you validate the heuristic's admissibility in practice?**

### What the Interviewer Is Testing
Scientific rigor; ability to validate algorithmic correctness empirically; understanding of admissible heuristics.

### Strong Answer

Validating optimality and admissibility requires both theoretical reasoning and empirical testing.

**Theoretical validation of admissibility:**
The corner pattern database is admissible by construction — it stores the exact minimum number of moves to solve the corners. Since solving the full cube requires at least as many moves as solving the corners alone, the corner heuristic never overestimates. However, this breaks if the database is incomplete: the BFS generator stops at depth 8, and unreached states return `0xF = 15`. Since the true maximum corner distance is ~11, returning 15 overestimates. So the heuristic as implemented is *not* strictly admissible for all states. I'd fix this by running BFS to completion.

**Empirical validation of optimality:**

1. **Compare against known results**: For specific well-studied scrambles, the optimal solution length is known. I'd test against these.

2. **Compare solvers**: For scrambles where BFS is feasible (depth ≤ 7), BFS is guaranteed optimal. I'd scramble at depth 1–7, solve with both BFS and IDA*, and verify they produce solutions of the same length. If IDA* ever returns a longer solution, the heuristic is overestimating somewhere.

3. **Property testing**: For random scrambles of depth N (made by applying N random moves), the optimal solution has length ≤ N. If IDA* returns a solution longer than N, something is wrong. (Note: it can be shorter than N because random moves can cancel each other.)

4. **Heuristic lower bound check**: For each state in the search, verify `h(n) ≤ optimal_distance(n)`. I can approximate this by: solve the state with BFS (for small states) and check that the heuristic value ≤ BFS solution length.

5. **Database self-consistency**: After generating the database, verify: `getNumMoves(solvedCube) == 0`. For each state at distance D, apply all 18 moves and verify the resulting state has distance ≤ D+1. This catches generation bugs.

The most practical immediate step is #2 — comparing BFS and IDA* solutions for depth 1–7 scrambles. It requires no external data and directly tests optimality.

### Likely Follow-Up
"If you find a case where IDA* gives a suboptimal solution, how do you debug it?"

### Strong Follow-Up Direction
Trace the search: for the problematic scramble, log the heuristic value at each state expansion. Find the state where `h(n) > true_distance(n)` — that's where admissibility breaks. Check whether that state's corner configuration is at depth > 8 in the database (returns 15 instead of the true value). If the database is complete and the heuristic is still overestimating, the bug is in `getDatabaseIndex()` — likely an indexing error in the Lehmer code or orientation calculation.

### Red Flags to Avoid
- Saying "IDA* is always optimal by definition" without noting that admissibility of the heuristic is a precondition.
- Not knowing that the depth-8 BFS limit breaks admissibility.
- Proposing only theoretical validation without empirical testing.
