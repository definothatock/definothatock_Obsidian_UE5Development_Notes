In Unreal Engine (and C++ in general), **lambdas** and **predicates** show up a lot in delegates, container algorithms (like `Algo::Sort`), async tasks, and Slate/UMG callbacks. You _can_ use UE without them, but knowing them will make your code much cleaner.

I’ll explain:

1. What a lambda is (with simple syntax examples)
2. What “capture” means (`[]`, `[&]`, `[=]`, etc.)
3. What a predicate is and how it relates to lambdas
4. Unreal-specific examples (Blueprint delegates, sorting `TArray`, timers, async)

---

## 1. What is a lambda in C++?

A **lambda** is an **anonymous function**: a function you define inline, usually where you need to pass some small operation as an argument.

**Normal function:**

```cpp
int Add(int a, int b)
{
    return a + b;
}
```

**Lambda that does the same:**

```cpp
auto AddLambda = [](int a, int b) {
    return a + b;
};

int Result = AddLambda(2, 3); // 5
```

Key points:

- `auto AddLambda` is the variable that _stores_ the lambda.
- `[]` is the **capture list** (we’ll get to that next).
- `(int a, int b)` is the parameter list.
- `{ return a + b; }` is the function body.

You often **don’t** name lambdas; you pass them directly:

```cpp
SomeFunctionThatNeedsACallback([](int Value) {
    UE_LOG(LogTemp, Log, TEXT("Value is %d"), Value);
});
```

---

## 2. The capture list: `[]`, `[&]`, `[=]`, `[this]`, etc.

The **capture list** controls which variables from the surrounding scope the lambda can use.

### 2.1 No capture: `[]`

```cpp
int X = 10;
auto Lambda = []() {
    // X; // ❌ cannot use X here, nothing is captured
};
```

If the lambda doesn’t use any external variables, `[]` is enough.

---

### 2.2 Capture by value: `[=]` or `[x]`

**By value** means: the lambda gets its _own copy_ of the variable.

```cpp
int X = 10;

auto Lambda = [=]() {
    int Y = X; // OK: X is copied into the lambda
};

// Or explicitly capture a specific variable:
auto Lambda2 = [X]() {
    int Y = X; // same idea
};
```

Be careful: if you change `X` after creating the lambda, the lambda still sees the **old value** (because it has a copy).

```cpp
int X = 10;
auto Lambda = [X]() {
    UE_LOG(LogTemp, Log, TEXT("X in lambda: %d"), X);
};

X = 20;
Lambda(); // prints 10, not 20
```

---

### 2.3 Capture by reference: `[&]` or `[&x]`

**By reference** means: the lambda refers to the _original variable_, not a copy.

```cpp
int X = 10;

auto Lambda = [&]() {
    X = 42; // modifies the original X
};

Lambda();
UE_LOG(LogTemp, Log, TEXT("X = %d"), X); // prints 42
```

Or explicitly:

```cpp
auto Lambda = [&X]() {
    X = 42;
};
```

This is very common but you must ensure the variables still **exist** when the lambda is called (important with async code, timers, etc.).

⚠️ Be careful: capturing by reference can dangle if the lambda outlives the referenced object.

---

### 2.4 Capturing `this`: `[this]` or `[=, this]`

In member functions, you’ll often see:

```cpp
void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    int LocalValue = 5;

    auto Lambda = [this, LocalValue]()
    {
        // Can use members of AMyActor via `this`
        UE_LOG(LogTemp, Log, TEXT("Actor name: %s, LocalValue: %d"),
               *GetName(), LocalValue);
    };

    Lambda();
}
```

You can also do:

```cpp
[this]() { /* ... */ }    // capture only this
[=, this]() { /* ... */ } // capture everything by value + this
[&, this]() { /* ... */ } // capture everything by ref + this
```

---

## 3. What is a predicate?

A **predicate** is just a function (or lambda) that **returns a `bool`** and is used to test a condition, like “is this element valid?”, “should this be kept?”, or “is A < B?”.

Examples:

```cpp
// Standalone function
bool IsEven(int X)
{
    return (X % 2) == 0;
}

// Lambda version (also a predicate)
auto IsEvenLambda = [](int X) {
    return (X % 2) == 0;
};
```

In Unreal (and C++ algorithms), predicates are often passed into container functions:

- Filtering arrays
- Sorting arrays
- Finding elements (e.g. `FindByPredicate`)

**So:**

- _Lambda_ = syntax feature (“anonymous function”).
- _Predicate_ = _role concept_ (“a function/lambda that returns bool and is used as a condition”).

