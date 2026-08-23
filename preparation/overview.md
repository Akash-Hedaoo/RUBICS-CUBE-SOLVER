# Rubik's Cube Solver — Comprehensive Technical Overview

This is a C++ engine that solves a 3×3 Rubik's Cube using graph search algorithms (DFS, BFS, IDDFS, IDA*), backed by a pre-computed corner pattern database heuristic. It includes three interchangeable internal cube representations (3D array, 1D array, bitboard) sharing a common abstract interface, and a webcam-based scanner module using OpenCV for physical cube input. The project is a portfolio/educational project demonstrating classical AI search, combinatorial optimization, bit manipulation, and computer vision.

---

## 1. Executive Summary

The Rubik's Cube Solver tackles the problem of finding an optimal or near-optimal solution to any scrambled 3×3 Rubik's Cube — a state space with approximately 4.3×10¹⁹ configurations. The project implements a progression of four search algorithms: Depth-First Search (depth-limited, non-optimal), Breadth-First Search (optimal but memory-heavy), Iterative Deepening DFS (optimal and memory-efficient), and IDA* (optimal with heuristic pruning). IDA* is the primary solver, powered by a pre-computed corner pattern database (~50 MB) that maps all 88,179,840 corner configurations to their minimum solve distances, enabling it to handle 13+ move scrambles.

The most distinctive engineering decision is the triple-representation architecture: the same Rubik's Cube can be internally modelled as a `char[6][3][3]` (intuitive), a `char[54]` (cache-friendly), or six `uint64_t` bitboards (O(1) face rotations via bit shifts). All three share the `RubiksCube` abstract base class, and all four solvers are C++ templates parameterised on the representation and its hash function — a textbook application of the Strategy pattern.

The project's main strengths are its clean polymorphic design, the well-implemented IDA* with an admissible heuristic, and the bitboard representation applying competitive-programming techniques to a puzzle domain. Its main limitations are the absence of automated tests, the single-heuristic scope (corners only, no edge database), hardcoded scanner color thresholds, and no deployment or CI/CD infrastructure. This is a portfolio-stage project with solid algorithmic depth but without production hardening.

---

## 2. Problem Statement

A scrambled 3×3 Rubik's Cube has roughly 4.3×10¹⁹ possible states, yet every state is solvable in at most 20 moves ("God's Number"). Finding a short solution algorithmically requires navigating an enormous search tree with a branching factor of 18 (6 faces × 3 move types: clockwise, counter-clockwise, 180°).

Without this project, solving a Rubik's Cube requires either memorised human algorithms (CFOP, Roux, etc.) that produce non-optimal solutions, or brute-force enumeration that is computationally infeasible for deep scrambles. This project removes that friction by providing an automated solver that finds optimal solutions using informed search.

The project assumes its user is either a developer exploring AI search algorithms, or someone with a physical Rubik's Cube who wants an automated solution via webcam input.

---

## 3. Target Users and Use Cases

### Primary Users
- **CS students and developers** studying graph search algorithms, heuristic design, and state-space representations — the project is structured as a pedagogical progression from DFS to IDA*.
- **Rubik's Cube enthusiasts** who want to scan a physical cube and get an optimal solution.

### Secondary Users
- **Interviewers or reviewers** evaluating the project as a demonstration of data structures, algorithms, OOP design, and bit manipulation.

### Main Use Cases (Verified)
1. **Algorithmic solving**: Scramble a virtual cube with `randomShuffleCube(n)`, then solve it with any of the four solvers.
2. **Webcam scanning + solving**: Scan all 6 faces of a physical cube via webcam, then solve using IDA* with the corner pattern database.
3. **Pattern database generation**: Generate the ~50 MB corner pattern database from scratch using BFS from the solved state.
4. **Representation benchmarking**: Compare the same algorithm across three different cube representations.

### Inferred Use Cases
- Comparing solver performance across different scramble depths to understand algorithmic scaling.
- Extending the heuristic system with additional pattern databases (edge databases) for deeper scrambles.

---

## 4. Core User Journey

The primary user journey in the active (uncommented) code path is: **scan a physical cube → solve with IDA* → display solution**.

### Step-by-Step Workflow

1. **User launches the executable** — `main()` in `main.cpp` runs.
2. **Webcam opens** — `CubeScanner(0)` initialises OpenCV `VideoCapture` on camera index 0.
3. **User aligns each face** — A 3×3 grid overlay is drawn on the camera feed. The user aligns a cube face and presses SPACE.
4. **Color classification** — For each of the 9 cells, a 5×5 pixel region is sampled, channel-wise median filtering is applied, the pixel is converted BGR→HSV, and hue thresholds classify the color (Red: 160–190, Orange: 3–19, Yellow: 20–30, Green: 60–90, Blue: 100–120, White: RGB-similarity check).
5. **Visual verification** — The detected face colors are displayed. The user presses N to confirm or R to rescan.
6. **Repeat for all 6 faces** — Colors are written to a `RubiksCubeBitboard` object via `cube.setColor()`.
7. **Corner pattern database loads** — `IDAstarSolver` constructor reads `Databases/cornerDepth5V1.txt` (~50 MB) into a `NibbleArray`-backed lookup table.
8. **IDA* search** — Iteratively deepening A* search with the corner heuristic explores the state space. The heuristic (`cornerDB.getNumMoves()`) provides a lower bound on remaining moves. States where `g(n) + h(n) > bound` are pruned.
9. **Solution reconstruction** — The solver traces back through `move_done[]` from solved state to initial state, reverses the move list, and returns it.
10. **Output** — Solution moves are printed to the console (e.g., `R U' F2 L ...`).

### Failure Points
- **Step 2**: Webcam fails to open → `runtime_error` thrown, program crashes.
- **Step 4**: Poor lighting or non-standard cube colors → misclassification. No automated validation that the scanned state is a valid Rubik's Cube configuration.
- **Step 7**: Database file missing or corrupt → either `fromFile()` returns false (unhandled) or throws "Database corrupt!".
- **Step 8**: Very deep scrambles (>13 moves) may take excessive time or memory in the IDA* priority queue.

