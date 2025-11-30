
## Core idea: `UObject` and Garbage Collection

Think of `UObject` as “managed objects” like in C# or Java:

- You never `delete` a `UObject` directly.
- The engine’s **GC** periodically checks: _“Is this object reachable from any root?”_
- Roots are:
    - Objects that are always kept alive (like the world, game instance, etc.), _and_
    - Any references reachable via `UPROPERTY` pointers from those roots.

So:

- **If you want the GC to keep something alive**, you must reference it with a `UPROPERTY` pointer (or other GC-visible mechanism).
- A raw pointer (`UObject*`) **without** `UPROPERTY` won’t prevent collection.
    - If the target object gets GC’d, your raw pointer becomes **dangling** (undefined behavior if dereferenced).

Keep that in mind: [[UE Pointers]] types are mostly about telling the GC and engine _how_ you’re referencing something.





Further Reads:
https://zhuanlan.zhihu.com/p/400473355
