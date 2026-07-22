# Some Important Concepts

A collection of notes on important concepts related to LLD, HLD etc.

## Table of Contents

- [Bloom Filter](#bloom-filter)

---

## Bloom Filter

A **Bloom filter** is a space-efficient, probabilistic data structure used to test whether an element **is possibly in a set** or **is definitely not in a set**.

> 🔗 Reference discussion: [claude.ai/share/5c296f2e-72f9-47f1-8681-171f0e8d965c](https://claude.ai/share/5c296f2e-72f9-47f1-8681-171f0e8d965c)

### Key Properties

- **No false negatives**: If the filter says an element is _not present_, it is guaranteed to not be in the set.
- **Possible false positives**: If the filter says an element _is present_, it might actually not be — there's a tunable probability of error.
- **Space efficient**: Uses a bit array instead of storing actual elements, making it much smaller than the underlying set.
- **No deletions** (in the basic version): You cannot remove an element once added, since bits may be shared between multiple elements. (A _Counting Bloom Filter_ variant supports deletion.)

### How It Works

1. Start with a bit array of size `m`, all bits initialized to `0`.
2. Use `k` independent hash functions, each mapping an element to one of the `m` positions.
3. **To add an element**: Hash it with all `k` hash functions and set the corresponding `k` bits to `1`.
4. **To check membership**: Hash the element with all `k` hash functions.
   - If **any** of the `k` bits is `0` → the element is **definitely not** in the set.
   - If **all** `k` bits are `1` → the element is **probably** in the set (could be a false positive).

### Example

```
Bit array (m = 10): [0 0 0 0 0 0 0 0 0 0]

Add "apple" -> hash1=1, hash2=4, hash3=7
Bit array:          [0 1 0 0 1 0 0 1 0 0]

Add "banana" -> hash1=3, hash2=4, hash3=9
Bit array:          [0 1 0 1 1 0 0 1 0 1]

Check "apple"  -> bits 1,4,7 are all 1 -> possibly present ✅
Check "grape"  -> hash1=2 -> bit 2 is 0 -> definitely not present ❌
```

### Complexity

| Operation | Time Complexity |
| --------- | --------------- |
| Insert    | O(k)            |
| Lookup    | O(k)            |
| Space     | O(m) bits       |

Where `k` is the number of hash functions and `m` is the bit array size (both constants relative to input size).
