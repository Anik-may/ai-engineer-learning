# Week 23: Coding Rounds

## Goal

Pass a 45-minute coding round in Python, and a practical "build a small
LLM feature" round.

## Two kinds of coding interview

| Kind | What they test | How to prepare |
| --- | --- | --- |
| Algorithms | Data structures, problem solving, clean code | Pattern practice |
| Practical / take-home | Build a small API, RAG function or agent | Your projects are the practice |

Bangladesh AI roles more often use the practical kind plus a short
algorithm screen. International roles use both.

## Algorithm patterns to cover

Do 3 to 4 medium problems per pattern. Quality over count.

| Pattern | Recognise it by |
| --- | --- |
| Hash map / set | "Find a pair", counts, duplicates |
| Two pointers | Sorted array, pairs, palindromes |
| Sliding window | Longest or shortest subarray or substring |
| Stack | Matching brackets, next greater element |
| Binary search | Sorted input, "minimum value that satisfies" |
| Linked list | Reverse, merge, cycle |
| Trees: DFS and BFS | Paths, levels, depth |
| Graphs | Grid islands, dependencies, shortest path |
| Heap | Top k, merge k lists, scheduling |
| Intervals | Merge, overlap |
| Dynamic programming, basic | "Number of ways", "minimum cost", 1-D and 2-D |
| Backtracking | Combinations, permutations |

Relevant to AI work and sometimes asked directly: top-k with a heap, cosine
similarity, LRU cache, rate limiter (token bucket), trie for autocomplete,
text chunking with overlap, deduplication.

## How to work a problem aloud

1. Restate the problem and ask about input size and edge cases.
2. Give a small example by hand.
3. State the brute-force idea and its complexity.
4. Improve it; name the pattern.
5. Write clean code, talking as you go.
6. Test with your example and one edge case.
7. State time and space complexity.

Silence loses interviews. Narrate your thinking.

## Python you must be fluent in

```python
from collections import Counter, defaultdict, deque
import heapq

counts = Counter(words).most_common(3)
groups = defaultdict(list)
q = deque([start]); q.append(x); q.popleft()
heapq.heappush(h, (priority, item)); heapq.heappop(h)
top = heapq.nlargest(5, scores, key=lambda s: s[1])
pairs = sorted(items, key=lambda x: (-x.score, x.name))
```

Also: list and dict comprehensions, slicing, `enumerate`, `zip`, sets,
`dataclass`, type hints, `async` and `await`, context managers, generators.

## Practical round: things you may be asked to build

1. A FastAPI endpoint that streams an LLM response.
2. A function that chunks text with overlap and tests for it.
3. A small RAG: embed, store, retrieve top-k, answer with citations.
4. A tool-calling loop with retries and a step limit.
5. A rate limiter or cache wrapper around a model call.
6. Structured extraction with Pydantic validation and a retry on invalid
   output.
7. An evaluation script: dataset in, scores out.

Build each once from an empty file in under 45 minutes, without copying
from your projects. Time yourself.

What reviewers look for: it runs; clear structure; input validation; error
handling and timeouts; no secrets in code; a few tests; a short README
stating trade-offs and what you would do next.

## Example to implement this week: LRU cache

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.data: OrderedDict[str, str] = OrderedDict()

    def get(self, key: str) -> str | None:
        if key not in self.data:
            return None
        self.data.move_to_end(key)
        return self.data[key]

    def put(self, key: str, value: str) -> None:
        self.data[key] = value
        self.data.move_to_end(key)
        if len(self.data) > self.capacity:
            self.data.popitem(last=False)
```

Then explain how you would add a time-to-live and make it safe across
several server instances (answer: move it to Redis).

## Using AI coding tools

Some interviews now allow or require them; several postings in the report
asked for daily use of AI coding tools. Ask the rules at the start. If
allowed, show judgement: state the plan first, review what the tool wrote,
test it, and explain it. Being unable to explain generated code fails the
round.

## Do

1. Two medium problems a day, timed at 25 minutes each, spoken aloud.
2. Keep an error log: problem, mistake, pattern. Review it every third day.
3. Build four of the seven practical tasks from an empty file.
4. One full mock coding interview with a person or recorded.

## Checkpoint

- [ ] You solve most medium problems in the covered patterns in 25 minutes.
- [ ] You built four practical tasks from scratch within time.
- [ ] You narrate while coding without long silences.
