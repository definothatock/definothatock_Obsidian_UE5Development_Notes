# Non-UObject, C++-only pointers (no GC)

These are equivalent to C++ smart pointers, used for normal C++ types (non-UObject):

1. `TUniquePtr<T>`
	- sole ownership (like `std::unique_ptr`).

2. `TSharedPtr<T>`
	- reference-counted shared ownership (like `std::shared_ptr`).

3. `TWeakPtr<T>`
	- non-owning weak reference to a `TSharedPtr<T>`.

4. Raw `T*`
	- normal C++ raw pointer.

These **do not** talk to Unreal GC. They’re for plain C++ memory management.


## Non-UObject pointers: `TSharedPtr`, `TUniquePtr`, `TWeakPtr`

These behave like standard C++ smart pointers, but are Unreal’s own implementation (plays nice with UE’s allocators, containers, etc.).

Used mainly for:
- Slate UI (`SWidget` and friends)
- Gameplay code that uses plain C++ structs/classes (non-UObject)
- Async tasks, caches, etc.

### 1.1 `TUniquePtr<T>`

**Analogy:** Exactly `std::unique_ptr`.
- Sole ownership. Cannot be copied (only moved).
- Object destroyed when the `TUniquePtr` goes out of scope or is reset.

```cpp
TUniquePtr<FMyData> Data = MakeUnique<FMyData>();
Data->Value = 5;

// Transferring ownership:
TUniquePtr<FMyData> Other = MoveTemp(Data); // Data becomes null
```

Use for:
- Strict single-owner objects
- Resource wrappers (file handles, sockets) in pure C++ parts of UE

### 1.2 `TSharedPtr<T>` and `TWeakPtr<T>`

**Analogy:** `std::shared_ptr` and `std::weak_ptr`.

`TSharedPtr<T>`:

```cpp
TSharedPtr<FMyData> Data = MakeShared<FMyData>();
Data->Value = 10;

TSharedPtr<FMyData> Copy = Data; // reference count +1
```

Object is deleted when last `TSharedPtr` is destroyed.

`TWeakPtr<T>`:

```cpp
TWeakPtr<FMyData> WeakData = Data;
...
if (TSharedPtr<FMyData> Pinned = WeakData.Pin())
{
    // Safe to use Pinned
}
```

Use when:
- Multiple subsystems may share ownership of a non-UObject.
- You want to refer to a shared object without extending its lifetime (`TWeakPtr`).

**Important:** These do **not** auto-serialize, do **not** participate in UE GC. They’re purely C++ lifetime tools.

---