Most of the time in UE5, a _predicate_ is implemented as a _lambda_.

---

## 4. Unreal-specific examples

### 4.1 Sorting a `TArray` with a lambda predicate

```cpp
TArray<int32> Numbers = { 5, 1, 7, 3, 2 };

// Sort ascending
Numbers.Sort([](int32 A, int32 B) {
    return A < B; // predicate: "should A come before B?"
});

// Sort descending
Numbers.Sort([](int32 A, int32 B) {
    return A > B;
});
```

Here `[](int32 A, int32 B) { return A < B; }` is a **lambda**, and it’s used as a **comparison predicate**.

---

### 4.2 Filtering a `TArray` with `Algo::RemoveIf` / `Algo::StableSort` etc.

Unreal has `Algo` utilities:

```cpp
#include "Algo/RemoveIf.h"

TArray<int32> Numbers = { 1, 2, 3, 4, 5, 6 };

// Remove odd numbers
Algo::RemoveIf(Numbers, [](int32 X) {
    return (X % 2) != 0; // predicate: "should this be removed?"
});

// Now Numbers contains { 2, 4, 6 }
```

Again: the lambda returns bool -> a predicate.

---

### 4.3 Delegates & lambdas

You’ll often bind delegates to lambdas.

**Example: Timer with a lambda**

```cpp
FTimerHandle TimerHandle;

GetWorldTimerManager().SetTimer(
    TimerHandle,
    [this]()   // capture this so we can call a member function
    {
        UE_LOG(LogTemp, Log, TEXT("Timer fired on %s"), *GetName());
    },
    2.0f,      // Rate
    false      // bLoop
);
```

**Example: Async task with lambda completion**

```cpp
AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this]()
{
    // Background work
    int32 Result = 42;

    // Back to game thread
    AsyncTask(ENamedThreads::GameThread, [this, Result]()
    {
        UE_LOG(LogTemp, Log, TEXT("Finished with result %d"), Result);
    });
});
```

Notice the capture patterns: `[this]`, `[this, Result]`.

---

### 4.4 Slate/UMG style: inline lambdas as callbacks

In Slate, you see this pattern a lot:

```cpp
SNew(SButton)
.Text(FText::FromString("Click Me"))
.OnClicked_Lambda([this]() -> FReply
{
    UE_LOG(LogTemp, Log, TEXT("Button clicked on %s"), *GetName());
    return FReply::Handled();
});
```

- `OnClicked_Lambda` explicitly expects a lambda.
- The lambda returns `FReply` (not `bool` here, but same idea: inline callback).

---

## 5. When should you use lambdas in UE5?

Typical use cases:

- **Short one-off callbacks** (timers, async, widget events).
- **Custom sorting/filtering logic** for `TArray`, `TSet`, `TMap`.
- **Predicates for “find” operations**, like “find all enemies with health < 0.5”.
- **Passing small bits of behavior** into reusable functions without defining a full named function.

Example: find first actor with low health:

```cpp
AActor* FindLowHealthEnemy(const TArray<AActor*>& Enemies, float Threshold)
{
    for (AActor* Enemy : Enemies)
    {
        if (!Enemy) continue;

        // Assume it has a GetHealth() method
        float Health = Enemy->GetHealth();
        if (Health < Threshold)
        {
            return Enemy;
        }
    }
    return nullptr;
}
```

With generic + lambda predicate:

```cpp
template <typename Predicate>
AActor* FindEnemy(const TArray<AActor*>& Enemies, Predicate Pred)
{
    for (AActor* Enemy : Enemies)
    {
        if (Enemy && Pred(Enemy))
        {
            return Enemy;
        }
    }
    return nullptr;
}

// Use it:
AActor* LowHealthEnemy = FindEnemy(Enemies, [](AActor* Enemy) {
    return Enemy->GetHealth() < 0.5f;
});
```

Now `FindEnemy` is reusable with _any_ condition, passed in as a **lambda predicate**.

---

## 6. Quick mental cheatsheet

- **Lambda syntax:**
    
    ```cpp
    [capture](params) -> return_type {
        // body
    };
    ```
    
    Return type often deduced; you can omit `-> return_type` if obvious.
    
- **Common captures:**
    
    - `[]` – capture nothing
    - `[&]` – capture everything by reference
    - `[=]` – capture everything by value
    - `[this]` – capture the current object (`this`)
    - `[&SomeVar]` – capture a specific var by reference
    - `[SomeVar]` – capture a specific var by value
- **Predicate** = function/lambda returning `bool` used for “should this pass?”, “is A < B?”, etc.
    

---