<img width="1807" height="1043" alt="image" src="https://github.com/user-attachments/assets/a7ffbdb1-2685-40c6-a6ce-6439d29ae97b" />


## 5. Feature Breakdown

### Fully Implemented Features

| Feature | Purpose | Key Components | Limitations |
|---|---|---|---|
| **DFS Solver** | Depth-limited search, non-optimal | `DFSSolver.h` — template class, recursive `dfs()` | Not optimal; exponential time; practical only for ~6 moves |
| **BFS Solver** | Guaranteed-optimal shortest-path search | `BFSSolver.h` — queue + `unordered_map` visited/back-pointers | Memory explodes at ~7 moves (O(18^d) space) |
| **IDDFS Solver** | Optimal, memory-efficient iterative deepening | `IDDFSSolver.h` — wraps `DFSSolver` with increasing depth | Redundant re-exploration; practical to ~7 moves |
| **IDA* Solver** | Optimal search with heuristic pruning | `IDAstarSolver.h` — priority queue + `CornerPatternDatabase` | Limited by corners-only heuristic; ~13 moves practical max |
| **3D Array Representation** | Intuitive `char[6][3][3]` cube model | `RubiksCube3dArray.cpp` | Slow face rotations (element-by-element swaps) |
| **1D Array Representation** | Cache-friendly `char[54]` model | `RubiksCube1dArray.cpp` | Moderate performance |
| **Bitboard Representation** | High-performance `uint64_t[6]` model | `RubiksCubeBitboard.cpp` | Complex bit manipulation; harder to debug |
| **Corner Pattern Database** | Pre-computed admissible heuristic for IDA* | `CornerPatternDatabase`, `CornerDBMaker`, `PatternDatabase`, `NibbleArray`, `PermutationIndexer` | Corners-only; 50 MB file size; BFS limited to depth 8 in generator |
| **Webcam Scanner** | Scan physical cube via camera | `CubeScanner.cpp` — OpenCV HSV classification | Hardcoded hue thresholds; no cube validity check |

### Stubbed/Legacy Features

- **`Model/PatternDatabase/PatternDatabase.h`**: A stub `PatternDatabaseEstimate<T>` class that always returns `getEstimate() = 0`. This was likely a placeholder before the full `PatternDatabases/` module was built. It is referenced in `CMakeLists.txt` but commented out in `IDAstarSolver.h`.

---

## 6. Technology Stack

| Layer | Technology | Where It Is Used | Why It Fits | Trade-Offs |
|---|---|---|---|---|
| Language | C++14 | Entire project | Low-level memory control for bitboard operations; templates for generic solvers; high performance for combinatorial search | Longer development cycle; manual memory management; no built-in test framework |
| Build System | CMake ≥ 3.20 | `CMakeLists.txt` | Cross-platform build generation; standard for C++ projects | Requires separate installation; verbose for small projects |
| Computer Vision | OpenCV 4.x | `Scanner/CubeScanner.cpp` | Mature, well-documented library for webcam capture, color space conversion, and image display | Heavy dependency (~100+ MB); only a fraction of its capabilities used |
| Data Storage | Binary file on disk | `Databases/cornerDepth5V1.txt` (~50 MB) | Simple, fast binary I/O for loading the pattern database | No versioning; no integrity checks beyond file size; platform-dependent byte order (assumed) |
| Portability Header | Custom `bits/stdc++.h` | `bits/stdc++.h` | Provides a GCC-like umbrella header for MSVC/cross-platform use | Non-standard; imports more than needed; slows compilation |

---

## 7. High-Level Architecture

The system follows a layered architecture with four major components:

1. **Model Layer** (`Model/`): Defines the `RubiksCube` abstract base class with pure virtual functions for all 18 moves, state querying (`getColor`, `isSolved`), and shared logic (`print`, `randomShuffleCube`, `move`, `invert`, corner extraction). Three concrete implementations provide different internal representations.

2. **Solver Layer** (`Solver/`): Four template-based solver classes, parameterised on `<T, H>` (representation type and hash function). Each solver contains a `T rubiksCube` member and a `solve()` method returning `vector<MOVE>`.

3. **Pattern Database Layer** (`PatternDatabases/`): A `PatternDatabase` abstract class backed by a `NibbleArray` (4-bit packed storage), with `CornerPatternDatabase` as the concrete implementation. `CornerDBMaker` generates the database via BFS. `PermutationIndexer<N,K>` converts permutations to lexicographic ranks using Lehmer codes.

4. **Scanner Layer** (`Scanner/`): `CubeScanner` wraps OpenCV's `VideoCapture`, providing webcam-based face scanning with HSV color classification and median filtering.

<img width="2603" height="205" alt="image" src="https://github.com/user-attachments/assets/452b7e60-dc3b-4b9f-8481-d03dd34fcb74" />


## 8. Module and Folder Map

