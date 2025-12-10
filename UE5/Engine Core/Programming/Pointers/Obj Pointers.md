| Pointer type               | Strong? | What it does                                                                     |     |
| -------------------------- | ------- | -------------------------------------------------------------------------------- | --- |
| `AActor*` (raw pointer)    | ❌       | Does **not** protect from GC. If the actor is destroyed, the pointer dangles.    |     |
| `TObjectPtr<AActor>`       | ✅       | Strong reference (since UE5). Keeps the actor referenced and safe from GC.       |     |
| `TStrongObjectPtr<AActor>` | ✅       | Also strong; explicitly manages retention/release semantics.                     |     |
| `TWeakObjectPtr<AActor>`   | ❌       | Weak reference. Tracks the actor but doesn’t keep it alive; auto-null when GC’d. |     |
When I say “strong reference,” I’m referring to a pointer (or reference) that _keeps the target UObject alive_ by participating in Unreal’s garbage-collection reference graph. If you hold a strong reference to an object, the GC sees that reference and won’t reclaim the object—even if nothing else points to it.

So when I recommended using `TWeakObjectPtr`, the idea was: you don’t _own_ that actor—just need to point to it temporarily—so you don’t want your pointer to prevent the actor from being collected. The weak pointer gives you a safe way to reference it without extending its lifetime or risking a dangling pointer after destruction.
