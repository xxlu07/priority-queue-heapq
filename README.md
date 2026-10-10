# priority queue python: heapq, thread-safe queues, max-heap tricks, and the interview problems they solve

Most people searching for a priority queue in Python need one of three things. They need a working implementation they can paste into a file, they need a max-heap even though every tutorial starts with a min-heap, or they just watched their code throw `TypeError: '<' not supported between instances of 'Task' and 'Task'` and want to know why.

All three come from the same place: Python ships two priority queue implementations in the standard library, `heapq` and `queue.PriorityQueue`, and neither one is a self-contained "PriorityQueue" class that behaves like Java's. Understanding the difference takes about five minutes and saves a surprising amount of debugging.

## The two stdlib answers, and how to pick one

Python's own documentation describes `heapq` as a heap queue algorithm — the thing textbooks call a priority queue. It's not a class. It's a set of functions that operate on a plain list, which means you get a list-based min-heap with no wrapper object, no locking, and no overhead beyond the list itself.

`queue.PriorityQueue` is the other option. It's a class, it's synchronized (thread-safe), and internally it's built on top of `heapq`: entries stay sorted and the lowest-valued entry comes out first. If you're pushing work items into a queue that multiple threads read from, this is the one you want. If you're solving a coding problem or a single-threaded batch job, `heapq` is faster and simpler.

A useful rule: `heapq` for algorithms, `queue.PriorityQueue` for producer/consumer threading.

## heapq in practice: push, pop, peek, heapify

`heapq` gives you four operations you'll use constantly.

python
import heapq

pq = []
heapq.heappush(pq, (5, "review PR"))
heapq.heappush(pq, (1, "fix prod bug"))
heapq.heappush(pq, (3, "write docs"))

print(pq[0])                 # (1, 'fix prod bug')  — peek, O(1)
print(heapq.heappop(pq))     # (1, 'fix prod bug')  — pop, O(log n)


The peek is the part people miss. Because the heap lives in a normal list, `pq[0]` is always the smallest item, and reading it costs nothing and doesn't modify anything.

If you already have a list and want it heap-ordered, don't loop and push — call `heapq.heapify(nums)`, which converts the list in place in linear time. Pushing n items one at a time is O(n log n); heapifying is O(n).

Two more functions are worth knowing because they're faster than what you'd write by hand:

- `heapq.heappushpop(heap, item)` pushes and pops in one step, and it's more efficient than a separate push followed by a pop. It returns the smaller of the two values and leaves the larger on the heap.
- `heapq.heapreplace(heap, item)` pops the current smallest and inserts the new item, keeping the heap size constant. It always returns an item from the heap and is the right tool for fixed-size heaps — top-k problems, sliding windows, and anything where you're keeping "the best k so far."

There's also `heapq.nlargest(n, iterable, key=None)` and `heapq.nsmallest(...)` if you just want the top n of a dataset. The docs are direct about the tradeoff: those two perform best for small values of n, `sorted()` wins for large n, and for `n == 1` you should just use `min()` or `max()`.

## Making a max-heap when heapq only speaks min

The single most common complaint about Python's heap module is that it has no max-heap. You have two real options.

**Option 1: negate the priority.** Fast, no extra objects, but it only works for numbers.

python
max_pq = []
for n in nums:
    heapq.heappush(max_pq, -n)

largest = -heapq.heappop(max_pq)


For tuples you negate the priority field and leave the payload alone: `(-priority, task)`. It's fine. It's also the version that breaks the moment your priorities aren't numeric.

**Option 2: a wrapper that ignores the payload.** This is the pattern Python's docs recommend when the heap entries are `(priority, data)` and the data items aren't comparable with each other.

python
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any = field(compare=False)


`order=True` generates the comparison methods from the fields, and `compare=False` on `item` removes it from those comparisons entirely. Now the heap only ever compares priorities. For a max-heap, store `-priority` or write a small wrapper class with a reversed `__lt__`. If you're doing this in an interview, say out loud that you're choosing the wrapper for readability over the negation hack — that's the kind of tradeoff statement that reads as experience rather than memorization.

## The bug that bites everyone: equal priorities with uncomparable payloads

Here's the failure mode that sends people searching in the first place. `heapq` docs are explicit that heap elements can be tuples, which is useful for attaching a priority to a record. The catch: when two entries share the same priority, Python falls through to comparing the second element. If those second elements are dictionaries, custom objects, or anything without a natural ordering, the heap raises a `TypeError` on insertion — sometimes only after ten thousand perfectly fine pushes.

The standard fix is a tie-breaker that can never be equal: a monotonically increasing counter, usually from `itertools.count()`.

python
import heapq, itertools