| Path | Responsibility | Important Notes |
|---|---|---|
| `main.cpp` | Entry point; orchestrates scanner, solver, and output | Active code path: webcam scan → IDA* solve. All other solver tests are commented out. |
| `Model/RubiksCube.h` | Abstract base class defining the cube interface | 18 pure virtual move functions; `FACE`, `COLOR`, `MOVE` enums; corner extraction methods |
| `Model/RubiksCube.cpp` | Shared logic: `print()`, `randomShuffleCube()`, `move()`, `invert()`, corner helpers | `randomShuffleCube` uses `srand(time(0))` — not cryptographically secure, but fine for puzzle scrambling |
| `Model/RubiksCube3dArray.cpp` | 3D array representation: `char cube[6][3][3]` | Includes `Hash3d` struct; `operator==` and `operator=` overloads |
| `Model/RubiksCube1dArray.cpp` | 1D flat array representation: `char cube[54]` | Includes `Hash1d` struct |
| `Model/RubiksCubeBitboard.cpp` | Bitboard representation: `uint64_t bitboard[6]` | One-hot color encoding; bit-shift face rotations; XOR-based `HashBitboard`; `getCorners()` method |
| `Solver/DFSSolver.h` | Depth-first search (depth-limited) | Default max depth = 8; recursive backtracking |
| `Solver/BFSSolver.h` | Breadth-first search (guaranteed optimal) | Back-pointer reconstruction via `move_done` map |
| `Solver/IDDFSSolver.h` | Iterative deepening DFS | Wraps `DFSSolver` with loop from depth 1 to max |
| `Solver/IDAstarSolver.h` | IDA* with corner pattern database heuristic | Priority queue ordered by `f(n) = g(n) + h(n)`; iteratively increases bound |
| `PatternDatabases/PatternDatabase.h/.cpp` | Abstract pattern database with `NibbleArray` storage | `setNumMoves` updates only if new value is smaller; binary file I/O |
| `PatternDatabases/CornerPatternDatabase.h/.cpp` | Corner-specific database: 8! × 3⁷ = 88,179,840 states mapped to size 100,179,840 | `getDatabaseIndex()` combines permutation rank × 2187 + orientation number |
| `PatternDatabases/CornerDBMaker.h/.cpp` | BFS-based database generator | Hardcoded depth limit of 8; uses `RubiksCubeBitboard` only |
| `PatternDatabases/PermutationIndexer.h` | Lehmer code → lexicographic rank converter | Template `<N, K=N>`; uses precomputed ones-count lookup and factorial tables |
| `PatternDatabases/NibbleArray.h/.cpp` | 4-bit packed array (two values per byte) | ~50% memory savings; bit manipulation for get/set |
| `PatternDatabases/math.h/.cpp` | `factorial()`, `pick()`, `choose()` utilities | Recursive factorial — fine for small N (max 8) |
| `Scanner/CubeScanner.h/.cpp` | Webcam face scanner with HSV classification | Hardcoded hue ranges; median 5×5 filtering; visual verification loop |
| `Databases/cornerDepth5V1.txt` | Pre-computed corner pattern database (~50 MB binary) | Loaded at runtime by `IDAstarSolver` |
| `bits/stdc++.h` | Custom umbrella header for cross-platform compatibility | Includes ~20 standard headers; enables code to use `#include <bits/stdc++.h>` on MSVC |
| `CMakeLists.txt` | CMake build configuration | C++14; links OpenCV; includes all source files |

A new engineer should start reading from `Model/RubiksCube.h` (the abstract interface), then `RubiksCubeBitboard.cpp` (the primary representation), then `IDAstarSolver.h` + `CornerPatternDatabase.cpp` (the core algorithm).

---

## 9. Data Model

The project does not use a traditional database. All state is in-memory during execution.

### Core Entity: Cube State

- **3D Array**: `char cube[6][3][3]` — each char is a color letter ('W', 'G', 'R', 'B', 'O', 'Y')
- **1D Array**: `char cube[54]` — flat mapping: `face * 9 + row * 3 + col`
- **Bitboard**: `uint64_t bitboard[6]` — each face is a 64-bit integer with 8 stickers × 8 bits each, using one-hot color encoding (bit position = color index). Center sticker is implicit.

### Corner Representation

For the pattern database, each cube state is decomposed into:
- **Corner permutation**: 8 corners, each identified by a 3-bit index encoding which of {W/Y, R/O, B/G} is present → 8! = 40,320 permutations
- **Corner orientation**: 7 independent orientations (the 8th is determined by the other 7), each with 3 possible values → 3⁷ = 2,187 combinations
- **Database index**: `rank(permutation) × 2187 + orientationNumber` → maps to 0..100,179,839

### State Transitions
- Each of 18 moves (L, L', L2, R, R', R2, U, U', U2, D, D', D2, F, F', F2, B, B', B2) transforms the cube state.
- `move()` applies a move; `invert()` applies the inverse.
- `isSolved()` checks if the current state matches the solved configuration.

### Search State (in solvers)
- **BFS**: `unordered_map<T, bool, H> visited` + `unordered_map<T, MOVE, H> move_done` (back-pointers)
- **IDA***: Same visited/move_done maps + `priority_queue<pair<Node, int>>` ordered by `f(n) = depth + estimate`

<img width="886" height="653" alt="image" src="https://github.com/user-attachments/assets/032deb16-fccd-4a14-aa49-7fa09553bf4c" />


---

## 10. API and Interface Design

This is a desktop application, not a web service. There are no HTTP endpoints, REST APIs, or RPC interfaces.

### Internal Interfaces

**`RubiksCube` (Abstract Base Class)**:
- `getColor(FACE, row, col)` → `COLOR` (pure virtual)
- `setColor(FACE, row, col, COLOR)` (pure virtual)
- `isSolved()` → `bool` (pure virtual)
- `move(MOVE)` → dispatches to the appropriate virtual move function
- `invert(MOVE)` → applies the inverse move
- 18 pure virtual move functions: `f()`, `fPrime()`, `f2()`, etc.
- `randomShuffleCube(n)` → applies n random moves, returns them
- `getCornerIndex(ind)` / `getCornerOrientation(ind)` → corner decomposition

**Solver Template Interface** (consistent across all four solvers):
- Constructor takes `T _rubiksCube` (and optionally `max_search_depth` or `fileName`)
- `solve()` → `vector<MOVE>` — the solution sequence
- `rubiksCube` member — holds the solved state after `solve()` completes

**PatternDatabase Interface**:
- `getDatabaseIndex(const RubiksCube&)` → `uint32_t` (pure virtual)
- `getNumMoves(const RubiksCube&)` → `uint8_t`
- `setNumMoves(const RubiksCube&, uint8_t)` → `bool`
- `toFile(string)` / `fromFile(string)` — binary serialization

### Error Handling Conventions
- Errors are handled via `throw` (string literals, not typed exceptions), `assert()`, and `runtime_error`.
- No consistent error handling strategy. `fromFile()` returns `false` on missing file but throws on size mismatch. The return value of `fromFile()` is not checked in `IDAstarSolver`.

