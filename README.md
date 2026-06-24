# 🧊 Rubik's Cube Solver

> An advanced C++ engine that solves a 3×3 Rubik's Cube using graph search algorithms, pattern databases, bitwise state representations, and real-time computer vision — demonstrating how classical AI, combinatorial optimization, and efficient data structures converge to crack one of the world's most iconic puzzles.

---

## 📑 Table of Contents

- [Why This Project Matters](#-why-this-project-matters)
- [Features](#-features)
- [Architecture Overview](#-architecture-overview)
- [Cube Representations — Three Ways to Model the Same Problem](#-cube-representations--three-ways-to-model-the-same-problem)
- [Solving Algorithms — From Brute Force to Optimal](#-solving-algorithms--from-brute-force-to-optimal)
- [Pattern Databases — The Secret to Near-Optimal Solutions](#-pattern-databases--the-secret-to-near-optimal-solutions)
- [Computer Vision Scanner](#-computer-vision-scanner)
- [Advanced Concepts Applied](#-advanced-concepts-applied)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
- [Usage](#-usage)
- [Complexity Analysis](#-complexity-analysis)
- [References & Further Reading](#-references--further-reading)
- [License](#-license)

---

## 🎯 Why This Project Matters

A 3×3 Rubik's Cube has **43,252,003,274,489,856,000** (43 quintillion) possible states — yet every single one can be solved in **≤ 20 moves** (["God's Number"](https://www.cube20.org/)). Bridging that gap between an astronomically large state space and an elegantly short solution is a deeply non-trivial problem in computer science.

This project explores that problem from first principles:

| Domain | What You'll Learn |
|---|---|
| **Algorithms & AI** | DFS, BFS, IDDFS, IDA* — the progression from blind search to informed, optimal search |
| **Data Structures** | Bitboards, hash maps, nibble arrays, priority queues, permutation indexing |
| **Combinatorics** | Lehmer codes, factorial number systems, group theory on the Rubik's cube group |
| **Computer Vision** | Real-time color classification using HSV color space with OpenCV |
| **Software Engineering** | Polymorphism, C++ templates, design patterns (Strategy, Template Method) |

---

## ✨ Features

- 🔄 **Four Solving Algorithms** — DFS, BFS, IDDFS, and IDA* with increasing sophistication
- 🧠 **Corner Pattern Database** — Pre-computed heuristic (~50 MB) enabling near-optimal solutions for deeply scrambled cubes
- ⚡ **Bitboard Representation** — Each face encoded as a 64-bit integer for O(1) move execution via bit manipulation
- 📷 **Webcam Scanner** — Scan all 6 faces of a physical Rubik's Cube using your webcam (OpenCV)
- 🏗️ **Three Cube Models** — 3D Array, 1D Array, and Bitboard, benchmarkable against each other through a shared abstract interface
- 🔁 **Polymorphic Solver Design** — C++ templates allow any solver to work with any representation

---

## 🏗 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                        main.cpp                         │
│  (Orchestrates scanning, solving, and output)           │
├──────────────┬──────────────────────┬───────────────────┤
│              │                      │                   │
▼              ▼                      ▼                   ▼
┌──────────┐  ┌────────────────┐  ┌──────────────┐  ┌──────────┐
│  Scanner │  │    Solvers     │  │    Model      │  │ Pattern  │
│          │  │                │  │               │  │ Database │
│ OpenCV   │  │ DFS            │  │ RubiksCube    │  │          │
│ Webcam   │  │ BFS            │  │ (Abstract)    │  │ Corner   │
│ HSV      │  │ IDDFS          │  │  ├─3D Array   │  │ DB Maker │
│ Color    │  │ IDA*           │  │  ├─1D Array   │  │ Nibble   │
│ Classify │  │                │  │  └─Bitboard   │  │ Array    │
└──────────┘  └────────────────┘  └──────────────┘  └──────────┘
```

---

## 🎲 Cube Representations — Three Ways to Model the Same Problem

The project implements three distinct internal representations of the Rubik's Cube, all sharing a common abstract interface (`RubiksCube` base class). This is a **textbook application of runtime polymorphism** and allows fair benchmarking of the same algorithm across different data layouts.

### 1. 3D Array Representation (`RubiksCube3dArray`)

```
char cube[6][3][3];  // 6 faces × 3 rows × 3 columns
```

- **Intuitive and readable** — maps directly to the physical cube
- Each face is a 3×3 matrix of color values
- Move operations involve explicit element swaps across faces
- **Best for**: Learning and debugging

### 2. 1D Array Representation (`RubiksCube1dArray`)

```
char cube[54];  // Flat array of all 54 stickers
```

- More **cache-friendly** than the 3D version (contiguous memory)
- Sticker `(face, row, col)` maps to index `face * 9 + row * 3 + col`
- **Best for**: Moderate performance with simple indexing

### 3. Bitboard Representation (`RubiksCubeBitboard`) ⚡

```cpp
uint64_t bitboard[6];  // One 64-bit integer per face
```

- Each face's 8 visible stickers are encoded in 8 bytes of a 64-bit integer
- Color is stored as a **one-hot encoded bit** (bit position = color index)
- Face rotations become **bit-shift operations** — executing in a single CPU cycle
- Side rotations use **bitmask extraction and insertion**
- Hashing the entire cube state reduces to **XOR of 6 integers**

```
Bit Layout per face (8 stickers, 8 bits each):
┌────┬────┬────┐
│ S0 │ S1 │ S2 │     S0-S7: 8 visible sticker positions
├────┼────┼────┤     Each sticker = 8 bits (one-hot color encoding)
│ S7 │ -- │ S3 │     Center (--) is implicit (always the face's color)
├────┼────┼────┤
│ S6 │ S5 │ S4 │     Total: 64 bits per face = 1 uint64_t
└────┴────┴────┘
```

> **Why this matters**: In competitive programming and game AI, bitboard representations are the industry standard (chess engines like Stockfish use them). This project applies the same principle to the Rubik's Cube.

---

## 🔍 Solving Algorithms — From Brute Force to Optimal

The project implements a **progression of four algorithms**, each improving on the last — a pedagogical journey through the landscape of search algorithms in AI.

### 1. Depth-First Search (DFS)

```
Explores as deep as possible before backtracking.
```

- **Branching factor**: 18 (6 faces × 3 move types: clockwise, counter-clockwise, 180°)
- **Depth-limited**: configurable `max_search_depth` (default = 8)
- **Space**: O(depth) — very memory-efficient
- **Time**: O(18^d) — exponential, impractical for deep scrambles
- **Guarantee**: May not find the shortest solution

### 2. Breadth-First Search (BFS)

```
Explores all states at depth d before moving to d+1.
```

- **Guaranteed optimal** (shortest path in unweighted graph)
- Uses `unordered_map` for visited-state tracking with **custom hash functions**
- **Back-pointer reconstruction**: stores `move_done[state] = move` to trace the solution path
- **Space**: O(18^d) — explodes quickly (infeasible beyond ~7 moves)

### 3. Iterative Deepening DFS (IDDFS)

```
Runs DFS with max_depth = 1, 2, 3, ... until solved.
```

- **Combines the best of both worlds**: optimal like BFS, memory-efficient like DFS
- Redundant re-exploration is acceptable because the branching factor dominates
- **Space**: O(depth)
- **Time**: O(18^d) but with a much smaller constant than naive BFS
- **Guarantee**: Finds shortest solution

### 4. IDA* (Iterative Deepening A*) ⭐

```
IDDFS + admissible heuristic from Corner Pattern Database
```

- The **crown jewel** of this project
- Uses a **pre-computed corner pattern database** as an admissible heuristic
- The heuristic provides a lower bound on the number of moves to solve the cube
- A* priority queue orders exploration by `f(n) = g(n) + h(n)`:
  - `g(n)` = depth (moves so far)
  - `h(n)` = estimated remaining moves (from pattern database lookup)
- **Prunes vast portions** of the search tree — can solve 13+ move scrambles
- **Guarantee**: Optimal solution (admissible heuristic never overestimates)

---

## 🗄 Pattern Databases — The Secret to Near-Optimal Solutions

The pattern database is a **pre-computed lookup table** that maps every possible configuration of the cube's **8 corner cubies** to the minimum number of moves required to solve just the corners.

### How It Works

1. **BFS from the solved state**: Starting from the solved cube, BFS explores all reachable states, recording the number of moves to reach each corner configuration
2. **Lehmer Code Indexing**: Each permutation of 8 corners is converted to a unique index using the [Lehmer code](https://en.wikipedia.org/wiki/Lehmer_code) in O(n) time
3. **Nibble Array Storage**: Each entry needs only 4 bits (moves ≤ 15), so two entries are packed per byte using a `NibbleArray`, halving memory usage
4. **Orientation Encoding**: Each corner has 3 possible orientations, giving `8! × 3^7 = 88,179,840` unique states

### Memory Optimization

| Approach | Size |
|---|---|
| Naive (1 byte per state) | ~88 MB |
| Nibble-packed (4 bits per state) | **~44 MB** |
| Actual database file | **~50 MB** |

### The PermutationIndexer

The `PermutationIndexer<N, K>` is a template class that converts any permutation of K items chosen from N into its lexicographic rank using precomputed factorials and a ones-count lookup table. This is a well-known technique in combinatorial mathematics.

```cpp
// Converts permutation → Lehmer code → base-10 index
uint32_t rank(const array<uint8_t, K>& perm) const;
```

---

## 📷 Computer Vision Scanner

The `CubeScanner` module lets you scan a physical Rubik's Cube using your webcam:

1. **Grid Overlay**: Draws a 3×3 alignment grid on the camera feed
2. **Capture**: Press `SPACE` to capture each face
3. **Color Classification**: Converts pixels from BGR → HSV color space, then classifies using hue thresholds:

| Color | Hue Range (OpenCV 0-180) |
|---|---|
| Red | 160–190° |
| Orange | 3–19° |
| Yellow | 20–30° |
| Green | 60–90° |
| Blue | 100–120° |
| White | RGB similarity check |

4. **Median Filtering**: Uses a 5×5 region median to reduce noise
5. **Visual Feedback**: Shows scanned face colors and a full cube net for verification
6. **Rescan Option**: Press `R` to rescan any face, `N` to confirm and move to the next

---

## 🎓 Advanced Concepts Applied

This project is a practical showcase of multiple advanced computer science concepts:

### 1. Graph Theory & State Space Search
The Rubik's Cube state space forms a **Cayley graph** of the Rubik's cube group — a finite group with ~4.3×10¹⁹ elements and 6 generators (face turns). Solving the cube is equivalent to finding a path from any node to the identity element.

### 2. Admissible Heuristics & A* Optimality
The corner pattern database is an **admissible heuristic** — it never overestimates the true distance. This guarantees that IDA* finds optimal solutions. This is the same mathematical framework used in robotics path planning and game AI.

### 3. Bit Manipulation & Cache Optimization
The bitboard representation applies techniques from competitive chess programming:
- **One-hot encoding** of colors into bit positions
- **Bit shifts** for O(1) face rotations
- **Bitmask extraction/insertion** for side rotations
- **XOR-based hashing** for O(1) state comparison

### 4. Combinatorial Indexing (Lehmer Codes)
The Lehmer code converts a permutation to its lexicographic rank — a technique from enumerative combinatorics with applications in:
- Database indexing of permutation states
- Cryptographic key generation
- Combinatorial game theory

### 5. Space-Efficient Storage (Nibble Arrays)
The `NibbleArray` packs two 4-bit values per byte, achieving 50% memory reduction. This same technique is used in:
- Genome sequence compression (2-bit nucleotide encoding)
- GPU texture compression formats

### 6. Object-Oriented Design & C++ Templates
- **Abstract base class** (`RubiksCube`) with pure virtual functions for all 18 moves
- **Template-based solvers** (`template<typename T, typename H>`) decouple algorithm from representation
- **Strategy Pattern**: Any solver can work with any cube representation
- **Custom hash functions** (`Hash3d`, `Hash1d`, `HashBitboard`) for `std::unordered_map`

### 7. Computer Vision Pipeline
The scanner implements a classic CV pipeline: capture → color space conversion → region sampling → median filtering → classification — the same approach used in industrial quality inspection and autonomous systems.

---

## 📁 Project Structure

```
rubiks-cube-solver/
│
├── main.cpp                          # Entry point — scanner + solver orchestration
├── CMakeLists.txt                    # Build configuration (CMake)
│
├── Model/                            # Rubik's Cube representations
│   ├── RubiksCube.h                  # Abstract base class (interface)
│   ├── RubiksCube.cpp                # Shared logic (print, shuffle, move dispatch)
│   ├── RubiksCube3dArray.cpp         # 3D array representation  [char cube[6][3][3]]
│   ├── RubiksCube1dArray.cpp         # 1D array representation  [char cube[54]]
│   └── RubiksCubeBitboard.cpp        # Bitboard representation  [uint64_t bitboard[6]]
│
├── Solver/                           # Search algorithms
│   ├── DFSSolver.h                   # Depth-First Search (depth-limited)
│   ├── BFSSolver.h                   # Breadth-First Search (optimal, memory-heavy)
│   ├── IDDFSSolver.h                 # Iterative Deepening DFS (optimal, memory-light)
│   └── IDAstarSolver.h              # IDA* with corner pattern database heuristic
│
├── PatternDatabases/                 # Heuristic computation for IDA*
│   ├── PatternDatabase.h/cpp         # Abstract pattern database (NibbleArray-backed)
│   ├── CornerPatternDatabase.h/cpp   # Corner-specific database (8! × 3^7 states)
│   ├── CornerDBMaker.h/cpp           # BFS-based database generator
│   ├── PermutationIndexer.h          # Lehmer code → lexicographic rank converter
│   ├── NibbleArray.h/cpp             # 4-bit packed array (50% memory savings)
│   └── math.h/cpp                    # Combinatorial math utilities (pick, factorial)
│
├── Scanner/                          # Computer vision module
│   ├── CubeScanner.h                 # Webcam-based 6-face color scanner
│   └── CubeScanner.cpp               # HSV classification + median filtering
│
├── Databases/                        # Pre-computed data
│   └── cornerDepth5V1.txt            # Corner pattern database (~50 MB)
│
└── bits/
    └── stdc++.h                      # Portable umbrella header (cross-platform)
```

---

## 🚀 Getting Started

### Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| **C++ Compiler** | C++14 or later | g++ (MinGW-W64) or MSVC |
| **CMake** | ≥ 3.20 | Build system |
| **OpenCV** | 4.x | Computer vision (webcam scanner) |

### Installing OpenCV (Windows with MSYS2)

```bash
# Install MSYS2 from https://www.msys2.org/
# Open MSYS2 UCRT64 terminal, then:

pacman -Syu
pacman -S mingw-w64-ucrt-x86_64-opencv mingw-w64-ucrt-x86_64-cmake mingw-w64-ucrt-x86_64-gcc
```

### Build

```bash
# Clone the repository
git clone https://github.com/Akash-Hedaoo/RUBICS-CUBE-SOLVER.git
cd RUBICS-CUBE-SOLVER

# Create build directory
mkdir build && cd build

# Configure (adjust generator as needed)
cmake -G "MinGW Makefiles" ..

# Build
mingw32-make
```

### Run

```bash
./rubiks_cube_solver.exe
```

> **Note**: The default mode opens your webcam to scan a physical cube. To test without a webcam, uncomment one of the solver test blocks in `main.cpp` (BFS, DFS, IDDFS, or IDA*).

---

## 💡 Usage

### Webcam Scanning Mode (Default)

1. Run the executable — your webcam will open
2. Align one face of the cube within the 3×3 grid overlay
3. Press **SPACE** to capture
4. Verify the detected colors in the preview window
5. Press **N** to confirm and move to the next face, or **R** to rescan
6. After all 6 faces are scanned, the solver runs automatically
7. The solution moves are printed to the console

### Solver-Only Mode (No Webcam)

Uncomment one of the solver test blocks in `main.cpp`:

```cpp
// Example: IDA* with random 5-move scramble
RubiksCubeBitboard cube;
vector<RubiksCube::MOVE> shuffle_moves = cube.randomShuffleCube(5);
cout << "Scramble: ";
for (auto move : shuffle_moves) cout << cube.getMove(move) << " ";

IDAstarSolver<RubiksCubeBitboard, HashBitboard> solver(cube, "Databases/cornerDepth5V1.txt");
auto solution = solver.solve();

cout << "\nSolution: ";
for (auto move : solution) cout << cube.getMove(move) << " ";
```

### Generating a New Pattern Database

```cpp
string fileName = "Databases/cornerDB.txt";
CornerDBMaker dbMaker(fileName, 0x99);
dbMaker.bfsAndStore();  // Takes several minutes — explores ~88M states
```

---

## 📊 Complexity Analysis

| Algorithm | Time Complexity | Space Complexity | Optimal? | Max Practical Depth |
|---|---|---|---|---|
| **DFS** | O(18^d) | O(d) | ❌ | ~6 moves |
| **BFS** | O(18^d) | O(18^d) | ✅ | ~7 moves |
| **IDDFS** | O(18^d) | O(d) | ✅ | ~7 moves |
| **IDA*** | O(18^d / pruning) | O(d) | ✅ | **13+ moves** |

> **Key insight**: IDA* with the corner pattern database prunes the effective branching factor from 18 down to approximately 3–5, making it orders of magnitude faster than uninformed search.

---

## 📚 References & Further Reading

1. **Korf, R.E.** (1997). *Finding Optimal Solutions to Rubik's Cube Using Pattern Databases*. AAAI.
2. **Rokicki, T., et al.** (2010). *God's Number is 20*. [cube20.org](https://www.cube20.org/)
3. **Korf, R.E.** (1985). *Depth-First Iterative-Deepening: An Optimal Admissible Tree Search*. Artificial Intelligence, 27(1).
4. **Hart, P.E., Nilsson, N.J., & Raphael, B.** (1968). *A Formal Basis for the Heuristic Determination of Minimum Cost Paths*. IEEE Transactions on Systems Science and Cybernetics.
5. **Lehmer, D.H.** (1960). *Teaching Combinatorial Tricks to a Computer*. Proceedings of Symposia in Applied Mathematics.

---

## 📄 License

This project is open source and available for educational purposes.

---

<p align="center">
  <i>Built with ❤️ and advanced algorithms</i>
</p>