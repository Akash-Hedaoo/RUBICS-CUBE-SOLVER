# Three-Minute Project Introduction

## Why

Every 3×3 Rubik's Cube can be solved in 20 moves or fewer — that's proven. But the state space has 43 quintillion configurations, and finding that short solution algorithmically is a genuinely hard search problem. I wanted to explore how classical AI search techniques handle this — starting from brute-force and progressively building toward informed, optimal search. The project also gave me a reason to work with bit manipulation, combinatorial indexing, and computer vision in a single codebase.

## What

I built a C++ engine that takes a scrambled 3×3 Rubik's Cube and finds an optimal or near-optimal solution. The system has two input modes: you can either scramble a virtual cube programmatically, or scan a real physical cube using your webcam. The solver then outputs the exact sequence of moves needed to solve it.

The core of the project is four search algorithms implemented as a progression: Depth-First Search, Breadth-First Search, Iterative Deepening DFS, and IDA*. IDA* is the primary solver — it uses a pre-computed corner pattern database as an admissible heuristic, which maps all 88 million possible corner configurations to their minimum solve distance. This lets it prune huge portions of the search tree and handle scrambles of 13 or more moves.

## How

The architecture has four layers. The **Model layer** defines an abstract `RubiksCube` base class with three concrete implementations: a 3D array for readability, a 1D array for cache-friendliness, and a bitboard representation where each face is a 64-bit integer with one-hot color encoding — so a face rotation becomes a single bit-shift operation. All three share the same interface, and all solvers are C++ templates parameterised on the representation and its hash function, so any solver works with any representation at zero runtime cost.

The **Pattern Database** is the most interesting engineering challenge. I pre-compute it by running BFS from the solved cube state, using Lehmer codes to convert each corner permutation into a unique array index, and storing the move counts in a nibble-packed array — two values per byte — cutting memory usage from about 84 MB to 50 MB. The heuristic is provably admissible, meaning it never overestimates, which guarantees IDA* finds optimal solutions.

One key trade-off: I implemented only a corner pattern database, not edges. This keeps memory manageable at ~50 MB but means the heuristic is weaker for edge-heavy scrambles. A combined corner-plus-edge database would be more powerful but would require over a gigabyte of storage.

The **scanner** uses OpenCV to capture each face from a webcam, converts pixels from BGR to HSV color space, applies median filtering for noise reduction, and classifies colors using hue thresholds.

## What Now

The project is at portfolio stage — algorithmically solid but not production-hardened. The biggest limitation is the corners-only heuristic, which limits practical solving depth to about 13 moves. Adding edge databases or implementing Kociemba's two-phase algorithm would extend that significantly. The scanner also uses hardcoded color thresholds, which can misclassify under poor lighting — a calibration step or ML-based classifier would fix that. On the engineering side, the project needs automated tests, proper error handling, and a CLI interface to replace the commented-out test blocks.

I'm happy to go deeper into any part — the bitboard encoding, the pattern database math, the IDA* implementation, or how I'd scale this toward production.

---

## Thirty-Second Version

I built a Rubik's Cube solver in C++ that finds optimal solutions using IDA* search with a pre-computed corner pattern database heuristic. The cube has 43 quintillion states, and the heuristic — covering all 88 million corner configurations using Lehmer code indexing and nibble-packed storage — prunes the search tree enough to solve 13+ move scrambles. I implemented three interchangeable cube representations, including a bitboard where face rotations are single bit-shift operations, and a webcam scanner using OpenCV for physical cube input. The main limitation is the corners-only heuristic; adding edge databases or switching to Kociemba's two-phase algorithm would handle deeper scrambles.

---

## Key Points to Remember

1. **43 quintillion states, 20-move max** — the problem is well-defined but computationally enormous.
2. **Four algorithms as a progression**: DFS → BFS → IDDFS → IDA*. Each solves a limitation of the previous one.
3. **Bitboard = chess engine technique**: One-hot encoding, bit-shift rotation, XOR hashing. Face rotation is ~1 CPU cycle.
4. **88 million corner states** indexed via Lehmer codes into a nibble array (4 bits per entry, 50% memory savings).
5. **Admissible heuristic guarantees optimality** — IDA* never finds a suboptimal solution.
6. **Template-based solvers** — compile-time polymorphism means zero runtime dispatch overhead in the search loop.
7. **Main limitation**: corners-only heuristic caps practical depth at ~13 moves. Edge databases or Kociemba's algorithm would extend this.
8. **Scanner weakness**: hardcoded HSV thresholds; no cube validity check after scanning.