---

## 11. Authentication and Authorization

Not applicable. This is a standalone desktop application with no user authentication, sessions, or permissions model.

---

## 12. Important Engineering Decisions

### Decision 1: Triple Cube Representation with Abstract Interface

| Aspect | Detail |
|---|---|
| **What was chosen** | Three concrete implementations (`3dArray`, `1dArray`, `Bitboard`) behind an abstract `RubiksCube` base class |
| **Evidence** | `RubiksCube.h` defines pure virtual functions; three `.cpp` files implement them differently |
| **Likely Reason** | Enables fair benchmarking of the same algorithm across different memory layouts; demonstrates OOP design for portfolio |
| **Benefit** | Any solver works with any representation via C++ templates; clean separation of concerns |
| **Cost** | Three implementations to maintain; virtual function call overhead (mitigated by templates) |
| **Alternative** | Single optimised representation (bitboard only) |
| **When to Reconsider** | If the project focused purely on solving speed, maintaining only the bitboard would reduce complexity |

### Decision 2: C++ Templates for Solver Genericity

| Aspect | Detail |
|---|---|
| **What was chosen** | `template<typename T, typename H>` on all solvers, parameterised on cube representation and hash |
| **Evidence** | All four solvers in `Solver/` are templates |
| **Likely Reason** | Avoids virtual dispatch overhead during search (compile-time polymorphism); enables custom hash per representation |
| **Benefit** | No runtime overhead for polymorphism in the hot search loop |
| **Cost** | All solver code must be in headers; longer compile times; template error messages are harder to read |
| **Alternative** | Virtual base class pointer + `std::function` hash — simpler but slower |
| **When to Reconsider** | If the project required runtime selection of representation (e.g., user chooses via CLI flag) |

### Decision 3: Bitboard One-Hot Color Encoding

| Aspect | Detail |
|---|---|
| **What was chosen** | Each sticker encoded as a one-hot bit in an 8-bit segment of a `uint64_t` (6 possible colors → 6 bits used out of 8) |
| **Evidence** | `RubiksCubeBitboard.cpp`: constructor sets `clr = 1 << side`; `getColor()` counts trailing bits |
| **Likely Reason** | Enables face rotations as a single 16-bit circular shift (`side >> 48 | side << 16`); XOR-based O(1) hashing |
| **Benefit** | Face rotation is a single CPU instruction; hashing 6 XORed `uint64_t` values is extremely fast |
| **Cost** | Wastes 2 bits per sticker (8 bits used, only 6 needed); `getColor()` requires a bit-counting loop |
| **Alternative** | 3-bit encoding (6 colors fit in 3 bits) → more stickers per word, but rotations become harder |
| **When to Reconsider** | If memory was the primary bottleneck (unlikely for a single cube state) |

### Decision 4: Corner-Only Pattern Database

