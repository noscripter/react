# Linked Structures & Double Buffering Study Guide

React’s Fiber architecture relies on linked structures and double buffering to schedule and commit UI updates. Use this guide to brush up on the computer science foundations behind those ideas and to practice related interview questions.

## Core Concepts

- **Linked List Fundamentals:** Fibers form a tree of nodes connected by `return`, `child`, and `sibling` pointers. Each level is effectively a linked list that React walks during reconciliation.
- **Double Buffering:** Each Fiber has an `alternate` pointer to its other version (`current` vs `workInProgress`). React swaps between these “buffers” so it can prepare an update without mutating the committed tree.
- **Incremental Traversal:** Because linked lists don’t require contiguous memory, React can pause and resume work at any node, which is essential for cooperative scheduling.

## Recommended Reading

### Linked Lists
- [CLRS, *Introduction to Algorithms*, Chapter 10.2 – Linked Lists](https://mitpress.mit.edu/9780262533058/introduction-to-algorithms/) (textbook reference)
- [LeetCode Explore Card: Linked List](https://leetcode.com/explore/learn/card/linked-list/)
- [MIT OCW 6.006 Lecture Notes – Linked Structures](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-fall-2011/pages/lecture-notes/) (see Lecture 2)

### Double Buffering
- [GPU Gems: Double Buffering Basics](https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects/chapter-6-high-quality-lights-via-logarithmic-shadow-mapping) (conceptual overview from graphics domain)
- [Game Programming Patterns – Double Buffer](https://gameprogrammingpatterns.com/double-buffer.html) (language-agnostic explanation)
- [React Fiber Architecture Overview](https://github.com/acdlite/react-fiber-architecture) (community write-up connecting buffering to React)

## Practice Problems

### Linked List Essentials (LeetCode)
1. **Reverse Linked List** – [LeetCode #206](https://leetcode.com/problems/reverse-linked-list/)  
   Practice pointer manipulation used heavily in Fiber traversals.
2. **Linked List Cycle Detection** – [LeetCode #141](https://leetcode.com/problems/linked-list-cycle-detection/)  
   Mirrors how React guards against bad polyfills or cycles in host trees.
3. **Merge k Sorted Lists** – [LeetCode #23](https://leetcode.com/problems/merge-k-sorted-lists/)  
   Reinforces priority-based merging similar to lane scheduling.
4. **Flatten a Multilevel Doubly Linked List** – [LeetCode #430](https://leetcode.com/problems/flatten-a-multilevel-doubly-linked-list/)  
   Comparable to unwinding nested children and siblings in a Fiber tree.

### Double Buffering & State Management
1. **Implement Double Buffer Simulation** – Write code that maintains two buffers (`current`, `workInProgress`) for a grid or image, swapping them per frame. Focus on:
   ```js
   function step(current, workInProgress) {
     for (let i = 0; i < current.length; i++) {
       workInProgress[i] = computeNext(current[i]);
     }
     return [workInProgress, current]; // swap buffers
   }
   ```
2. **Functional Reactivity Exercise** – Build a simple scheduler that enqueues tasks with priorities, processes them in rounds, and keeps a “current” vs “pending” queue to mimic React’s double buffering of work units.
3. **Cache-Friendly Linked List Update** – Implement a data structure that mirrors React’s `alternate` pattern: each node keeps a `current` value and a `staged` value, and a `commit()` method swaps them atomically.

## Interview Question Prompts

- Explain how React’s Fiber “linked list of work” compares to a classic singly linked list. What trade-offs does this structure make compared to arrays when pausing work?
- Design a system that renders frames to screen using double buffering. How do you prevent tearing, and how would you coordinate swaps if rendering takes longer than a frame?
- Given a tree where each node has `child` and `sibling` pointers, write a function that traverses nodes in depth-first order without recursion. Discuss how this mirrors React’s commit phase.
- If you needed to detect whether two buffers have diverged (e.g., `current` vs `workInProgress`), how would you compare them efficiently? How does React avoid comparing entire trees?

Use these resources alongside the main `docs/interview-questions.md` file to reinforce the systems-level thinking React expects from core contributors.