counter = itertools.count()
pq = []

def add_task(task, priority=0):
    heapq.heappush(pq, (priority, next(counter), task))


That gives you two things at once. The counter never ties, so the payload is never compared. And because it increases with insertion order, two tasks with the same priority come out in the order they arrived — a stable FIFO behavior that a bare `(priority, task)` tuple doesn't guarantee.

The harder problem is changing a task's priority after it's already in the heap, or deleting it outright. Python's docs walk through the workaround, and it's worth internalizing: you can't surgically remove an arbitrary entry from a heap without breaking the invariant, so you mark it as removed instead. Keep a dictionary mapping each task to its entry, push a new entry with the new priority, and mark the old one with a sentinel value like `'<removed-task>'`. When you pop, skip anything marked removed until you find a live entry. It's lazy deletion, and it's how `decrease-key` gets implemented on top of a Python min-heap.

If you want that dictionary-backed heap interface without writing it yourself, packages like `heapdict` offer it — a dict-shaped priority queue where keys are elements and values are priorities, with min-max ordering. It's a third-party install, so it's not available in an interview, but it's a reasonable choice in production code where you genuinely need `decrease-key`.

## heapq vs queue.PriorityQueue, side by side

The thread-safety question is where the two diverge, and it's the only question that usually matters for the choice.

|  | `heapq` | `queue.PriorityQueue` |
| --- | --- | --- |
| Type | Functions operating on a list | Class |
| Thread-safe | No — you add your own locks | Yes, uses locks internally |
| Underlying structure | A plain list | Built on `heapq` |
| Blocking operations | No | Yes — `put` blocks on a full queue, `get` blocks on an empty one |
| Typical entry form | `(priority, data)` tuple | `(priority_number, data)` tuple |
| Coordination API | None | `task_done()`, `join()` |
| Best for | Algorithms, single-threaded work, top-k | Producer/consumer threading |

One detail worth flagging: the docs note that `queue`'s classes use locks to block competing threads but are not designed to handle reentrancy within a thread. So don't call `put` on a queue from inside a task that a worker is already processing off that same queue unless you've thought through the deadlock carefully.

For the multi-threaded pattern, the worker side looks like this:

python
from queue import PriorityQueue
import threading, random

pq = PriorityQueue()

def worker():
    while True:
        pri, job = pq.get()
        print(f"processing {job} (priority {pri})")
        pq.task_done()

threading.Thread(target=worker, daemon=True).start()

for job in ["build", "test", "deploy"]:
    pq.put((random.randint(1, 10), job))

pq.join()


`join()` returns once `task_done()` has been called for every item that was put. That's the mechanism `heapq` simply doesn't have, and it's the reason `queue.PriorityQueue` exists.

## Complexity, so you don't have to look it up mid-interview

| Operation | heapq | queue.PriorityQueue |
| --- | --- | --- |
| Find min (peek) | O(1) — `heap[0]` | O(1) via `queue[0]` semantics; `get()` is O(log n) |
| Insert | O(log n) | O(log n) |
| Pop min | O(log n) | O(log n) |
| Build from n items | O(n) with `heapify` | Not offered; n inserts |
| Delete arbitrary / change priority | Not supported — lazy deletion | Not supported — lazy deletion |
| `nlargest` / `nsmallest` | O(n log k) with a size-k heap | n/a |

The line that trips people up is heapify. Building a heap from an unsorted list is linear, not O(n log n), and interviewers do ask why.

## Where priority queues actually show up in interview problems

This is the part that's easy to miss if you learn `heapq` as an isolated data structure. AlgoMonster's own analysis of interview problem frequency ranks priority queue and heap as **medium ROI** — not as dominant as DFS, BFS, and two pointers, but showing up more often than most candidates expect. Their stated classic problems are the median of a data stream and k closest points.

Their interview decision flowchart routes two specific questions to heap-based solutions: "kth smallest/largest?" and "merge many sorted lists?" (the latter jointly with divide and conquer). Weighted shortest-path problems route to Dijkstra's algorithm, which is a priority queue with a distance array bolted on.

So the practical pattern list looks like this:

1. **Top k / kth largest or smallest** — keep a min-heap of size k, push each element, pop whenever the size exceeds k. At the end the heap holds the k largest. Complexity O(n log k), which beats sorting when k is small.
2. **Merge k sorted lists** — push the head of each list into a min-heap, pop the smallest, then push the next node from whichever list it came from.
3. **Median from a data stream** — two heaps. A max-heap for the lower half, a min-heap for the upper half, rebalanced after each insert. This is the one where the negation trick gets used most.
4. **K closest points to origin** — a size-k max-heap keyed on squared distance.
5. **Dijkstra** — a min-heap of `(distance, node)` pairs, plus lazy deletion of stale entries when you pop a node you've already finalized.

