# 🎯 LeetCode Patterns 1-10: Complete Visual Guide
### *Master Core Problem-Solving Patterns with ASCII, Memory Techniques & Socratic Learning*

---

## 📖 Table of Contents

- [Pattern 1: Sliding Window](#pattern-1-sliding-window)
- [Pattern 2: Two Pointers](#pattern-2-two-pointers)
- [Pattern 3: Fast-Slow Pointers](#pattern-3-fast-slow-pointers)
- [Pattern 4: Prefix Sum](#pattern-4-prefix-sum)
- [Pattern 5: Hash Map / Frequency Counting](#pattern-5-hash-map--frequency-counting)
- [Pattern 6: Stack Pattern](#pattern-6-stack-pattern)
- [Pattern 7: Monotonic Stack](#pattern-7-monotonic-stack)
- [Pattern 8: Binary Search](#pattern-8-binary-search)
- [Pattern 9: Binary Search on Answer](#pattern-9-binary-search-on-answer)
- [Pattern 10: Backtracking](#pattern-10-backtracking)

---

## Pattern 1: Sliding Window

> **Definition:** A technique that maintains a window of elements in an array or string, expanding or contracting it to find optimal subarrays that satisfy certain conditions.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Instead of checking every possible subarray (O(n²)), we maintain a "window" that slides through the array, adjusting its boundaries based on conditions.
- 💭 **Intuition:** Imagine looking through a physical window at a landscape—you can slide it left/right and expand/shrink it without re-examining everything you've already seen.
- 🔢 **Mathematical Reasoning:** We avoid redundant calculations by reusing information from the previous window state (add new element, remove old element).
- ⚡ **When to Use:** Problems involving contiguous subarrays/substrings with constraints (max/min length, sum, distinct elements).

---

### **ASCII Diagram**

```
Fixed-Size Window:
Array: [1, 3, 5, 2, 8, 7, 6]
        [1, 3, 5]              ← Window size = 3, Sum = 9
           [3, 5, 2]           ← Slide right: Remove 1, Add 2
              [5, 2, 8]        ← Slide right: Remove 3, Add 8

Dynamic Window:
Array: [1, 3, 5, 2, 8, 7, 6]   Target sum ≤ 10
        L→
        [1, 3, 5]              ← Sum = 9 (valid)
        [1, 3, 5, 2]           ← Sum = 11 (invalid, shrink)
           L→
           [3, 5, 2]           ← Sum = 10 (valid)
```

---

### **Memorization Technique**

🎬 **Visualization:** Imagine a **magnifying glass sliding over an ant trail**. The glass can zoom in (shrink) or zoom out (expand) as it moves, always keeping ants in focus. The ants represent array elements, and the glass represents your window.

💥 **Exaggeration:** The magnifying glass is GIANT and GLOWING, leaving a trail of light behind it so you never re-examine the same ants twice!

🔗 **Association:** "SLIDE" = **S**hrink, **L**oop, **I**ncrement, **D**ynamic, **E**xpand

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Fixed-Size Window** | Window size remains constant | Maximum sum of subarray of size k |
| **Dynamic Shrinking Window** | Expand right, shrink left when invalid | Longest substring without repeating characters |
| **Dynamic Expanding Window** | Expand both sides based on conditions | Minimum window substring |

---

### **Socratic Teaching Round**

#### **Question 1:** Why can't we just check every possible subarray?
**Answer:** Checking every subarray takes O(n²) time because for each starting position (n positions), we examine all possible ending positions (n positions). With sliding window, we maintain a moving window in O(n) time.

#### **Question 2:** When should the window shrink vs. expand?
**Answer:** **Expand** when we need more elements to satisfy the condition. **Shrink** when the current window violates the condition (becomes invalid).

#### **Question 3:** What data structure helps track window contents?
**Answer:** Hash map (dictionary) for character/element frequency counting, or simple variables for sums/counts.

#### **Question 4:** How do we know if a problem uses sliding window?
**Answer:** Look for keywords: "contiguous," "subarray," "substring," constraints on elements within a range, optimization of window properties.

#### **Question 5:** What's the difference between fixed and dynamic windows?
**Answer:** **Fixed:** Window size is predetermined (size k). **Dynamic:** Window size changes based on validity conditions.

---

### **Problems**

---

#### **EASY: Maximum Sum Subarray of Size K**

**A. Problem Statement:**
Given an array of integers and a number k, find the maximum sum of any contiguous subarray of size k.

**B. ASCII Visualization:**
```
Array: [2, 1, 5, 1, 3, 2]   k = 3

Step 1: [2, 1, 5] → Sum = 8
Step 2:    [1, 5, 1] → Sum = 7
Step 3:       [5, 1, 3] → Sum = 9 ✓ Max
Step 4:          [1, 3, 2] → Sum = 6
```

**C. Pattern Recognition Question:**
*Why is this sliding window and not nested loops?*

**D. Step-by-Step Reasoning:**
1. Initialize the first window of size k
2. Calculate its sum
3. Slide the window: remove leftmost element, add new rightmost element
4. Update maximum sum at each step
5. Return maximum

**E. Python Solution:**
```python
def max_sum_subarray(arr, k):
    # Edge case: array smaller than k
    if len(arr) < k:
        return None
    
    # Calculate sum of first window
    window_sum = sum(arr[:k])
    max_sum = window_sum
    
    # Slide the window from left to right
    for i in range(k, len(arr)):
        # Remove leftmost element of previous window, add new element
        window_sum = window_sum - arr[i - k] + arr[i]
        max_sum = max(max_sum, window_sum)
    
    return max_sum

# Example usage
print(max_sum_subarray([2, 1, 5, 1, 3, 2], 3))  # Output: 9
```

**F. Memory Cue:**
*"Window SUM slides like a TRAIN: lose one car at the back, gain one at the front!"*

---

#### **HARD: Longest Substring Without Repeating Characters**

**A. Problem Statement:**
Given a string, find the length of the longest substring without repeating characters.

**B. ASCII Visualization:**
```
String: "abcabcbb"

Window:  a b c a b c b b
         L→
         [a b c]           ← Length = 3, no repeats
         [a b c a]         ← 'a' repeats! Shrink from left
             L→
             [b c a]       ← Length = 3, no repeats
             [b c a b]     ← 'b' repeats! Shrink
               L→
               [c a b]     ← Length = 3 ✓
```

**C. Pattern Recognition Question:**
*How does the hash map help us detect repeats immediately?*

**D. Step-by-Step Reasoning:**
1. Use two pointers (left, right) and a hash map to track character positions
2. Expand right pointer, add characters to map
3. If duplicate found, shrink from left until duplicate is removed
4. Track maximum window size throughout
5. Return maximum length

**E. Python Solution:**
```python
def length_of_longest_substring(s):
    # Hash map to store character and its index
    char_map = {}
    left = 0
    max_length = 0
    
    for right in range(len(s)):
        # If character already in map and within current window
        if s[right] in char_map and char_map[s[right]] >= left:
            # Move left pointer to right of the duplicate
            left = char_map[s[right]] + 1
        
        # Update character position
        char_map[s[right]] = right
        
        # Update maximum length
        max_length = max(max_length, right - left + 1)
    
    return max_length

# Example usage
print(length_of_longest_substring("abcabcbb"))  # Output: 3
```

**F. Memory Cue:**
*"UNIQUE window: HashMap is the BOUNCER—kicks out duplicates by moving the door (left pointer)!"*

---

#### **VERY HARD: Minimum Window Substring**

**A. Problem Statement:**
Given strings s and t, find the minimum window substring of s that contains all characters of t (including duplicates).

**B. ASCII Visualization:**
```
s = "ADOBECODEBANC"    t = "ABC"

Need: A=1, B=1, C=1

Window: A D O B E C O D E B A N C
        L→                          R→
        [A D O B E C]               ← Contains A,B,C ✓ Length=6
          L→
          [D O B E C]               ← Missing A ✗
                  ...shrinking & expanding...
                      L→      R→
                      [B A N C]     ← Contains A,B,C ✓ Length=4 (minimum)
```

**C. Pattern Recognition Question:**
*Why do we need TWO frequency maps (required vs. window)?*

**D. Step-by-Step Reasoning:**
1. Create frequency map for string t (required characters)
2. Use two pointers (left, right) and a window frequency map
3. Expand right: add characters to window until all requirements met
4. Shrink left: remove characters while maintaining validity
5. Track minimum window size when valid
6. Return minimum window substring

**E. Python Solution:**
```python
from collections import Counter

def min_window(s, t):
    if not s or not t:
        return ""
    
    # Frequency map of characters in t
    required = Counter(t)
    # Track how many unique characters in t are satisfied in current window
    formed = 0
    required_chars = len(required)
    
    # Window character frequency map
    window_counts = {}
    
    # Left and right pointers
    left = 0
    # Result: (window_length, left, right)
    result = (float('inf'), 0, 0)
    
    for right in range(len(s)):
        # Add character from right to window
        char = s[right]
        window_counts[char] = window_counts.get(char, 0) + 1
        
        # Check if frequency of current char matches required frequency
        if char in required and window_counts[char] == required[char]:
            formed += 1
        
        # Try to shrink window until it's no longer valid
        while left <= right and formed == required_chars:
            char = s[left]
            
            # Update result if this window is smaller
            if right - left + 1 < result[0]:
                result = (right - left + 1, left, right)
            
            # Remove leftmost character from window
            window_counts[char] -= 1
            if char in required and window_counts[char] < required[char]:
                formed -= 1
            
            # Move left pointer ahead
            left += 1
    
    # Return minimum window or empty string
    return "" if result[0] == float('inf') else s[result[1]:result[2] + 1]

# Example usage
print(min_window("ADOBECODEBANC", "ABC"))  # Output: "BANC"
```

**F. Memory Cue:**
*"Two TREASURE MAPS: one shows what you NEED (required), one shows what you HAVE (window). Match them perfectly while shrinking the search area!"*

---

### **Summary in 3 Sentences**

Sliding window optimizes subarray/substring problems by maintaining a moving range instead of checking all possibilities. The window dynamically expands to explore and shrinks to eliminate invalid states, using hash maps or counters to track contents. Master when to expand (need more elements) versus shrink (condition violated) for O(n) efficiency.

---

## Pattern 2: Two Pointers

> **Definition:** A technique using two reference points (pointers) that move through a data structure, often from opposite ends or at different speeds, to solve problems efficiently without nested loops.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Two pointers eliminate the need for nested loops by intelligently moving through the array based on comparisons, reducing O(n²) to O(n).
- 💭 **Intuition:** Picture two people walking toward each other from opposite ends of a hallway—they meet in the middle, examining everything once.
- 🔢 **Mathematical Reasoning:** By comparing elements at both pointers and moving them based on conditions, we cover all necessary comparisons in a single pass.
- ⚡ **When to Use:** Sorted arrays, palindromes, pair/triplet sums, removing duplicates, partitioning arrays.

---

### **ASCII Diagram**

```
Opposite Direction Pointers:
Array: [1, 2, 3, 4, 5, 6]
        L→             ←R       Compare arr[L] + arr[R]
           L→       ←R          Move based on sum vs target
              L→ ←R             Continue until L meets R

Same Direction Pointers:
Array: [0, 0, 1, 1, 2, 3]
        S→                      Slow pointer (writes)
        F→                      Fast pointer (reads)
        S  F→                   Fast finds new element
        S→ F→                   Slow writes, both advance
```

---

### **Memorization Technique**

🎬 **Visualization:** Imagine two **ROBOTS on a conveyor belt**. One robot (left) moves slow and MARKS items, the other (right) zooms ahead SCANNING for special items. They work together but never waste time checking the same item twice.

💥 **Exaggeration:** The robots have GIANT LASER POINTERS shooting beams at array elements, and they NEVER look backward (one-pass only)!

🔗 **Association:** "TWO POINTERS" = **T**wo **W**alkers **O**pposite, **P**artition **O**r **I**terate **N**ever **T**wice, **E**fficient **R**eaching **S**olution

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Opposite Direction** | Start at both ends, move toward center | Two sum in sorted array |
| **Same Direction (Fast-Slow)** | Both start at beginning, different speeds | Remove duplicates |
| **Sliding Window Variant** | Two pointers define a window | Covered in Pattern 1 |

---

### **Socratic Teaching Round**

#### **Question 1:** Why does two pointers work best on sorted arrays?
**Answer:** Sorting provides order, letting us make decisions about pointer movement based on comparisons (if sum too large, move right pointer left; if too small, move left pointer right).

#### **Question 2:** What's the difference between two pointers and sliding window?
**Answer:** Sliding window focuses on **continuous subarrays** with specific properties. Two pointers is broader: can start from opposite ends, same direction at different speeds, or track two separate positions for various purposes.

#### **Question 3:** How do we know which pointer to move?
**Answer:** Based on the problem condition: for sum problems, move the pointer that helps reach target; for duplicates, move fast pointer to find next unique element.

#### **Question 4:** Can two pointers work on unsorted arrays?
**Answer:** Yes, but limited scenarios (e.g., palindrome checking, partitioning). Most two-pointer problems benefit from sorted order.

#### **Question 5:** What's the time complexity advantage?
**Answer:** Reduces O(n²) nested loops to O(n) single pass, as each pointer traverses the array once.

---

### **Problems**

---

#### **EASY: Two Sum II (Sorted Array)**

**A. Problem Statement:**
Given a sorted array and a target, find two numbers that add up to the target. Return their indices (1-indexed).

**B. ASCII Visualization:**
```
Array: [2, 7, 11, 15]  Target: 9

Step 1: L→         ←R
        [2]     [15]     → 2 + 15 = 17 > 9 (move R left)

Step 2: L→      ←R
        [2]  [11]        → 2 + 11 = 13 > 9 (move R left)

Step 3: L→   ←R
        [2] [7]          → 2 + 7 = 9 ✓ Found!
```

**C. Pattern Recognition Question:**
*Why does moving the right pointer left when sum is too large guarantee we don't miss solutions?*

**D. Step-by-Step Reasoning:**
1. Place left pointer at start (smallest value)
2. Place right pointer at end (largest value)
3. Calculate sum of elements at both pointers
4. If sum equals target: return indices
5. If sum too large: decrease by moving right pointer left
6. If sum too small: increase by moving left pointer right
7. Repeat until pointers meet

**E. Python Solution:**
```python
def two_sum_sorted(numbers, target):
    left = 0
    right = len(numbers) - 1
    
    while left < right:
        current_sum = numbers[left] + numbers[right]
        
        if current_sum == target:
            # Return 1-indexed positions
            return [left + 1, right + 1]
        elif current_sum < target:
            # Need larger sum, move left pointer right
            left += 1
        else:
            # Need smaller sum, move right pointer left
            right -= 1
    
    return []  # No solution found

# Example usage
print(two_sum_sorted([2, 7, 11, 15], 9))  # Output: [1, 2]
```

**F. Memory Cue:**
*"SQUEEZE the array like a TUBE OF TOOTHPASTE: too much (move right), too little (move left), just right (done)!"*

---

#### **HARD: 3Sum**

**A. Problem Statement:**
Given an array of integers, find all unique triplets that sum to zero.

**B. ASCII Visualization:**
```
Array: [-1, 0, 1, 2, -1, -4]  Target: 0
Sorted: [-4, -1, -1, 0, 1, 2]

Fix i at -1:
        i   L→         ←R
       [-1][-1]  [0]  [2]    → -1 + (-1) + 2 = 0 ✓

        i      L→   ←R
       [-1] [-1] [1][2]      → -1 + (-1) + 2 = 0 (duplicate, skip)

        i         L→ ←R
       [-1]   [0] [1]        → -1 + 0 + 1 = 0 ✓
```

**C. Pattern Recognition Question:**
*How does fixing one element turn 3Sum into a 2Sum problem?*

**D. Step-by-Step Reasoning:**
1. Sort the array (enables two-pointer technique)
2. Fix first element (iterate through array)
3. Use two pointers (left, right) on remaining elements
4. For each fixed element, find pairs that sum to -element
5. Skip duplicates to avoid duplicate triplets
6. Collect all valid triplets

**E. Python Solution:**
```python
def three_sum(nums):
    nums.sort()  # Sort array
    result = []
    
    for i in range(len(nums) - 2):
        # Skip duplicate values for first element
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        
        # Two pointer approach for remaining elements
        left = i + 1
        right = len(nums) - 1
        target = -nums[i]  # We want nums[left] + nums[right] = -nums[i]
        
        while left < right:
            current_sum = nums[left] + nums[right]
            
            if current_sum == target:
                result.append([nums[i], nums[left], nums[right]])
                
                # Skip duplicate values for left pointer
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                # Skip duplicate values for right pointer
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                
                left += 1
                right -= 1
            elif current_sum < target:
                left += 1
            else:
                right -= 1
    
    return result

# Example usage
print(three_sum([-1, 0, 1, 2, -1, -4]))  # Output: [[-1, -1, 2], [-1, 0, 1]]
```

**F. Memory Cue:**
*"3Sum is 2Sum wearing a HAT: fix the hat (first element), then solve 2Sum on the body (rest of array)!"*

---

#### **VERY HARD: Trapping Rain Water**

**A. Problem Statement:**
Given an array representing elevation heights, calculate how much water can be trapped after it rains.

**B. ASCII Visualization:**
```
Height: [0,1,0,2,1,0,1,3,2,1,2,1]

Elevation view:
       █
   █   █ █   █
   █ █ █ █ █ █
───█─█─█─█─█─█─  (Water represented by ≈)

Water trapped:
       █
   █≈≈≈█≈█≈≈≈█
   █≈█≈█≈█≈█≈█
───█─█─█─█─█─█─

Two Pointers Approach:
L→              ←R
Track max_left and max_right heights
Move pointer with smaller max height
```

**C. Pattern Recognition Question:**
*Why can we calculate water at each position based only on the smaller of the two maximum heights (left vs right)?*

**D. Step-by-Step Reasoning:**
1. Use two pointers (left at start, right at end)
2. Track maximum height seen from left and right
3. Water at position = min(max_left, max_right) - current_height
4. Move the pointer with smaller maximum height (limiting factor)
5. Update maximum heights as we go
6. Accumulate trapped water

**E. Python Solution:**
```python
def trap_rain_water(height):
    if not height:
        return 0
    
    left = 0
    right = len(height) - 1
    max_left = height[left]
    max_right = height[right]
    water = 0
    
    while left < right:
        if max_left < max_right:
            # Left side is limiting factor
            left += 1
            max_left = max(max_left, height[left])
            # Water trapped at current position
            water += max_left - height[left]
        else:
            # Right side is limiting factor
            right -= 1
            max_right = max(max_right, height[right])
            # Water trapped at current position
            water += max_right - height[right]
    
    return water

# Example usage
print(trap_rain_water([0,1,0,2,1,0,1,3,2,1,2,1]))  # Output: 6
```

**F. Memory Cue:**
*"Water finds the LOWEST WALL: two pointers are WALLS closing in, and water fills up to the shorter wall's height!"*

---

### **Summary in 3 Sentences**

Two pointers eliminate nested loops by using two reference points that move through arrays based on logical conditions. The pattern works best on sorted data where pointer movement decisions depend on comparisons (move left vs right, when to advance). Master the choice between opposite-direction (converging) and same-direction (different speeds) pointer patterns.

---

## Pattern 3: Fast-Slow Pointers

> **Definition:** A cycle detection technique using two pointers moving at different speeds through a linked list—the fast pointer moves twice as fast as the slow pointer, and they meet if a cycle exists.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** If a cycle exists, the fast pointer will eventually "lap" the slow pointer (like runners on a circular track). This proves cycle existence without extra space.
- 💭 **Intuition:** Imagine two runners on a track: one runs twice as fast. If the track loops, the faster runner will catch up to the slower one.
- 🔢 **Mathematical Reasoning:** In a cycle of length C, after C steps, fast pointer gains C positions on slow pointer. Since fast moves 2 steps and slow moves 1 step, the gap closes by 1 each iteration, guaranteeing they meet.
- ⚡ **When to Use:** Cycle detection in linked lists, finding middle element, detecting duplicate numbers with constant space.

---

### **ASCII Diagram**

```
Cycle Detection (Tortoise and Hare):

No Cycle:
1 → 2 → 3 → 4 → NULL
S   F           ← Fast reaches NULL (no cycle)

With Cycle:
    S           F
    ↓           ↓
1 → 2 → 3 → 4 → 5
        ↑       ↓
        8 ← 7 ← 6

Step 1: S at 1, F at 1
Step 2: S at 2, F at 3
Step 3: S at 3, F at 5
Step 4: S at 4, F at 7
Step 5: S at 5, F at 3
Step 6: S at 6, F at 5
Step 7: S at 7, F at 7  ← They meet! Cycle detected

Finding Middle:
1 → 2 → 3 → 4 → 5 → NULL
S       F           ← When F reaches end, S is at middle
```

---

### **Memorization Technique**

🎬 **Visualization:** Picture a **TORTOISE and HARE racing** on a circular track. The hare runs at DOUBLE speed. If the track loops back, the hare will definitely lap the tortoise. If the track ends, the hare reaches the finish line first.

💥 **Exaggeration:** The hare has ROCKET BOOTS (2x speed) and leaves FIRE TRAILS. The tortoise has a GLOWING SHELL. When they meet, their colors CLASH creating LIGHTNING—that's your cycle!

🔗 **Association:** "FLOYD'S ALGORITHM" = **F**ast **L**aps **O**ver **Y**ou **D**efinitely → **S**low = **A**lways **L**ags → **G**ame **O**ver **R**eunion **I**ndicates **T**rack **H**as **M**eet

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Cycle Detection** | Detect if cycle exists | Linked list cycle |
| **Cycle Start** | Find where cycle begins | After detecting cycle, reset one pointer |
| **Middle Element** | Find middle of linked list | Fast reaches end, slow at middle |
| **Array Cycle** | Detect duplicates treating array as linked list | Find duplicate number |

---

### **Socratic Teaching Round**

#### **Question 1:** Why does the fast pointer move 2x speed instead of 3x or 4x?
**Answer:** 2x is optimal: guarantees meeting within one cycle traversal, minimizes steps, and simplifies the math. Higher speeds might skip over the slow pointer in the cycle.

#### **Question 2:** What if the cycle is very small (length 1 or 2)?
**Answer:** The algorithm still works. Fast pointer will meet slow pointer immediately or within one iteration since the gap closes by 1 each step.

#### **Question 3:** How do we find where the cycle starts after detection?
**Answer:** Reset one pointer to head, keep other at meeting point. Move both at same speed (1 step). They meet at cycle start. This works due to mathematical properties: distance from head to cycle start equals distance from meeting point to cycle start.

#### **Question 4:** Can this pattern work on arrays?
**Answer:** Yes! Treat array as implicit linked list where `arr[i]` points to `arr[arr[i]]`. Used in problems like "Find Duplicate Number."

#### **Question 5:** What's the space complexity advantage?
**Answer:** O(1) space compared to hash set approach O(n). We only use two pointers regardless of input size.

---

### **Problems**

---

#### **EASY: Linked List Cycle Detection**

**A. Problem Statement:**
Given a linked list, determine if it has a cycle.

**B. ASCII Visualization:**
```
Linked List with Cycle:

1 → 2 → 3 → 4
        ↑   ↓
        6 ← 5

Iteration:
Step 1: Slow=1, Fast=1
Step 2: Slow=2, Fast=3
Step 3: Slow=3, Fast=5
Step 4: Slow=4, Fast=3
Step 5: Slow=5, Fast=5  ← MEET! Cycle exists ✓

No Cycle:
1 → 2 → 3 → 4 → NULL
            S   F   ← Fast reaches NULL ✓
```

**C. Pattern Recognition Question:**
*Why must the fast pointer eventually meet the slow pointer if a cycle exists?*

**D. Step-by-Step Reasoning:**
1. Initialize slow and fast pointers at head
2. Move slow pointer 1 step per iteration
3. Move fast pointer 2 steps per iteration
4. If fast pointer reaches NULL, no cycle
5. If slow and fast meet, cycle detected
6. Return boolean result

**E. Python Solution:**
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def has_cycle(head):
    # Edge case: empty list or single node
    if not head or not head.next:
        return False
    
    # Initialize both pointers at head
    slow = head
    fast = head
    
    # Move pointers until fast reaches end or they meet
    while fast and fast.next:
        slow = slow.next          # Move slow 1 step
        fast = fast.next.next     # Move fast 2 steps
        
        # Cycle detected!
        if slow == fast:
            return True
    
    # Fast reached end, no cycle
    return False

# Example usage
# Create cycle: 1 → 2 → 3 → 4 → 2 (cycle back)
node1 = ListNode(1)
node2 = ListNode(2)
node3 = ListNode(3)
node4 = ListNode(4)
node1.next = node2
node2.next = node3
node3.next = node4
node4.next = node2  # Creates cycle
print(has_cycle(node1))  # Output: True
```

**F. Memory Cue:**
*"HARE catches TORTOISE on a CIRCULAR TRACK but NEVER on a STRAIGHT ROAD!"*

---

#### **HARD: Find Cycle Start**

**A. Problem Statement:**
Given a linked list with a cycle, find the node where the cycle begins.

**B. ASCII Visualization:**
```
Linked List:
    Cycle Start
        ↓
1 → 2 → 3 → 4 → 5
        ↑       ↓
        8 ← 7 ← 6

Phase 1: Detect cycle (fast & slow meet)
Meeting point: node 5

Phase 2: Find cycle start
Reset slow to head (node 1)
Keep fast at meeting point (node 5)
Move both at 1 step each:

Slow: 1 → 2 → 3
Fast: 5 → 6 → 7 → 8 → 3

They meet at node 3 (cycle start) ✓
```

**C. Pattern Recognition Question:**
*Why does resetting one pointer to head and moving both at same speed find the cycle start?*

**D. Step-by-Step Reasoning:**
1. First, detect cycle using fast-slow pointers
2. After meeting, reset slow pointer to head
3. Keep fast pointer at meeting point
4. Move both pointers one step at a time
5. They will meet at the cycle start node
6. This works because: distance(head to start) = distance(meeting to start)

**E. Python Solution:**
```python
def detect_cycle(head):
    if not head or not head.next:
        return None
    
    # Phase 1: Detect if cycle exists
    slow = head
    fast = head
    
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        
        if slow == fast:
            # Cycle detected, proceed to phase 2
            break
    else:
        # No cycle found
        return None
    
    # Phase 2: Find cycle start
    slow = head  # Reset slow to head
    # Fast stays at meeting point
    
    while slow != fast:
        slow = slow.next
        fast = fast.next  # Move both at same speed (1 step)
    
    # Both pointers now at cycle start
    return slow

# Example usage
node1 = ListNode(1)
node2 = ListNode(2)
node3 = ListNode(3)
node4 = ListNode(4)
node1.next = node2
node2.next = node3
node3.next = node4
node4.next = node2  # Cycle starts at node2
cycle_start = detect_cycle(node1)
print(cycle_start.val if cycle_start else None)  # Output: 2
```

**F. Memory Cue:**
*"After MEETING in the cycle, send one runner back to START LINE. When they run at SAME SPEED, they shake hands at the CYCLE ENTRANCE!"*

---

#### **VERY HARD: Find Duplicate Number**

**A. Problem Statement:**
Given an array of n+1 integers where each integer is between 1 and n (inclusive), find the duplicate number. You cannot modify the array and must use O(1) extra space.

**B. ASCII Visualization:**
```
Array: [1, 3, 4, 2, 2]  (indices: 0-4)
Treat as linked list: arr[i] points to arr[arr[i]]

Index:  0  1  2  3  4
Value: [1, 3, 4, 2, 2]

Implicit Linked List:
Start(0) → 1 → 3 → 2 → 4 → 2 (cycle!)
                   ↑________|

0 → arr[0]=1 → arr[1]=3 → arr[3]=2 → arr[2]=4 → arr[4]=2 (back to index 2)

Cycle exists with duplicate 2 as cycle start!

Fast-Slow Movement:
Slow: 0 → 1 → 3 → 2
Fast: 0 → 3 → 4 → 3 (they meet at 2)
```

**C. Pattern Recognition Question:**
*How does treating the array as a linked list convert this into a cycle detection problem?*

**D. Step-by-Step Reasoning:**
1. Treat array indices as linked list nodes
2. Each value arr[i] points to index arr[i]
3. Since there's a duplicate, two indices point to same value
4. This creates a cycle (multiple paths to same node)
5. Use Floyd's algorithm to detect cycle
6. The cycle start is the duplicate number
7. Apply two-phase approach: detect cycle, then find start

**E. Python Solution:**
```python
def find_duplicate(nums):
    # Phase 1: Find intersection point in cycle
    slow = nums[0]
    fast = nums[0]
    
    # Move until they meet
    while True:
        slow = nums[slow]           # Move 1 step
        fast = nums[nums[fast]]     # Move 2 steps
        
        if slow == fast:
            break
    
    # Phase 2: Find cycle start (the duplicate)
    slow = nums[0]  # Reset slow to start
    # Fast stays at meeting point
    
    while slow != fast:
        slow = nums[slow]
        fast = nums[fast]  # Both move 1 step
    
    # Both point to duplicate number
    return slow

# Example usage
print(find_duplicate([1, 3, 4, 2, 2]))  # Output: 2
print(find_duplicate([3, 1, 3, 4, 2]))  # Output: 3
```

**F. Memory Cue:**
*"Array is a TREASURE MAP where numbers are DIRECTIONS. The DUPLICATE is a place with TWO PATHS leading to it—that's your CYCLE START!"*

---

### **Summary in 3 Sentences**

Fast-slow pointers detect cycles by moving at different speeds—if they meet, a cycle exists. The pattern uses O(1) space and works on linked lists or arrays treated as implicit linked lists. Master the two-phase approach: detect cycle (fast moves 2x), then find cycle start (reset one pointer, move both at 1x).

---

## Pattern 4: Prefix Sum

> **Definition:** A preprocessing technique that creates an array where each element stores the cumulative sum from the start, enabling O(1) range sum queries after O(n) preprocessing.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Computing range sums repeatedly is expensive (O(n) per query). Prefix sum trades O(n) preprocessing for O(1) queries by storing cumulative information.
- 💭 **Intuition:** Imagine a water tank with level markers—each marker shows total water up to that point. To find water between two markers, subtract lower from higher.
- 🔢 **Mathematical Reasoning:** sum(i, j) = prefix[j] - prefix[i-1]. The difference gives the range sum without re-adding elements.
- ⚡ **When to Use:** Multiple range sum queries, subarray sum problems, finding subarrays with specific properties.

---

### **ASCII Diagram**

```
Original Array:
Index:  0   1   2   3   4   5
Array: [3,  1,  4,  2,  5,  1]

Prefix Sum Array:
Index:  0   1   2   3   4   5
Prefix: [3,  4,  8, 10, 15, 16]
         ↑   ↑   ↑
         |   |   sum[0:2] = 3+1+4 = 8
         |   sum[0:1] = 3+1 = 4
         sum[0:0] = 3

Range Sum Query [2, 4]:
Sum = prefix[4] - prefix[1]
    = 15 - 4
    = 11
    = (4 + 2 + 5) ✓

Visual:
[3, 1, 4, 2, 5, 1]
 ----  --------    ← We want this range [2:4]
prefix[4] = 3+1+4+2+5 = 15
prefix[1] = 3+1 = 4
Difference = 11 = 4+2+5 ✓
```

---

### **Memorization Technique**

🎬 **Visualization:** Think of a **STAIRCASE where each step shows TOTAL HEIGHT climbed**. To find height between step 3 and step 7, subtract step 3's total from step 7's total—you get the climb between them!

💥 **Exaggeration:** The staircase is MADE OF GOLD BARS, each bar labeled with RUNNING TOTAL. You're a PIRATE calculating treasure in any range by SUBTRACTING two totals!

🔗 **Association:** "PREFIX SUM" = **P**reprocess **R**unning **E**xtraction **F**ast **I**nstant **X**traction → **S**um **U**sing **M**inus

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **1D Prefix Sum** | Linear array cumulative sums | Range sum query |
| **2D Prefix Sum** | Matrix cumulative sums | Rectangle sum query |
| **Prefix with HashMap** | Count subarrays with target sum | Subarray sum equals k |
| **Difference Array** | Range updates efficiently | Range addition queries |

---

### **Socratic Teaching Round**

#### **Question 1:** Why is prefix sum O(1) for queries but O(n) to build?
**Answer:** We iterate through the array once (O(n)) to compute cumulative sums. After that, any range sum is just two array accesses and one subtraction (O(1)).

#### **Question 2:** How do we handle sum from index 0 to i?
**Answer:** Simply use prefix[i] directly, no subtraction needed. For general range [i, j], use prefix[j] - prefix[i-1]. Edge case: if i=0, result is just prefix[j].

#### **Question 3:** What if we need multiple updates to the array?
**Answer:** Prefix sum is best for static arrays with many queries. For updates, consider segment trees or Fenwick trees (Binary Indexed Tree).

#### **Question 4:** How does prefix sum help find subarrays with target sum?
**Answer:** Use hashmap: if prefix[j] - prefix[i] = target, then prefix[j] - target = prefix[i]. Store prefix sums in hashmap and check if (current_prefix - target) exists.

#### **Question 5:** Can prefix sum work with operations other than addition?
**Answer:** Yes! Prefix XOR (for XOR queries), prefix product (for product queries), prefix max/min (for range max/min). The concept extends to any associative operation.

---

### **Problems**

---

#### **EASY: Range Sum Query - Immutable**

**A. Problem Statement:**
Given an integer array, handle multiple queries to calculate sum of elements between indices i and j (inclusive).

**B. ASCII Visualization:**
```
Array: [-2, 0, 3, -5, 2, -1]

Build Prefix Sum:
Index:    0   1   2   3   4   5
Original:[-2,  0,  3, -5,  2, -1]
Prefix:  [-2, -2,  1, -4, -2, -3]
          ↑   ↑    ↑   ↑   ↑   ↑
          |   |    |   |   |   sum[0:5]=-2+0+3-5+2-1=-3
          |   |    |   |   sum[0:4]=-2+0+3-5+2=-2
          |   |    |   sum[0:3]=-2+0+3-5=-4
          |   |    sum[0:2]=-2+0+3=1
          |   sum[0:1]=-2+0=-2
          sum[0:0]=-2

Query: Sum[2, 5] = prefix[5] - prefix[1] = -3 - (-2) = -1
Verify: 3 + (-5) + 2 + (-1) = -1 ✓
```

**C. Pattern Recognition Question:**
*Why do we subtract prefix[i-1] instead of prefix[i] for range [i, j]?*

**D. Step-by-Step Reasoning:**
1. Build prefix sum array in constructor
2. prefix[i] = sum of elements from index 0 to i
3. For range [i, j]: sum = prefix[j] - prefix[i-1]
4. Handle edge case when i=0 (just return prefix[j])
5. Each query now takes O(1) time

**E. Python Solution:**
```python
class NumArray:
    def __init__(self, nums):
        # Build prefix sum array
        self.prefix = [0] * len(nums)
        self.prefix[0] = nums[0]
        
        for i in range(1, len(nums)):
            self.prefix[i] = self.prefix[i - 1] + nums[i]
    
    def sumRange(self, left, right):
        # Range sum query in O(1)
        if left == 0:
            return self.prefix[right]
        return self.prefix[right] - self.prefix[left - 1]

# Example usage
obj = NumArray([-2, 0, 3, -5, 2, -1])
print(obj.sumRange(0, 2))  # Output: 1  (sum of -2, 0, 3)
print(obj.sumRange(2, 5))  # Output: -1 (sum of 3, -5, 2, -1)
print(obj.sumRange(0, 5))  # Output: -3 (sum of entire array)
```

**F. Memory Cue:**
*"Prefix sum is a RUNNING TOTAL at a MARATHON—to find distance between mile 5 and mile 8, subtract mile 5 total from mile 8 total!"*

---

#### **HARD: Subarray Sum Equals K**

**A. Problem Statement:**
Given an array and integer k, find the total number of continuous subarrays whose sum equals k.

**B. ASCII Visualization:**
```
Array: [1, 2, 3, 4]  k = 3

Prefix sums: [0, 1, 3, 6, 10]
             (added 0 at start)

For each prefix[j], check if (prefix[j] - k) exists:

j=0: prefix=0, need -3? No
j=1: prefix=1, need -2? No
j=2: prefix=3, need 0? Yes! (subarray [1,2])
j=3: prefix=6, need 3? Yes! (subarray [3])
j=4: prefix=10, need 7? No

Found 2 subarrays: [1,2] and [3]

HashMap tracks:
{0: 1}           ← count=0 subarrays with sum 0
{0: 1, 1: 1}     ← count=1 subarray with sum 1
{0: 1, 1: 1, 3: 1} ← found! 3-3=0 exists
...
```

**C. Pattern Recognition Question:**
*How does storing prefix sums in a hashmap let us count subarrays in one pass?*

**D. Step-by-Step Reasoning:**
1. Use hashmap to store frequency of prefix sums seen
2. Initialize with {0: 1} (empty subarray has sum 0)
3. Track running sum (current prefix sum)
4. For each position, check if (current_sum - k) exists in map
5. If yes, add its frequency to result (those are valid subarrays)
6. Add current sum to hashmap
7. Continue through array

**E. Python Solution:**
```python
def subarray_sum(nums, k):
    # HashMap: prefix_sum -> frequency
    prefix_count = {0: 1}  # Empty subarray has sum 0
    current_sum = 0
    count = 0
    
    for num in nums:
        # Update running sum
        current_sum += num
        
        # Check if (current_sum - k) exists
        # If yes, we found subarray(s) with sum = k
        if current_sum - k in prefix_count:
            count += prefix_count[current_sum - k]
        
        # Add current sum to hashmap
        prefix_count[current_sum] = prefix_count.get(current_sum, 0) + 1
    
    return count

# Example usage
print(subarray_sum([1, 2, 3], 3))      # Output: 2 ([1,2] and [3])
print(subarray_sum([1, 1, 1], 2))      # Output: 2 ([1,1] twice)
print(subarray_sum([1, -1, 1, -1], 0)) # Output: 4
```

**F. Memory Cue:**
*"PREFIX SUM + HASHMAP = TIME MACHINE: 'Have I seen (current - target) before?' HashMap remembers ALL previous moments!"*

---

#### **VERY HARD: 2D Matrix Range Sum Query**

**A. Problem Statement:**
Given a 2D matrix, handle multiple queries to find sum of elements inside a rectangle defined by top-left (row1, col1) and bottom-right (row2, col2).

**B. ASCII Visualization:**
```
Matrix:
  0 1 2 3
0 3 0 1 4
1 5 6 3 2
2 1 2 0 1
3 4 1 0 3

2D Prefix Sum (cumulative from [0,0] to [i,j]):
  0  1  2  3
0 3  3  4  8
1 8 14 18 24
2 9 17 21 28
3 13 22 26 36

Query: Sum of rectangle (1,1) to (2,2)
(shaded area in original matrix):
  6 3
  2 0  → Sum should be 11

Formula: prefix[2][2] - prefix[0][2] - prefix[2][0] + prefix[0][0]
       = 21 - 4 - 9 + 3 = 11 ✓

Visual explanation:
prefix[2][2] = entire area from (0,0) to (2,2)
prefix[0][2] = subtract unwanted top area
prefix[2][0] = subtract unwanted left area
prefix[0][0] = add back (subtracted twice)
```

**C. Pattern Recognition Question:**
*Why do we need to add back prefix[row1-1][col1-1] in the formula?*

**D. Step-by-Step Reasoning:**
1. Build 2D prefix sum: prefix[i][j] = sum of rectangle from (0,0) to (i,j)
2. For each cell, prefix[i][j] = matrix[i][j] + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1]
3. For query (row1, col1) to (row2, col2):
   - Take full rectangle: prefix[row2][col2]
   - Subtract top unwanted: prefix[row1-1][col2]
   - Subtract left unwanted: prefix[row2][col1-1]
   - Add back overlap (subtracted twice): prefix[row1-1][col1-1]
4. Handle edge cases with boundary checks

**E. Python Solution:**
```python
class NumMatrix:
    def __init__(self, matrix):
        if not matrix or not matrix[0]:
            self.prefix = []
            return
        
        rows, cols = len(matrix), len(matrix[0])
        # Create prefix sum matrix (extra row/col for easier boundary handling)
        self.prefix = [[0] * (cols + 1) for _ in range(rows + 1)]
        
        # Build 2D prefix sum
        for i in range(1, rows + 1):
            for j in range(1, cols + 1):
                self.prefix[i][j] = (
                    matrix[i-1][j-1] +              # Current cell
                    self.prefix[i-1][j] +           # Sum above
                    self.prefix[i][j-1] -           # Sum to left
                    self.prefix[i-1][j-1]           # Remove double-counted
                )
    
    def sumRegion(self, row1, col1, row2, col2):
        # Adjust for 1-indexed prefix array
        row1 += 1
        col1 += 1
        row2 += 1
        col2 += 1
        
        # Calculate rectangle sum using inclusion-exclusion
        return (
            self.prefix[row2][col2] -
            self.prefix[row1-1][col2] -
            self.prefix[row2][col1-1] +
            self.prefix[row1-1][col1-1]
        )

# Example usage
matrix = [
    [3, 0, 1, 4],
    [5, 6, 3, 2],
    [1, 2, 0, 1],
    [4, 1, 0, 3]
]
obj = NumMatrix(matrix)
print(obj.sumRegion(1, 1, 2, 2))  # Output: 11
print(obj.sumRegion(2, 1, 3, 3))  # Output: 6
```

**F. Memory Cue:**
*"2D prefix sum is STACKED PIZZA BOXES: to get boxes in middle, take ALL boxes, remove top stack, remove left stack, add back corner (removed TWICE)!"*

---

### **Summary in 3 Sentences**

Prefix sum preprocesses cumulative sums enabling O(1) range queries through subtraction of cumulative values. The pattern extends to 2D matrices using inclusion-exclusion principle and combines with hashmaps for counting subarrays with target properties. Master the core formula: range[i,j] = prefix[j] - prefix[i-1], and understand why we add/subtract in 2D cases.

---

## Pattern 5: Hash Map / Frequency Counting

> **Definition:** Using hash tables (dictionaries) to store element frequencies, positions, or mappings, enabling O(1) lookups to solve problems involving duplicates, pairs, or element relationships.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Hash maps trade space for time, providing instant lookups instead of linear searches. They're perfect for tracking "have I seen this before?"
- 💭 **Intuition:** Like a library index card system—instead of searching every shelf (O(n)), check the index card (O(1)) to find exactly where something is.
- 🔢 **Mathematical Reasoning:** Hash functions map keys to array indices, giving average O(1) access time. Collisions are handled via chaining or open addressing.
- ⚡ **When to Use:** Finding pairs/complements, counting frequencies, detecting duplicates, anagrams, tracking positions.

---

### **ASCII Diagram**

```
Frequency Counting:
Array: [1, 2, 2, 3, 1, 4, 2]

HashMap:
┌─────┬─────┐
│ Key │ Cnt │
├─────┼─────┤
│  1  │  2  │  ← Appears 2 times
│  2  │  3  │  ← Appears 3 times
│  3  │  1  │  ← Appears 1 time
│  4  │  1  │  ← Appears 1 time
└─────┴─────┘

Two Sum Pattern:
Array: [2, 7, 11, 15]  Target: 9

For each number, check if (target - number) exists:
┌─────────┬───────┐
│ Number  │ Check │ HashMap State
├─────────┼───────┼──────────────
│ 2       │ 9-2=7?│ {2: 0}
│ 7       │ 9-7=2?│ {2: 0, 7: 1} ← Found! 2 exists
└─────────┴───────┴──────────────

Position Tracking:
String: "abcabcbb"

HashMap tracks last seen position:
┌─────┬────────┐
│ Char│ Index  │
├─────┼────────┤
│ 'a' │   3    │  ← Last seen at index 3
│ 'b' │   6    │  ← Last seen at index 6
│ 'c' │   5    │  ← Last seen at index 5
└─────┴────────┘
```

---

### **Memorization Technique**

🎬 **Visualization:** Imagine a **MAGIC FILING CABINET** where drawers INSTANTLY OPEN to the right file. Each drawer is labeled (key), and inside is the information you need (value). No searching through all drawers—go directly to the one you want!

💥 **Exaggeration:** The cabinet has GLOWING NEON LABELS and SHOOTS OUT the drawer you need like a VENDING MACHINE. Each lookup is INSTANT with SPARKLES!

🔗 **Association:** "HASH MAP" = **H**ave **A** **S**uper **H**andy → **M**agical **A**rray **P**airing

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Frequency Counter** | Count occurrences of elements | Most common element |
| **Complement Lookup** | Check if target - current exists | Two sum |
| **Position Tracker** | Store indices of elements | First unique character |
| **Grouping** | Group related elements | Group anagrams |

---

### **Socratic Teaching Round**

#### **Question 1:** Why is hash map lookup O(1) on average but not always?
**Answer:** Hash functions can cause collisions (multiple keys mapping to same bucket). With good hash functions and load factors, collisions are rare, giving amortized O(1). Worst case (all collisions) is O(n).

#### **Question 2:** When should we use hash map vs sorting?
**Answer:** Hash map: O(n) time, O(n) space, preserves order. Sorting: O(n log n) time, O(1) space (in-place). Choose based on time/space constraints and whether order matters.

#### **Question 3:** How do hash maps help with the Two Sum problem?
**Answer:** For each number x, we instantly check if (target - x) exists in O(1) time. Without hash map, we'd need nested loops O(n²).

#### **Question 4:** What's the difference between hash set and hash map?
**Answer:** Hash set stores only keys (membership testing). Hash map stores key-value pairs (additional information). Use sets for existence checks, maps for associated data.

#### **Question 5:** Can hash maps handle custom objects as keys?
**Answer:** Yes, if objects are hashable (immutable and have __hash__ method in Python). Tuples work as keys, lists don't. This is useful for memoization and caching.

---

### **Problems**

---

#### **EASY: Two Sum**

**A. Problem Statement:**
Given an array of integers and a target, return indices of two numbers that add up to the target.

**B. ASCII Visualization:**
```
Array: [2, 7, 11, 15]  Target: 9

Pass through array:
Index 0: num=2, need 9-2=7
  HashMap: {2: 0}  (store for future)

Index 1: num=7, need 9-7=2
  HashMap: {2: 0, 7: 1}
  Check: Is 2 in map? YES! ✓
  Found pair: indices [0, 1]

Visual:
[2, 7, 11, 15]
 ↑  ↑
 |__|__ These sum to 9!
```

**C. Pattern Recognition Question:**
*Why do we store values in the hashmap BEFORE checking if complement exists?*

**D. Step-by-Step Reasoning:**
1. Initialize empty hashmap to store {number: index}
2. For each number in array:
   a. Calculate complement = target - current number
   b. Check if complement exists in hashmap
   c. If yes: return [hashmap[complement], current_index]
   d. If no: store current number and index in hashmap
3. Continue until pair found
4. Return empty if no pair exists (edge case)

**E. Python Solution:**
```python
def two_sum(nums, target):
    # HashMap to store {number: index}
    seen = {}
    
    for i, num in enumerate(nums):
        complement = target - num
        
        # Check if complement exists in map
        if complement in seen:
            return [seen[complement], i]
        
        # Store current number and its index
        seen[num] = i
    
    return []  # No solution found

# Example usage
print(two_sum([2, 7, 11, 15], 9))   # Output: [0, 1]
print(two_sum([3, 2, 4], 6))        # Output: [1, 2]
print(two_sum([3, 3], 6))           # Output: [0, 1]
```

**F. Memory Cue:**
*"Two Sum is SHOPPING with a BUDDY: you need $9 total, you have $2, check if buddy has $7 in their WALLET (hashmap)!"*

---

#### **HARD: Group Anagrams**

**A. Problem Statement:**
Given an array of strings, group anagrams together. Anagrams are words with same characters in different order.

**B. ASCII Visualization:**
```
Input: ["eat", "tea", "tan", "ate", "nat", "bat"]

Sorting each word as key:
"eat" → "aet"
"tea" → "aet"  ← Same key!
"tan" → "ant"
"ate" → "aet"  ← Same key!
"nat" → "ant"  ← Same key!
"bat" → "abt"

HashMap groups:
┌─────┬──────────────────┐
│ Key │ Anagram Group    │
├─────┼──────────────────┤
│"aet"│["eat","tea","ate"]│
│"ant"│["tan","nat"]     │
│"abt"│["bat"]           │
└─────┴──────────────────┘

Output: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

**C. Pattern Recognition Question:**
*Why is sorted string a good hash key for anagrams?*

**D. Step-by-Step Reasoning:**
1. Use hashmap with sorted string as key
2. For each word:
   a. Sort characters to create key
   b. Add word to list at that key
3. Anagrams automatically group (same sorted key)
4. Return all groups as list of lists
5. Alternative: Use character frequency as key (for optimization)

**E. Python Solution:**
```python
from collections import defaultdict

def group_anagrams(strs):
    # HashMap: sorted_string -> list of anagrams
    anagram_map = defaultdict(list)
    
    for word in strs:
        # Sort characters to create key
        # Anagrams will have same sorted key
        sorted_key = ''.join(sorted(word))
        
        # Add word to corresponding group
        anagram_map[sorted_key].append(word)
    
    # Return all anagram groups
    return list(anagram_map.values())

# Alternative: Using character frequency as key (faster)
def group_anagrams_optimized(strs):
    anagram_map = defaultdict(list)
    
    for word in strs:
        # Create frequency array as key
        count = [0] * 26  # 26 lowercase letters
        for char in word:
            count[ord(char) - ord('a')] += 1
        
        # Convert list to tuple (hashable)
        key = tuple(count)
        anagram_map[key].append(word)
    
    return list(anagram_map.values())

# Example usage
strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
print(group_anagrams(strs))
# Output: [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']]
```

**F. Memory Cue:**
*"Anagrams are PUZZLE PIECES from same picture—SORT the pieces to see they MATCH! HashMap is the PUZZLE BOX holding matching sets!"*

---

#### **VERY HARD: Longest Consecutive Sequence**

**A. Problem Statement:**
Given an unsorted array of integers, find the length of the longest consecutive elements sequence in O(n) time.

**B. ASCII Visualization:**
```
Array: [100, 4, 200, 1, 3, 2]

Add all to HashSet:
{100, 4, 200, 1, 3, 2}

Find sequence starts (no left neighbor):
- 100: Check 99? No → Start of sequence
  100 → No 101 (length = 1)

- 4: Check 3? Yes → Not a start (skip)

- 200: Check 199? No → Start of sequence
  200 → No 201 (length = 1)

- 1: Check 0? No → Start of sequence
  1 → 2 → 3 → 4 → 5? No
  [1, 2, 3, 4] (length = 4) ✓ Longest!

Visual:
100 (alone)
200 (alone)
1 → 2 → 3 → 4 (consecutive chain)

Result: 4
```

**C. Pattern Recognition Question:**
*Why do we only count sequences starting from numbers with no left neighbor?*

**D. Step-by-Step Reasoning:**
1. Add all numbers to hash set (O(n) for lookups)
2. For each number, check if it's a sequence start:
   - If (num - 1) exists, it's NOT a start (skip)
   - If (num - 1) doesn't exist, it IS a start
3. From each start, count consecutive numbers:
   - Check num+1, num+2, num+3... in set
   - Continue until gap found
4. Track maximum sequence length
5. Return maximum
6. Key insight: Each number checked at most twice (once as potential start, once in sequence)

**E. Python Solution:**
```python
def longest_consecutive(nums):
    if not nums:
        return 0
    
    # Add all numbers to set for O(1) lookup
    num_set = set(nums)
    max_length = 0
    
    for num in num_set:
        # Only start counting if this is the beginning of a sequence
        # (no left neighbor exists)
        if num - 1 not in num_set:
            current_num = num
            current_length = 1
            
            # Count consecutive numbers
            while current_num + 1 in num_set:
                current_num += 1
                current_length += 1
            
            # Update maximum length
            max_length = max(max_length, current_length)
    
    return max_length

# Example usage
print(longest_consecutive([100, 4, 200, 1, 3, 2]))  # Output: 4
print(longest_consecutive([0, 3, 7, 2, 5, 8, 4, 6, 0, 1]))  # Output: 9
```

**F. Memory Cue:**
*"Consecutive sequence is a TRAIN: only count from the ENGINE (no left neighbor), then follow the CARS (consecutive numbers) until CABOOSE!"*

---

### **Summary in 3 Sentences**

Hash maps enable O(1) lookups by trading space for time, perfect for tracking frequencies, positions, and existence checks. The pattern shines in complement searches (Two Sum), grouping by properties (anagrams), and detecting relationships between elements. Master knowing when to store values before or after checking, and how to choose appropriate keys (sorted strings, tuples, etc.).

---

## Pattern 6: Stack Pattern

> **Definition:** A Last-In-First-Out (LIFO) data structure where elements are added and removed from the same end, used for tracking nested structures, reversing order, and maintaining recent history.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Stacks naturally model nested/recursive structures and "undo" operations. The most recent item is always accessible, perfect for matching pairs or tracking state.
- 💭 **Intuition:** Like a stack of plates—you can only add or remove from the top. The last plate placed is the first one removed.
- 🔢 **Mathematical Reasoning:** LIFO ordering ensures proper nesting validation and reverse processing. Parentheses matching, function calls, and backtracking all follow this pattern.
- ⚡ **When to Use:** Parentheses matching, next greater element, expression evaluation, undo mechanisms, DFS traversal.

---

### **ASCII Diagram**

```
Stack Operations:
Empty: []

Push 1: [1]       ← Top
Push 2: [1, 2]    ← Top
Push 3: [1, 2, 3] ← Top

Pop:    [1, 2]    ← Returns 3
Pop:    [1]       ← Returns 2

Peek:   [1]       ← Returns 1 (doesn't remove)

Parentheses Matching:
String: "({[]})"

Process:
'(' → Push → ['(']
'{' → Push → ['(', '{']
'[' → Push → ['(', '{', '[']
']' → Pop '[' → Match ✓ → ['(', '{']
'}' → Pop '{' → Match ✓ → ['(']
')' → Pop '(' → Match ✓ → []

Empty stack = Valid! ✓

Stack for Reversing:
Input:  [1, 2, 3, 4]
Push all: [1, 2, 3, 4]
          ↑ bottom  ↑ top

Pop all:  [4, 3, 2, 1] (reversed!)
```

---

### **Memorization Technique**

🎬 **Visualization:** Picture a **TOWER OF PANCAKES**—you can only add pancakes on TOP and eat from the TOP. The last pancake placed is the first one eaten. No pulling from the middle!

💥 **Exaggeration:** Each pancake is LABELED with GLOWING NUMBERS. When you pop, the pancake SHOOTS UP like a SPRING with FIREWORKS! The stack keeps PERFECT MEMORY of what's inside.

🔗 **Association:** "STACK" = **S**tore **T**op **A**ccessible **C**heck **K**eep-Last-First

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Basic Stack** | Push, pop, peek operations | Parentheses validation |
| **Min/Max Stack** | Track minimum/maximum efficiently | Get min in O(1) |
| **Monotonic Stack** | Maintain increasing/decreasing order | Next greater element (Pattern 7) |
| **Expression Evaluation** | Process operators and operands | Calculator, postfix notation |

---

### **Socratic Teaching Round**

#### **Question 1:** Why can't we use a queue for parentheses matching?
**Answer:** Queues are FIFO (first in, first out). For matching nested structures, we need the MOST RECENT opening bracket to match the current closing bracket—that's LIFO (stack).

#### **Question 2:** What's the time complexity of stack operations?
**Answer:** Push, pop, and peek are all O(1) because we only access the top element. No searching or shifting required.

#### **Question 3:** How does a stack help with function call management?
**Answer:** Each function call pushes its context onto the call stack. When a function returns, its context pops off. Nested calls naturally follow LIFO order.

#### **Question 4:** Can we implement a stack using an array?
**Answer:** Yes! Array with a pointer to the top position. Push increments pointer, pop decrements. Dynamic arrays (Python lists) work perfectly.

#### **Question 5:** When should we use stack vs recursion?
**Answer:** They're equivalent! Recursion uses the system call stack. Explicit stacks give more control, avoid stack overflow, and let us pause/resume. Use recursion for clarity, stacks for iteration.

---

### **Problems**

---

#### **EASY: Valid Parentheses**

**A. Problem Statement:**
Given a string containing only '(', ')', '{', '}', '[', ']', determine if the input string is valid (all brackets properly closed and nested).

**B. ASCII Visualization:**
```
Example 1: "({[]})"  ✓ Valid

Stack progression:
'(' → ['(']
'{' → ['(', '{']
'[' → ['(', '{', '[']
']' → Pop '[' matches ✓ → ['(', '{']
'}' → Pop '{' matches ✓ → ['(']
')' → Pop '(' matches ✓ → []

Empty stack at end = Valid!

Example 2: "([)]"  ✗ Invalid

Stack progression:
'(' → ['(']
'[' → ['(', '[']
')' → Pop '[' NO MATCH ✗

Wrong closing bracket = Invalid!
```

**C. Pattern Recognition Question:**
*Why must the stack be empty at the end for valid parentheses?*

**D. Step-by-Step Reasoning:**
1. Create empty stack
2. Create mapping of closing to opening brackets
3. For each character:
   - If opening bracket: push to stack
   - If closing bracket: 
     a. Check if stack empty (no matching opening)
     b. Pop from stack and verify it matches
4. After processing all characters, stack must be empty
5. Return True if valid, False otherwise

**E. Python Solution:**
```python
def is_valid_parentheses(s):
    # Stack to track opening brackets
    stack = []
    
    # Mapping of closing to opening brackets
    closing_to_opening = {
        ')': '(',
        '}': '{',
        ']': '['
    }
    
    for char in s:
        if char in closing_to_opening:
            # Closing bracket found
            # Check if stack is empty or top doesn't match
            if not stack or stack[-1] != closing_to_opening[char]:
                return False
            stack.pop()  # Valid match, remove opening bracket
        else:
            # Opening bracket, push to stack
            stack.append(char)
    
    # Valid only if all brackets matched (stack empty)
    return len(stack) == 0

# Example usage
print(is_valid_parentheses("()"))        # Output: True
print(is_valid_parentheses("()[]{}"))    # Output: True
print(is_valid_parentheses("(]"))        # Output: False
print(is_valid_parentheses("([)]"))      # Output: False
print(is_valid_parentheses("{[]}"))      # Output: True
```

**F. Memory Cue:**
*"Opening brackets are DOORS opening into ROOMS (stack). Closing brackets must EXIT the MOST RECENT room first—LIFO!"*

---

#### **HARD: Min Stack**

**A. Problem Statement:**
Design a stack that supports push, pop, top, and retrieving the minimum element in O(1) time.

**B. ASCII Visualization:**
```
Operations on MinStack:

Push 5:
Main:  [5]
Min:   [5]  ← Current min

Push 3:
Main:  [5, 3]
Min:   [5, 3]  ← New min

Push 7:
Main:  [5, 3, 7]
Min:   [5, 3, 3]  ← Keep tracking min

Push 2:
Main:  [5, 3, 7, 2]
Min:   [5, 3, 3, 2]  ← New minimum!

getMin() → 2 (peek min stack)

Pop:
Main:  [5, 3, 7]
Min:   [5, 3, 3]  ← Min reverts to 3

getMin() → 3
```

**C. Pattern Recognition Question:**
*Why do we need a second stack to track minimums?*

**D. Step-by-Step Reasoning:**
1. Use two stacks: main stack and min stack
2. Main stack: stores all values normally
3. Min stack: tracks minimum at each level
4. Push: Add to main stack, add min(new_val, current_min) to min stack
5. Pop: Remove from both stacks
6. GetMin: Peek top of min stack (O(1))
7. Alternative: Store (value, min) pairs in single stack

**E. Python Solution:**
```python
class MinStack:
    def __init__(self):
        # Main stack stores values
        self.stack = []
        # Min stack stores minimum at each level
        self.min_stack = []
    
    def push(self, val):
        # Always push to main stack
        self.stack.append(val)
        
        # Push to min stack: either new val or current min
        if not self.min_stack:
            self.min_stack.append(val)
        else:
            self.min_stack.append(min(val, self.min_stack[-1]))
    
    def pop(self):
        # Pop from both stacks
        if self.stack:
            self.stack.pop()
            self.min_stack.pop()
    
    def top(self):
        # Return top of main stack
        return self.stack[-1] if self.stack else None
    
    def getMin(self):
        # Return top of min stack (current minimum)
        return self.min_stack[-1] if self.min_stack else None

# Example usage
min_stack = MinStack()
min_stack.push(-2)
min_stack.push(0)
min_stack.push(-3)
print(min_stack.getMin())  # Output: -3
min_stack.pop()
print(min_stack.top())     # Output: 0
print(min_stack.getMin())  # Output: -2
```

**F. Memory Cue:**
*"MinStack is TWIN TOWERS: one tower holds VALUES, the other holds the CHAMPION (minimum) at each floor!"*

---

#### **VERY HARD: Basic Calculator**

**A. Problem Statement:**
Implement a basic calculator to evaluate a string expression containing digits, '+', '-', '(', ')' and spaces.

**B. ASCII Visualization:**
```
Expression: "2 + (3 - (4 + 5))"

Stack-based evaluation:
Start: result=0, sign=+1, stack=[]

'2': result = 0 + 1*2 = 2

'+': sign = +1

'(': Push (result=2, sign=+1) to stack
     Reset result=0, sign=+1

'3': result = 0 + 1*3 = 3

'-': sign = -1

'(': Push (result=3, sign=-1) to stack
     Reset result=0, sign=+1

'4': result = 0 + 1*4 = 4

'+': sign = +1

'5': result = 4 + 1*5 = 9

')': Pop (3, -1): result = 3 + (-1)*9 = -6

')': Pop (2, +1): result = 2 + (+1)*(-6) = -4

Final: -4 ✓
```

**C. Pattern Recognition Question:**
*How does the stack help us handle nested parentheses?*

**D. Step-by-Step Reasoning:**
1. Use stack to save state when entering parentheses
2. Track current result and sign
3. For each character:
   - Digit: Build number, apply sign when complete
   - '+' or '-': Update sign
   - '(': Push current (result, sign) to stack, reset
   - ')': Pop from stack, apply to inner result
4. Process number when hitting operator or closing paren
5. Return final result

**E. Python Solution:**
```python
def calculate(s):
    stack = []
    result = 0
    sign = 1  # 1 for positive, -1 for negative
    num = 0
    
    for i, char in enumerate(s):
        if char.isdigit():
            # Build multi-digit number
            num = num * 10 + int(char)
        
        elif char in ['+', '-']:
            # Apply previous number with its sign
            result += sign * num
            num = 0
            # Update sign for next number
            sign = 1 if char == '+' else -1
        
        elif char == '(':
            # Save current result and sign to stack
            stack.append(result)
            stack.append(sign)
            # Reset for expression inside parentheses
            result = 0
            sign = 1
        
        elif char == ')':
            # Complete current number
            result += sign * num
            num = 0
            # Pop sign and previous result
            result *= stack.pop()  # Multiply by sign before '('
            result += stack.pop()  # Add result before '('
    
    # Add last number if any
    result += sign * num
    
    return result

# Example usage
print(calculate("1 + 1"))              # Output: 2
print(calculate(" 2-1 + 2 "))          # Output: 3
print(calculate("(1+(4+5+2)-3)+(6+8)"))  # Output: 23
print(calculate("2 + (3 - (4 + 5))"))    # Output: -4
```

**F. Memory Cue:**
*"Calculator stack is TIME TRAVEL: each '(' SAVES your progress (checkpoint), each ')' LOADS checkpoint and CONTINUES!"*

---

### **Summary in 3 Sentences**

Stacks provide LIFO access perfect for nested structures, matching pairs, and reversing operations. The pattern excels at validating parentheses, tracking state during traversal, and evaluating expressions. Master when to push (save state/opening bracket), when to pop (match closing/restore state), and how auxiliary stacks track metadata (like minimums).

---

## Pattern 7: Monotonic Stack

> **Definition:** A stack that maintains elements in monotonically increasing or decreasing order by popping elements that violate the order when pushing new elements.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Monotonic stacks efficiently find the next greater/smaller element for all positions in O(n) time by maintaining order and eliminating impossible candidates.
- 💭 **Intuition:** Imagine people standing in line by height. When a taller person arrives, everyone shorter behind them can't be "next taller" anymore—remove them!
- 🔢 **Mathematical Reasoning:** By maintaining monotonic order, each element is pushed and popped at most once, guaranteeing O(n) complexity instead of O(n²) brute force.
- ⚡ **When to Use:** Next greater/smaller element, temperature problems, histogram areas, stock span, visibility problems.

---

### **ASCII Diagram**

```
Monotonic Decreasing Stack (for Next Greater Element):

Array: [2, 1, 2, 4, 3]
Goal: Find next greater element for each position

Process:
2: Stack [] → Push 2 → [2]

1: Stack [2] → 1 < 2 (maintains decreasing) → [2, 1]

2: Stack [2, 1] → 2 > 1 (violates!)
   Pop 1: next greater for 1 is 2 ✓
   2 = 2 (equal, push) → [2, 2]

4: Stack [2, 2] → 4 > 2 (violates!)
   Pop 2: next greater for 2 is 4 ✓
   Pop 2: next greater for 2 is 4 ✓
   Push 4 → [4]

3: Stack [4] → 3 < 4 (maintains decreasing) → [4, 3]

Result: [4, 2, 4, -1, -1]
         ↑  ↑  ↑   ↑   ↑
         Each element's next greater

Visual:
       4
     /   \
    2     3
   / \
  2   1

Stack maintains "potential next greater" candidates
```

---

### **Memorization Technique**

🎬 **Visualization:** Picture a **MOUNTAIN RANGE where peaks are JEALOUS**. When a NEW TALL PEAK arrives, it KICKS OUT all shorter peaks behind it because they can NEVER be "next taller" anymore!

💥 **Exaggeration:** Each number is a TOWER with GROWING AMBITIONS. Taller towers BULLDOZE smaller ones in their path. The stack is a SKYLINE that only keeps DOMINANT towers!

🔗 **Association:** "MONOTONIC" = **M**aintain **O**rder **N**o **O**utsiders **T**opple **O**lder **N**umbers **I**f **C**ompeting

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Increasing Stack** | Maintain increasing order (find next smaller) | Stock span |
| **Decreasing Stack** | Maintain decreasing order (find next greater) | Next greater element |
| **Index Storage** | Store indices instead of values | Temperature span |
| **Circular Array** | Handle wraparound with modulo | Next greater in circular array |

---

### **Socratic Teaching Round**

#### **Question 1:** Why does monotonic stack work in O(n) time?
**Answer:** Each element is pushed exactly once and popped at most once. Total operations = 2n pushes/pops = O(n).

#### **Question 2:** When do we use increasing vs decreasing stack?
**Answer:** **Decreasing stack**: Find next GREATER element (pop when current > top). **Increasing stack**: Find next SMALLER element (pop when current < top).

#### **Question 3:** Should we store values or indices in the stack?
**Answer:** **Indices** are more versatile: we can access values via indices, calculate distances/spans, and handle duplicate values. Use indices unless only values matter.

#### **Question 4:** How do we handle elements with no next greater/smaller?
**Answer:** Elements remaining in stack after processing all elements have no answer. Assign default (-1, infinity, or array length depending on problem).

#### **Question 5:** What's the difference between monotonic stack and regular stack?
**Answer:** Regular stack: no ordering constraint. Monotonic stack: actively maintains order by popping violating elements. This ordering enables O(n) next greater/smaller queries.

---

### **Problems**

---

#### **EASY: Next Greater Element I**

**A. Problem Statement:**
Given two arrays nums1 and nums2 where nums1 is a subset of nums2, find the next greater element for each element in nums1 in nums2.

**B. ASCII Visualization:**
```
nums1 = [4, 1, 2]
nums2 = [1, 3, 4, 2]

Build next greater map for nums2:
Process nums2 with decreasing stack:

1: Stack [] → [1]
3: Stack [1] → 3>1, pop 1 (next greater of 1 is 3) → [3]
4: Stack [3] → 4>3, pop 3 (next greater of 3 is 4) → [4]
2: Stack [4] → 2<4, push → [4, 2]

Map: {1→3, 3→4, 4→-1, 2→-1}

Query nums1: [4, 1, 2]
Result: [-1, 3, -1]
```

**C. Pattern Recognition Question:**
*Why do we build a hashmap instead of querying the stack repeatedly?*

**D. Step-by-Step Reasoning:**
1. Process nums2 with monotonic decreasing stack
2. For each element, pop all smaller elements (they found their next greater)
3. Store mappings in hashmap: {element → next_greater}
4. Query nums1 elements from hashmap
5. Return -1 for elements with no next greater

**E. Python Solution:**
```python
def next_greater_element(nums1, nums2):
    # Build next greater map for nums2
    next_greater = {}
    stack = []
    
    for num in nums2:
        # Pop all smaller elements (num is their next greater)
        while stack and stack[-1] < num:
            smaller = stack.pop()
            next_greater[smaller] = num
        
        stack.append(num)
    
    # Remaining elements have no next greater
    for num in stack:
        next_greater[num] = -1
    
    # Build result for nums1
    return [next_greater[num] for num in nums1]

# Example usage
print(next_greater_element([4,1,2], [1,3,4,2]))  # Output: [-1, 3, -1]
print(next_greater_element([2,4], [1,2,3,4]))    # Output: [3, -1]
```

**F. Memory Cue:**
*"Monotonic stack is a TALENT SHOW: taller talent ELIMINATES shorter ones. Hashmap is the SCOREBOARD showing who beat whom!"*

---

#### **HARD: Daily Temperatures**

**A. Problem Statement:**
Given array of daily temperatures, return array where each element is the number of days until a warmer temperature. If no warmer day, use 0.

**B. ASCII Visualization:**
```
Temperatures: [73, 74, 75, 71, 69, 72, 76, 73]
              0   1   2   3   4   5   6   7

Stack stores indices for temperature comparison:

i=0, T=73: Stack [] → [0]

i=1, T=74: 74>73, pop 0
          answer[0] = 1-0 = 1 ✓
          Stack [1]

i=2, T=75: 75>74, pop 1
          answer[1] = 2-1 = 1 ✓
          Stack [2]

i=3, T=71: 71<75 → [2, 3]

i=4, T=69: 69<71 → [2, 3, 4]

i=5, T=72: 72>69, pop 4 (answer[4] = 5-4 = 1)
          72>71, pop 3 (answer[3] = 5-3 = 2)
          72<75 → [2, 5]

i=6, T=76: 76>72, pop 5 (answer[5] = 6-5 = 1)
          76>75, pop 2 (answer[2] = 6-2 = 4)
          Stack [6]

i=7, T=73: 73<76 → [6, 7]

Result: [1, 1, 4, 2, 1, 1, 0, 0]
```

**C. Pattern Recognition Question:**
*Why do we store indices instead of temperatures in the stack?*

**D. Step-by-Step Reasoning:**
1. Initialize result array with zeros
2. Use monotonic decreasing stack (stores indices)
3. For each day:
   - While current temp > stack top temp: pop and calculate span
   - Span = current_index - popped_index
   - Push current index to stack
4. Remaining indices have no warmer day (already 0)
5. Return result array

**E. Python Solution:**
```python
def daily_temperatures(temperatures):
    n = len(temperatures)
    answer = [0] * n
    stack = []  # Monotonic stack storing indices
    
    for i in range(n):
        current_temp = temperatures[i]
        
        # Pop all indices with cooler temperatures
        while stack and temperatures[stack[-1]] < current_temp:
            prev_index = stack.pop()
            # Calculate days until warmer temperature
            answer[prev_index] = i - prev_index
        
        # Push current index
        stack.append(i)
    
    # Remaining indices have no warmer day (answer already 0)
    return answer

# Example usage
temps = [73, 74, 75, 71, 69, 72, 76, 73]
print(daily_temperatures(temps))  # Output: [1, 1, 4, 2, 1, 1, 0, 0]
```

**F. Memory Cue:**
*"Temperature stack is a WAITING LIST: people wait for WARMER days. When warm day comes, calculate WAIT TIME (index difference)!"*

---

#### **VERY HARD: Largest Rectangle in Histogram**

**A. Problem Statement:**
Given array representing histogram bar heights, find the area of the largest rectangle that can be formed.

**B. ASCII Visualization:**
```
Heights: [2, 1, 5, 6, 2, 3]

Histogram:
      6
    5 ▓
    ▓ ▓     3
  2 ▓ ▓ 2 ▓
  ▓ ▓ ▓ ▓ ▓
  ▓ 1 ▓ ▓ ▓
  0 1 2 3 4 5 (indices)

Largest rectangle: height=5, width=2 (indices 2-3)
Area = 5 * 2 = 10

Monotonic Stack Approach:
Keep stack increasing (heights)

i=0, h=2: [] → [0]
i=1, h=1: 1<2, pop 0
          width = 1-(-1)-1 = 1
          area = 2*1 = 2
          [1]
i=2, h=5: [1, 2]
i=3, h=6: [1, 2, 3]
i=4, h=2: 2<6, pop 3
          width = 4-2-1 = 1
          area = 6*1 = 6
          2<5, pop 2
          width = 4-1-1 = 2
          area = 5*2 = 10 ✓ Max!
          [1, 4]
i=5, h=3: [1, 4, 5]
```

**C. Pattern Recognition Question:**
*Why does popping an element from the stack represent calculating a rectangle with that height?*

**D. Step-by-Step Reasoning:**
1. Use monotonic increasing stack (stores indices)
2. For each bar:
   - While current height < stack top height:
     - Pop index (this height can't extend further right)
     - Width = current_index - left_boundary - 1
     - Left boundary = new stack top (or -1 if empty)
     - Calculate area = height * width
   - Push current index
3. After processing all bars, pop remaining (extend to end)
4. Track maximum area throughout
5. Key insight: Popped bar is shortest in its width range

**E. Python Solution:**
```python
def largest_rectangle_area(heights):
    max_area = 0
    stack = []  # Monotonic increasing stack (stores indices)
    
    for i, h in enumerate(heights):
        # Maintain increasing stack
        while stack and heights[stack[-1]] > h:
            # Pop: this height can't extend further
            height_index = stack.pop()
            height = heights[height_index]
            
            # Calculate width
            # Left boundary is previous element in stack (or -1)
            left_boundary = stack[-1] if stack else -1
            width = i - left_boundary - 1
            
            # Calculate area
            area = height * width
            max_area = max(max_area, area)
        
        stack.append(i)
    
    # Process remaining bars (extend to end)
    while stack:
        height_index = stack.pop()
        height = heights[height_index]
        left_boundary = stack[-1] if stack else -1
        width = len(heights) - left_boundary - 1
        area = height * width
        max_area = max(max_area, area)
    
    return max_area

# Example usage
print(largest_rectangle_area([2, 1, 5, 6, 2, 3]))  # Output: 10
print(largest_rectangle_area([2, 4]))               # Output: 4
```

**F. Memory Cue:**
*"Histogram stack is BUILDING BLOCKS: each block STRETCHES as far as possible until TALLER block STOPS it. Pop = measure that block's KINGDOM!"*

---

### **Summary in 3 Sentences**

Monotonic stacks maintain order by popping elements that violate monotonicity, efficiently solving next greater/smaller problems in O(n) time. Use decreasing stack for next greater elements, increasing stack for next smaller elements. Master storing indices to calculate distances and understanding that popping represents finding the answer for that element.

---

## Pattern 8: Binary Search

> **Definition:** A divide-and-conquer algorithm that repeatedly halves the search space by comparing the target with the middle element in a sorted array, achieving O(log n) time complexity.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Sorted data enables elimination of half the search space at each step. Instead of checking every element (O(n)), we make logarithmic comparisons.
- 💭 **Intuition:** Like opening a dictionary—you don't start at page 1. You open the middle, see if your word comes before or after, then repeat on the relevant half.
- 🔢 **Mathematical Reasoning:** Each comparison halves the remaining elements: n → n/2 → n/4 → ... → 1. Total steps = log₂(n).
- ⚡ **When to Use:** Searching in sorted arrays, finding boundaries, optimization problems, rotated arrays, matrix search.

---

### **ASCII Diagram**

```
Binary Search Visualization:
Array: [1, 3, 5, 7, 9, 11, 13, 15]  Target: 9

Step 1: Check middle
[1, 3, 5, 7, | 9, 11, 13, 15]
            mid=7
7 < 9 → Search right half

Step 2: Check middle of right half
              [9, 11, | 13, 15]
                      mid=11
11 > 9 → Search left half

Step 3: Check middle of remaining
              [9]
              mid=9
9 == 9 → Found! ✓

Binary Search Tree (Decision Flow):
           [1...15]
              ↓
         mid=7 (< 9)
              ↓
          [9...15]
              ↓
        mid=11 (> 9)
              ↓
            [9]
            ↓
          FOUND!

Left/Right Pointer Movement:
Start:  L                    R
        [1, 3, 5, 7, 9, 11, 13, 15]
               M (7 < 9)
        
After:           L           R
                [9, 11, 13, 15]
                    M (11 > 9)
        
Final:           L=R
                 [9]
                Found!
```

---

### **Memorization Technique**

🎬 **Visualization:** Picture a **PHONE BOOK falling open**. You see a name, realize yours is EARLIER/LATER, and TEAR THE BOOK IN HALF, throwing away the wrong half. Repeat until ONE PAGE remains!

💥 **Exaggeration:** Each half you throw makes a GIANT EXPLOSION. The book SHRINKS EXPONENTIALLY with GLOWING EDGES showing the search space. Final page SPARKLES!

🔗 **Association:** "BINARY SEARCH" = **B**isect **I**ntervals **N**arrow **A**rea **R**apidly **Y**ielding → **S**plit **E**liminate **A**im **R**ecursive **C**hopping **H**alves

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Exact Match** | Find specific target | Classic binary search |
| **First/Last Occurrence** | Find boundaries in duplicates | Find first/last position |
| **Closest Element** | Find nearest value | Find closest to target |
| **Rotated Array** | Handle rotated sorted array | Search in rotated array |
| **2D Matrix** | Binary search in matrix | Search 2D matrix |

---

### **Socratic Teaching Round**

#### **Question 1:** Why does binary search require a sorted array?
**Answer:** Sorting provides ordering information. When we compare with mid element, we know ALL elements left are smaller, ALL elements right are larger. This enables safe elimination of entire halves.

#### **Question 2:** What's the most common bug in binary search?
**Answer:** Integer overflow when calculating mid: `(left + right) / 2` can overflow. Use `left + (right - left) / 2` or `(left + right) >> 1` instead.

#### **Question 3:** How do we handle the case when target doesn't exist?
**Answer:** Loop terminates when left > right. Return -1 or the position where element should be inserted (left pointer position).

#### **Question 4:** When do we use `left <= right` vs `left < right`?
**Answer:** `left <= right`: Check all elements including when left==right. `left < right`: Stop when one element remains (useful for finding boundaries).

#### **Question 5:** Can binary search work on answer spaces (not arrays)?
**Answer:** Yes! If we can check "is X achievable" in some time, we can binary search the range of X. Called "binary search on answer" (Pattern 9).

---

### **Problems**

---

#### **EASY: Binary Search (Classic)**

**A. Problem Statement:**
Given a sorted array and a target value, return the index of the target if found, otherwise return -1.

**B. ASCII Visualization:**
```
Array: [-1, 0, 3, 5, 9, 12]  Target: 9

Iteration 1:
L                      R
[-1, 0, 3, 5, 9, 12]
         M (mid=3, val=5)
5 < 9 → Search right

Iteration 2:
            L         R
         [5, 9, 12]
             M (mid=4, val=9)
9 == 9 → Found at index 4! ✓
```

**C. Pattern Recognition Question:**
*Why do we update left = mid + 1 (not left = mid) when going right?*

**D. Step-by-Step Reasoning:**
1. Initialize left = 0, right = len(array) - 1
2. While left <= right:
   a. Calculate mid = left + (right - left) // 2
   b. If array[mid] == target: return mid
   c. If array[mid] < target: search right half (left = mid + 1)
   d. If array[mid] > target: search left half (right = mid - 1)
3. If not found, return -1
4. Key: Update left/right to mid ± 1 (we already checked mid)

**E. Python Solution:**
```python
def binary_search(nums, target):
    left = 0
    right = len(nums) - 1
    
    while left <= right:
        # Avoid integer overflow
        mid = left + (right - left) // 2
        
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            # Search right half
            left = mid + 1
        else:
            # Search left half
            right = mid - 1
    
    # Target not found
    return -1

# Example usage
print(binary_search([-1, 0, 3, 5, 9, 12], 9))   # Output: 4
print(binary_search([-1, 0, 3, 5, 9, 12], 2))   # Output: -1
```

**F. Memory Cue:**
*"Binary search is SCISSORS cutting paper: each cut HALVES the problem until you find the TARGET piece!"*

---

#### **HARD: Find First and Last Position**

**A. Problem Statement:**
Given a sorted array with duplicates and a target value, find the starting and ending position of the target. Return [-1, -1] if not found.

**B. ASCII Visualization:**
```
Array: [5, 7, 7, 8, 8, 8, 10]  Target: 8
                 ↑     ↑
              first  last

Finding First Occurrence:
L                       R
[5, 7, 7, 8, 8, 8, 10]
         M (val=8, found but keep searching left)
         
L           R
[5, 7, 7, 8]
      M (val=7 < 8, search right)
      
      L     R
      [7, 8]
         M (val=8, found, keep searching left)
         
      L=R
      [8]
      First = 3 ✓

Finding Last Occurrence:
(Similar process, but search RIGHT when found)
Last = 5 ✓

Result: [3, 5]
```

**C. Pattern Recognition Question:**
*Why do we continue binary searching even after finding the target?*

**D. Step-by-Step Reasoning:**
1. Use two binary searches:
   - First: Find leftmost occurrence (continue left when found)
   - Last: Find rightmost occurrence (continue right when found)
2. For first occurrence:
   - When found: right = mid - 1 (keep searching left)
   - Track potential answer
3. For last occurrence:
   - When found: left = mid + 1 (keep searching right)
   - Track potential answer
4. Return [first, last] or [-1, -1] if not found

**E. Python Solution:**
```python
def search_range(nums, target):
    def find_first():
        left, right = 0, len(nums) - 1
        first = -1
        
        while left <= right:
            mid = left + (right - left) // 2
            
            if nums[mid] == target:
                first = mid  # Found, but keep searching left
                right = mid - 1
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1
        
        return first
    
    def find_last():
        left, right = 0, len(nums) - 1
        last = -1
        
        while left <= right:
            mid = left + (right - left) // 2
            
            if nums[mid] == target:
                last = mid  # Found, but keep searching right
                left = mid + 1
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1
        
        return last
    
    first = find_first()
    if first == -1:
        return [-1, -1]
    
    last = find_last()
    return [first, last]

# Example usage
print(search_range([5, 7, 7, 8, 8, 10], 8))  # Output: [3, 4]
print(search_range([5, 7, 7, 8, 8, 10], 6))  # Output: [-1, -1]
```

**F. Memory Cue:**
*"Finding boundaries is like FINDING EDGES of a LAKE: keep searching LEFT for west shore, keep searching RIGHT for east shore!"*

---

#### **VERY HARD: Search in Rotated Sorted Array**

**A. Problem Statement:**
Given a rotated sorted array (rotated at unknown pivot), find a target value. Array has no duplicates.

**B. ASCII Visualization:**
```
Original: [1, 2, 3, 4, 5, 6, 7]
Rotated:  [4, 5, 6, 7, 1, 2, 3]  (rotated at index 4)
           ↑ pivot

Target: 1

Key Insight: One half is always sorted!

Step 1: Check middle
[4, 5, 6, | 7, 1, 2, 3]
         mid=7

Left half [4,5,6,7] is sorted (4<7)
Is target in [4,7]? No (1 not in range)
→ Search right half

Step 2: Check middle of right half
          [1, | 2, 3]
           mid=2

Left half [1] is sorted
Is target in [1,2]? Yes! (1 in range)
→ Search left half

Step 3:
          [1]
          Found! ✓

Decision Tree:
If left half sorted:
  If target in left range → search left
  Else → search right
If right half sorted:
  If target in right range → search right
  Else → search left
```

**C. Pattern Recognition Question:**
*How do we determine which half is properly sorted after rotation?*

**D. Step-by-Step Reasoning:**
1. At least one half is always sorted (key insight!)
2. Compare nums[left] with nums[mid]:
   - If nums[left] <= nums[mid]: left half is sorted
   - Else: right half is sorted
3. Check if target is in the sorted half's range
4. If yes: search that half
5. If no: search the other half
6. Handle edge cases (target equals mid, left, or right)

**E. Python Solution:**
```python
def search_rotated(nums, target):
    left, right = 0, len(nums) - 1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        # Found target
        if nums[mid] == target:
            return mid
        
        # Determine which half is sorted
        if nums[left] <= nums[mid]:
            # Left half is sorted
            if nums[left] <= target < nums[mid]:
                # Target in left sorted range
                right = mid - 1
            else:
                # Target in right half
                left = mid + 1
        else:
            # Right half is sorted
            if nums[mid] < target <= nums[right]:
                # Target in right sorted range
                left = mid + 1
            else:
                # Target in left half
                right = mid - 1
    
    return -1  # Target not found

# Example usage
print(search_rotated([4, 5, 6, 7, 0, 1, 2], 0))  # Output: 4
print(search_rotated([4, 5, 6, 7, 0, 1, 2], 3))  # Output: -1
print(search_rotated([1], 0))                     # Output: -1
```

**F. Memory Cue:**
*"Rotated array is a BROKEN STAIRCASE: one side is INTACT (sorted), climb that side if target is there, else try the BROKEN side!"*

---

### **Summary in 3 Sentences**

Binary search eliminates half the search space at each step by comparing with the middle element, achieving O(log n) efficiency on sorted data. The pattern extends to finding boundaries (first/last occurrences), handling rotations, and searching 2D matrices. Master the core invariant: at each step, identify which half contains the target based on comparisons and ordering properties.

---

## Pattern 9: Binary Search on Answer

> **Definition:** Using binary search not on an array, but on the range of possible answers, testing each candidate answer's feasibility to find the optimal solution.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** When direct calculation is hard but verification is easy, we can binary search the answer space. If answer X works, try smaller; if X fails, try larger.
- 💭 **Intuition:** Like the "guess my number" game with hints of "too high" or "too low"—binary search finds the answer efficiently.
- 🔢 **Mathematical Reasoning:** Many optimization problems have monotonic properties: if X works, all values > X work (or vice versa). This monotonicity enables binary search.
- ⚡ **When to Use:** "Minimize maximum," "maximize minimum," capacity/allocation problems, time-bound problems, feasibility checks.

---

### **ASCII Diagram**

```
Problem: Split array into m subarrays, minimize largest sum

Array: [7, 2, 5, 10, 8]  m = 2

Answer space: [max_element, sum_of_all]
              [10, 32]

Binary Search on Answer:
Low=10                    High=32
        Mid=21
Can split with max_sum ≤ 21?
[7, 2, 5] = 14 ✓
[10, 8] = 18 ✓
Yes! Try smaller answer

Low=10     High=21
     Mid=15
Can split with max_sum ≤ 15?
[7, 2, 5] = 14 ✓
[10] = 10 ✓
[8] = 8 ✓
Needs 3 subarrays (> m=2) ✗
Try larger answer

Low=16  High=21
    Mid=18
Can split with max_sum ≤ 18?
[7, 2, 5] = 14 ✓
[10, 8] = 18 ✓
Yes! Answer = 18 ✓

Feasibility Function:
For each candidate answer, check if achievable
If achievable → good candidate, try better
If not achievable → need larger value
```

---

### **Memorization Technique**

🎬 **Visualization:** Picture a **BUDGET NEGOTIATION**: you propose a budget (candidate answer), client says "too high" or "acceptable." You binary search the PRICE RANGE until finding the MINIMUM ACCEPTABLE price!

💥 **Exaggeration:** Each budget proposal is a NEON NUMBER on a GIANT SCALE. The scale GLOWS GREEN (feasible) or RED (infeasible). You're ZOOMING through numbers at LIGHTNING SPEED!

🔗 **Association:** "BINARY SEARCH ON ANSWER" = **B**udget **I**s **N**egotiated **A**djusting **R**ange **Y**ielding → **S**olution **O**ptimal **A**nswer

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Minimize Maximum** | Find minimum of maximum values | Split array, allocate tasks |
| **Maximize Minimum** | Find maximum of minimum values | Aggressive cows |
| **Capacity Problems** | Find minimum capacity needed | Shipping packages |
| **Time-bound** | Find minimum time to complete task | Koko eating bananas |

---

### **Socratic Teaching Round**

#### **Question 1:** How do we know when to use binary search on answer?
**Answer:** Look for keywords: "minimize the maximum," "maximize the minimum," and when you can VERIFY a solution but can't COMPUTE it directly. If checking "is X feasible?" is easier than "what's the optimal X?", use this pattern.

#### **Question 2:** What's the feasibility function?
**Answer:** A helper function that tests "given answer X, is it achievable?" Returns True/False. This function depends on the problem and is usually the core logic.

#### **Question 3:** How do we set the initial search range?
**Answer:** **Lower bound**: Minimum possible answer (often max element, 0, or 1). **Upper bound**: Maximum possible answer (often sum of all, length, or theoretical max).

#### **Question 4:** When do we move left vs right?
**Answer:** **If feasible**: Current answer works, try better (smaller for minimize, larger for maximize). **If infeasible**: Current answer fails, need worse (larger for minimize, smaller for maximize).

#### **Question 5:** Why is this called "binary search on answer" not "binary search on array"?
**Answer:** We're not searching an array—we're searching the SPACE of possible answers. The "array" is conceptual: [min_possible_answer ... max_possible_answer].

---

### **Problems**

---

#### **EASY: Koko Eating Bananas**

**A. Problem Statement:**
Koko has n piles of bananas with pile[i] bananas. Guards return in h hours. Find minimum eating speed k (bananas/hour) so she can eat all bananas within h hours.

**B. ASCII Visualization:**
```
Piles: [3, 6, 7, 11]  h = 8 hours

Question: Minimum k (bananas/hour)?

Try k=4:
Pile 3: ⌈3/4⌉ = 1 hour
Pile 6: ⌈6/4⌉ = 2 hours
Pile 7: ⌈7/4⌉ = 2 hours
Pile 11: ⌈11/4⌉ = 3 hours
Total: 1+2+2+3 = 8 hours ✓ (exactly h)

Try k=3:
Pile 3: ⌈3/3⌉ = 1 hour
Pile 6: ⌈6/3⌉ = 2 hours
Pile 7: ⌈7/3⌉ = 3 hours
Pile 11: ⌈11/3⌉ = 4 hours
Total: 1+2+3+4 = 10 hours ✗ (> h)

Binary search range: [1, max(piles)]
                      [1, 11]

k=1,2,3 → Too slow (> h hours)
k=4,5,6...11 → Fast enough (≤ h hours)
Answer: k=4 (minimum that works)
```

**C. Pattern Recognition Question:**
*Why is the answer space monotonic (if k works, k+1 also works)?*

**D. Step-by-Step Reasoning:**
1. Search range: [1, max(piles)]
2. Feasibility function: Can finish in h hours at speed k?
   - For each pile: hours = ceil(pile / k)
   - Sum all hours, check if ≤ h
3. Binary search:
   - If feasible: Try smaller k (right = mid - 1)
   - If infeasible: Need larger k (left = mid + 1)
4. Return left (minimum feasible speed)

**E. Python Solution:**
```python
import math

def min_eating_speed(piles, h):
    def can_finish(k):
        """Check if Koko can eat all bananas at speed k within h hours"""
        hours = 0
        for pile in piles:
            # Ceiling division: hours to eat this pile
            hours += math.ceil(pile / k)
        return hours <= h
    
    # Binary search on answer (eating speed)
    left = 1
    right = max(piles)
    
    while left < right:
        mid = left + (right - left) // 2
        
        if can_finish(mid):
            # mid works, try smaller speed
            right = mid
        else:
            # mid too slow, need faster
            left = mid + 1
    
    return left

# Example usage
print(min_eating_speed([3, 6, 7, 11], 8))  # Output: 4
print(min_eating_speed([30, 11, 23, 4, 20], 5))  # Output: 30
```

**F. Memory Cue:**
*"Koko's eating speed is DIAL on OVEN: turn it UP if too slow, turn it DOWN if fast enough. Binary search finds MINIMUM working temperature!"*

---

#### **HARD: Split Array Largest Sum**

**A. Problem Statement:**
Given array and integer m, split array into m non-empty subarrays. Minimize the largest sum among these subarrays.

**B. ASCII Visualization:**
```
Array: [7, 2, 5, 10, 8]  m = 2

All possible splits:
[7] [2, 5, 10, 8] → max(7, 25) = 25
[7, 2] [5, 10, 8] → max(9, 23) = 23
[7, 2, 5] [10, 8] → max(14, 18) = 18 ✓ Minimum!
[7, 2, 5, 10] [8] → max(24, 8) = 24

Answer space: [10, 32]  (max element to sum of all)

Binary search: Is max_sum = X achievable with m splits?

X = 21:
Greedy split: [7,2,5]=14, [10,8]=18
Both ≤ 21, used 2 splits ✓ Feasible

X = 15:
Greedy split: [7,2,5]=14, [10]=10, [8]=8
Used 3 splits > m=2 ✗ Infeasible

X = 18:
Greedy split: [7,2,5]=14, [10,8]=18
Both ≤ 18, used 2 splits ✓ Answer!
```

**C. Pattern Recognition Question:**
*Why does the greedy approach (keep adding to current subarray until exceeding limit) work for the feasibility check?*

**D. Step-by-Step Reasoning:**
1. Search range: [max(array), sum(array)]
2. Feasibility function: Can split with max_sum ≤ X using ≤ m subarrays?
   - Greedy: Keep adding elements to current subarray
   - If adding exceeds X, start new subarray
   - Count total subarrays needed
3. Binary search:
   - If feasible (≤ m subarrays): Try smaller X (right = mid)
   - If infeasible (> m subarrays): Need larger X (left = mid + 1)
4. Return left (minimum max_sum)

**E. Python Solution:**
```python
def split_array(nums, m):
    def can_split(max_sum):
        """Check if we can split into ≤ m subarrays with each sum ≤ max_sum"""
        subarrays = 1
        current_sum = 0
        
        for num in nums:
            if current_sum + num > max_sum:
                # Start new subarray
                subarrays += 1
                current_sum = num
                
                if subarrays > m:
                    return False
            else:
                current_sum += num
        
        return True
    
    # Binary search on answer (max subarray sum)
    left = max(nums)  # Minimum possible (each element separate)
    right = sum(nums)  # Maximum possible (all in one subarray)
    
    while left < right:
        mid = left + (right - left) // 2
        
        if can_split(mid):
            # mid works, try smaller
            right = mid
        else:
            # mid too small, need larger
            left = mid + 1
    
    return left

# Example usage
print(split_array([7, 2, 5, 10, 8], 2))  # Output: 18
print(split_array([1, 2, 3, 4, 5], 2))   # Output: 9
```

**F. Memory Cue:**
*"Split array is PIZZA DELIVERY: minimize the HEAVIEST box. Binary search the WEIGHT LIMIT, greedily pack boxes until limit hit!"*

---

#### **VERY HARD: Capacity to Ship Packages**

**A. Problem Statement:**
Ship packages within d days. Each day, load packages in order (no reordering). Find minimum ship capacity to ship all packages within d days.

**B. ASCII Visualization:**
```
Weights: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]  days = 5

Question: Minimum capacity C?

Try C = 15:
Day 1: [1,2,3,4,5] = 15
Day 2: [6,7] = 13
Day 3: [8] = 8
Day 4: [9] = 9
Day 5: [10] = 10
Total: 5 days ✓ Feasible

Try C = 10:
Day 1: [1,2,3,4] = 10
Day 2: [5] = 5
Day 3: [6] = 6
Day 4: [7] = 7
Day 5: [8] = 8
Day 6: [9] = 9  ✗ Need 6 days (> 5)

Try C = 13:
Day 1: [1,2,3,4] = 10
Day 2: [5,6] = 11
Day 3: [7] = 7
Day 4: [8] = 8
Day 5: [9] = 9
Can't fit 10! Need day 6 ✗

Try C = 15: Works! ✓ Answer

Search range: [max(weights), sum(weights)]
              [10, 55]
```

**C. Pattern Recognition Question:**
*Why must packages be loaded in order (why can't we reorder for better packing)?*

**D. Step-by-Step Reasoning:**
1. Search range: [max(weights), sum(weights)]
2. Feasibility function: Can ship all in d days with capacity C?
   - Simulate loading: add packages until capacity reached
   - Start new day when next package doesn't fit
   - Count total days needed
3. Binary search:
   - If feasible (≤ d days): Try smaller C (right = mid)
   - If infeasible (> d days): Need larger C (left = mid + 1)
4. Return left (minimum capacity)

**E. Python Solution:**
```python
def ship_within_days(weights, days):
    def can_ship(capacity):
        """Check if can ship all packages in ≤ days with given capacity"""
        days_needed = 1
        current_load = 0
        
        for weight in weights:
            if current_load + weight > capacity:
                # Start new day
                days_needed += 1
                current_load = weight
                
                if days_needed > days:
                    return False
            else:
                current_load += weight
        
        return True
    
    # Binary search on answer (ship capacity)
    left = max(weights)  # Must fit heaviest package
    right = sum(weights)  # Ship all in one day
    
    while left < right:
        mid = left + (right - left) // 2
        
        if can_ship(mid):
            # mid works, try smaller capacity
            right = mid
        else:
            # mid too small, need larger
            left = mid + 1
    
    return left

# Example usage
weights = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
print(ship_within_days(weights, 5))  # Output: 15
print(ship_within_days([3, 2, 2, 4, 1, 4], 3))  # Output: 6
```

**F. Memory Cue:**
*"Ship capacity is ELEVATOR WEIGHT LIMIT: binary search the minimum CAPACITY that fits all people across multiple TRIPS (days)!"*

---

### **Summary in 3 Sentences**

Binary search on answer optimizes by searching the solution space rather than input space, using a feasibility function to guide the search. The pattern works when checking "is X achievable?" is easier than computing optimal X directly, and the answer space has monotonic properties. Master identifying minimize maximum/maximize minimum problems and writing robust feasibility checks.

---

## Pattern 10: Backtracking

> **Definition:** A systematic exhaustive search algorithm that builds solutions incrementally, abandoning candidates ("backtracking") as soon as it determines they cannot lead to valid solutions.

---

### **Why This Pattern Exists (Meta-Reasoning)**

- 🔍 **Core Idea:** Try all possibilities by making choices, exploring consequences, and undoing choices (backtracking) when they lead to dead ends. More efficient than brute force by pruning impossible branches early.
- 💭 **Intuition:** Like exploring a maze: walk forward, mark your path, if you hit a dead end, walk back (backtrack) to the last decision point and try a different path.
- 🔢 **Mathematical Reasoning:** Explores decision tree depth-first, pruning branches that violate constraints. Time complexity often O(b^d) where b=branching factor, d=depth, but pruning reduces this significantly.
- ⚡ **When to Use:** Permutations, combinations, subsets, N-Queens, Sudoku, constraint satisfaction, path finding with choices.

---

### **ASCII Diagram**

```
Backtracking Decision Tree (Generate all subsets of [1,2,3]):

                    []
                   /  \
              [1]       []
             /  \      /  \
        [1,2]  [1]  [2]   []
        /  \   /\   /\    /\
    [1,2,3][1,2][1,3][1][2,3][2][3][]
    
Choose → Explore → Unchoose (Backtrack)

Visual Process:
State: []
  ↓ Choose 1
State: [1]
  ↓ Choose 2
State: [1, 2]
  ↓ Choose 3
State: [1, 2, 3] ✓ Valid solution
  ↑ Backtrack (remove 3)
State: [1, 2]
  ↑ Backtrack (remove 2)
State: [1]
  ↓ Choose 3
State: [1, 3] ✓ Valid solution
  ...

Backtracking Template:
def backtrack(state):
    if is_solution(state):
        save(state)
        return
    
    for choice in get_choices(state):
        # Make choice
        state.add(choice)
        
        # Explore
        backtrack(state)
        
        # Undo choice (backtrack)
        state.remove(choice)
```

---

### **Memorization Technique**

🎬 **Visualization:** Picture a **LABYRINTH with COLORED CHALK**. You draw a path as you walk (make choices). Hit a wall? ERASE your last steps (backtrack) and try a different turn. Your chalk leaves marks showing "been here, tried that!"

💥 **Exaggeration:** Each choice is a GLOWING PORTAL you jump through. Dead ends have GIANT RED X's. When you backtrack, you REWIND TIME like a video game, erasing your footsteps with MAGIC!

🔗 **Association:** "BACKTRACK" = **B**uild **A**ttempt **C**hoose **K**eep **T**rying **R**everse **A**bandon **C**ontinue **K**eep

---

### **Key Variations**

| Variation | Description | Example |
|-----------|-------------|---------|
| **Permutations** | All orderings of elements | Generate permutations |
| **Combinations** | Choosing k elements from n | Combination sum |
| **Subsets** | All possible subsets | Power set |
| **Constraint Satisfaction** | Satisfy constraints at each step | N-Queens, Sudoku |
| **Path Finding** | Find all valid paths | Word search, maze |

---

### **Socratic Teaching Round**

#### **Question 1:** What's the difference between backtracking and brute force?
**Answer:** Brute force generates ALL possibilities then filters. Backtracking prunes branches early using constraints, never exploring invalid subtrees. Much more efficient!

#### **Question 2:** When do we backtrack (undo a choice)?
**Answer:** After fully exploring that choice's subtree. Whether we found solutions or hit dead ends, we backtrack to try other choices at the current level.

#### **Question 3:** Why is backtracking often implemented recursively?
**Answer:** Recursion naturally maintains the "choice stack"—the call stack tracks our path. Each recursive call represents a choice, returning represents backtracking.

#### **Question 4:** How do we avoid duplicates in backtracking?
**Answer:** Sort input and skip duplicate elements, or use sets. When making choices, start from next index (not 0) to avoid reusing elements.

#### **Question 5:** What's the template for backtracking problems?
**Answer:** 
1. Define the state (what we're building)
2. Base case: when is solution complete?
3. For each valid choice: make choice → recurse → undo choice
4. Add pruning conditions to skip invalid branches

---

### **Problems**

---

#### **EASY: Generate All Subsets**

**A. Problem Statement:**
Given an array of unique integers, return all possible subsets (the power set).

**B. ASCII Visualization:**
```
Input: [1, 2, 3]

Decision Tree:
                []
               /  \
          Include  Exclude
            1        1
           /\       /\
          2  -     2  -
         /\  /\   /\  /\
        3 - 3 - 3 - 3 -

All subsets:
[] (exclude all)
[1] (include 1)
[1, 2] (include 1, 2)
[1, 2, 3] (include all)
[1, 3] (include 1, 3)
[2] (include 2)
[2, 3] (include 2, 3)
[3] (include 3)

Backtracking Path:
[] → [1] → [1,2] → [1,2,3] → backtrack → [1,2]
   → [1,3] → backtrack → [1] → backtrack → []
   → [2] → [2,3] → backtrack → [2] → backtrack
   → [3] → backtrack → []
```

**C. Pattern Recognition Question:**
*Why do we make a copy of the current subset before adding it to results?*

**D. Step-by-Step Reasoning:**
1. Start with empty subset
2. For each element, make two choices:
   - Include element in subset
   - Exclude element from subset
3. At each point, current state is a valid subset
4. Use recursion with index to track position
5. Base case: processed all elements
6. Backtrack by removing last added element

**E. Python Solution:**
```python
def subsets(nums):
    result = []
    current = []
    
    def backtrack(start):
        # Every state is a valid subset
        result.append(current[:])  # Make a copy!
        
        # Try adding each remaining element
        for i in range(start, len(nums)):
            # Make choice: include nums[i]
            current.append(nums[i])
            
            # Explore with this choice
            backtrack(i + 1)
            
            # Undo choice (backtrack)
            current.pop()
    
    backtrack(0)
    return result

# Alternative: Iterative approach using bit manipulation
def subsets_iterative(nums):
    result = []
    n = len(nums)
    
    # 2^n possible subsets (each element in/out)
    for mask in range(1 << n):
        subset = []
        for i in range(n):
            # Check if i-th bit is set
            if mask & (1 << i):
                subset.append(nums[i])
        result.append(subset)
    
    return result

# Example usage
print(subsets([1, 2, 3]))
# Output: [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
```

**F. Memory Cue:**
*"Subsets are LIGHT SWITCHES: each element is ON (included) or OFF (excluded). Try all 2^n combinations by FLIPPING switches!"*

---

#### **HARD: Combination Sum**

**A. Problem Statement:**
Given an array of distinct integers and a target, find all unique combinations where chosen numbers sum to target. Same number can be used unlimited times.

**B. ASCII Visualization:**
```
Input: [2, 3, 6, 7]  Target: 7

Decision Tree (pruned):
                  []
                /  |  \  \
              2   3   6   7
             /|\  |
           2 3 6  3
          /|  |
         2 3  6
        /
       2
      (8>7, prune)

Valid Paths:
[2,2,3] → 2+2+3 = 7 ✓
[7] → 7 = 7 ✓

Backtracking with Target:
start=0, target=7, current=[]
  ↓ choose 2
start=0, target=5, current=[2]
  ↓ choose 2
start=0, target=3, current=[2,2]
  ↓ choose 2
start=0, target=1, current=[2,2,2] (1<2, backtrack)
  ↑ remove 2
  ↓ choose 3
start=1, target=0, current=[2,2,3] ✓ Solution!
```

**C. Pattern Recognition Question:**
*Why do we pass 'start' index instead of always starting from 0?*

**D. Step-by-Step Reasoning:**
1. Sort candidates (helps with pruning)
2. Track current combination and remaining target
3. For each candidate from start index:
   - If candidate > remaining: skip (pruning)
   - Add candidate to combination
   - Recurse with same start (can reuse numbers)
   - Remove candidate (backtrack)
4. Base case: remaining == 0 (found solution)
5. Start index prevents duplicates ([2,3] vs [3,2])

**E. Python Solution:**
```python
def combination_sum(candidates, target):
    result = []
    current = []
    
    def backtrack(start, remaining):
        # Found valid combination
        if remaining == 0:
            result.append(current[:])
            return
        
        # Try each candidate from start
        for i in range(start, len(candidates)):
            candidate = candidates[i]
            
            # Pruning: if candidate too large, skip
            if candidate > remaining:
                continue
            
            # Make choice: include candidate
            current.append(candidate)
            
            # Explore: can reuse same number (start=i not i+1)
            backtrack(i, remaining - candidate)
            
            # Undo choice (backtrack)
            current.pop()
    
    backtrack(0, target)
    return result

# Example usage
print(combination_sum([2, 3, 6, 7], 7))
# Output: [[2, 2, 3], [7]]
print(combination_sum([2, 3, 5], 8))
# Output: [[2, 2, 2, 2], [2, 3, 3], [3, 5]]
```

**F. Memory Cue:**
*"Combination sum is COIN CHANGE with INFINITE COINS: keep adding coins (same index) until you hit TARGET or overshoot!"*

---

#### **VERY HARD: N-Queens**

**A. Problem Statement:**
Place n queens on an n×n chessboard such that no two queens attack each other (same row, column, or diagonal).

**B. ASCII Visualization:**
```
4-Queens Solution:

Board:
. Q . .    Row 0: Queen at col 1
. . . Q    Row 1: Queen at col 3
Q . . .    Row 2: Queen at col 0
. . Q .    Row 3: Queen at col 2

Constraint Checking:
Queens attack: same row, col, diagonal

Diagonals:
- Main diagonal: row - col = constant
- Anti-diagonal: row + col = constant

Backtracking Process:
Row 0: Try col 0
  ↓
Row 1: Try cols, col 2 valid
  ↓
Row 2: Try cols, none valid! Backtrack
  ↑
Row 1: Try next col, col 3 valid
  ↓
Row 2: Try cols, col 1 valid
  ↓
Row 3: Try cols, none valid! Backtrack
  ...continue until solution found

Decision Tree (partial):
        []
       / | \ \
      Q  Q Q Q  (row 0, try each col)
      0  1 2 3
     /
    Q   (row 1)
   /|\
  ...
```

**C. Pattern Recognition Question:**
*Why do we track columns and diagonals in sets instead of checking the board each time?*

**D. Step-by-Step Reasoning:**
1. Place queens row by row (one per row guaranteed)
2. For each row, try each column
3. Check if placement valid:
   - Column not used
   - Main diagonal (row-col) not attacked
   - Anti-diagonal (row+col) not attacked
4. If valid: mark column/diagonals, recurse to next row
5. If reach last row: found solution
6. Backtrack: unmark column/diagonals, try next position
7. Use sets for O(1) constraint checking

**E. Python Solution:**
```python
def solve_n_queens(n):
    result = []
    board = [['.'] * n for _ in range(n)]
    
    # Track attacked columns and diagonals
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col
    
    def backtrack(row):
        # Placed all queens successfully
        if row == n:
            # Convert board to required format
            solution = [''.join(row) for row in board]
            result.append(solution)
            return
        
        # Try placing queen in each column
        for col in range(n):
            # Check if position is safe
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue
            
            # Make choice: place queen
            board[row][col] = 'Q'
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)
            
            # Explore next row
            backtrack(row + 1)
            
            # Undo choice (backtrack)
            board[row][col] = '.'
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)
    
    backtrack(0)
    return result

# Example usage
solutions = solve_n_queens(4)
for sol in solutions:
    for row in sol:
        print(row)
    print()

# Output:
# .Q..
# ...Q
# Q...
# ..Q.
# 
# ..Q.
# Q...
# ...Q
# .Q..
```

**F. Memory Cue:**
*"N-Queens is CHESS WARFARE: place GENERALS (queens) so none can ATTACK each other. Try positions, mark DANGER ZONES (sets), backtrack if CONFLICT!"*

---

### **Summary in 3 Sentences**

Backtracking systematically explores all possibilities by making choices, recursing to explore consequences, and undoing choices to try alternatives. The pattern prunes impossible branches early using constraints, making it more efficient than brute force enumeration. Master the choose-explore-unchoose template and understand when to use sets/arrays for efficient constraint checking.

---

## 🎓 Conclusion

Congratulations! You've completed **Patterns 1-10** of the Ultimate LeetCode Pattern Learning System. These core patterns form the foundation for solving hundreds of coding problems efficiently.

### **What You've Mastered:**

✅ **Pattern 1: Sliding Window** - Optimize subarray problems in O(n)  
✅ **Pattern 2: Two Pointers** - Eliminate nested loops with intelligent pointer movement  
✅ **Pattern 3: Fast-Slow Pointers** - Detect cycles and find midpoints  
✅ **Pattern 4: Prefix Sum** - Answer range queries in O(1) after O(n) preprocessing  
✅ **Pattern 5: Hash Map / Frequency Counting** - Instant lookups and relationship tracking  
✅ **Pattern 6: Stack Pattern** - Handle nested structures and LIFO operations  
✅ **Pattern 7: Monotonic Stack** - Find next greater/smaller elements in O(n)  
✅ **Pattern 8: Binary Search** - Divide and conquer on sorted data  
✅ **Pattern 9: Binary Search on Answer** - Optimize by searching solution space  
✅ **Pattern 10: Backtracking** - Systematically explore all possibilities with pruning  

---

### **Next Steps:**

Continue your journey with **Patterns 11-20** (Advanced Patterns):
- Pattern 11: DFS (Depth-First Search)
- Pattern 12: BFS (Breadth-First Search)
- Pattern 13: Graphs
- Pattern 14: Dynamic Programming (1D)
- Pattern 15: Dynamic Programming (2D Grid)
- Pattern 16: Greedy Strategy
- Pattern 17: Intervals Pattern
- Pattern 18: Heap / Priority Queue
- Pattern 19: Bit Manipulation
- Pattern 20: Union-Find (DSU)

---

### **How to Practice:**

1. **Review each pattern daily** for 10 minutes
2. **Solve 3-5 problems per pattern** on LeetCode
3. **Draw ASCII diagrams** for each problem
4. **Teach concepts to others** to reinforce learning
5. **Create flashcards** using memory techniques

---

### **Remember:**

> *"Patterns are not just solutions—they're ways of seeing problems. Once you recognize the pattern, the solution becomes obvious."*

Keep practicing, stay curious, and happy coding! 🚀

---

*Created with 💙 for visual learners and pattern masters*