| Aspect | Detail |
|---|---|
| **What was chosen** | Pre-computed heuristic based only on the 8 corner cubies (ignoring 12 edge cubies) |
| **Evidence** | `CornerPatternDatabase` indexes `8! × 3^7 = 88,179,840` states into a `NibbleArray` of size `100,179,840` |
| **Likely Reason** | Corners alone produce a useful admissible heuristic within manageable memory (~50 MB); implementing edge databases would require hundreds of MB |
| **Benefit** | Sufficient to solve 13+ move scrambles in reasonable time; database generation completes in minutes |
| **Cost** | Weaker heuristic than combined corner+edge databases; may not efficiently solve very deep scrambles (15+ moves) |
| **Alternative** | Full corner + edge databases (~1–2 GB), or Kociemba's two-phase algorithm |
| **When to Reconsider** | When solving very deep scrambles (approaching God's Number of 20) consistently is required |

### Decision 5: IDA* with Priority Queue (vs. Standard Recursive IDA*)

| Aspect | Detail |
|---|---|
| **What was chosen** | IDA* uses a `priority_queue` with `visited` and `move_done` maps — more like A* with iterative bound increases |
| **Evidence** | `IDAstarSolver.h`: `priority_queue<pair<Node, int>>` ordered by `f(n)`, plus `unordered_map<T, bool, H> visited` |
| **Likely Reason** | Avoids re-exploring nodes within a single bound iteration; combines IDA*'s iterative deepening with A*'s best-first expansion |
| **Benefit** | More efficient exploration within each bound; back-pointer reconstruction for solution path |
| **Cost** | Higher memory usage than pure recursive IDA* (stores visited set and priority queue); loses IDA*'s main advantage of O(d) memory |
| **Alternative** | Pure recursive IDA* with no visited set (standard textbook implementation) — O(d) memory, but re-explores nodes |
| **When to Reconsider** | If memory becomes the bottleneck for very deep searches, switching to recursive IDA* would save memory at the cost of repeated work |

### Decision 6: BFS Depth Limit of 8 in Database Generator

| Aspect | Detail |
|---|---|
| **What was chosen** | `CornerDBMaker::bfsAndStore()` breaks at `curr_depth == 9`, limiting BFS to depth 8 |
| **Evidence** | `CornerDBMaker.cpp` line: `if (curr_depth == 9) break;` |
| **Likely Reason** | Balancing database completeness vs. generation time and memory. At depth 8, most of the 88M corner states are reached |
| **Benefit** | Database generates in a few minutes; file is ~50 MB |
| **Cost** | Some corner configurations at depth 9+ will have `0xF` (15) as their entry — the heuristic returns max value for unseen states, which is still admissible but less informative |
| **Alternative** | BFS to full completion (all states reached) — would take longer but produce a more informative heuristic |
| **When to Reconsider** | If solver performance on deep scrambles needs improvement, completing the database would help |

### Decision 7: Custom `bits/stdc++.h` for Cross-Platform Compatibility

| Aspect | Detail |
|---|---|
| **What was chosen** | A custom umbrella header at `bits/stdc++.h` including ~30 standard library headers |
| **Evidence** | `bits/stdc++.h` file; `CMakeLists.txt` includes `${CMAKE_SOURCE_DIR}` to resolve it |
| **Likely Reason** | GCC provides `<bits/stdc++.h>` but MSVC/Clang do not; this enables the competitive-programming style `#include <bits/stdc++.h>` across platforms |
| **Benefit** | No need to track individual includes per file; familiar to competitive programmers |
| **Cost** | Pulls in many unnecessary headers; slows compilation; non-standard practice |
| **Alternative** | Use explicit includes per file |
| **When to Reconsider** | In any production or team setting; compile times matter at scale |

---

## 13. Reliability and Failure Handling

### Likely Failure Points

| Failure | Handling | Risk |
|---|---|---|
| **Webcam unavailable** | `CubeScanner` throws `runtime_error("Failed to open webcam")` | Program crashes with unhandled exception unless caught in main |
| **Database file missing** | `PatternDatabase::fromFile()` returns `false` | `IDAstarSolver` constructor does not check return value — solver runs with uninitialised database (all entries = `0xFF` = 15), causing extremely slow or broken search |
| **Database file corrupt (wrong size)** | `fromFile()` throws `"Database corrupt! Failed to open Reader"` (raw string, not `std::exception`) | Crashes with uncatchable-by-type exception |
| **Invalid cube state from scanner** | No validation | If the scanned state is not a valid Rubik's Cube (e.g., duplicate colors, impossible configuration), the solver may never find a solution and run indefinitely |
| **Deep scramble (>13 moves)** | No timeout or depth limit on IDA* | Search may take impractical time; memory grows with priority queue and visited set |
| **Color misclassification** | No automated detection | Solution may be incorrect or solver may fail |

### Missing Safeguards
- No timeout mechanism for solvers.
- No validation that the scanned cube configuration is a valid permutation.
- No graceful error recovery — errors either crash the program or silently produce wrong results.
- `assert()` is used in `NibbleArray::get/set` and `BFSSolver::solve` — these are stripped in release builds, removing safety checks.

---

## 14. Performance and Scalability

### Performance Characteristics

| Component | Characteristic | Notes |
|---|---|---|
| **DFS** | O(18^d) time, O(d) space | Practical for d ≤ 6 |
| **BFS** | O(18^d) time and space | Memory exhaustion at d ≈ 7 (18^7 ≈ 612M states) |
| **IDDFS** | O(18^d) time, O(d) space | Better than BFS on memory, but redundant re-exploration |
| **IDA*** | O(18^d / pruning) time | Corner heuristic reduces effective branching factor to ~3–5; practical for d ≤ 13 |
| **Pattern DB loading** | ~50 MB disk read | One-time cost at startup |
| **Pattern DB lookup** | O(1) per query | `getDatabaseIndex()` involves `PermutationIndexer::rank()` (O(8) with precomputed tables) + orientation arithmetic |
| **Bitboard move execution** | ~1–2 CPU cycles for face rotation | Bit shift + mask operations |

### Bottlenecks
- **IDA* memory**: The priority queue and visited/move_done `unordered_map` can grow large for deep scrambles. Each entry in `visited` stores a full `RubiksCubeBitboard` (48 bytes for 6 × `uint64_t`) plus hash map overhead.
- **Corner heuristic weakness**: The corners-only heuristic underestimates significantly for edge-heavy scrambles, causing more exploration.
- **Hash collisions**: `HashBitboard` uses simple XOR of 6 `uint64_t` values — this has high collision rates because XOR is symmetric and many cube states differ in only one face.
- **`randomShuffleCube` RNG**: Uses `srand(time(0))` + `rand()` — not a concern for performance but produces low-quality randomness.

### What Should Be Measured Before Optimising
- Time-to-solve vs. scramble depth for each algorithm
- Peak memory usage of IDA* for various scramble depths
- Hash collision rate of `HashBitboard` in BFS/IDA* visited sets
- Move execution speed across the three representations

---

## 15. Security and Privacy Review

### Observed Issues

| Issue | Severity | Details |
|---|---|---|
| **Raw string exception throws** | Low | `PatternDatabase.cpp` throws `"Database corrupt!"` and `"Failed to open..."` — these are `const char*`, not `std::exception` subclasses. Standard catch blocks won't catch them unless `catch(...)` is used. |
| **No input validation on scanned cube** | Medium | A malformed cube state (e.g., 7 white stickers on one face) is accepted without validation, leading to undefined solver behavior. |
| **`srand(time(0))` for shuffle** | Low | Predictable seed; trivial for testing but should not be used if scramble unpredictability matters. |
| **No file path sanitization** | Low | The database file path is hardcoded in source. No user input is passed to file operations, so injection is not a current risk. |

### General Production Recommendations (if applicable)
- Webcam access without user consent notification (standard for desktop apps, but worth noting).
- No logging of sensitive data (no sensitive data exists in this project).
- No network access — the application is entirely offline.

---

## 16. Testing and Quality Strategy

### Existing Tests
The repository contains **no automated tests**. There are no test files, no test framework integration, and no CI/CD pipelines.

### Manual Test Code
`main.cpp` contains extensive commented-out test blocks (~245 lines) that manually verify:
- Cube creation, printing, and `isSolved()` across all three representations
- Equality and assignment operators
- `unordered_map` usage with custom hash functions
- Each solver (DFS, BFS, IDDFS, IDA*) with random scrambles
- `CornerPatternDatabase` index/orientation functions
- `CornerDBMaker` database generation

These function as integration smoke tests when uncommented but are not automated or CI-runnable.

### What Appears Untested
- Edge cases: already-solved cube, single-move scramble, invalid cube state
- Scanner color classification accuracy under different lighting conditions
- Pattern database integrity after generation
- Solver correctness for known test cases (e.g., specific scramble → known optimal solution)
- Move correctness: no test verifies that `move(X)` followed by `invert(X)` returns to the original state for all representations

### Recommended Testing Pyramid
1. **Unit Tests**: Move correctness (apply + invert = identity), corner index/orientation calculations, `NibbleArray` get/set, `PermutationIndexer` rank for known permutations
2. **Integration Tests**: Scramble → solve → verify `isSolved()` for each solver × representation combination
3. **Regression Tests**: Known scramble sequences with known optimal solutions
4. **Property Tests**: `randomShuffleCube(n)` → solve → verify `isSolved()` with n = 1..8

---

## 17. Deployment and Operations

### Local Development
- **Prerequisites**: C++14 compiler, CMake ≥ 3.20, OpenCV 4.x
- **Build**: `mkdir build && cd build && cmake -G "MinGW Makefiles" .. && mingw32-make`
- **Run**: `./rubiks_cube_solver.exe` (requires webcam for default mode)
- **Alternative mode**: Uncomment solver test blocks in `main.cpp` for webcam-free testing

### What Is Absent
- **No Docker configuration**: No `Dockerfile` or `docker-compose.yml`.
- **No CI/CD**: No GitHub Actions, Jenkins, or equivalent workflows.
- **No automated builds**: No build scripts beyond CMake.
- **No release artifacts**: No pre-built binaries or installers.
- **No logging**: No logging framework; only `cout` console output.
- **No monitoring or alerting**: Not applicable for a desktop tool.
- **No database migrations**: Not applicable (binary file, not a database).
- **No environment configuration**: No `.env` file; the database path is hardcoded in source.

---

## 18. Current Strengths

1. **Clean polymorphic architecture**: The `RubiksCube` abstract base class with three concrete implementations is a well-executed OOP design. The template-based solvers decouple algorithm from representation without runtime dispatch overhead. (`RubiksCube.h`, all solver headers)

2. **Well-implemented IDA* with admissible heuristic**: The corner pattern database is mathematically correct — it uses Lehmer code indexing and 3-base orientation encoding to map ~88M states, and the `NibbleArray` halves memory usage. The heuristic is provably admissible. (`CornerPatternDatabase.cpp`, `PermutationIndexer.h`, `NibbleArray.cpp`)

3. **Bitboard representation**: Applying chess-engine techniques (one-hot encoding, bit-shift rotations, XOR hashing) to the Rubik's Cube is creative and performant. Face rotation reduces to `(side >> 48) | (side << 16)`. (`RubiksCubeBitboard.cpp`)

4. **Algorithmic progression**: The four solvers form a pedagogical ladder from naive to optimal informed search, each building on the previous. DFS → BFS → IDDFS → IDA* demonstrates understanding of search algorithm trade-offs.

5. **Complete scanner pipeline**: The `CubeScanner` implements a full computer vision pipeline: webcam capture → grid overlay → median filtering → HSV color classification → visual verification → cube state injection. (`CubeScanner.cpp`)

6. **Practical combinatorial indexing**: The `PermutationIndexer<N,K>` template with precomputed ones-count lookup and factorial tables is a correct, efficient implementation of Lehmer code ranking. (`PermutationIndexer.h`)

7. **Thoughtful memory optimisation**: The `NibbleArray` packs two 4-bit values per byte, reducing the ~88 MB naive storage to ~50 MB. The nibble-packing implementation with bitwise operations for even/odd positions is clean. (`NibbleArray.cpp`)

---

## 19. Current Limitations and Technical Debt

### Critical

| # | Limitation | Impact | Practical Improvement | Acceptable Now? |
|---|---|---|---|---|
| 1 | **No cube validity check after scanning** | Solver may run indefinitely or produce garbage for invalid cube states | Add a validation function that checks: exactly 9 of each color, valid corner/edge permutation parity, valid corner/edge orientation sum | No — this is a user-facing bug |

### High

| # | Limitation | Impact | Practical Improvement | Acceptable Now? |
|---|---|---|---|---|
| 2 | **`fromFile()` return value not checked** | If the database file is missing, the solver runs with uninitialised data — producing wrong results silently | Add error handling in `IDAstarSolver` constructor | No — silent failure |
| 3 | **No automated tests** | Cannot verify correctness after changes; refactoring is risky | Add a test framework (Google Test or Catch2) with basic solver correctness tests | Acceptable for portfolio, but a significant gap |
| 4 | **Hardcoded scanner hue thresholds** | Color misclassification under non-ideal lighting | Add calibration mode or adaptive thresholding | Acceptable for demo |

### Medium

| # | Limitation | Impact | Practical Improvement | Acceptable Now? |
|---|---|---|---|---|
| 5 | **Corners-only heuristic** | Limited solving depth (~13 moves practical max) | Add edge pattern databases for a stronger combined heuristic | Yes — adequate for demo |
| 6 | **No solver timeout** | Deep scrambles may run for hours | Add a configurable timeout or iteration limit | Yes — user can kill the process |
| 7 | **String-literal exception throws** | Uncatchable by standard `catch(std::exception&)` blocks | Use `std::runtime_error` instead of raw strings | Yes — minor |
| 8 | **`HashBitboard` uses simple XOR** | High collision rate in unordered maps | Use a proper hash combiner (boost::hash_combine or FNV) | Yes — performance impact is measurable but not critical at current scale |
| 9 | **Classes defined in `.cpp` files** | `RubiksCube3dArray`, `RubiksCube1dArray`, `RubiksCubeBitboard` are defined in `.cpp` files without separate headers | Separate declarations into `.h` files | Yes — works but unconventional |

### Low

| # | Limitation | Impact | Practical Improvement | Acceptable Now? |
|---|---|---|---|---|
| 10 | **`srand(time(0))` for randomness** | Predictable scrambles; not thread-safe | Use `<random>` with `mt19937` | Yes |
| 11 | **Custom `bits/stdc++.h`** | Slower compilation; non-standard | Use explicit per-file includes | Yes |
| 12 | **Commented-out code in `main.cpp`** | 245 lines of dead code; hard to navigate | Extract test blocks into a test suite or remove | Yes |

---

## 20. Production Readiness Gap

This project is a **portfolio/educational project**, not a production system. The following would be needed to move it toward production-grade quality:

| Area | Current State | Needed |
|---|---|---|
| **Input Validation** | No cube state validation | Validate scanned cube is a legal Rubik's Cube permutation |
| **Error Handling** | Mix of throws, asserts, unchecked returns | Consistent exception hierarchy; handle all error paths |
| **Testing** | No automated tests | Unit + integration test suite; CI-runnable |
| **Configuration** | Hardcoded paths and parameters | Configuration file or CLI arguments for database path, camera index, solver selection |
| **Logging** | `cout` only | Structured logging with severity levels |
| **Build/CI** | Manual CMake only | GitHub Actions workflow for build + test on push |
| **Documentation** | Good README; no API docs | Doxygen or inline documentation for public interfaces |
| **Performance** | No benchmarks | Automated benchmark suite comparing solvers/representations |
| **Packaging** | Source-only | CMake install targets; pre-built binaries for releases |
| **Cross-Platform** | Targets MinGW on Windows | Test on MSVC, GCC (Linux), Clang (macOS) |

---

## 21. Improvement Roadmap

### Immediate: Next 1–2 Weeks

| Improvement | Why It Matters | Impact | Complexity | Dependencies | Success Metric |
|---|---|---|---|---|---|
| Add cube state validation after scanning | Prevents silent solver failures on invalid input | High | Low | None | `isValidCubeState()` returns false for known-bad configurations |
| Check `fromFile()` return value in IDA* constructor | Prevents running with uninitialised database | High | Low | None | Solver throws meaningful error if DB file is missing |
| Fix string-literal throws to `std::runtime_error` | Enables proper exception handling | Medium | Low | None | All exceptions are `std::exception` subclasses |
| Extract test blocks from `main.cpp` | Cleaner codebase; reusable tests | Medium | Low | None | `main.cpp` < 30 lines; tests in separate files |

### Near Term: Next 1–2 Months

| Improvement | Why It Matters | Impact | Complexity | Dependencies | Success Metric |
|---|---|---|---|---|---|
| Add automated test suite (Google Test or Catch2) | Enables confident refactoring; verifies correctness | High | Medium | Test framework dependency | All solvers pass correctness tests for known scrambles |
| CLI argument parsing for configuration | Enables webcam-free mode, solver selection, database path | High | Medium | None | `./solver --no-webcam --solver=bfs --depth=5` works |
| Improve hash function (`HashBitboard`) | Reduces collision rate; improves BFS/IDA* performance | Medium | Low | None | Measurable speedup on BFS with 6+ move scrambles |
| Add scanner calibration mode | Improves color classification under varied lighting | Medium | Medium | OpenCV | User runs calibration once; stored thresholds used thereafter |

### Medium Term: Next 3–6 Months

| Improvement | Why It Matters | Impact | Complexity | Dependencies | Success Metric |
|---|---|---|---|---|---|
| Add edge pattern databases | Significantly stronger heuristic; solve 15+ move scrambles | High | High | Significant memory (~500 MB–2 GB for full edge DBs) | IDA* solves random 15-move scrambles in < 60 seconds |
| Implement Kociemba's two-phase algorithm | Industry-standard approach for fast solving (< 22 moves guaranteed) | High | High | Different algorithmic approach | Any scramble solved in < 1 second |
| GitHub Actions CI/CD | Automated build + test on every push | Medium | Medium | Test suite exists | Green CI badge on README |
| Cross-platform build support | Enables Linux/macOS usage | Medium | Medium | Platform-specific testing | Builds on GCC + Clang + MSVC |

---

## 22. Metrics That Should Be Tracked

### Performance Metrics
| Metric | Why It Matters |
|---|---|
| **Time-to-solve vs. scramble depth** | Measures algorithm efficiency; reveals practical depth limits |
| **States explored vs. scramble depth** | Quantifies heuristic pruning effectiveness |
| **Peak memory usage per solver** | Identifies memory bottleneck for BFS/IDA* |
| **Move execution time per representation** | Validates bitboard performance advantage |

### Quality Metrics
| Metric | Why It Matters |
|---|---|
| **Solution optimality rate** | Percentage of solutions that match known optimal move count |
| **Scanner classification accuracy** | Percentage of correctly identified sticker colors under standard lighting |
| **Hash collision rate** | Impacts search performance in BFS/IDA* |

### Reliability Metrics
| Metric | Why It Matters |
|---|---|
| **Solver completion rate** | Percentage of scrambles that solve within a time limit |
| **Scanner retry rate** | How often users press R to rescan — indicates classification quality |

The repository does not currently track any of these metrics. No benchmarking infrastructure exists.

---

## 23. Key Project Stories for Interviews

### Story 1: The Bitboard Representation — Borrowing from Chess Engines

- **Context**: Needed a fast cube state representation for search-heavy solving.
- **Challenge**: Face rotations involve moving 8 stickers simultaneously across a 3D structure — element-by-element swaps are slow.
- **Decision**: Encoded each face as a 64-bit integer with one-hot color encoding. Face rotation becomes a single bit shift: `(side >> 48) | (side << 16)`.
- **Result**: O(1) face rotations and XOR-based O(1) hashing, dramatically faster than array-based approaches.
- **Learning**: Techniques from one domain (chess bitboards) can be creatively adapted to structurally different problems (Rubik's Cube).
- **Follow-up**: Profile actual speedup vs. 3D array; the one-hot encoding wastes 2 bits per sticker and `getColor()` requires bit-counting — worth measuring whether this matters in practice.

### Story 2: Choosing IDA* Over Standard A*

- **Context**: BFS provides optimal solutions but runs out of memory at ~7 moves. Need to solve 13+ move scrambles.
- **Challenge**: Standard A* stores the entire open set in memory — still too large for the Rubik's Cube state space.
- **Decision**: Implemented IDA* with iterative bound increases. Each iteration uses a priority queue with visited tracking (more like bounded A* than pure recursive IDA*).
- **Result**: Can solve 13+ move scrambles with manageable memory by restarting with higher bounds.
- **Learning**: The trade-off between memory and repeated work is fundamental to search; IDA* trades redundant computation for bounded memory.
- **Follow-up**: The current implementation uses a priority queue + visited set, which is heavier than textbook recursive IDA*. Measuring memory usage vs. a pure recursive implementation would clarify whether this hybrid approach is beneficial.

### Story 3: Corner Pattern Database Design

- **Context**: IDA* needs an admissible heuristic to prune the 18-branching-factor search tree.
- **Challenge**: The full Rubik's Cube has ~4.3×10¹⁹ states — impossible to pre-compute distances for all of them.
- **Decision**: Pre-compute distances for corner configurations only (8! × 3⁷ ≈ 88M states) using BFS from the solved state. Store in a `NibbleArray` (4 bits per entry) for ~50% memory savings.
- **Result**: The heuristic reduces the effective branching factor from 18 to ~3–5, enabling 13+ move solves.
- **Learning**: Pattern databases work because subproblem lower bounds are admissible. The Lehmer code indexing technique was essential — without it, mapping permutations to array indices would require a hash table.
- **Follow-up**: The BFS generator stops at depth 8. Completing the BFS to cover all states, or adding edge databases, would strengthen the heuristic.

### Story 4: Polymorphism vs. Performance Trade-off

- **Context**: Wanted to support multiple cube representations while keeping solver code generic.
- **Challenge**: Virtual function calls in the inner search loop (called millions of times) could significantly slow down solving.
- **Decision**: Used C++ templates instead of virtual dispatch for the solver-representation interface. The `RubiksCube` base class uses virtual functions for shared utilities (`print`, corner extraction), but solvers use compile-time polymorphism.
- **Result**: No runtime dispatch overhead in the hot loop; each solver is specialised at compile time for its representation.
- **Learning**: Templates and virtual functions solve different problems. Templates for performance-critical generic code; virtual functions for runtime flexibility.
- **Follow-up**: Benchmark the actual overhead of virtual dispatch vs. templates in this specific use case.

### Story 5: Scanner Color Classification Under Real-World Conditions

- **Context**: Needed to classify 6 colors from webcam images of a physical Rubik's Cube.
- **Challenge**: BGR color values vary dramatically with lighting conditions, cube material, and camera quality.
- **Decision**: Convert to HSV color space and use hue thresholds for classification. Added a 5×5 median filter to reduce noise. White is detected via RGB channel similarity rather than hue.
- **Result**: Works under controlled lighting but struggles with shadows, reflections, and ambient color casts.
- **Learning**: Fixed hue thresholds are fragile. HSV separates brightness from color (useful), but real-world conditions still cause misclassification.
- **Follow-up**: A calibration step or ML-based classifier would be more robust.

### Story 6: The NibbleArray — When Half a Byte Is Enough

- **Context**: The corner pattern database has ~88M entries, each needing to store a value 0–15 (minimum moves).
- **Challenge**: 88M bytes = ~84 MB. Want to reduce memory footprint.
- **Decision**: Implemented a `NibbleArray` that packs two 4-bit values per byte using bitwise operations (high nibble for even indices, low nibble for odd indices).
- **Result**: ~50% memory reduction (84 MB → ~50 MB including overhead).
- **Learning**: When values fit in 4 bits, nibble packing is a straightforward optimisation with significant impact at scale. The same technique is used in genome sequence compression.
- **Follow-up**: The `storageSize()` is `size/2 + 1` — the +1 handles odd sizes. An off-by-one bug here would be catastrophic.

---

## 24. Facts, Inferences, and Assumptions

### Verified from the Repository

- Four search algorithms are implemented: DFS, BFS, IDDFS, IDA*.
- Three cube representations exist: 3D array (`char[6][3][3]`), 1D array (`char[54]`), bitboard (`uint64_t[6]`).
- All representations share the `RubiksCube` abstract base class with pure virtual functions.
- All solvers are C++ templates parameterised on `<T, H>`.
- The corner pattern database indexes `8! × 3^7` states using Lehmer code ranking and orientation encoding.
- The `NibbleArray` packs two 4-bit values per byte.
- The `CornerDBMaker` BFS generator stops at depth 8.
- The active `main()` code path scans via webcam and solves with IDA*.
- The database file `cornerDepth5V1.txt` is ~50 MB.
- The `CubeScanner` uses OpenCV with hardcoded HSV hue thresholds.
- The project builds with CMake, requires C++14, and links against OpenCV.
- No automated tests exist. All test code is commented out in `main.cpp`.
- No CI/CD, Docker, or deployment configuration exists.
- `fromFile()` return value is not checked in the IDA* constructor.
- Exceptions are thrown as raw string literals, not `std::exception` subclasses.

### Strongly Inferred

- The bitboard representation is the intended production representation — it is used in the active code path, the `CornerDBMaker`, and is the only representation with a `getCorners()` method.
- The 3D array representation was built first (most intuitive), then optimised progressively to 1D and bitboard.
- The `Model/PatternDatabase/PatternDatabase.h` stub (returning 0) was a precursor to the full `PatternDatabases/` module.
- The database file name `cornerDepth5V1.txt` likely indicates "depth 5 version 1" from an earlier iteration, though the current generator goes to depth 8.
- The IDA* implementation's use of a priority queue + visited set trades IDA*'s O(d) memory advantage for reduced redundant exploration.

### Assumptions Requiring Confirmation

- Whether the project has been benchmarked (solve time vs. scramble depth) across representations — no benchmark data exists in the repository.
- Whether the database file in `Databases/` was generated by the included `CornerDBMaker` code or an external tool — the file size (~50 MB) is consistent with the code's output.
- Whether the scanner has been tested with a real physical Rubik's Cube under varied conditions — no test results or calibration data exists.
- The exact reason for the database size being `100,179,840` rather than `88,179,840` (8! × 3^7) — there appears to be some padding; this may be intentional for alignment or an earlier size calculation. The candidate should verify this constant.