If you're practicing these, the `heapreplace` pattern is worth drilling until it's automatic:

python
def top_k_largest(nums, k):
    heap = nums[:k]
    heapq.heapify(heap)          # O(k)
    for n in nums[k:]:
        if n > heap[0]:          # beats the current smallest kept
            heapq.heapreplace(heap, n)
    return heap


Note what's happening: the smallest element of the heap is the gate. Keeping the k largest in a min-heap feels backwards until you see that `heap[0]` is exactly the value you'd need to evict.

If you'd rather work through these patterns in a structured order instead of picking problems at random, AlgoMonster builds its curriculum around pattern selection — 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, written by Google engineers and competitive programmers. 👉 [See how AlgoMonster structures its heap and priority queue material](https://algo.monster?fpr=li69)

## Common mistakes, collected

A few things that cause most of the real-world bugs:

- **Mutating a list after heapifying it.** If you sort the list, or assign into an index, the heap invariant is gone and every later pop returns garbage. Push through `heappush` or rebuild with `heapify`.
- **Assuming a heap is sorted.** It isn't. `heap[0]` is the minimum; everything past that is only partially ordered.
- **Popping from an empty heap.** `heappop` raises `IndexError`. `heapq` has no `is_empty()` — check `if not pq:` or `len(pq) == 0`.
- **Comparing heaps with `<` and expecting the max.** Since a heap is a list, `heap1 < heap2` does a lexicographic list comparison, which has nothing to do with heap contents.
- **Reaching for `PriorityQueue` inside a tight single-threaded loop.** The locking is pure overhead when no other thread touches the queue.

One more: in a coding interview, you can generally use the built-in library for heaps rather than implementing sift-up and sift-down yourself. The AlgoMonster discussion around its Heap Fundamentals lesson covers this directly, and the consensus in that thread is that language standard-library heaps are fair game. Knowing the O(log n) mechanics matters for explaining your complexity, not for typing out a `__lt__`-based heap from scratch while the clock runs.

## What AlgoMonster costs, and whether it fits a heap learner

If you're using priority queue problems as practice rather than as a one-off task, the useful question is whether a structured curriculum is worth it versus free documentation. AlgoMonster's angle is that it teaches selection — which pattern this problem needs — rather than just supplying problems. It's text and interactive diagrams, with no video, which is either the best or worst thing about it depending on how you learn.

Here's the current plan set from AlgoMonster's subscribe page:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems | $0 | — | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Monthly | Full access to all patterns, lessons and problems; unlimited AI assistant and Speedrun; company-specific questions | $18.99/mo | Monthly, cancel anytime | [Start Pro monthly](https://algo.monster/subscribe?fpr=li69) |
| Pro — Yearly | Everything above plus 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, member discount on live bootcamps | ~$99/year at the current promotion ($8.25/mo billed annually; list price $198/year) | 12 months of full access | [Compare the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates | ~$189 one-time at the current promotion (list price $378) | Billed once | [Check the lifetime price](https://algo.monster/subscribe?fpr=li69) |

Two honest caveats, because the numbers above move. AlgoMonster rotates promotions, and different campaign links have surfaced different prices — one capture of the same subscribe page showed $69.99/year and $89 lifetime, and third-party reviews have quoted everything from $89 to $459. Treat the table as the standard promotional pricing and confirm the total on the checkout screen before you pay. Also note that several blog reviews claim there is no monthly option; the subscribe page has shown one, so check the live page rather than the reviews.

Who this makes sense for: someone whose gap is pattern recognition, who can already write correct Python, and who has a few weeks. The heap material specifically sits inside a larger pattern library, so you're paying for the surrounding structure as much as for the priority queue chapter. If your actual problem is "I need a max-heap in Python by Friday," the answer is three lines of negation and you don't need to buy anything.

If you want to see how the patterns are laid out before spending anything, the free tier and their public problem-statistics page are both reasonable places to start. 👉 [Browse the free AlgoMonster material](https://algo.monster?fpr=li69)

## The short version

Use `heapq` with a plain list and `(priority, counter, payload)` tuples for single-threaded work and interview problems. Add a counter whenever your payloads can't be compared, or you'll get a `TypeError` the first time two priorities tie. Negate for max-heaps unless readability matters more than a little memory. Switch to `queue.PriorityQueue` when multiple threads share the queue and you want blocking `put`/`get` with `join()` and `task_done()`.

And if you're learning this for interviews rather than for a script: heaps are medium-ROI. Worth knowing cold, not worth grinding for three weeks at the expense of DFS, BFS, and two pointers.
