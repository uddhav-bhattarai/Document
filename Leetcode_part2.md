# 🎯 LeetCode Patterns 11-20: Advanced Mastery Guide
### *Master Advanced Patterns with ASCII, Memory Techniques & Socratic Learning*

---

## 📋 Table of Contents

- [🌲 Pattern 11: DFS (Depth-First Search)](#-pattern-11-dfs-depth-first-search)
- [🌊 Pattern 12: BFS (Breadth-First Search)](#-pattern-12-bfs-breadth-first-search)
- [🕸️ Pattern 13: Graphs](#️-pattern-13-graphs)
- [🔢 Pattern 14: Dynamic Programming (1D)](#-pattern-14-dynamic-programming-1d)
- [🔲 Pattern 15: Dynamic Programming (2D Grid)](#-pattern-15-dynamic-programming-2d-grid)
- [💰 Pattern 16: Greedy Strategy](#-pattern-16-greedy-strategy)
- [📊 Pattern 17: Intervals Pattern](#-pattern-17-intervals-pattern)
- [⛰️ Pattern 18: Heap / Priority Queue](#️-pattern-18-heap--priority-queue)
- [🔢 Pattern 19: Bit Manipulation](#-pattern-19-bit-manipulation-1)
- [🌲 Pattern 20: Union-Find (DSU)](#-pattern-20-union-find-dsu)

---

## 🌲 Pattern 11: DFS (Depth-First Search)

> **Definition:** A traversal algorithm that explores as far as possible along each branch before backtracking, using recursion or an explicit stack.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Go deep first, backtrack when stuck
- 💭 **Visual Intuition:** Like exploring a maze by always taking the first unexplored path
- 🔢 **Mathematical Reasoning:** Uses call stack (LIFO) to maintain state
- ⚡ **When to Use:** Path finding, tree traversal, connected components, cycle detection

---

### **ASCII Diagram**

```
Tree DFS Traversal:
      1
     / \
    2   3
   / \
  4   5

Visit Order: 1 → 2 → 4 → 5 → 3

Call Stack Evolution:
[1]
[1,2]
[1,2,4]  ← deepest
[1,2]
[1,2,5]
[1]
[1,3]
[]

Grid DFS (finding island):
1 1 0 0
1 1 0 0    DFS from (0,0):
0 0 1 1    ↓→↓→ marks all connected 1's
           
Path: (0,0) → (0,1) → (1,1) → (1,0)
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Deep Sea Diver"** - Imagine a diver going as deep as possible before coming up |
| 📖 Story Method | "The explorer who never turns back until hitting a wall" |
| 💥 Exaggeration | Picture a mole digging straight down, NEVER sideways first |
| 🔗 Association | **DFS = Depth First = Dive First** |

**Memory Cue:**
> *"DFS is like reading a book: you finish Chapter 1 completely before starting Chapter 2."*

---

### **Key Variations**

1. **Recursive DFS** - Natural call stack
2. **Iterative DFS** - Explicit stack (for avoiding stack overflow)
3. **DFS with Backtracking** - Track and undo choices (Sudoku, N-Queens)
4. **DFS with Memoization** - Cache results (becomes DP)

---

### **Socratic Teaching Round**

**Q1:** Why does DFS use a stack (recursion) while BFS uses a queue?
> **A1:** Stack (LIFO) ensures we explore the *most recently discovered* node first, going deep. Queue (FIFO) explores *oldest* nodes first, going wide.

**Q2:** When would DFS be *worse* than BFS?
> **A2:** When finding shortest path (BFS guarantees shortest in unweighted graphs), or when solution is near the root (BFS finds it faster).

**Q3:** How do you detect cycles in DFS?
> **A3:** Track nodes in current path with a "visiting" state. If you revisit a "visiting" node, cycle found.

**Q4:** Why does DFS risk stack overflow?
> **A4:** Each recursive call adds a frame to the call stack. Deep trees (e.g., linked list) can exhaust stack memory.

**Q5:** How is backtracking related to DFS?
> **A5:** Backtracking IS DFS with state restoration. You explore a path (DFS), mark choices, then "undo" (backtrack) to try alternatives.

---

### **Problem 1 (Easy): Number of Islands**

**A. Problem Statement**
Given a 2D grid of '1's (land) and '0's (water), count the number of islands. An island is surrounded by water and formed by connecting adjacent lands horizontally or vertically.

**B. ASCII Visualization**
```
Grid:
1 1 0 0 0
1 1 0 0 0
0 0 1 0 0
0 0 0 1 1

Island 1:  Island 2:  Island 3:
█ █ . . .  . . . . .  . . . . .
█ █ . . .  . . █ . .  . . . . .
. . . . .  . . . . .  . . . █ █

DFS from each '1', marking visited:
Start (0,0) → DFS marks entire island → count++
Start (2,2) → DFS marks it → count++
Start (3,3) → DFS marks connected cells → count++
Result: 3 islands
```

**C. Pattern Recognition Question**
*How do you know this is a DFS problem?*
> We need to **explore all connected components** - DFS naturally finds all connected cells.

**D. Step-by-Step Reasoning**
1. Scan grid for first unvisited '1'
2. When found, increment island counter
3. Run DFS from that cell to mark entire island as visited
4. DFS explores all 4 directions recursively
5. Repeat until grid fully scanned

**E. Python Solution**
```python
def numIslands(grid):
    if not grid:
        return 0
    
    rows, cols = len(grid), len(grid[0])
    islands = 0
    
    def dfs(r, c):
        # Base cases: out of bounds or water or visited
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] == '0':
            return
        
        # Mark current cell as visited (sink the island)
        grid[r][c] = '0'
        
        # Explore all 4 directions (go DEEP in each direction)
        dfs(r + 1, c)  # down
        dfs(r - 1, c)  # up
        dfs(r, c + 1)  # right
        dfs(r, c - 1)  # left
    
    # Scan entire grid
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':  # Found new island
                islands += 1
                dfs(r, c)  # Sink entire island
    
    return islands
```

**F. Memory Cue**
> *"Sink the island like a ship - DFS floods all connected parts."*

---

### **Problem 2 (Hard): Course Schedule II (Topological Sort)**

**A. Problem Statement**
Return the ordering of courses you should take to finish all courses, given prerequisites. If impossible, return empty array.

**B. ASCII Visualization**
```
Courses: [0, 1, 2, 3]
Prerequisites: [[1,0], [2,0], [3,1], [3,2]]
Meaning: Take 0 before 1, 0 before 2, etc.

Graph (Adjacency List):
0 → [1, 2]
1 → [3]
2 → [3]
3 → []

DFS Topological Sort:
Start DFS from each unvisited node
Post-order adds to result when all children done

DFS(0): Visit 0 → DFS(1) → DFS(3) → add 3 → add 1 → DFS(2) → add 2 → add 0
Result: [3, 1, 2, 0] (reverse post-order)
Valid order: 0 → 1 → 2 → 3 or 0 → 2 → 1 → 3

Cycle Detection:
0 → 1 → 0  (visiting 0 again while in its DFS path)
```

**C. Pattern Recognition Question**
*Why is this DFS, not BFS?*
> DFS with post-order naturally produces topological ordering. We need to finish dependencies (go deep) before adding parent.

**D. Step-by-Step Reasoning**
1. Build adjacency list from prerequisites
2. Use 3 states: unvisited, visiting (in current path), visited
3. DFS from each unvisited node
4. If we revisit a "visiting" node → cycle detected → impossible
5. Add to result in post-order (after all children processed)
6. Reverse result for correct topological order

**E. Python Solution**
```python
def findOrder(numCourses, prerequisites):
    # Build adjacency list: course → [courses that depend on it]
    adj = {i: [] for i in range(numCourses)}
    for course, prereq in prerequisites:
        adj[prereq].append(course)
    
    # States: 0=unvisited, 1=visiting (in path), 2=visited (done)
    state = [0] * numCourses
    result = []
    
    def dfs(course):
        if state[course] == 1:  # Cycle detected!
            return False
        if state[course] == 2:  # Already processed
            return True
        
        state[course] = 1  # Mark as visiting (in current path)
        
        # Visit all dependent courses (go DEEP first)
        for next_course in adj[course]:
            if not dfs(next_course):
                return False
        
        state[course] = 2  # Mark as visited (done)
        result.append(course)  # Add in POST-ORDER
        return True
    
    # Try DFS from every course
    for course in range(numCourses):
        if not dfs(course):
            return []  # Cycle found, impossible
    
    return result[::-1]  # Reverse post-order for topo sort
```

**F. Memory Cue**
> *"Traffic lights: Red (visiting/in-path) = stop, cycle! Green (visited) = go ahead. Add to result when LEAVING (post-order)."*

---

### **Problem 3 (Very Hard): Sudoku Solver**

**A. Problem Statement**
Write a program to solve a Sudoku puzzle by filling empty cells (marked with '.').

**B. ASCII Visualization**
```
Input Board:
5 3 . | . 7 . | . . .
6 . . | 1 9 5 | . . .
. 9 8 | . . . | . 6 .
------+-------+------
8 . . | . 6 . | . . 3
4 . . | 8 . 3 | . . 1
7 . . | . 2 . | . . 6
------+-------+------
. 6 . | . . . | 2 8 .
. . . | 4 1 9 | . . 5
. . . | . 8 . | . 7 9

DFS Backtracking Process:
1. Find empty cell (0,2)
2. Try digit 1: valid? → DFS next cell
3. If future fails → BACKTRACK, try 2
4. Continue until solution or exhausted

Decision Tree (partial):
         Try '1' at (0,2)
        /              \
    Valid?            Invalid? → try '2'
      ↓
  Try '1' at (0,3)
    /         \
  Valid?    Invalid?
    ↓
  ...continue or BACKTRACK
```

**C. Pattern Recognition Question**
*Why is this the hardest DFS problem?*
> It combines: DFS (exploration), backtracking (undo choices), and constraint satisfaction (validate each placement).

**D. Step-by-Step Reasoning**
1. Find next empty cell ('.')
2. Try digits 1-9 in that cell
3. For each digit, check if valid (row, col, 3x3 box constraints)
4. If valid, place digit and DFS to next cell
5. If DFS succeeds → solution found
6. If DFS fails → BACKTRACK (remove digit), try next digit
7. If all digits fail → return False (backtrack further)

**E. Python Solution**
```python
def solveSudoku(board):
    def is_valid(row, col, num):
        # Check row
        if num in board[row]:
            return False
        
        # Check column
        if num in [board[i][col] for i in range(9)]:
            return False
        
        # Check 3x3 box
        box_row, box_col = 3 * (row // 3), 3 * (col // 3)
        for i in range(box_row, box_row + 3):
            for j in range(box_col, box_col + 3):
                if board[i][j] == num:
                    return False
        return True
    
    def dfs():
        # Find next empty cell
        for row in range(9):
            for col in range(9):
                if board[row][col] == '.':
                    # Try each digit 1-9
                    for num in '123456789':
                        if is_valid(row, col, num):
                            board[row][col] = num  # Make choice
                            
                            if dfs():  # DFS to next cell
                                return True  # Solution found!
                            
                            board[row][col] = '.'  # BACKTRACK
                    
                    return False  # No valid digit found
        
        return True  # All cells filled successfully
    
    dfs()  # Modifies board in-place
```

**F. Memory Cue**
> *"Pencil and eraser: Write a number (DFS), if wrong erase it (backtrack), try next number."*

---

### **Summary in 3 Sentences**

DFS explores deeply before backtracking, using recursion or a stack to maintain state. It's ideal for path finding, connected components, topological sorting, and backtracking problems. Remember: "Go deep first, backtrack when stuck, mark visited to avoid cycles."

---

## 🌊 Pattern 12: BFS (Breadth-First Search)

> **Definition:** A traversal algorithm that explores all neighbors at the current depth before moving to nodes at the next depth level, using a queue.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Explore level-by-level, like ripples in water
- 💭 **Visual Intuition:** Like spreading paint outward from a center point
- 🔢 **Mathematical Reasoning:** Uses queue (FIFO) to process nodes in discovery order
- ⚡ **When to Use:** Shortest path (unweighted), level-order traversal, minimum steps

---

### **ASCII Diagram**

```
Tree BFS Traversal:
      1
     / \
    2   3
   / \   \
  4   5   6

Level-by-level:
Level 0: 1
Level 1: 2, 3
Level 2: 4, 5, 6

Queue Evolution:
[1]           → process 1, add children
[2, 3]        → process 2, add children
[3, 4, 5]     → process 3, add children
[4, 5, 6]     → process 4
[5, 6]        → process 5
[6]           → process 6
[]            → done

Grid BFS (shortest path):
S . . . E
. # # . .
. . . . .

BFS ripple:
S . . . E
1 . . . .
2 . . . .
3 . . . .

S=start (distance 0)
Numbers show distance from S
E found at distance 4
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Ripple Effect"** - Stone dropped in pond, circles expand outward |
| 📖 Story Method | "The explorer who maps every room on one floor before going downstairs" |
| 💥 Exaggeration | Imagine water flooding a building floor-by-floor |
| 🔗 Association | **BFS = Breadth First = Broad First = Wide First** |

**Memory Cue:**
> *"BFS is like a concert crowd wave - spreads outward level by level."*

---

### **Key Variations**

1. **Standard BFS** - Level-by-level traversal with queue
2. **Multi-Source BFS** - Start from multiple points simultaneously
3. **Bidirectional BFS** - Search from both start and end
4. **0-1 BFS** - Modified for graphs with 0 and 1 edge weights

---

### **Socratic Teaching Round**

**Q1:** Why is BFS guaranteed to find the shortest path in unweighted graphs?
> **A1:** BFS explores nodes in increasing order of distance from source. First time you reach a node is via the shortest path.

**Q2:** When would BFS use more memory than DFS?
> **A2:** In wide trees/graphs. BFS queue holds all nodes at current level (can be huge). DFS stack only holds path from root to current node (usually smaller).

**Q3:** How do you track levels in BFS?
> **A3:** Two methods: (1) Add level number with each node in queue, or (2) Process entire level at once using `len(queue)`.

**Q4:** Why use BFS for shortest path instead of DFS?
> **A4:** DFS might find a long path first. You'd need to explore ALL paths to ensure you found the shortest. BFS guarantees shortest on first discovery.

**Q5:** Can BFS work on weighted graphs?
> **A5:** Only if all weights are equal (unweighted). For weighted graphs, use Dijkstra's algorithm.

---

### **Problem 1 (Easy): Binary Tree Level Order Traversal**

**A. Problem Statement**
Given a binary tree, return the level order traversal of its nodes' values (i.e., from left to right, level by level).

**B. ASCII Visualization**
```
Input Tree:
    3
   / \
  9  20
    /  \
   15   7

BFS Process:
Queue: [3]           → Output: [[3]]
Queue: [9, 20]       → Output: [[3], [9, 20]]
Queue: [15, 7]       → Output: [[3], [9, 20], [15, 7]]
Queue: []            → Done

Visual Levels:
Level 0:    3
Level 1:   9  20
Level 2:  15  7

Result: [[3], [9, 20], [15, 7]]
```

**C. Pattern Recognition Question**
*What keyword signals BFS?*
> "Level by level" or "layer by layer" - BFS naturally processes nodes level-wise.

**D. Step-by-Step Reasoning**
1. Initialize queue with root node
2. For each level: process all nodes currently in queue
3. Track level size before processing (`len(queue)`)
4. For each node: add value to current level, add children to queue
5. Add completed level to result

**E. Python Solution**
```python
from collections import deque

def levelOrder(root):
    if not root:
        return []
    
    result = []
    queue = deque([root])  # Start with root
    
    while queue:
        level = []
        level_size = len(queue)  # Number of nodes at current level
        
        # Process all nodes at current level
        for _ in range(level_size):
            node = queue.popleft()  # FIFO - oldest node first
            level.append(node.val)
            
            # Add children for next level (left to right)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        
        result.append(level)
    
    return result
```

**F. Memory Cue**
> *"Queue = line at grocery store. Process everyone in current line before new arrivals."*

---

### **Problem 2 (Hard): Word Ladder**

**A. Problem Statement**
Given two words (beginWord and endWord), and a dictionary's word list, find the length of shortest transformation sequence from beginWord to endWord, such that: only one letter can be changed at a time, and each transformed word must exist in the word list.

**B. ASCII Visualization**
```
beginWord = "hit", endWord = "cog"
wordList = ["hot","dot","dog","lot","log","cog"]

BFS Transformation Graph:
        hit
         ↓ (change i→o)
        hot
       /   \
   (h→d)   (t→l)
     ↓       ↓
    dot     lot
     ↓       ↓
   (t→g)  (t→g)
     ↓       ↓
    dog     log
      \     /
       \   /  (o→o, d/l→c)
        ↓ ↓
        cog

BFS Level-by-Level:
Level 0: hit (distance 0)
Level 1: hot (distance 1)
Level 2: dot, lot (distance 2)
Level 3: dog, log (distance 3)
Level 4: cog (distance 4)

Shortest path: hit → hot → dot → dog → cog = 5 words
```

**C. Pattern Recognition Question**
*Why is this a BFS problem?*
> We need the SHORTEST transformation sequence. BFS guarantees shortest path in unweighted graphs (each transformation = 1 step).

**D. Step-by-Step Reasoning**
1. Start BFS from beginWord
2. For each word, generate all possible 1-letter changes
3. If change exists in wordList and unvisited → add to queue
4. Track distance/level for each word
5. When endWord found → return distance
6. If queue empty without finding endWord → return 0 (impossible)

**E. Python Solution**
```python
from collections import deque

def ladderLength(beginWord, endWord, wordList):
    if endWord not in wordList:
        return 0
    
    wordList = set(wordList)  # O(1) lookup
    queue = deque([(beginWord, 1)])  # (word, distance)
    visited = {beginWord}
    
    while queue:
        word, distance = queue.popleft()
        
        # Try changing each letter
        for i in range(len(word)):
            # Try all 26 letters
            for c in 'abcdefghijklmnopqrstuvwxyz':
                next_word = word[:i] + c + word[i+1:]
                
                if next_word == endWord:  # Found target!
                    return distance + 1
                
                if next_word in wordList and next_word not in visited:
                    visited.add(next_word)
                    queue.append((next_word, distance + 1))
    
    return 0  # No transformation found
```

**F. Memory Cue**
> *"BFS = Best First Search for shortest paths. Like GPS finding quickest route."*

---

### **Problem 3 (Very Hard): Shortest Path in Binary Matrix**

**A. Problem Statement**
Given an n x n binary matrix grid, return the length of the shortest clear path in the matrix. If there is no clear path, return -1. A clear path is from top-left to bottom-right through cells with value 0, and you can move in 8 directions.

**B. ASCII Visualization**
```
Grid (0=clear, 1=blocked):
0 0 0
1 1 0
1 1 0

BFS Ripple (numbers = distance):
1 2 3
X X 4
X X 5

Start: (0,0) distance=1
End: (2,2) distance=5

8-Directional Movement from (r,c):
(-1,-1) (-1,0) (-1,1)
( 0,-1)  (r,c) ( 0,1)
( 1,-1) ( 1,0) ( 1,1)

Larger Example:
0 1 1 0 0
0 0 0 0 1
1 0 1 0 0
0 0 0 1 0
0 0 0 0 0

BFS finds shortest:
1 X X 4 5
2 3 4 5 X
X 4 X 6 7
5 5 6 X 8
6 6 7 8 9
```

**C. Pattern Recognition Question**
*Why multi-source BFS wouldn't help here?*
> Single source (top-left) and single destination (bottom-right). Multi-source BFS is for problems starting from multiple points simultaneously.

**D. Step-by-Step Reasoning**
1. Check if start (0,0) or end (n-1,n-1) is blocked → return -1
2. Initialize queue with start position and distance 1
3. BFS explores all 8 directions from each cell
4. Mark visited cells (change to 1) to avoid revisiting
5. When reaching bottom-right → return distance
6. If queue empties → return -1 (no path)

**E. Python Solution**
```python
from collections import deque

def shortestPathBinaryMatrix(grid):
    n = len(grid)
    if grid[0][0] == 1 or grid[n-1][n-1] == 1:
        return -1
    
    # 8 directions: up, down, left, right, and 4 diagonals
    directions = [(-1,-1),(-1,0),(-1,1),(0,-1),(0,1),(1,-1),(1,0),(1,1)]
    
    queue = deque([(0, 0, 1)])  # (row, col, distance)
    grid[0][0] = 1  # Mark as visited
    
    while queue:
        r, c, dist = queue.popleft()
        
        # Reached bottom-right corner
        if r == n-1 and c == n-1:
            return dist
        
        # Explore all 8 directions
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            
            # Check bounds and if cell is clear (0)
            if 0 <= nr < n and 0 <= nc < n and grid[nr][nc] == 0:
                grid[nr][nc] = 1  # Mark as visited
                queue.append((nr, nc, dist + 1))
    
    return -1  # No path found
```

**F. Memory Cue**
> *"BFS = waves spreading on beach, reaching all nearby sand (8 directions) before going further."*

---

### **Summary in 3 Sentences**

BFS explores level-by-level using a queue, guaranteeing shortest path in unweighted graphs. It's ideal for shortest path problems, level-order traversal, and minimum step scenarios. Remember: "Queue holds the frontier, process layer by layer like ripples in water."

---

## 🕸️ Pattern 13: Graphs

> **Definition:** Graph algorithms solve problems involving nodes (vertices) connected by edges, handling relationships, connectivity, paths, and cycles in complex networks.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Model relationships and connections between entities
- 💭 **Visual Intuition:** Like a map of cities (nodes) connected by roads (edges)
- 🔢 **Mathematical Reasoning:** Graph theory provides tools for analyzing networks
- ⚡ **When to Use:** Social networks, dependencies, pathfinding, connectivity problems

---

### **ASCII Diagram**

```
Basic Graph Types:

Undirected Graph:        Directed Graph:
    A --- B                 A --> B
    |     |                 ↓     ↓
    C --- D                 C <-- D

Weighted Graph:          Cyclic Graph:
    A -5- B                 A → B
    |     |                 ↓   ↑
    3     2                 C → D
    |     |                 ↑___↓
    C -4- D

Adjacency List Representation:
Graph: A-B, A-C, B-D
A: [B, C]
B: [A, D]
C: [A]
D: [B]

Adjacency Matrix:
  A B C D
A[0 1 1 0]
B[1 0 0 1]
C[1 0 0 0]
D[0 1 0 0]
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Social Network"** - People (nodes) connected by friendships (edges) |
| 📖 Story Method | "Cities connected by roads - some one-way, some two-way, some with tolls" |
| 💥 Exaggeration | Imagine Facebook's ENTIRE network as one giant web |
| 🔗 Association | **Graph = Network = Web = Connections** |

**Memory Cue:**
> *"Graphs are like your brain - neurons (nodes) connected by synapses (edges)."*

---

### **Key Variations**

1. **Directed vs Undirected** - One-way vs two-way connections
2. **Weighted vs Unweighted** - With or without edge costs
3. **Cyclic vs Acyclic** - Contains cycles or not (DAG)
4. **Connected vs Disconnected** - All nodes reachable or not
5. **Dense vs Sparse** - Many or few edges relative to nodes

---

### **Socratic Teaching Round**

**Q1:** When should you use adjacency list vs adjacency matrix?
> **A1:** Adjacency list for sparse graphs (few edges) - saves space. Matrix for dense graphs or when you need O(1) edge lookup.

**Q2:** How do you detect if a graph has a cycle?
> **A2:** In undirected: DFS with parent tracking (if you reach visited node that's not parent → cycle). In directed: DFS with "visiting" state (reach visiting node → cycle).

**Q3:** What's the difference between DFS and BFS on graphs?
> **A3:** DFS explores deep paths (good for cycle detection, topological sort). BFS explores level-by-level (good for shortest path). Both can traverse entire graph.

**Q4:** Why are graphs harder than trees?
> **A4:** Graphs can have cycles (infinite loops if not careful), multiple paths between nodes, and disconnected components. Trees are special simple graphs (no cycles, one path between nodes).

**Q5:** What's a DAG and why is it important?
> **A5:** Directed Acyclic Graph - no cycles. Important for: task scheduling (topological sort), dependencies, expression evaluation. Can be processed in linear order.

---

### **Problem 1 (Easy): Find if Path Exists in Graph**

**A. Problem Statement**
Given n nodes (0 to n-1) and a list of bidirectional edges, determine if there's a valid path from source to destination.

**B. ASCII Visualization**
```
n = 6, edges = [[0,1],[0,2],[3,5],[5,4],[4,3]]
source = 0, destination = 5

Graph:
0---1    3---5
|        |   |
2        +---4

Components:
Component 1: {0, 1, 2}
Component 2: {3, 4, 5}

Path from 0 to 5? NO (different components)

Example with path:
edges = [[0,1],[1,2],[2,3],[3,5]]
0---1---2---3---5
Path exists: 0→1→2→3→5
```

**C. Pattern Recognition Question**
*How do you know this is a graph connectivity problem?*
> Keywords: "path exists", "nodes", "edges" - classic graph traversal to check if two nodes are connected.

**D. Step-by-Step Reasoning**
1. Build adjacency list from edges
2. Use BFS/DFS starting from source
3. Mark visited nodes to avoid infinite loops
4. If we reach destination → return True
5. If traversal completes without reaching destination → return False

**E. Python Solution**
```python
from collections import deque, defaultdict

def validPath(n, edges, source, destination):
    # Edge case: source is destination
    if source == destination:
        return True
    
    # Build adjacency list (undirected graph)
    graph = defaultdict(list)
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)  # Bidirectional
    
    # BFS to find path
    queue = deque([source])
    visited = {source}
    
    while queue:
        node = queue.popleft()
        
        # Check all neighbors
        for neighbor in graph[node]:
            if neighbor == destination:
                return True  # Path found!
            
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    
    return False  # No path exists
```

**F. Memory Cue**
> *"BFS like sending a scout - if scout reaches destination, path exists."*

---

### **Problem 2 (Hard): Clone Graph**

**A. Problem Statement**
Given a reference of a node in a connected undirected graph, return a deep copy (clone) of the graph. Each node contains a value and a list of neighbors.

**B. ASCII Visualization**
```
Original Graph:
    1 --- 2
    |     |
    4 --- 3

Cloning Process (BFS with HashMap):
Step 1: Clone node 1
  Original: 1
  Clone:    1' (neighbors empty)
  Map: {1 → 1'}

Step 2: Process neighbors of 1 (nodes 2, 4)
  Clone 2: Map: {1 → 1', 2 → 2'}
  Clone 4: Map: {1 → 1', 2 → 2', 4 → 4'}
  Connect: 1'.neighbors = [2', 4']

Step 3: Process neighbors of 2 (nodes 1, 3)
  1 already cloned (in map)
  Clone 3: Map: {1 → 1', 2 → 2', 4 → 4', 3 → 3'}
  Connect: 2'.neighbors = [1', 3']

Step 4: Process remaining connections
  3'.neighbors = [2', 4']
  4'.neighbors = [1', 3']

Result: Cloned Graph
    1'--- 2'
    |     |
    4'--- 3'
```

**C. Pattern Recognition Question**
*Why use a HashMap for cloning?*
> HashMap maps original nodes → cloned nodes. Prevents duplicate clones and allows connecting neighbors correctly.

**D. Step-by-Step Reasoning**
1. Use BFS/DFS to traverse original graph
2. For each node: create clone and store in HashMap
3. When processing neighbors: check if already cloned (in HashMap)
4. If not cloned: create clone, add to HashMap and queue
5. Connect cloned node's neighbors using HashMap lookups

**E. Python Solution**
```python
from collections import deque

class Node:
    def __init__(self, val=0, neighbors=None):
        self.val = val
        self.neighbors = neighbors if neighbors is not None else []

def cloneGraph(node):
    if not node:
        return None
    
    # Map: original node → cloned node
    clones = {}
    
    # Clone the starting node
    clones[node] = Node(node.val)
    
    # BFS to traverse and clone entire graph
    queue = deque([node])
    
    while queue:
        current = queue.popleft()
        
        # Process all neighbors
        for neighbor in current.neighbors:
            if neighbor not in clones:
                # First time seeing this node - clone it
                clones[neighbor] = Node(neighbor.val)
                queue.append(neighbor)
            
            # Connect current's clone to neighbor's clone
            clones[current].neighbors.append(clones[neighbor])
    
    return clones[node]  # Return cloned starting node
```

**F. Memory Cue**
> *"HashMap is your phonebook - translates original person to their clone twin."*

---

### **Problem 3 (Very Hard): Alien Dictionary (Topological Sort)**

**A. Problem Statement**
Given a sorted dictionary of an alien language, derive the order of characters in the alien alphabet. If no valid order exists, return "".

**B. ASCII Visualization**
```
Input: ["wrt","wrf","er","ett","rftt"]

Compare adjacent words to find character order:
"wrt" vs "wrf" → t comes before f (t < f)
"wrf" vs "er"  → w comes before e (w < e)
"er" vs "ett"  → r comes before t (r < t)
"ett" vs "rftt" → e comes before r (e < r)

Build Directed Graph (edges = "before" relationships):
w → e → r → t → f
    ↓_____↑

Adjacency List:
w: [e]
e: [r]
r: [t]
t: [f]

Topological Sort (DFS post-order):
DFS(w) → DFS(e) → DFS(r) → DFS(t) → DFS(f)
Post-order: f, t, r, e, w
Reverse: "wertf" ✓

Cycle Example (invalid):
["abc", "bca", "cab"]
a → b → c → a (cycle!)
No valid ordering → return ""
```

**C. Pattern Recognition Question**
*How is this a graph problem?*
> Character ordering creates directed edges (dependencies). Finding valid order = topological sort of the character graph.

**D. Step-by-Step Reasoning**
1. Build graph by comparing adjacent words character-by-character
2. First differing character creates an edge (char1 → char2)
3. Use topological sort (DFS with post-order) to find character order
4. Detect cycles during DFS → invalid dictionary → return ""
5. Return reversed post-order as final ordering

**E. Python Solution**
```python
from collections import defaultdict, deque

def alienOrder(words):
    # Initialize graph with all unique characters
    graph = {c: set() for word in words for c in word}
    
    # Build graph by comparing adjacent words
    for i in range(len(words) - 1):
        word1, word2 = words[i], words[i + 1]
        min_len = min(len(word1), len(word2))
        
        # Check for invalid case: prefix comes after full word
        if len(word1) > len(word2) and word1[:min_len] == word2[:min_len]:
            return ""  # Invalid: "abc" before "ab"
        
        # Find first differing character
        for j in range(min_len):
            if word1[j] != word2[j]:
                graph[word1[j]].add(word2[j])  # word1[j] < word2[j]
                break
    
    # Topological sort using DFS
    # States: 0=unvisited, 1=visiting, 2=visited
    state = {c: 0 for c in graph}
    result = []
    
    def dfs(char):
        if state[char] == 1:  # Cycle detected
            return False
        if state[char] == 2:  # Already processed
            return True
        
        state[char] = 1  # Mark as visiting
        
        for neighbor in graph[char]:
            if not dfs(neighbor):
                return False
        
        state[char] = 2  # Mark as visited
        result.append(char)  # Add in post-order
        return True
    
    # Run DFS on all characters
    for char in graph:
        if not dfs(char):
            return ""  # Cycle found
    
    return "".join(reversed(result))  # Reverse post-order
```

**F. Memory Cue**
> *"Alien dictionary is like course prerequisites - some letters must come before others."*

---

### **Summary in 3 Sentences**

Graphs model relationships between entities using nodes and edges, requiring traversal algorithms like DFS and BFS. Common problems include path finding, connectivity, cloning, and topological sorting (ordering with dependencies). Remember: "Build adjacency list, track visited nodes, watch for cycles."

---

## 🔢 Pattern 14: Dynamic Programming (1D)

> **Definition:** Dynamic Programming (1D) solves optimization problems by breaking them into overlapping subproblems, storing results in a 1D array to avoid redundant calculations.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Optimal solution built from optimal solutions of smaller subproblems
- 💭 **Visual Intuition:** Like climbing stairs - each step builds on previous steps
- 🔢 **Mathematical Reasoning:** Recurrence relation: dp[i] = f(dp[i-1], dp[i-2], ...)
- ⚡ **When to Use:** Optimization (min/max), counting ways, decision problems with choices

---

### **ASCII Diagram**

```
Fibonacci Example (Classic DP):
F(n) = F(n-1) + F(n-2)

Without DP (Exponential Time):
              F(5)
           /        \
        F(4)         F(3)
       /    \       /    \
    F(3)   F(2)  F(2)   F(1)
    / \     / \   / \
  F(2) F(1)...   ... (many repeated calculations!)

With DP (Linear Time):
dp[0] dp[1] dp[2] dp[3] dp[4] dp[5]
  0     1     1     2     3     5
  ↑     ↑     ↑     ↑     ↑     ↑
base  base  sum   sum   sum   sum

Each value computed once, stored, reused

Climbing Stairs (1 or 2 steps):
dp[i] = ways to reach step i

Step:  0  1  2  3  4  5
Ways:  1  1  2  3  5  8
       ↑  ↑  └──┘
     base +  dp[i] = dp[i-1] + dp[i-2]
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Building Blocks"** - Each block (state) built from previous blocks |
| 📖 Story Method | "Climbing stairs: count ways to reach each step by summing previous steps" |
| 💥 Exaggeration | Imagine a GIANT ladder where you remember how many ways to reach each rung |
| 🔗 Association | **DP = Don't rePeat = Dynamic Programming** |

**Memory Cue:**
> *"DP is like a recipe book - save results (ingredients) so you don't recalculate (cook again)."*

---

### **Key Variations**

1. **Bottom-Up (Tabulation)** - Build table from base cases upward
2. **Top-Down (Memoization)** - Recursion with caching
3. **Space Optimized** - Only store last few states instead of entire array
4. **Decision DP** - Choose between options (take/skip item)

---

### **Socratic Teaching Round**

**Q1:** What makes a problem suitable for DP?
> **A1:** Two properties: (1) Optimal substructure - optimal solution contains optimal solutions to subproblems, (2) Overlapping subproblems - same subproblems solved multiple times.

**Q2:** What's the difference between greedy and DP?
> **A2:** Greedy makes locally optimal choice at each step (may not be globally optimal). DP explores all choices and picks globally optimal solution.

**Q3:** How do you convert recursion to DP?
> **A3:** (1) Identify recurrence relation, (2) Identify base cases, (3) Create DP array, (4) Fill array bottom-up, or add memoization to recursion.

**Q4:** When should you use bottom-up vs top-down DP?
> **A4:** Bottom-up (tabulation) is faster, uses iteration, clear space complexity. Top-down (memoization) is more intuitive, recursive, only computes needed states.

**Q5:** How do you optimize space in 1D DP?
> **A5:** If dp[i] only depends on previous k states, keep only those k values (rolling array). Example: Fibonacci only needs last 2 values.

---

### **Problem 1 (Easy): Climbing Stairs**

**A. Problem Statement**
You are climbing a staircase with n steps. You can climb 1 or 2 steps at a time. How many distinct ways can you climb to the top?

**B. ASCII Visualization**
```
n = 5 stairs

Ways to reach each step:
Step 0: 1 way (start)
Step 1: 1 way (one 1-step)
Step 2: 2 ways (1+1 or 2)
Step 3: 3 ways (1+1+1, 1+2, 2+1)
Step 4: 5 ways
Step 5: 8 ways

Visual:
      ╔═5═╗  8 ways
    ╔═4═╝    5 ways
  ╔═3═╝      3 ways
╔═2═╝        2 ways
1            1 way
START        1 way

Recurrence: dp[i] = dp[i-1] + dp[i-2]
Why? To reach step i, you either:
  - Come from step i-1 (take 1 step)
  - Come from step i-2 (take 2 steps)

dp array:
i:   0  1  2  3  4  5
dp:  1  1  2  3  5  8
```

**C. Pattern Recognition Question**
*How do you recognize this as DP?*
> "Count ways" + "choices at each step" + "optimal substructure" → DP pattern.

**D. Step-by-Step Reasoning**
1. Define dp[i] = ways to reach step i
2. Base cases: dp[0] = 1, dp[1] = 1
3. Recurrence: dp[i] = dp[i-1] + dp[i-2]
4. Fill array from left to right
5. Return dp[n]

**E. Python Solution**
```python
def climbStairs(n):
    # Base cases
    if n <= 2:
        return n
    
    # DP array: dp[i] = ways to reach step i
    dp = [0] * (n + 1)
    dp[0] = 1  # One way to stay at start
    dp[1] = 1  # One way to reach step 1
    
    # Fill DP table
    for i in range(2, n + 1):
        # Ways to reach i = ways from i-1 + ways from i-2
        dp[i] = dp[i-1] + dp[i-2]
    
    return dp[n]

# Space-Optimized Version (O(1) space):
def climbStairs_optimized(n):
    if n <= 2:
        return n
    
    # Only need last 2 values
    prev2 = 1  # dp[i-2]
    prev1 = 1  # dp[i-1]
    
    for i in range(2, n + 1):
        current = prev1 + prev2
        prev2 = prev1
        prev1 = current
    
    return prev1
```

**F. Memory Cue**
> *"Stairs = Fibonacci. Each step is sum of previous two ways."*

---

### **Problem 2 (Hard): House Robber**

**A. Problem Statement**
You are a robber planning to rob houses along a street. Each house has a certain amount of money. Adjacent houses have security systems connected - you cannot rob two adjacent houses. Return maximum amount you can rob.

**B. ASCII Visualization**
```
houses = [2, 7, 9, 3, 1]

Decision at each house:
House 0: Rob 2 (no choice)
House 1: Rob 7 OR skip and keep 2? → Take 7
House 2: Rob 9 + house 0 (2) = 11 OR skip and keep 7? → Take 11
House 3: Rob 3 + house 1 (7) = 10 OR skip and keep 11? → Keep 11
House 4: Rob 1 + house 2 (11) = 12 OR skip and keep 11? → Take 12

DP Table:
i:    0   1   2   3   4
$:    2   7   9   3   1
dp:   2   7  11  11  12
      ↑   ↑   ↑
     rob rob rob+dp[i-2] vs dp[i-1]

Recurrence:
dp[i] = max(
  rob house i + dp[i-2],  // Rob this, skip previous
  dp[i-1]                  // Skip this, keep previous max
)

Visual Decision Tree:
         [2,7,9,3,1]
        /          \
   Rob 2          Skip 2
   [7,9,3,1]      [7,9,3,1]
     /  \            / \
  Rob 7 Skip 7   Rob 7 Skip 7
   ...    ...      ...   ...
```

**C. Pattern Recognition Question**
*Why can't we use greedy (always rob highest value)?*
> Greedy might take high value but miss even higher combination. Example: [2,1,1,2] - greedy takes 2s (total 4), optimal takes 1s (total 2). Wait, greedy works here! Better example: [5,1,1,5] - greedy takes 5+5=10 but can't due to adjacency. Need DP to explore all valid combinations.

**D. Step-by-Step Reasoning**
1. Define dp[i] = max money robbing houses 0 to i
2. Base cases: dp[0] = nums[0], dp[1] = max(nums[0], nums[1])
3. Recurrence: dp[i] = max(nums[i] + dp[i-2], dp[i-1])
4. At each house: choose to rob (add to dp[i-2]) or skip (keep dp[i-1])
5. Return dp[n-1]

**E. Python Solution**
```python
def rob(nums):
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]
    
    n = len(nums)
    dp = [0] * n
    
    # Base cases
    dp[0] = nums[0]
    dp[1] = max(nums[0], nums[1])
    
    # Fill DP table
    for i in range(2, n):
        # Either rob current house + max from i-2
        # Or skip current house and keep max from i-1
        dp[i] = max(nums[i] + dp[i-2], dp[i-1])
    
    return dp[n-1]

# Space-Optimized O(1):
def rob_optimized(nums):
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]
    
    prev2 = nums[0]  # dp[i-2]
    prev1 = max(nums[0], nums[1])  # dp[i-1]
    
    for i in range(2, len(nums)):
        current = max(nums[i] + prev2, prev1)
        prev2 = prev1
        prev1 = current
    
    return prev1
```

**F. Memory Cue**
> *"Robber's dilemma: Take current + two-houses-ago, OR skip current keep previous max."*

---

### **Problem 3 (Very Hard): Longest Increasing Subsequence**

**A. Problem Statement**
Given an integer array nums, return the length of the longest strictly increasing subsequence.

**B. ASCII Visualization**
```
nums = [10, 9, 2, 5, 3, 7, 101, 18]

LIS ending at each position:
10: [10] length=1
9:  [9] length=1 (can't extend 10)
2:  [2] length=1
5:  [2,5] length=2
3:  [2,3] length=2
7:  [2,5,7] or [2,3,7] length=3
101:[2,5,7,101] or [2,3,7,101] length=4
18: [2,5,7,18] or [2,3,7,18] length=4

DP Table:
i:   0  1  2  3  4  5  6   7
num: 10 9  2  5  3  7  101 18
dp:  1  1  1  2  2  3  4   4
     ↑           ↑     ↑
   base     2<5,so +1  max

For each i, check all j < i:
  if nums[j] < nums[i]:
    dp[i] = max(dp[i], dp[j] + 1)

Visual of building LIS:
[10]→[9]→[2]→[2,5]→[2,3]→[2,5,7]→[2,5,7,101]
                         →[2,3,7]→[2,3,7,101]
                                 →[2,5,7,18]
```

**C. Pattern Recognition Question**
*Why is this DP and not greedy?*
> Greedy (always take next bigger) fails. Example: [1,3,2,4] - greedy takes [1,3] then stuck, optimal is [1,2,4] or [1,3,4]. DP checks all valid extensions.

**D. Step-by-Step Reasoning**
1. Define dp[i] = length of LIS ending at index i
2. Initialize all dp[i] = 1 (each element is LIS of length 1)
3. For each i, check all previous j < i:
   - If nums[j] < nums[i], we can extend: dp[i] = max(dp[i], dp[j] + 1)
4. Return max(dp) - longest among all ending positions

**E. Python Solution**
```python
def lengthOfLIS(nums):
    if not nums:
        return 0
    
    n = len(nums)
    # dp[i] = length of LIS ending at index i
    dp = [1] * n  # Each element is LIS of length 1
    
    # For each position i
    for i in range(1, n):
        # Check all previous positions j
        for j in range(i):
            # If nums[j] < nums[i], we can extend LIS ending at j
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    
    # Return longest LIS among all ending positions
    return max(dp)

# Optimized O(n log n) using Binary Search + Patience Sorting:
def lengthOfLIS_optimized(nums):
    import bisect
    
    # tails[i] = smallest tail element of LIS of length i+1
    tails = []
    
    for num in nums:
        # Find position to insert/replace
        pos = bisect.bisect_left(tails, num)
        
        if pos == len(tails):
            tails.append(num)  # Extend LIS
        else:
            tails[pos] = num  # Replace with smaller value
    
    return len(tails)
```

**F. Memory Cue**
> *"LIS = Look back, Include if Smaller. Check all previous, extend best valid sequence."*

---

### **Summary in 3 Sentences**

1D Dynamic Programming solves optimization problems by storing solutions to subproblems in a 1D array, avoiding redundant calculations. Key insight: dp[i] is built from previous states dp[i-1], dp[i-2], etc., following a recurrence relation. Remember: "Identify recurrence, set base cases, fill table bottom-up or memoize top-down."

---

## 🔲 Pattern 15: Dynamic Programming (2D Grid)

> **Definition:** 2D DP solves optimization problems on grids or involving two sequences, storing results in a 2D table where dp[i][j] represents the solution for state (i, j).

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Optimal solution depends on two dimensions/parameters
- 💭 **Visual Intuition:** Like filling a spreadsheet cell-by-cell based on neighbors
- 🔢 **Mathematical Reasoning:** Recurrence: dp[i][j] = f(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
- ⚡ **When to Use:** Grid paths, sequence alignment, matching problems, 2-choice decisions

---

### **ASCII Diagram**

```
Grid Path Problem (Top-left to bottom-right):
Start → → → → End
↓ ↓ ↓ ↓ ↓   ↓
↓ → → → → → ↓
↓ ↓ ↓ ↓ ↓ ↓ ↓
↓ → → → → → End

dp[i][j] = ways/min-cost to reach cell (i,j)

Example: Count paths (can only move right or down)
  0 1 2 3
0 1 1 1 1
1 1 2 3 4
2 1 3 6 10

dp[i][j] = dp[i-1][j] + dp[i][j-1]
            (from top)  (from left)

LCS (Longest Common Subsequence):
Text1: "ABCD"
Text2: "AEBD"

DP Table:
    ""  A  E  B  D
""   0  0  0  0  0
A    0  1  1  1  1
B    0  1  1  2  2
C    0  1  1  2  2
D    0  1  1  2  3

If text1[i] == text2[j]:
  dp[i][j] = dp[i-1][j-1] + 1
Else:
  dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Spreadsheet Cascade"** - Each cell computed from neighbors above/left |
| 📖 Story Method | "Robot walking on grid - count paths by adding ways from top and left" |
| 💥 Exaggeration | Imagine MASSIVE chess board where each square knows best path from all previous squares |
| 🔗 Association | **2D DP = Two Dimensions = Table = Matrix = Grid** |

**Memory Cue:**
> *"2D DP is like Sudoku - fill each cell using values from already-filled cells."*

---

### **Key Variations**

1. **Grid Path Problems** - Count paths, minimum cost, obstacles
2. **Sequence Alignment** - LCS, edit distance, string matching
3. **Knapsack 2D** - Items with two constraints (weight, volume)
4. **Matrix Chain** - Optimal parenthesization, substring problems

---

### **Socratic Teaching Round**

**Q1:** What's the difference between 1D and 2D DP?
> **A1:** 1D DP solves problems with one parameter (single sequence/choice). 2D DP solves problems with two parameters (two sequences, grid coordinates, or two-dimensional choices).

**Q2:** How do you identify if a problem needs 2D DP?
> **A2:** Look for: (1) Two sequences being compared/matched, (2) Grid/matrix structure, (3) Two independent parameters in state definition.

**Q3:** What are the typical recurrence patterns in 2D DP?
> **A3:** (1) Grid: dp[i][j] from top/left neighbors, (2) LCS-style: diagonal if match, else max of top/left, (3) Edit distance: min of three neighbors + cost.

**Q4:** How do you optimize 2D DP space?
> **A4:** If dp[i][j] only depends on previous row, use rolling array (two rows). If depends on diagonal, keep previous row + one extra value.

**Q5:** Why is bottom-up preferred for 2D DP?
> **A5:** Clearer dependency structure (fill row-by-row or column-by-column). Top-down memoization works but harder to visualize for 2D tables.

---

### **Problem 1 (Easy): Unique Paths**

**A. Problem Statement**
Robot is located at top-left of m x n grid. Robot can only move down or right. How many unique paths are there to reach bottom-right corner?

**B. ASCII Visualization**
```
3x3 Grid:
S → → E
↓ ↓ ↓ ↓
↓ → → ↓
↓ ↓ ↓ ↓
↓ → → E

DP Table (number of ways to reach each cell):
  0 1 2
0 1 1 1  (only one way: all right)
1 1 2 3  (top + left)
2 1 3 6  (top + left)

Cell (2,2): 6 ways
Paths:
1. R R D D
2. R D R D
3. R D D R
4. D R R D
5. D R D R
6. D D R R

Recurrence:
dp[0][j] = 1 (only right moves)
dp[i][0] = 1 (only down moves)
dp[i][j] = dp[i-1][j] + dp[i][j-1]
          (from top)   (from left)
```

**C. Pattern Recognition Question**
*How do you know this needs 2D DP?*
> Two parameters (row, column) define state. Each cell's solution depends on two neighbors (2D structure).

**D. Step-by-Step Reasoning**
1. Define dp[i][j] = number of paths to cell (i, j)
2. Base cases: First row and column all = 1 (only one direction)
3. Recurrence: dp[i][j] = dp[i-1][j] + dp[i][j-1]
4. Fill table row by row or column by column
5. Return dp[m-1][n-1]

**E. Python Solution**
```python
def uniquePaths(m, n):
    # Create DP table
    dp = [[0] * n for _ in range(m)]
    
    # Base cases: first row and column
    for i in range(m):
        dp[i][0] = 1  # Only one way: all down
    for j in range(n):
        dp[0][j] = 1  # Only one way: all right
    
    # Fill DP table
    for i in range(1, m):
        for j in range(1, n):
            # Paths to (i,j) = paths from top + paths from left
            dp[i][j] = dp[i-1][j] + dp[i][j-1]
    
    return dp[m-1][n-1]

# Space-Optimized O(n):
def uniquePaths_optimized(m, n):
    # Only need previous row
    dp = [1] * n  # First row all 1s
    
    for i in range(1, m):
        for j in range(1, n):
            dp[j] += dp[j-1]  # dp[j] has value from above, dp[j-1] from left
    
    return dp[n-1]
```

**F. Memory Cue**
> *"Robot adds paths from top and left - like merging rivers flowing into one pond."*

---

### **Problem 2 (Hard): Longest Common Subsequence**

**A. Problem Statement**
Given two strings text1 and text2, return the length of their longest common subsequence. A subsequence is a sequence that can be derived by deleting some or no characters without changing the order.

**B. ASCII Visualization**
```
text1 = "ABCDE"
text2 = "ACE"

LCS = "ACE" (length 3)

DP Table:
      ""  A  C  E
  "" [ 0  0  0  0]
  A  [ 0  1  1  1]
  B  [ 0  1  1  1]
  C  [ 0  1  2  2]
  D  [ 0  1  2  2]
  E  [ 0  1  2  3]

Building dp[4][3] (text1[3]='D', text2[2]='E'):
D ≠ E → max(dp[3][3], dp[4][2]) = max(2, 2) = 2

Building dp[5][3] (text1[4]='E', text2[2]='E'):
E = E → dp[4][2] + 1 = 2 + 1 = 3 ✓

Visual Matching:
text1: A B C D E
       ↓   ↓   ↓
text2: A   C   E

Recurrence:
if text1[i-1] == text2[j-1]:
    dp[i][j] = dp[i-1][j-1] + 1  (match! extend diagonal)
else:
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])  (skip one char)
```

**C. Pattern Recognition Question**
*Why compare two sequences with 2D DP?*
> Each cell (i,j) represents "LCS of first i characters of text1 and first j characters of text2". Two sequences → two dimensions.

**D. Step-by-Step Reasoning**
1. Define dp[i][j] = LCS length of text1[0:i] and text2[0:j]
2. Base cases: dp[0][j] = 0, dp[i][0] = 0 (empty string has LCS 0)
3. If characters match: extend previous diagonal result + 1
4. If characters don't match: take max of skipping either character
5. Return dp[m][n]

**E. Python Solution**
```python
def longestCommonSubsequence(text1, text2):
    m, n = len(text1), len(text2)
    
    # Create DP table with extra row/column for base case
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    # Fill DP table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                # Characters match: extend diagonal LCS
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                # Characters don't match: take max of skipping one
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    
    return dp[m][n]

# Space-Optimized O(n):
def longestCommonSubsequence_optimized(text1, text2):
    m, n = len(text1), len(text2)
    
    # Only need previous row + one diagonal value
    prev = [0] * (n + 1)
    
    for i in range(1, m + 1):
        curr = [0] * (n + 1)
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                curr[j] = prev[j-1] + 1  # Diagonal
            else:
                curr[j] = max(prev[j], curr[j-1])  # Top or left
        prev = curr
    
    return prev[n]
```

**F. Memory Cue**
> *"LCS = Look for Matching Characters. Match → diagonal+1, No match → max(top, left)."*

---

### **Problem 3 (Very Hard): Edit Distance**

**A. Problem Statement**
Given two strings word1 and word2, return the minimum number of operations required to convert word1 to word2. You can: insert, delete, or replace a character.

**B. ASCII Visualization**
```
word1 = "horse"
word2 = "ros"

DP Table (minimum operations):
      ""  r  o  s
  "" [ 0  1  2  3]  (insert r, o, s)
  h  [ 1  1  2  3]
  o  [ 2  2  1  2]
  r  [ 3  2  2  2]
  s  [ 4  3  3  2]
  e  [ 5  4  4  3]

Building dp[1][1] (word1[0]='h', word2[0]='r'):
h ≠ r → min(
    dp[0][0] + 1 = 1,  (replace h→r)
    dp[0][1] + 1 = 2,  (delete h)
    dp[1][0] + 1 = 2   (insert r)
) = 1

Building dp[2][2] (word1[1]='o', word2[1]='o'):
o = o → dp[1][1] = 1 (no operation needed!)

Path to solution:
horse → rorse (replace h→r) = 1
rorse → rose  (delete r)    = 2
rose  → ros   (delete e)    = 3

Recurrence:
if word1[i-1] == word2[j-1]:
    dp[i][j] = dp[i-1][j-1]  (no operation)
else:
    dp[i][j] = 1 + min(
        dp[i-1][j-1],  (replace)
        dp[i-1][j],    (delete from word1)
        dp[i][j-1]     (insert to word1)
    )
```

**C. Pattern Recognition Question**
*Why are there three choices at each step?*
> Three operations available: replace (use both chars), delete (skip word1 char), insert (skip word2 char). Each corresponds to a neighbor in DP table.

**D. Step-by-Step Reasoning**
1. Define dp[i][j] = min operations to convert word1[0:i] to word2[0:j]
2. Base cases: dp[i][0] = i (delete all i chars), dp[0][j] = j (insert all j chars)
3. If characters match: no operation needed, take diagonal value
4. If don't match: take min of three operations + 1
5. Return dp[m][n]

**E. Python Solution**
```python
def minDistance(word1, word2):
    m, n = len(word1), len(word2)
    
    # Create DP table
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    # Base cases
    for i in range(m + 1):
        dp[i][0] = i  # Delete all characters from word1
    for j in range(n + 1):
        dp[0][j] = j  # Insert all characters of word2
    
    # Fill DP table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                # Characters match: no operation needed
                dp[i][j] = dp[i-1][j-1]
            else:
                # Take minimum of three operations
                dp[i][j] = 1 + min(
                    dp[i-1][j-1],  # Replace
                    dp[i-1][j],    # Delete
                    dp[i][j-1]     # Insert
                )
    
    return dp[m][n]

# Space-Optimized O(n):
def minDistance_optimized(word1, word2):
    m, n = len(word1), len(word2)
    
    # Only need previous row + diagonal value
    prev = list(range(n + 1))
    
    for i in range(1, m + 1):
        curr = [i]  # First column
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                curr.append(prev[j-1])  # Diagonal
            else:
                curr.append(1 + min(prev[j-1], prev[j], curr[j-1]))
        prev = curr
    
    return prev[n]
```

**F. Memory Cue**
> *"Edit distance: Match=free, No match=pick cheapest of Replace/Delete/Insert (diagonal/top/left)."*

---

### **Summary in 3 Sentences**

2D DP solves problems involving two sequences or grid coordinates by storing solutions in a 2D table. Each cell dp[i][j] is computed from neighboring cells (top, left, diagonal) based on problem-specific recurrence. Remember: "Two dimensions = two parameters, fill table row-by-row, watch for match vs no-match patterns."

---

## 💰 Pattern 16: Greedy Strategy

> **Definition:** Greedy algorithms make locally optimal choices at each step, hoping to find a global optimum. Works when local optimum leads to global optimum.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Make the best immediate choice without looking ahead
- 💭 **Visual Intuition:** Like taking the shortest visible path without mapping entire route
- 🔢 **Mathematical Reasoning:** Works when problem has greedy-choice property and optimal substructure
- ⚡ **When to Use:** Optimization problems where local best → global best (scheduling, intervals, coins)

---

### **ASCII Diagram**

```
Greedy vs DP Comparison:

Coin Change (Greedy CAN FAIL):
Coins: [1, 3, 4], Target: 6
Greedy: Take 4 first → [4, 1, 1] = 3 coins
Optimal: [3, 3] = 2 coins (DP needed!)

Activity Selection (Greedy WORKS):
Activities (start, end):
A: |-----|
B:   |-----|
C:     |---|
D:        |-----|

Greedy: Pick earliest ending
1. Pick C (ends earliest)
2. Pick D (next earliest that doesn't overlap)
Result: [C, D] = optimal

Jump Game (Greedy):
nums = [2,3,1,1,4]
Positions: 0 1 2 3 4

Max reach from each:
0: can reach 0+2=2
1: can reach 1+3=4 ✓
2: can reach 2+1=3
3: can reach 3+1=4
4: goal

Greedy: Track farthest reachable
Step 0→1: farthest = 2
Step 1→2: farthest = max(2, 1+3) = 4 ≥ goal ✓
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Hungry Person"** - Always grab closest food without planning ahead |
| 📖 Story Method | "Greedy pirate takes biggest treasure chest first" |
| 💥 Exaggeration | Imagine someone who NEVER looks beyond immediate benefit |
| 🔗 Association | **Greedy = Immediate = Now = Local Best** |

**Memory Cue:**
> *"Greedy is like shopping on empty stomach - grab what looks best NOW."*

---

### **Key Variations**

1. **Activity Selection** - Maximize non-overlapping intervals
2. **Greedy on Sorted Data** - Sort first, then make greedy choices
3. **Two-Pointer Greedy** - Move pointers greedily based on condition
4. **Huffman Coding Style** - Build optimal structure greedily

---

### **Socratic Teaching Round**

**Q1:** When does greedy work vs when does it fail?
> **A1:** Greedy works when local optimum → global optimum (greedy-choice property). Fails when need to "sacrifice" short-term for long-term gain (like coin change with [1,3,4]).

**Q2:** How do you prove a greedy algorithm is correct?
> **A2:** Show: (1) Greedy-choice property - locally optimal choice is part of globally optimal solution, (2) Optimal substructure - optimal solution contains optimal solutions to subproblems.

**Q3:** What's the relationship between sorting and greedy?
> **A3:** Many greedy algorithms start by sorting (intervals by end time, items by value/weight ratio). Sorting reveals the "locally optimal" choices.

**Q4:** Why is greedy faster than DP?
> **A4:** Greedy makes one pass with O(n) or O(n log n) time. DP explores many states with O(n²) or higher. But greedy only works for special problems.

**Q5:** How do you recognize a greedy problem?
> **A5:** Look for: "maximize/minimize", "scheduling", "interval", "earliest/latest". Try greedy first - if it works, great! If not, use DP.

---

### **Problem 1 (Easy): Best Time to Buy and Sell Stock**

**A. Problem Statement**
You are given an array prices where prices[i] is the price of a stock on day i. Maximize profit by choosing a single day to buy and different day in future to sell. Return maximum profit.

**B. ASCII Visualization**
```
prices = [7, 1, 5, 3, 6, 4]

Price Graph:
7 |█
6 |    █
5 |  █
4 |        █
3 |    ░
2 |
1 |█
  +------------
  0 1 2 3 4 5

Greedy Strategy:
Track minimum price seen so far
Calculate profit if selling today

Day 0: min=7, profit=0
Day 1: min=1, profit=0 (1-1)
Day 2: min=1, profit=4 (5-1) ← potential
Day 3: min=1, profit=2 (3-1)
Day 4: min=1, profit=5 (6-1) ← best!
Day 5: min=1, profit=3 (4-1)

Buy at 1, sell at 6 → profit = 5
```

**C. Pattern Recognition Question**
*Why is this greedy, not DP?*
> We only need to track one piece of information (minimum so far). No need to explore multiple states or combinations.

**D. Step-by-Step Reasoning**
1. Initialize min_price = infinity, max_profit = 0
2. For each price:
   - Update max_profit if (price - min_price) is better
   - Update min_price if current price is lower
3. Return max_profit

**E. Python Solution**
```python
def maxProfit(prices):
    if not prices:
        return 0
    
    min_price = float('inf')  # Track minimum price seen
    max_profit = 0            # Track maximum profit
    
    for price in prices:
        # Update minimum price (best buy opportunity so far)
        min_price = min(min_price, price)
        
        # Calculate profit if selling today
        profit = price - min_price
        
        # Update maximum profit
        max_profit = max(max_profit, profit)
    
    return max_profit
```

**F. Memory Cue**
> *"Buy low, sell high: Track lowest price seen, calculate profit at each step."*

---

### **Problem 2 (Hard): Jump Game II**

**A. Problem Statement**
Given an array nums where nums[i] is the maximum jump length from position i, return minimum number of jumps to reach the last index. You can assume you can always reach the last index.

**B. ASCII Visualization**
```
nums = [2, 3, 1, 1, 4]
Index:  0  1  2  3  4

Jump Ranges:
From 0: can reach [1, 2]
From 1: can reach [2, 3, 4] ✓
From 2: can reach [3]
From 3: can reach [4]

Greedy BFS-like approach:
Level 0: Position 0
Level 1: Positions reachable in 1 jump: [1, 2]
Level 2: Positions reachable in 2 jumps: [3, 4] ✓

Visual:
0 → [1, 2] → [3, 4]
    jump 1     jump 2

Current window: what we can reach with current jumps
Next window: farthest we can reach with one more jump

Step-by-step:
Position: 0 1 2 3 4
Jump:     0 1 1 2 2
          ^ ^^^^ ^^^
        start  |   |
           window  end
```

**C. Pattern Recognition Question**
*Why greedy instead of BFS?*
> Greedy tracks the "frontier" of reachable positions without explicit queue. Simpler and same O(n) time.

**D. Step-by-Step Reasoning**
1. Track current jump's end and farthest reachable
2. For each position, update farthest reachable
3. When reaching end of current jump, increment jump count
4. Current jump's end becomes the farthest we could reach

**E. Python Solution**
```python
def jump(nums):
    if len(nums) <= 1:
        return 0
    
    jumps = 0
    current_end = 0     # End of current jump's range
    farthest = 0        # Farthest position reachable
    
    # Don't need to jump from last position
    for i in range(len(nums) - 1):
        # Update farthest position reachable
        farthest = max(farthest, i + nums[i])
        
        # Reached end of current jump's range
        if i == current_end:
            jumps += 1
            current_end = farthest  # New range for next jump
            
            # If we can reach the end, no need to continue
            if current_end >= len(nums) - 1:
                break
    
    return jumps
```

**F. Memory Cue**
> *"Jump game: Track your frontier. When you reach edge, take next jump to farthest point."*

---

### **Problem 3 (Very Hard): Merge Intervals**

**A. Problem Statement**
Given an array of intervals where intervals[i] = [start_i, end_i], merge all overlapping intervals and return an array of the non-overlapping intervals.

**B. ASCII Visualization**
```
intervals = [[1,3],[2,6],[8,10],[15,18]]

Timeline:
1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18
[-----]
   [--------]
                  [----]
                                          [--------]

After sorting by start:
[1,3], [2,6], [8,10], [15,18]

Merge process:
Start with [1,3]
Compare [2,6]: overlaps (2 ≤ 3) → merge to [1,6]
Compare [8,10]: no overlap (8 > 6) → add [1,6], start new [8,10]
Compare [15,18]: no overlap (15 > 10) → add [8,10], add [15,18]

Result: [[1,6],[8,10],[15,18]]

Overlap condition:
Current: [1, 6]
Next:    [2, 9]
         ^
  next.start ≤ current.end → OVERLAP

Merge: [1, max(6, 9)] = [1, 9]
```

**C. Pattern Recognition Question**
*Why sort first?*
> Sorting by start time ensures we process intervals in order. Only need to check if next interval overlaps with current (greedy).

**D. Step-by-Step Reasoning**
1. Sort intervals by start time
2. Initialize result with first interval
3. For each subsequent interval:
   - If overlaps with last in result: merge (extend end)
   - If doesn't overlap: add as new interval
4. Return merged intervals

**E. Python Solution**
```python
def merge(intervals):
    if not intervals:
        return []
    
    # Sort by start time
    intervals.sort(key=lambda x: x[0])
    
    merged = [intervals[0]]
    
    for current in intervals[1:]:
        last_merged = merged[-1]
        
        # Check if current overlaps with last merged interval
        if current[0] <= last_merged[1]:
            # Overlap: merge by extending end time
            last_merged[1] = max(last_merged[1], current[1])
        else:
            # No overlap: add as new interval
            merged.append(current)
    
    return merged
```

**F. Memory Cue**
> *"Sort intervals, walk through, merge if touching/overlapping, else start new."*

---

### **Summary in 3 Sentences**

Greedy algorithms make locally optimal choices at each step, working only when local optimum leads to global optimum. Common patterns include sorting first, tracking frontiers/boundaries, and making immediate best choices. Remember: "Fast but only works for special problems - try greedy first, fall back to DP if needed."

---

## 📊 Pattern 17: Intervals Pattern

> **Definition:** Interval problems involve managing, merging, or finding relationships between ranges [start, end]. Common operations: overlap detection, merging, insertion, intersection.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Efficiently handle time ranges, schedules, or continuous segments
- 💭 **Visual Intuition:** Like managing calendar appointments or train schedules
- 🔢 **Mathematical Reasoning:** Intervals can be sorted, compared, and merged in O(n log n) time
- ⚡ **When to Use:** Scheduling, meeting rooms, calendar problems, range operations

---

### **ASCII Diagram**

```
Basic Interval Operations:

Overlap Detection:
A: [1, 5]    ████████
B: [3, 7]       ████████
Overlap: [3, 5] ███

No Overlap:
A: [1, 3]  ████
B: [5, 7]        ████

Merging:
[1,3] [2,6] [8,10]
 ████ █████  ████
Result: [1,6] [8,10]
        ████████ ████

Insertion:
Existing: [1,3] [6,9]
New: [2,5]
Result: [1,5] [6,9]

Meeting Rooms:
[[0,30],[5,10],[15,20]]
0        10        20        30
[████████████████████████████]  Room 1
     [████]                      Room 2
          [████]                 Room 3
Minimum rooms needed: 2 (overlap at time 5-10 and 15-20)
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Train Schedule"** - Trains arriving/departing, tracks needed for overlaps |
| 📖 Story Method | "Calendar with meetings - highlight overlaps, merge adjacent" |
| 💥 Exaggeration | Imagine managing THOUSANDS of appointments, need efficient merging |
| 🔗 Association | **Intervals = Ranges = Segments = Time Slots** |

**Memory Cue:**
> *"Intervals like puzzle pieces - sort them, slide together if touching, count overlaps."*

---

### **Key Variations**

1. **Overlap Detection** - Check if two intervals overlap
2. **Merge Intervals** - Combine overlapping intervals
3. **Insert Interval** - Add new interval and merge
4. **Meeting Rooms** - Count minimum rooms/resources needed
5. **Interval Intersection** - Find overlapping parts

---

### **Socratic Teaching Round**

**Q1:** How do you check if two intervals overlap?
> **A1:** Intervals [a, b] and [c, d] overlap if: c ≤ b AND a ≤ d. Equivalently: max(a, c) ≤ min(b, d).

**Q2:** Why sort intervals before processing?
> **A2:** Sorting by start time makes it easy to check overlaps with neighbors only. Without sorting, you'd need to check all pairs (O(n²)).

**Q3:** What's the difference between "merge intervals" and "meeting rooms"?
> **A3:** Merge combines overlapping into one. Meeting rooms counts maximum simultaneous overlaps (need that many rooms/resources).

**Q4:** How do you handle intervals that touch but don't overlap [1,3] and [3,5]?
> **A4:** Depends on problem: touching may count as overlap ([1,5]) or separate ([1,3] and [3,5]). Check problem definition carefully.

**Q5:** What data structure helps with interval problems?
> **A5:** Sorting is key. For dynamic queries, use sweep line algorithm or segment trees. For simple cases, arrays with sorting suffice.

---

### **Problem 1 (Easy): Meeting Rooms**

**A. Problem Statement**
Given an array of meeting time intervals where intervals[i] = [start_i, end_i], determine if a person could attend all meetings (no overlaps).

**B. ASCII Visualization**
```
Example 1: [[0,30],[5,10],[15,20]]
Timeline:
0        5    10   15   20        30
[████████████████████████████████]
     [████]                          OVERLAP!
          [████]                     OVERLAP!

Cannot attend all meetings

Example 2: [[7,10],[2,4]]
Timeline:
0   2  4    7  10
    [█]     [█]
No overlaps → Can attend all

Sorting helps:
[[0,30],[5,10],[15,20]]
→ Sort: [[0,30],[5,10],[15,20]]
Check: 5 < 30 → overlap found
```

**C. Pattern Recognition Question**
*Why sort by start time?*
> After sorting, only need to check if next meeting starts before current ends. No need to check all pairs.

**D. Step-by-Step Reasoning**
1. Sort intervals by start time
2. For each pair of consecutive intervals:
   - If next start < current end → overlap → return false
3. If no overlaps found → return true

**E. Python Solution**
```python
def canAttendMeetings(intervals):
    if not intervals:
        return True
    
    # Sort by start time
    intervals.sort(key=lambda x: x[0])
    
    # Check consecutive pairs for overlap
    for i in range(1, len(intervals)):
        prev_end = intervals[i-1][1]
        curr_start = intervals[i][0]
        
        if curr_start < prev_end:  # Overlap detected
            return False
    
    return True
```

**F. Memory Cue**
> *"Sort meetings chronologically, check if any starts before previous ends."*

---

### **Problem 2 (Hard): Meeting Rooms II**

**A. Problem Statement**
Given an array of meeting time intervals, return the minimum number of conference rooms required.

**B. ASCII Visualization**
```
intervals = [[0,30],[5,10],[15,20]]

Timeline with rooms:
0    5    10   15   20        30
[█████████████████████████████]  Room 1
     [███]                        Room 2  
          [███]                   Room 2 (reused)

At time 5: 2 meetings overlap → need 2 rooms
At time 15: 2 meetings overlap → still 2 rooms
Maximum overlap: 2 rooms needed

Sweep Line Approach:
Events:  0s 5s 10e 15s 20e 30e
         +1 +1 -1  +1  -1  -1

Count:   1  2  1   2   1   0
         ^  ^max      

Alternative: Min Heap
Sort by start: [[0,30],[5,10],[15,20]]
Heap tracks end times of ongoing meetings

Step 0: Add 30 to heap [30]
Step 1: 5 < 30 → Add 10 to heap [10, 30]
Step 2: 15 > 10 → Remove 10, add 20 [20, 30]
Max heap size: 2
```

**C. Pattern Recognition Question**
*Why use a min heap?*
> Min heap tracks earliest ending meeting. If new meeting starts after earliest end, we can reuse that room (pop from heap).

**D. Step-by-Step Reasoning**
1. Sort intervals by start time
2. Use min heap to track end times of ongoing meetings
3. For each meeting:
   - If starts after earliest ending (heap top): reuse room (pop heap)
   - Add current meeting's end time to heap
4. Max heap size = rooms needed

**E. Python Solution**
```python
import heapq

def minMeetingRooms(intervals):
    if not intervals:
        return 0
    
    # Sort by start time
    intervals.sort(key=lambda x: x[0])
    
    # Min heap to track end times
    heap = []
    
    for interval in intervals:
        start, end = interval
        
        # If earliest ending meeting has ended, reuse room
        if heap and heap[0] <= start:
            heapq.heappop(heap)
        
        # Add current meeting's end time
        heapq.heappush(heap, end)
    
    # Heap size = rooms needed
    return len(heap)

# Alternative: Sweep Line
def minMeetingRooms_sweep(intervals):
    events = []
    for start, end in intervals:
        events.append((start, 1))   # Meeting starts (+1 room)
        events.append((end, -1))     # Meeting ends (-1 room)
    
    events.sort()  # Sort by time
    
    rooms = 0
    max_rooms = 0
    
    for time, delta in events:
        rooms += delta
        max_rooms = max(max_rooms, rooms)
    
    return max_rooms
```

**F. Memory Cue**
> *"Heap tracks active meetings. Size of heap = rooms needed simultaneously."*

---

### **Problem 3 (Very Hard): Insert Interval**

**A. Problem Statement**
Given a sorted list of non-overlapping intervals and a new interval, insert the new interval and merge if necessary. Return sorted list of non-overlapping intervals.

**B. ASCII Visualization**
```
intervals = [[1,3],[6,9]], newInterval = [2,5]

Timeline:
1  2  3  4  5  6  7  8  9
[████]                        Existing [1,3]
   [████████]                 New [2,5]
               [████████]     Existing [6,9]

Step 1: Add intervals before new interval
[1,3] starts before 2, but overlaps
Skip adding separately

Step 2: Merge overlapping intervals
[1,3] overlaps with [2,5] → merge to [1,5]
[6,9] doesn't overlap with [1,5]

Step 3: Add intervals after
Add [6,9]

Result: [[1,5],[6,9]]

Complex Example:
intervals = [[1,2],[3,5],[6,7],[8,10],[12,16]]
newInterval = [4,8]

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16
[█]                                      Keep [1,2]
    [███]                                Merge
      [█████]                            Merge
        [███]                            Merge
          [███]                          Merge
                                [██████] Keep [12,16]

Result: [[1,2],[3,10],[12,16]]
```

**C. Pattern Recognition Question**
*Why process in three phases?*
> (1) Add non-overlapping intervals before new interval, (2) Merge all overlapping intervals, (3) Add non-overlapping intervals after.

**D. Step-by-Step Reasoning**
1. Phase 1: Add all intervals that end before new interval starts
2. Phase 2: Merge new interval with all overlapping intervals
   - Update start to min, end to max
3. Phase 3: Add remaining intervals after merged interval
4. Return result

**E. Python Solution**
```python
def insert(intervals, newInterval):
    result = []
    i = 0
    n = len(intervals)
    
    # Phase 1: Add all intervals that end before newInterval starts
    while i < n and intervals[i][1] < newInterval[0]:
        result.append(intervals[i])
        i += 1
    
    # Phase 2: Merge overlapping intervals
    # Merge all intervals that overlap with newInterval
    while i < n and intervals[i][0] <= newInterval[1]:
        # Expand newInterval to include current interval
        newInterval[0] = min(newInterval[0], intervals[i][0])
        newInterval[1] = max(newInterval[1], intervals[i][1])
        i += 1
    
    result.append(newInterval)  # Add merged interval
    
    # Phase 3: Add remaining intervals
    while i < n:
        result.append(intervals[i])
        i += 1
    
    return result
```

**F. Memory Cue**
> *"Three buckets: before (keep), overlapping (merge), after (keep)."*

---

### **Summary in 3 Sentences**

Interval problems involve sorting ranges by start time and checking for overlaps between consecutive intervals. Common techniques include merging overlapping intervals, using heaps to track active intervals, and sweep-line algorithms for counting simultaneous events. Remember: "Sort intervals, check neighbors for overlap, use heap when tracking multiple active ranges."

---

## ⛰️ Pattern 18: Heap / Priority Queue

> **Definition:** A heap is a specialized tree-based data structure that maintains the property that the parent is always greater (max heap) or smaller (min heap) than its children, enabling O(log n) insertions and O(1) access to min/max element.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Efficiently maintain and access minimum or maximum element
- 💭 **Visual Intuition:** Like a pyramid where the top is always the smallest/largest
- 🔢 **Mathematical Reasoning:** Complete binary tree with heap property, O(log n) operations
- ⚡ **When to Use:** Top K elements, median finding, scheduling, merging sorted arrays

---

### **ASCII Diagram**

```
Min Heap Structure:
        1
       / \
      3   2
     / \ / \
    7  5 4  6

Properties:
- Parent ≤ Children
- Root = minimum element
- Complete binary tree

Array Representation:
[1, 3, 2, 7, 5, 4, 6]
 0  1  2  3  4  5  6

For index i:
- Parent: (i-1)//2
- Left child: 2*i + 1
- Right child: 2*i + 2

Operations:
Insert 0:
       1                    0
      / \                  / \
     3   2    →           1   2
    / \ / \              / \ / \
   7  5 4  6 0          3  5 4  6 7

Bubble up: 0 swaps with 1, then with root

Extract Min:
       1                    2
      / \                  / \
     3   2    →           3   4
    / \ / \              / \ /
   7  5 4  6            7  5 6

Bubble down: Replace root with last, bubble down
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Mountain Peak"** - Smallest always bubbles to top (min heap) |
| 📖 Story Method | "Priority line at airport - VIPs (high priority) always at front" |
| 💥 Exaggeration | Imagine MASSIVE crowd where most important person magically floats to front |
| 🔗 Association | **Heap = Priority Queue = Sorted-ish = Top Element** |

**Memory Cue:**
> *"Heap is like foam bubbles - smallest/largest always rises to surface."*

---

### **Key Variations**

1. **Min Heap** - Parent smaller than children, get minimum in O(1)
2. **Max Heap** - Parent larger than children, get maximum in O(1)
3. **K-way Merge** - Merge multiple sorted arrays using heap
4. **Top K Elements** - Maintain heap of size K for top K items
5. **Median Finder** - Two heaps (max + min) to track median

---

### **Socratic Teaching Round**

**Q1:** Why is heap better than sorted array for priority queue?
> **A1:** Heap insertion is O(log n), sorted array is O(n). Heap is O(log n) for extract-min, sorted array is O(1) but O(n) to maintain sort.

**Q2:** How is a heap different from a BST?
> **A2:** Heap only guarantees parent-child relationship (partial order), BST guarantees all left < parent < all right (total order). Heap is complete tree, BST isn't necessarily.

**Q3:** When would you use max heap vs min heap?
> **A3:** Min heap for smallest element (Dijkstra's, merge K sorted lists). Max heap for largest element. For top K smallest, use max heap of size K.

**Q4:** Why use two heaps for finding median?
> **A4:** Max heap stores smaller half, min heap stores larger half. Median is at top of one heap. Keeps halves balanced in O(log n) per insertion.

**Q5:** What's the time complexity of building a heap from array?
> **A5:** O(n) using heapify (bottom-up). O(n log n) if inserting one by one (top-down). Heapify is more efficient.

---

### **Problem 1 (Easy): Kth Largest Element in Array**

**A. Problem Statement**
Given an integer array nums and an integer k, return the kth largest element in the array. Note that it is the kth largest element in sorted order, not the kth distinct element.

**B. ASCII Visualization**
```
nums = [3,2,1,5,6,4], k = 2

Approach 1: Sort
Sorted: [1,2,3,4,5,6]
Kth largest (2nd): 5

Approach 2: Min Heap of size k
Heap stores k largest elements
Root = kth largest

Process:
3: heap = [3]
2: heap = [2,3]           (min heap, but shows k largest)
1: heap = [1,2,3]         (size exceeds k, remove min)
   → heap = [2,3]
5: heap = [2,3,5]         (remove 2)
   → heap = [3,5]
6: heap = [3,5,6]         (remove 3)
   → heap = [5,6]
4: 4 < 5 (min), don't add

Min Heap Structure (stores 2 largest):
     5
      \
       6

Root = kth largest = 5

Why min heap of size k?
- Keeps k largest elements
- Root = smallest of k largest = kth largest
```

**C. Pattern Recognition Question**
*Why use min heap, not max heap?*
> Min heap of size k keeps k largest elements. Root is smallest of these k = kth largest. Max heap would give us largest, not kth largest.

**D. Step-by-Step Reasoning**
1. Create min heap of size k
2. Add first k elements to heap
3. For remaining elements:
   - If element > heap top (current kth largest): remove top, add element
4. Heap top = kth largest

**E. Python Solution**
```python
import heapq

def findKthLargest(nums, k):
    # Min heap of size k
    heap = []
    
    for num in nums:
        heapq.heappush(heap, num)
        
        # Keep heap size at most k
        if len(heap) > k:
            heapq.heappop(heap)  # Remove smallest
    
    # Root of min heap = kth largest
    return heap[0]

# Alternative: Max Heap (negate values)
def findKthLargest_maxheap(nums, k):
    # Python heapq is min heap, negate for max heap
    heap = [-num for num in nums]
    heapq.heapify(heap)
    
    # Pop k-1 times to get kth largest
    for _ in range(k - 1):
        heapq.heappop(heap)
    
    return -heap[0]  # Negate back

# Alternative: Quickselect O(n) average
def findKthLargest_quickselect(nums, k):
    # Convert to 0-indexed: kth largest = (n-k)th smallest
    k = len(nums) - k
    
    def quickselect(l, r):
        pivot = nums[r]
        p = l
        
        for i in range(l, r):
            if nums[i] <= pivot:
                nums[p], nums[i] = nums[i], nums[p]
                p += 1
        
        nums[p], nums[r] = nums[r], nums[p]
        
        if p > k:
            return quickselect(l, p - 1)
        elif p < k:
            return quickselect(p + 1, r)
        else:
            return nums[p]
    
    return quickselect(0, len(nums) - 1)
```

**F. Memory Cue**
> *"Min heap of size k = keeps k largest, root is kth largest (smallest of the k)."*

---

### **Problem 2 (Hard): Find Median from Data Stream**

**A. Problem Statement**
Design a data structure that supports adding integers from a data stream and finding the median of all elements so far.

**B. ASCII Visualization**
```
Stream: [5, 15, 1, 3]

Using Two Heaps:
Max Heap (left half)  |  Min Heap (right half)
Stores smaller half   |  Stores larger half

After 5:
maxHeap: [5]  |  minHeap: []
Median: 5

After 15:
maxHeap: [5]  |  minHeap: [15]
Median: (5 + 15) / 2 = 10

After 1:
maxHeap: [5, 1]  |  minHeap: [15]
Rebalance (maxHeap too large):
maxHeap: [1]  |  minHeap: [5, 15]
Median: 5 (top of minHeap, odd count)

After 3:
maxHeap: [3, 1]  |  minHeap: [5, 15]
Median: (3 + 5) / 2 = 4

Visual:
Smaller half ← [1, 3] | [5, 15] → Larger half
              MAX     MIN
              top     top
               ↓       ↓
              3       5
Median = (3 + 5) / 2 = 4

Invariants:
1. maxHeap.size() ≈ minHeap.size() (differ by at most 1)
2. All elements in maxHeap ≤ all elements in minHeap
```

**C. Pattern Recognition Question**
*Why two heaps instead of one sorted structure?*
> Two heaps give O(log n) insertion and O(1) median retrieval. Sorted array would be O(n) insertion. Two heaps partition data around median efficiently.

**D. Step-by-Step Reasoning**
1. Max heap stores smaller half (left side)
2. Min heap stores larger half (right side)
3. When adding number:
   - Add to appropriate heap based on value
   - Rebalance if heaps differ by more than 1
4. Median is:
   - Average of two tops if equal size
   - Top of larger heap if odd count

**E. Python Solution**
```python
import heapq

class MedianFinder:
    def __init__(self):
        # Max heap (negate values) for smaller half
        self.small = []
        # Min heap for larger half
        self.large = []
    
    def addNum(self, num: int) -> None:
        # Add to max heap (small half) by default
        heapq.heappush(self.small, -num)
        
        # Ensure every element in small ≤ every element in large
        if self.small and self.large and (-self.small[0] > self.large[0]):
            val = -heapq.heappop(self.small)
            heapq.heappush(self.large, val)
        
        # Balance heaps (sizes differ by at most 1)
        if len(self.small) > len(self.large) + 1:
            val = -heapq.heappop(self.small)
            heapq.heappush(self.large, val)
        
        if len(self.large) > len(self.small) + 1:
            val = heapq.heappop(self.large)
            heapq.heappush(self.small, -val)
    
    def findMedian(self) -> float:
        # Odd count: return top of larger heap
        if len(self.small) > len(self.large):
            return -self.small[0]
        
        if len(self.large) > len(self.small):
            return self.large[0]
        
        # Even count: average of both tops
        return (-self.small[0] + self.large[0]) / 2.0
```

**F. Memory Cue**
> *"Two heaps like balanced scale: max heap (left), min heap (right), median at center."*

---

### **Problem 3 (Very Hard): Merge K Sorted Lists**

**A. Problem Statement**
You are given an array of k linked-lists lists, each linked-list is sorted in ascending order. Merge all the linked-lists into one sorted linked-list and return it.

**B. ASCII Visualization**
```
Input:
List 1: 1 → 4 → 5
List 2: 1 → 3 → 4
List 3: 2 → 6

Min Heap Approach:
Step 1: Add first node of each list to heap
Heap: [1(L1), 1(L2), 2(L3)]
Min = 1(L1)

Step 2: Remove min, add next from that list
Result: 1 →
Heap: [1(L2), 2(L3), 4(L1)]
Min = 1(L2)

Step 3: Continue...
Result: 1 → 1 →
Heap: [2(L3), 3(L2), 4(L1)]
Min = 2(L3)

Continue until heap empty:
Result: 1 → 1 → 2 → 3 → 4 → 4 → 5 → 6

Why Heap?
- Always get minimum among k lists in O(log k)
- Total time: O(N log k) where N = total nodes
- Better than comparing all k lists each time O(N*k)

Heap Structure:
       1(L2)
      /     \
   2(L3)   4(L1)
  /    \
3(L2)  4(L2)
```

**C. Pattern Recognition Question**
*Why not merge lists two at a time?*
> Merging two at a time: O(N) * k merges = O(Nk). Using heap: O(N log k) - much faster for large k.

**D. Step-by-Step Reasoning**
1. Add first node from each list to min heap
2. While heap not empty:
   - Remove minimum node (smallest value)
   - Add to result
   - If that node has next, add next to heap
3. Return merged list

**E. Python Solution**
```python
import heapq

class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
    
    # Make ListNode comparable for heap
    def __lt__(self, other):
        return self.val < other.val

def mergeKLists(lists):
    # Min heap to store nodes
    heap = []
    
    # Add first node of each list to heap
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, lst)
    
    # Dummy head for result
    dummy = ListNode(0)
    current = dummy
    
    # Process heap
    while heap:
        # Get minimum node
        min_node = heapq.heappop(heap)
        
        # Add to result
        current.next = min_node
        current = current.next
        
        # If this node has next, add to heap
        if min_node.next:
            heapq.heappush(heap, min_node.next)
    
    return dummy.next

# Alternative: Divide and Conquer
def mergeKLists_divideconquer(lists):
    if not lists:
        return None
    
    def merge2Lists(l1, l2):
        dummy = ListNode(0)
        current = dummy
        
        while l1 and l2:
            if l1.val < l2.val:
                current.next = l1
                l1 = l1.next
            else:
                current.next = l2
                l2 = l2.next
            current = current.next
        
        current.next = l1 or l2
        return dummy.next
    
    # Merge pairs of lists repeatedly
    while len(lists) > 1:
        merged = []
        
        for i in range(0, len(lists), 2):
            l1 = lists[i]
            l2 = lists[i + 1] if i + 1 < len(lists) else None
            merged.append(merge2Lists(l1, l2))
        
        lists = merged
    
    return lists[0]
```

**F. Memory Cue**
> *"K-way merge: heap keeps track of k frontrunners, always pick smallest."*

---

### **Summary in 3 Sentences**

Heaps maintain partial ordering where parent is always smaller (min heap) or larger (max heap) than children, enabling efficient O(log n) insertions and O(1) min/max access. Common applications include top K elements, median finding with two heaps, and merging K sorted structures. Remember: "Heap = priority queue, use min heap for smallest, max heap for largest, two heaps for median."

---

## 🔢 Pattern 19: Bit Manipulation

> **Definition:** Bit manipulation uses bitwise operators to directly manipulate bits in binary representations of numbers, enabling efficient solutions to certain problems.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Operate directly on binary representation for speed and elegance
- 💭 **Visual Intuition:** Like flipping switches on/off or reading binary signals
- 🔢 **Mathematical Reasoning:** Bitwise operations are O(1), can replace arithmetic operations
- ⚡ **When to Use:** Set operations, flags, optimization, XOR properties, power of 2 checks

---

### **ASCII Diagram**

```
Binary Representation:
Decimal: 13
Binary:  1101
Bits:    ↓↓↓↓
         8421

Bitwise Operators:
AND (&):  1101 & 1011 = 1001  (both 1)
OR  (|):  1101 | 1011 = 1111  (either 1)
XOR (^):  1101 ^ 1011 = 0110  (different)
NOT (~):  ~1101 = 0010        (flip bits)
LEFT (<<): 1101 << 1 = 11010  (multiply by 2)
RIGHT(>>): 1101 >> 1 = 110    (divide by 2)

Common Tricks:
Check if power of 2:
  n & (n-1) == 0
  Example: 8 (1000) & 7 (0111) = 0 ✓

Get lowest set bit:
  n & -n
  Example: 12 (1100) & -12 = 4 (0100)

Count set bits:
  Brian Kernighan's:
  n & (n-1) removes rightmost set bit
  12: 1100
   &  1011 (11)
   =  1000 (8)
   &  0111 (7)
   =  0000
  2 operations = 2 set bits

XOR Properties:
a ^ a = 0     (cancel out)
a ^ 0 = a     (identity)
a ^ b ^ a = b (find unique)
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Light Switches"** - Each bit is an on/off switch |
| 📖 Story Method | "Binary code like Morse code - dots and dashes (0s and 1s)" |
| 💥 Exaggeration | Imagine GIANT scoreboard with bulbs - flip patterns |
| 🔗 Association | **Bit = Binary digit = Switch = Flag** |

**Memory Cue:**
> *"Bit manipulation is like playing with LEGO blocks - combine, flip, shift pieces."*

---

### **Key Variations**

1. **XOR Tricks** - Finding unique elements, swapping without temp
2. **Bit Masking** - Setting, clearing, toggling, checking specific bits
3. **Power of 2** - Checking and manipulating powers of 2
4. **Counting Bits** - Brian Kernighan's algorithm
5. **Subset Generation** - Using bits to represent subsets

---

### **Socratic Teaching Round**

**Q1:** Why is XOR useful for finding unique elements?
> **A1:** XOR has property: a ^ a = 0 and a ^ 0 = a. All duplicates cancel out, leaving only unique element.

**Q2:** How does n & (n-1) detect power of 2?
> **A2:** Power of 2 has exactly one bit set (1000). n-1 flips all bits after that bit (0111). ANDing gives 0 only for powers of 2.

**Q3:** Why are bitwise operations faster than arithmetic?
> **A3:** Bitwise operations are single CPU instructions, directly manipulate bits. Arithmetic may need multiple cycles.

**Q4:** How do you set/clear/toggle a specific bit?
> **A4:** Set: n | (1 << i), Clear: n & ~(1 << i), Toggle: n ^ (1 << i), Check: n & (1 << i).

**Q5:** What's the relationship between left shift and multiplication?
> **A5:** Left shift by k = multiply by 2^k. Right shift by k = divide by 2^k. Much faster than multiplication/division.

---

### **Problem 1 (Easy): Single Number**

**A. Problem Statement**
Given a non-empty array of integers nums, every element appears twice except for one. Find that single one in linear time and constant space.

**B. ASCII Visualization**
```
nums = [4, 1, 2, 1, 2]

XOR Properties:
a ^ a = 0  (same numbers cancel)
a ^ 0 = a  (identity)
XOR is commutative and associative

Process:
4 ^ 1 ^ 2 ^ 1 ^ 2
= 4 ^ (1 ^ 1) ^ (2 ^ 2)  (reorder)
= 4 ^ 0 ^ 0
= 4

Binary visualization:
4:  100
1:  001
2:  010
1:  001
2:  010

Step by step:
100 (4)
001 (1)
---  XOR
101

010 (2)
---  XOR
111

001 (1)
---  XOR
110

010 (2)
---  XOR
100 (4) ✓
```

**C. Pattern Recognition Question**
*Why does XOR solve this elegantly?*
> XOR's self-canceling property: duplicates become 0, leaving only the unique element. No extra space needed.

**D. Step-by-Step Reasoning**
1. Initialize result = 0
2. XOR all numbers together
3. Duplicates cancel out (a ^ a = 0)
4. Result is the single unique number

**E. Python Solution**
```python
def singleNumber(nums):
    result = 0
    
    for num in nums:
        result ^= num  # XOR accumulation
    
    return result

# One-liner using reduce
from functools import reduce
import operator

def singleNumber_oneliner(nums):
    return reduce(operator.xor, nums, 0)
```

**F. Memory Cue**
> *"XOR is the canceler - duplicates vanish, unique survives."*

---

### **Problem 2 (Hard): Count Bits**

**A. Problem Statement**
Given an integer n, return an array ans of length n + 1 such that for each i (0 ≤ i ≤ n), ans[i] is the number of 1's in the binary representation of i.

**B. ASCII Visualization**
```
n = 5
Output: [0, 1, 1, 2, 1, 2]

Binary representations:
0: 0000 → 0 ones
1: 0001 → 1 one
2: 0010 → 1 one
3: 0011 → 2 ones
4: 0100 → 1 one
5: 0101 → 2 ones

DP Pattern:
i & (i-1) removes rightmost 1 bit

For i = 5 (0101):
i-1 = 4 (0100)
i & (i-1) = 0101 & 0100 = 0100 (4)

count[5] = count[4] + 1
         = 1 + 1 = 2 ✓

For i = 6 (0110):
i-1 = 5 (0101)
i & (i-1) = 0110 & 0101 = 0100 (4)

count[6] = count[4] + 1
         = 1 + 1 = 2 ✓

Recurrence:
count[i] = count[i & (i-1)] + 1
```

**C. Pattern Recognition Question**
*How is this DP with bit manipulation?*
> We reuse previously computed results. i & (i-1) gives a smaller number we've already processed, just add 1 for the removed bit.

**D. Step-by-Step Reasoning**
1. Initialize array of size n+1
2. Base case: count[0] = 0
3. For each i from 1 to n:
   - i & (i-1) removes rightmost set bit
   - count[i] = count[i & (i-1)] + 1
4. Return array

**E. Python Solution**
```python
def countBits(n):
    result = [0] * (n + 1)
    
    for i in range(1, n + 1):
        # Remove rightmost 1 bit, add 1
        result[i] = result[i & (i - 1)] + 1
    
    return result

# Alternative: Using i >> 1 (divide by 2)
def countBits_alt(n):
    result = [0] * (n + 1)
    
    for i in range(1, n + 1):
        # count[i] = count[i//2] + (i % 2)
        result[i] = result[i >> 1] + (i & 1)
    
    return result
```

**F. Memory Cue**
> *"Brian Kernighan: i & (i-1) removes one bit, reuse previous count + 1."*

---

### **Problem 3 (Very Hard): Maximum XOR of Two Numbers**

**A. Problem Statement**
Given an integer array nums, return the maximum result of nums[i] XOR nums[j], where 0 ≤ i ≤ j < n.

**B. ASCII Visualization**
```
nums = [3, 10, 5, 25, 2, 8]

Binary representations:
3:  00011
10: 01010
5:  00101
25: 11001
2:  00010
8:  01000

Maximum XOR pairs:
5  ^ 25 = 00101 ^ 11001 = 11100 = 28 ✓ (maximum)
3  ^ 8  = 00011 ^ 01000 = 01011 = 11
10 ^ 2  = 01010 ^ 00010 = 01000 = 8

Trie Approach:
Build prefix tree of binary representations
For each number, find path that maximizes XOR

Trie for [3, 10]:
         ROOT
        /    \
       0      1
      /        \
     0          1
    /            \
   0              0
  /                \
 1                  0
  \                  \
   1                  1

For 5 (00101), traverse opposite bits when possible:
Start at ROOT
Bit 1 (0): Have both 0 and 1, choose 1 (opposite) → 1
Bit 2 (0): At 1 node, choose 1 if exists → 1
...
Build maximum XOR greedily
```

**C. Pattern Recognition Question**
*Why use a trie instead of checking all pairs?*
> All pairs = O(n²). Trie approach = O(32n) ≈ O(n) since we process fixed 32 bits per number. Trie lets us greedily choose opposite bits for maximum XOR.

**D. Step-by-Step Reasoning**
1. Build binary trie with all numbers (32 bits each)
2. For each number, traverse trie:
   - Try to go opposite direction at each bit (maximize XOR)
   - If opposite exists, take it; else take available path
3. Track maximum XOR found
4. Return maximum

**E. Python Solution**
```python
class TrieNode:
    def __init__(self):
        self.children = {}

class Solution:
    def findMaximumXOR(self, nums):
        # Build trie with all numbers
        root = TrieNode()
        
        # Insert all numbers into trie (32 bits)
        for num in nums:
            node = root
            for i in range(31, -1, -1):  # Process from MSB to LSB
                bit = (num >> i) & 1
                if bit not in node.children:
                    node.children[bit] = TrieNode()
                node = node.children[bit]
        
        max_xor = 0
        
        # For each number, find maximum XOR
        for num in nums:
            node = root
            current_xor = 0
            
            for i in range(31, -1, -1):
                bit = (num >> i) & 1
                # Try to go opposite direction for max XOR
                opposite = 1 - bit
                
                if opposite in node.children:
                    current_xor |= (1 << i)  # Set this bit in result
                    node = node.children[opposite]
                else:
                    node = node.children[bit]
            
            max_xor = max(max_xor, current_xor)
        
        return max_xor

# Alternative: Greedy with Set (builds maximum XOR bit by bit)
def findMaximumXOR_greedy(nums):
    max_xor = 0
    mask = 0
    
    # Build XOR from MSB to LSB
    for i in range(31, -1, -1):
        mask |= (1 << i)  # Add current bit to mask
        prefixes = {num & mask for num in nums}
        
        # Try to set current bit in result
        temp = max_xor | (1 << i)
        
        # Check if we can achieve this XOR with some pair
        for prefix in prefixes:
            if temp ^ prefix in prefixes:
                max_xor = temp
                break
    
    return max_xor
```

**F. Memory Cue**
> *"Trie for XOR: opposite bits make bigger XOR, greedily choose opposite path."*

---

### **Summary in 3 Sentences**

Bit manipulation operates directly on binary representations using bitwise operators (AND, OR, XOR, shifts) for efficient solutions. Key techniques include XOR for finding unique elements, n & (n-1) for removing rightmost bit, and tries for maximizing XOR. Remember: "XOR cancels duplicates, bit masks set/clear flags, shifts multiply/divide by powers of 2."

---

## 🌲 Pattern 20: Union-Find (DSU)

> **Definition:** Union-Find (Disjoint Set Union) is a data structure that efficiently tracks and merges disjoint sets, supporting near-constant time union and find operations through path compression and union by rank.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Track connected components and efficiently merge them
- 💭 **Visual Intuition:** Like grouping people into teams and checking if two people are on same team
- 🔢 **Mathematical Reasoning:** Amortized O(α(n)) per operation where α is inverse Ackermann (practically O(1))
- ⚡ **When to Use:** Connectivity, cycle detection, network problems, Kruskal's MST

---

### **ASCII Diagram**

```
Union-Find Structure:

Initial State (5 elements):
0   1   2   3   4
↓   ↓   ↓   ↓   ↓
(each is its own parent)

After union(0, 1):
  0      2   3   4
 ↙       ↓   ↓   ↓
1

After union(2, 3):
  0      2       4
 ↙      ↙        ↓
1      3

After union(0, 2):
    0           4
   ↙ ↘          ↓
  1   2
      ↓
      3

Find(3): Follow path to root
3 → 2 → 0 (root found)

Path Compression:
After find(3), compress path:
    0           4
   ↙|↘          ↓
  1 3 2
(all point directly to root)

Union by Rank:
Keep trees shallow by attaching smaller tree under larger

Example:
Tree A (rank 2):    Tree B (rank 1):
    0                   2
   ↙ ↘                  ↓
  1   3                 4

Union(A, B): Attach B under A
        0
       ↙|↘
      1 3 2
          ↓
          4
```

---

### **Memorization Technique**

| Technique | Application |
|-----------|-------------|
| 🎬 Visualization | **"Family Trees"** - Each group has a patriarch (root), find ancestry |
| 📖 Story Method | "Social network: people form groups, find if two people are in same group" |
| 💥 Exaggeration | Imagine millions of islands, track which merge into continents |
| 🔗 Association | **Union = Merge, Find = Search Root = Check Connection** |

**Memory Cue:**
> *"Union-Find is like organizing a party - merge friend groups, check if two people know each other."*

---

### **Key Variations**

1. **Basic Union-Find** - Simple parent array
2. **Path Compression** - Flatten tree during find for speed
3. **Union by Rank** - Attach smaller tree under larger
4. **Union by Size** - Track size instead of rank
5. **With Rollback** - Support undo operations

---

### **Socratic Teaching Round**

**Q1:** Why is Union-Find better than DFS for connectivity?
> **A1:** Union-Find is O(α(n)) amortized per query, practically O(1). DFS is O(V+E) per query. For multiple connectivity queries, Union-Find is much faster.

**Q2:** What's the difference between rank and size in union?
> **A2:** Rank approximates tree height (used for union by rank). Size is exact number of elements (used for union by size). Both keep trees balanced.

**Q3:** How does path compression improve performance?
> **A3:** Makes trees flatter by pointing nodes directly to root during find. Future finds become faster. Reduces tree height from O(n) to O(log n) or better.

**Q4:** When would you use Union-Find over BFS/DFS?
> **A4:** When you have many connectivity queries or need to dynamically merge components. BFS/DFS better for one-time traversal or when you need full path.

**Q5:** How does Union-Find detect cycles in undirected graphs?
> **A5:** When adding edge (u, v), if find(u) == find(v), they're already connected → cycle exists. Otherwise, union them.

---

### **Problem 1 (Easy): Number of Provinces**

**A. Problem Statement**
There are n cities. Some are connected directly. A province is a group of directly or indirectly connected cities. Return the total number of provinces.

**B. ASCII Visualization**
```
isConnected = [
  [1,1,0],
  [1,1,0],
  [0,0,1]
]

Graph representation:
0 --- 1    2
(connected) (isolated)

Union-Find process:
Initial: parent = [0, 1, 2]
         count = 3

Check (0,1): connected
  union(0, 1)
  parent = [0, 0, 2]
  count = 2

Check (0,2): not connected
Check (1,2): not connected

Final provinces: 2

Visual:
Province 1: {0, 1}
Province 2: {2}

Example 2:
isConnected = [
  [1,0,0],
  [0,1,0],
  [0,0,1]
]

0   1   2  (no connections)
Provinces: 3
```

**C. Pattern Recognition Question**
*Why Union-Find instead of DFS?*
> Union-Find naturally counts connected components as we merge. DFS works but Union-Find is more elegant for this problem.

**D. Step-by-Step Reasoning**
1. Initialize Union-Find with n elements
2. For each connection in matrix:
   - If cities i and j are connected, union them
3. Count number of distinct roots (components)
4. Return count

**E. Python Solution**
```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.count = n  # Number of components
    
    def find(self, x):
        # Path compression: point directly to root
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x, y):
        root_x = self.find(x)
        root_y = self.find(y)
        
        if root_x == root_y:
            return  # Already in same set
        
        # Union by rank: attach smaller tree under larger
        if self.rank[root_x] < self.rank[root_y]:
            self.parent[root_x] = root_y
        elif self.rank[root_x] > self.rank[root_y]:
            self.parent[root_y] = root_x
        else:
            self.parent[root_y] = root_x
            self.rank[root_x] += 1
        
        self.count -= 1  # Merged two components

def findCircleNum(isConnected):
    n = len(isConnected)
    uf = UnionFind(n)
    
    # Process all connections
    for i in range(n):
        for j in range(i + 1, n):
            if isConnected[i][j] == 1:
                uf.union(i, j)
    
    return uf.count
```

**F. Memory Cue**
> *"Provinces = connected groups. Union cities, count remaining roots."*

---

### **Problem 2 (Hard): Accounts Merge**

**A. Problem Statement**
Given a list of accounts where each element is a list of strings, first element is name, rest are emails. Merge accounts if they share at least one email. Return merged accounts.

**B. ASCII Visualization**
```
accounts = [
  ["John","john@mail.com","john_work@mail.com"],
  ["John","john_another@mail.com"],
  ["Mary","mary@mail.com"],
  ["John","john@mail.com","john_another@mail.com"]
]

Email connections:
john@mail.com <---> john_work@mail.com (account 0)
john@mail.com <---> john_another@mail.com (account 3)

Union-Find on emails:
Initial groups:
{john@mail.com}
{john_work@mail.com}
{john_another@mail.com}
{mary@mail.com}

After union(john@mail.com, john_work@mail.com):
{john@mail.com, john_work@mail.com}
{john_another@mail.com}
{mary@mail.com}

After union(john@mail.com, john_another@mail.com):
{john@mail.com, john_work@mail.com, john_another@mail.com}
{mary@mail.com}

Result:
[
  ["John","john@mail.com","john_another@mail.com","john_work@mail.com"],
  ["Mary","mary@mail.com"]
]

Visual:
john@mail.com ---- john_work@mail.com
       |
       +---------- john_another@mail.com

(all connected → one account)
```

**C. Pattern Recognition Question**
*Why use Union-Find for email merging?*
> Emails form connected components through shared accounts. Union-Find efficiently merges these components.

**D. Step-by-Step Reasoning**
1. Map each email to its owner name
2. For each account, union all emails in it
3. Group emails by their root (component)
4. Sort emails in each group
5. Return merged accounts with name + sorted emails

**E. Python Solution**
```python
from collections import defaultdict

class UnionFind:
    def __init__(self):
        self.parent = {}
    
    def find(self, x):
        if x not in self.parent:
            self.parent[x] = x
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x, y):
        root_x = self.find(x)
        root_y = self.find(y)
        if root_x != root_y:
            self.parent[root_x] = root_y

def accountsMerge(accounts):
    uf = UnionFind()
    email_to_name = {}
    
    # Union all emails in each account
    for account in accounts:
        name = account[0]
        first_email = account[1]
        
        for email in account[1:]:
            email_to_name[email] = name
            uf.union(first_email, email)
    
    # Group emails by root
    components = defaultdict(list)
    for email in email_to_name:
        root = uf.find(email)
        components[root].append(email)
    
    # Build result
    result = []
    for emails in components.values():
        name = email_to_name[emails[0]]
        result.append([name] + sorted(emails))
    
    return result
```

**F. Memory Cue**
> *"Emails like friends: shared email connects accounts, union them all together."*

---

### **Problem 3 (Very Hard): Redundant Connection II**

**A. Problem Statement**
In a directed graph, each node has at most one parent. Given edges that form a rooted tree plus one extra edge, return the edge that should be removed to make it a valid tree.

**B. ASCII Visualization**
```
Example: edges = [[1,2],[1,3],[2,3]]

Graph:
  1
 ↙ ↘
2 → 3

Node 3 has TWO parents (1 and 2) ← Invalid!

Possible removals:
Remove [1,3]: Valid tree
     1
    ↙
   2 → 3

Remove [2,3]: Valid tree
     1
    ↙ ↘
   2   3

Answer: [2,3] (last edge that causes issue)

Cases to handle:
1. Node with two parents
2. Cycle without two-parent node

Case 1: Two parents
  1
 ↙ ↘
2   3
    ↑
    +--- 4

Node 3 has parents 1 and 4

Case 2: Cycle
1 → 2
↑   ↓
4 ← 3

No node has two parents, but cycle exists

Strategy:
1. Detect if any node has two parents
2. Use Union-Find to detect cycle
3. Return appropriate edge based on case
```

**C. Pattern Recognition Question**
*Why is this harder than undirected redundant connection?*
> Directed graph has two failure modes: node with two parents OR cycle. Need to handle both cases differently.

**D. Step-by-Step Reasoning**
1. Check if any node has two parents (two incoming edges)
2. If yes, mark those two candidate edges
3. Use Union-Find to detect cycle:
   - Skip second candidate edge temporarily
   - If still forms cycle → remove first candidate
   - If no cycle → remove second candidate
4. If no two-parent node, return edge that creates cycle

**E. Python Solution**
```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n + 1))
    
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x, y):
        root_x = self.find(x)
        root_y = self.find(y)
        if root_x == root_y:
            return False  # Cycle detected
        self.parent[root_x] = root_y
        return True

def findRedundantDirectedConnection(edges):
    n = len(edges)
    parent = [0] * (n + 1)
    candidate1 = candidate2 = None
    
    # Check for node with two parents
    for u, v in edges:
        if parent[v] != 0:
            # Node v has two parents
            candidate1 = [parent[v], v]  # First edge to v
            candidate2 = [u, v]          # Second edge to v
            break
        parent[v] = u
    
    # Use Union-Find to detect cycle
    uf = UnionFind(n)
    
    for u, v in edges:
        # Skip second candidate if exists
        if candidate2 and u == candidate2[0] and v == candidate2[1]:
            continue
        
        if not uf.union(u, v):
            # Cycle detected
            if candidate1:
                return candidate1  # First edge caused the issue
            return [u, v]  # This edge creates cycle
    
    # No cycle found, remove second candidate
    return candidate2
```

**F. Memory Cue**
> *"Directed tree: check two-parent node first, then use Union-Find for cycle."*

---

### **Summary in 3 Sentences**

Union-Find efficiently tracks and merges disjoint sets using path compression and union by rank for near-constant time operations. It excels at connectivity queries, cycle detection, and grouping elements into connected components. Remember: "Find follows path to root and compresses, union merges trees by attaching smaller under larger, count tracks number of components."

---

## 🎯 Quick Reference: Patterns 11-20

| Pattern | Key Concept | Time Complexity | When to Use |
|---------|-------------|-----------------|-------------|
| **DFS** | Explore deep first, backtrack | O(V+E) | Path finding, cycles, backtracking |
| **BFS** | Level-by-level exploration | O(V+E) | Shortest path, level order |
| **Graphs** | Nodes & edges relationships | Varies | Networks, dependencies, connections |
| **DP 1D** | Sequential optimization | O(n) | Fibonacci, stairs, house robber |
| **DP 2D** | Grid/two-sequence optimization | O(n²) | Paths, LCS, edit distance |
| **Greedy** | Locally optimal choices | O(n log n) | Intervals, scheduling, optimization |
| **Intervals** | Range management | O(n log n) | Meetings, calendars, overlaps |
| **Heap** | Maintain min/max efficiently | O(log n) | Top K, median, priority queue |
| **Bit Manipulation** | Direct bit operations | O(1) | Flags, XOR tricks, powers of 2 |
| **Union-Find** | Merge & query disjoint sets | O(α(n)) ≈ O(1) | Connectivity, grouping, MST |

---

## 🎓 Final Mastery Tips

### **Pattern Recognition Checklist**

**See these keywords?** → **Think this pattern:**
- "All paths", "cycle", "backtrack" → **DFS**
- "Shortest path", "level order", "minimum steps" → **BFS**
- "Network", "connected", "dependencies" → **Graphs**
- "Optimal", "count ways", "max/min" → **DP**
- "Locally optimal", "earliest", "greedy" → **Greedy**
- "Overlapping ranges", "schedule", "merge" → **Intervals**
- "Top K", "median", "smallest/largest" → **Heap**
- "Single number", "XOR", "bits" → **Bit Manipulation**
- "Connected components", "groups", "merge sets" → **Union-Find**

---

## 🚀 Next Steps

1. **Practice Each Pattern** - Solve all 3 problems per pattern
2. **Mix Patterns** - Many hard problems combine patterns
3. **Time Yourself** - Build speed with pattern recognition
4. **Draw Diagrams** - Always visualize before coding
5. **Teach Others** - Best way to solidify understanding

---

## 💡 Remember

> *"Patterns are not about memorizing solutions—they're about recognizing structure. Once you see the pattern, the solution flows naturally."*

---

### 🎉 Congratulations!

You've completed the **Advanced LeetCode Patterns (11-20)** mastery guide. You now have:

- ✅ 10 advanced algorithmic patterns
- ✅ 30 fully explained problems (Easy, Hard, Very Hard)
- ✅ ASCII visualizations for every concept
- ✅ Memory techniques for long-term retention
- ✅ Socratic understanding of the "why" behind patterns

**Keep practicing, keep visualizing, and keep mastering!** 🚀

---

*Master these patterns, and watch your interview success rate soar!* 💪
