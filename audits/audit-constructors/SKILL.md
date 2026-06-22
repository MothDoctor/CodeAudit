---
name: audit-constructors
description: Audit C++/Unreal classes for redundant constructors that can be removed — empty-bodied constructors with no initializer list or only a pure Super() forward, which the compiler/UHT-generated constructor already provides. Use when asked to find empty/trivial/unneeded constructors, declared-but-do-nothing constructors, or constructors to delete. Finds every case in one pass, lists them as a table, then removes the declaration and definition (or converts USTRUCT default ctors to = default).
---

# Redundant constructor audit

## What is removable

A constructor that does nothing the compiler/UHT-generated one would not. Delete both the header declaration and the cpp definition.

- Empty body, no initializer list: `UFoo::UFoo() {}`.
- Empty body, pure forward only: `UFoo::UFoo(const FObjectInitializer& OI) : Super(OI) {}` (or `: Super(Initializer)`).

For a `UCLASS`, removing the only constructor is safe: `GENERATED_BODY()` / UHT supplies a default constructor, and for the `(const FObjectInitializer&)` signature the generated one forwards the initializer to `Super` correctly.

## What is NOT removable (leave alone)

- Any non-empty body: `CreateDefaultSubobject`, setting `*Class` defaults, `StackTag = ...`, `bAutoActivate = true`, appending to arrays, `static_assert`, etc.
- Any initializer list that sets a member to a non-default value: `: MyPlayerConnectionType(EPlayerConnectionType::ActivePlayer)`, `: Index(0)` where the member has no in-class default, `: DisplayGamma(2.2f)`, etc.

## Special cases (do not blind-delete)

- **USTRUCT default constructor with a sibling parameterized constructor.** The parameterized ctor suppresses the implicit default, and USTRUCT reflection needs a default ctor. Do NOT remove. Convert the empty `{}` to `= default` in the header and delete the cpp definition. Example: `FLocalMessageOk()` alongside `FLocalMessageOk(const FText&, const FText&)`.
- **Redundant null member inits.** `: PrimaryWidgetStack(nullptr)` or `: WorldContextObject(nullptr), LocalPlayer(nullptr)` restate the default (`TObjectPtr` and raw `UObject*` members are already null-initialized), so the constructor is effectively trivial and removable. Flag these explicitly since the initializer list is non-empty but does nothing.
- **`: Super(FObjectInitializer::Get())` that substitutes a fresh initializer.** When a constructor takes `const FObjectInitializer& ObjectInitializer` but forwards `FObjectInitializer::Get()` instead, it discards the caller's initializer. That is almost certainly a bug. Removing the constructor both deletes dead code and fixes it (the generated ctor forwards the real initializer). Recommend removal but call out the behavior fix. A no-arg ctor doing `: Super(FObjectInitializer::Get())` is lower confidence (may be intentional for the subclass) — flag, do not auto-remove.

## Discovery: one pass, no blind spots

Constructor definitions sit at column 0 as `Class::Class(`, while method definitions have a return-type prefix (`void Class::Method(`). So a single anchored grep isolates constructors and destructors:

- `Grep` pattern `^\w+::~?\w+\(` over `*.cpp` with `-A 7` to capture the initializer list and body. Use `head_limit: 0` and page with `offset` if the output is truncated, so no file is dropped.
- Do NOT restrict the class-name prefix. An earlier `^[UAF]\w+::~?[UAF]\w+\(` pattern silently skipped Slate (`S`), interface (`I`), and template (`T`) classes — e.g. a Slate widget's empty `SFoo::SFoo()` never appeared. The `^` anchor alone already separates constructors/destructors from methods, because a method definition leads with its return type (`void Class::Method(`) so the class name is not at column 0. Discard the few non-ctor matches the broader pattern catches (`operator==`, etc.) by eye.
- This scan covers constructor/destructor *definitions in `.cpp` files only*. Constructors defined inline in headers (forwarding ctors, small structs) are a separate pass and are not claimed to be covered — say so if you only ran the cpp scan.
- A few constructors may still escape the `-A 7` window (long initializer lists). For any whose body you could not see, read the file's top region directly. Do not classify a constructor you have not seen the full body of.
- Enumerate header declarations too (`Grep` the class for the ctor signature) — both the declaration and the definition must go. Watch for the default-argument form `(const FObjectInitializer& OI = FObjectInitializer::Get())` in the header.

Classify each constructor into: removable (empty + trivial), USTRUCT-default (-> `= default`), redundant-null-init (removable), `Super(FObjectInitializer::Get())` (decision), or keep (real work).

## Report before editing

List candidates as a table first, then get confirmation (or proceed if told "remove all" / "go with A and B"). Columns:

| Class | cpp:line | Initializer | Action |
|---|---|---|---|
| `UFoo` | Foo.cpp:13 | none | remove decl + def |
| `UBar` | Bar.cpp:11 | `: Super(OI)` | remove decl + def |
| `FMsgOk` | Msg.cpp:10 | none (USTRUCT) | `= default`, drop def |

Group by action (remove entirely / convert to `= default` / decision-needed / keep) so the user can approve a subset.

## Applying the removal

- Read each file before editing (the edit tool requires it).
- Delete the cpp definition AND its trailing blank line so two blank lines do not collapse together. These ctors are usually the first thing after the includes / `UE_INLINE_GENERATED_CPP_BY_NAME(...)`, so removing the block leaves the includes followed by the next function with correct spacing.
- Delete the header declaration line.
- For USTRUCT default ctors: change the header line to `FMsgOk() = default;` and delete the cpp definition.
- A class may use `GENERATED_UCLASS_BODY()` instead of `GENERATED_BODY()`. `GENERATED_UCLASS_BODY()` declares a `(const FObjectInitializer&)` constructor that the class must define. If you remove that definition, also switch the macro to `GENERATED_BODY()`, otherwise the link breaks.

## C++ style

- No `auto`. Spell out types.
- Do not introduce in-class default member initializers as part of this audit unless asked — that is a separate refactor. This audit only removes constructors that are already redundant.
