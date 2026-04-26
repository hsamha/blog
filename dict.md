[Back](./README.md)

# Dictionary & ConcurrentDictionary

Both `Dictionary<TKey,TValue>` and `ConcurrentDictionary<TKey,TValue>` store key-value pairs, but they are designed for very different threading scenarios.

## Comparison Table

| Aspect                      | `Dictionary<TKey,TValue>`                 | `ConcurrentDictionary<TKey,TValue>`     |
| --------------------------- | ----------------------------------------- | --------------------------------------- |
| Thread safety               | Not thread-safe for concurrent writes     | Thread-safe for concurrent reads/writes |
| Namespace                   | `System.Collections.Generic`              | `System.Collections.Concurrent`         |
| Best use case               | Single-threaded or externally locked code | Multi-threaded shared access            |
| Performance (single thread) | Usually faster                            | Slight overhead due to synchronization  |
| Concurrent reads            | Usually safe if no writes occur           | Safe                                    |
| Concurrent writes           | Unsafe                                    | Safe                                    |
| Add if missing atomically   | No                                        | Yes (`TryAdd`, `GetOrAdd`)              |
| Update atomically           | No                                        | Yes (`TryUpdate`, `AddOrUpdate`)        |
| Need manual `lock`          | Often yes in multi-threading              | Usually no                              |
| Risk of race conditions     | High in shared access                     | Greatly reduced                         |
| Enumeration during writes   | Can throw / inconsistent                  | Supported snapshot-style enumeration    |

## Problems with Normal Dictionary in Multi-threading

When multiple threads use a normal `Dictionary` at the same time—especially if at least one thread writes—you can get:

| Problem            | Example                                           |
| ------------------ | ------------------------------------------------- |
| Race conditions    | Two threads both try to add same key              |
| Lost updates       | Two threads increment counter, one overwrite wins |
| Exceptions         | “Collection was modified” during enumeration      |
| Corrupted state    | Internal structure becomes inconsistent           |
| Check-then-act bug | Thread A checks key missing, Thread B adds first  |

### Example problem

```csharp id="4aqbrm"
if (!dict.ContainsKey("x"))
{
    dict["x"] = CreateValue();
}
```

Two threads can both pass `ContainsKey()` and both create/add values.

## How ConcurrentDictionary Solves It

It provides **atomic operations** that combine steps safely.

| Problem in Dictionary              | ConcurrentDictionary Solution |
| ---------------------------------- | ----------------------------- |
| Check then add race                | `GetOrAdd()`                  |
| Two writers overwrite unexpectedly | `AddOrUpdate()`               |
| Unsafe add duplicate key           | `TryAdd()`                    |
| Update based on old value          | `TryUpdate()`                 |
| Manual lock everywhere             | Internal synchronization      |
| Enumeration while writing          | Safe concurrent enumeration   |

### Example safe version

```csharp id="5jv3j6"
var value = dict.GetOrAdd("x", _ => CreateValue());
```

Only one thread wins creation/addition.

## Real-world Counter Example

### Normal Dictionary (unsafe)

```csharp id="1fjg9s"
dict["hits"] = dict["hits"] + 1;
```

Two threads may read `5`, both write `6` → one increment lost.

### ConcurrentDictionary (safe)

```csharp id="zh2ow7"
dict.AddOrUpdate("hits", 1, (_, old) => old + 1);
```

Increment happens atomically.

## Rule of Thumb

| Scenario                                 | Recommended Choice     |
| ---------------------------------------- | ---------------------- |
| Local method data                        | `Dictionary`           |
| Shared cache across tasks/threads        | `ConcurrentDictionary` |
| Maximum single-thread speed              | `Dictionary`           |
| Background service with parallel workers | `ConcurrentDictionary` |

## Short Summary

`Dictionary` is faster and simpler, but unsafe for shared concurrent writes.
`ConcurrentDictionary` adds built-in thread safety and atomic operations to solve race conditions and synchronization problems.

---
[Back](./README.md)
