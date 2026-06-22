---
name: audit-weakptr-resolve
description: Audit C++/Unreal code for redundant TWeakObjectPtr resolves — the "double lookup" where a weak pointer is validated with IsValid() (or implicitly) and then dereferenced again via .Get(), operator->, implicit T* conversion, or *. Use when asked to check/fix double lookup of weak pointers, redundant IsValid/Get, or weak-pointer dereference patterns. Finds every case in a single pass, lists them as a table, then collapses each to one .Get() with a null check.
---

# TWeakObjectPtr redundant-resolve audit

## What counts as a defect

A `TWeakObjectPtr` (or `TWeakInterfacePtr`, `TWeakPtr`) resolved more than once when one resolve would do. Each of these resolves the weak pointer against the GC object array, so two of them on the same value is wasted work:

- `Ptr.IsValid()` then `Ptr.Get()` — the canonical double lookup.
- `Ptr.IsValid()` then `Ptr->Member` — `operator->` resolves again.
- `Ptr.IsValid()` then `Func(Ptr)` or `Cast<X>(Ptr)` — implicit `T*` conversion resolves again.
- `Ptr->A()` ... `Ptr->B()` — every `operator->` is a fresh resolve, even with no `IsValid()` guard.

The fix is always the same: resolve once into a typed local, null-check the local, operate on the local.

```cpp
// before
if (WeakThing.IsValid())
{
    WeakThing.Get()->DoStuff();      // or WeakThing->DoStuff();
}

// after
if (UThing* Thing = WeakThing.Get())
{
    Thing->DoStuff();
}
```

## Two defects are bundled, not one

`IsValid()` followed by a deref hides a second, more serious bug. `IsValid()` only proves the weak pointer itself is live. It says nothing about intermediate accessors in the chain:

```cpp
// IsValid() guards LocalPlayer, NOT GetSlateUser() which can return null
if (!LocalPlayer.IsValid() || LocalPlayer->GetSlateUser()->GetFocusedWidget() == nullptr)
```

When you collapse the resolve, also null-check every intermediate accessor result that can return null (`GetSlateUser()`, `GetPersonalGameSettings()`, etc.). Cross-check sibling call sites: if other callers guard the same accessor with `if (X = ...)` or `ensure(X)` or an early `== nullptr` return, the unguarded one is a latent crash, not a style nit. Report it as a correctness fix, and note the behavior change (crash -> graceful decline).

## Discovery: one pass, no blind spots

Do NOT scope candidates to "files that declare `TWeakObjectPtr` AND call `.Get()`". That intersection has three blind spots, each of which has hidden a real defect:

1. Weak pointers declared in the engine and reached through a parameter or member (e.g. `AController::StartSpot`, `Player->StartSpot`) never appear in a project-local `TWeakObjectPtr<` grep.
2. Files that deref only through `operator->` (never `.Get()`) drop out of any `.Get()`-keyed search.
3. Same-named members in different classes (a weak `GameState` in one class, a `TObjectPtr GameState` in another) create false matches that waste a pass.

Run BOTH searches below and union the results. Together they have no blind spot:

**Search A — every `IsValid()`, filter to weak receivers.** This catches all guarded cases, including engine weak pointers reached via accessor or parameter:
- `Grep` for `\.IsValid\(\)` across the whole source tree.
- For each hit, identify the receiver's type. Keep `TWeakObjectPtr` / `TWeakInterfacePtr` / `TWeakPtr`. Discard `TSharedPtr` / `TSharedRef` (their `IsValid()`/`Get()` are trivial pointer checks, not GC resolves), `FGameplayTag::IsValid()`, `FInputDeviceId::IsValid()`, `FTimerHandle::IsValid()`, and other value-type `IsValid()`.

**Search B — every weak identifier, find unguarded `operator->`/implicit uses.** This catches the no-`IsValid` per-use resolves:
- `Grep` for `TWeakObjectPtr\s*<` to enumerate every weak identifier: members, function parameters (including the default `TWeakObjectPtr<>` = `UObject`), locals, and the element type of `TArray` / `TSet` / `TMap` of weak pointers.
- For each identifier `Name`, grep `Name->`, `*Name`, and `Name` passed bare as a call argument or `Cast<>()` operand. Each is a resolve; if there is more than one, or one plus an `IsValid()`, it is a candidate.

Use the authoritative weak-identifier set from Search B to resolve the type questions Search A raises — that is how you discard the same-named non-weak members in other classes.

## Report before editing

List every candidate as a table first, then get confirmation (or proceed if the task said "fix all"). Columns:

| Site (file:line) | Identifier | Resolves before | Defect | Fix |
|---|---|---|---|---|
| Foo.cpp:90 | `WeakValueSetting` | `IsValid()` + `.Get()` | redundant resolve | one `.Get()` local |
| Bar.cpp:195 | `WeakLocalPlayer` | `IsValid()` + `operator->` + null chain | redundant resolve AND unguarded `GetSlateUser()` | local + guard intermediate |

Also keep a short "verified clean" list (identifiers already on a single `.Get()`, and same-named non-weak members you ruled out) so the audit is auditable and the reader can see nothing was skipped.

## Naming the deref local

Keep the deref local's name identical across functions for the same member (do not call it `PlayerController` in one function and `Controller` in another). Do not rename the cached weak member itself as part of this audit.

## C++ style (match the surrounding code)

- No `auto`. Spell out the pointee type in the deref local, including in range-for and structured bindings.
- One resolve, one local, null-check the local. Preserve the original `&&` / `||` short-circuit order when collapsing a compound condition (resolve the weak pointer before the condition, but keep the relative order of the other operands).
- Preserve existing behavior unless the change fixes a crash. If you add a guard that converts a latent null-deref into an early-out, call that out explicitly as a behavior change.
- `const` the deref local when the surrounding method is const or the value is only read.
