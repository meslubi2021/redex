---
id: passes
title: Passes
---

ReDex has a large set of optimization passes that is constantly evolving.
Information in this document may be outdated, inspect the code if necessary.

## AccessMarkingPass

Final objects and private methods can be optimized more aggressively than
virtual objects and public methods.

Devirtualization can result in [NullPointerException]. Two Redex passes perform
devirtualization of methods: `AccessMarkingPass` devirtualizes methods not using
`this`. [`MethodInlinePass`](#methodinlinepass) inlines monomorphic virtual
calls.

The [app's config file](config.md) can override `AccessMarkingPass` behavior.
`finalize_methods`, `finalize_classes`, and `privatize_methods` default to
`true`. `finalize_fields` defaults to `false`.

```
"AccessMarkingPass": {
  "finalize_fields": true
},
```

Pass ordering dependencies:
* `AccessMarkingPass` should be run early as it enables other optimizations.

See related:
* [`MethodDevirtualizationPass`](#methoddevirtualizationpass)

## AnnoKillPass

`AnnoKillPass` originally removed only annotations with no static references in
the code--"build-visible" annotations. It was expanded to remove annotations
referenced statically, but not used at runtime--"runtime-visible" annotations.

`AnnoKillPass` reads configuration options from the [app's config
file](config.md) specifying annotations to be kept or killed. An additional
option specifies whether Redex should attempt to match signatures for removal.

```
"AnnoKillPass" : {
  "keep_annos": [
    "Landroid/view/ViewDebug$CapturedViewProperty;",
    "Landroid/view/ViewDebug$ExportedProperty;"
  ],
  "kill_bad_signatures" : true,
  "kill_annos" : [
    "Lcom/google/inject/BindingAnnotation;"
  ]
},
```

See related:
* [`SystemAnnoKillPass`](#systemannokillpass)

## BridgePass

BridgePass removes bridge methods created by the `javac` compiler as part of
type erasure for covariant generics.

Example of a bridge method in pseudo-bytecode:
```
check-cast*   (for checking covariant arg types)
invoke-{direct,virtual,static}  bridged-method
move-result
return
```

`BridgePass` inlines the target of the bridging, the "bridgee", into the bridge
method by replacing the `invoke-` and adjusting check-casts as needed. The
bridgee can then be deleted. The optimization is not applied if the bridgee is
referenced elsewhere in the code.

Pass ordering dependencies:
* [`ResultPropagationPass`](#resultpropagationpass) should be run after
  `BridgePass` to avoid pattern matching conflicts.

See related:
 * [`ResultPropagationPass`](#resultpropagationpass)
 * [`SynthPass`](#synthpass)

## CheckBreadcrumbsPass

`CheckBreadcrumbsPass` validates Redex codegen against leftover references to
deleted types, methods, or fields.

* Verifies that there are no references to a deleted class definition remaining
  in DEX files (essentially an `internal` class that is not in scope).
* Verifies that the target of a field and method reference exists on the class
  it is defined on.

Redex will warn if it finds dangling references or illegal references to
entities.

## ConstantPropagationPass

`ConstantPropagationPass` substitutes the values of constants into expressions
at compile time. Constant propagation can eliminate multiple expressions,
resulting in a constant load.

`ConstantPropagationPass` does a whole program analysis to replace instructions
with single destination registers with constant loads. The analysis is run
iteratively until a fixed point or configurable limit is reached.

`CostantPropagationPass` should be run before dead code elimination (DCE) passes
as it can create dead code.

See related:
* [`LocalDCEPass`](#localdcepass)
* [`RemoveUnreachablePass`](#removeunreachablepass)

## CopyPropagationPass

`CopyPropagationPass` removes writes of duplicated values to registers in a
basic block. If value `A` and value `B` are aliases, then any moves between
these registers are unnecessary and can be eliminated. Duplicated source
registers can also be deduplicated.

`CopyPropagationPass` can also remove duplicated instructions if the source and
the destination are aliased.

Example: `v0` and `v1` contain the same value and can be treated locally as
aliases:

```
const v0, 0
const v1, 0
invoke-static v0 foo
invoke-static v1 bar
```

can be transformed into

```
const v0, 0
invoke-static v0 foo
invoke-static v0 bar
```

`CopyPropagationPass` should be run before dead code elimination (DCE) passes as
it can create dead code.

See related:
* [`LocalDCEPass`](#localdcepass)
* [`RemoveUnreachablePass`](#removeunreachablepass)

## DedupBlocksPass

Dedup blocks inside of a method. Duplicated blocks are those with the same code
and the same successor. Duplicated blocks can have different predecessors.

`DedupBlocksPass` identifies one of the blocks as the canonical version, then
redirects all predecessors to the canonical block. The pass currenly only
identifies blocks with a single successor, but in the future may identify blocks
with multiple sucessors.

Stack traces for deduplicated blocks will always report the same line number,
but the predecessor line numbers will be correct.

`DedupBlocksPass` should be run after [`InterDexPass`](#interdexpass).

## DelInitPass

DelInitPass deletes unreferenced methods and fields that have no reachable
constructor, as well as constructors for classes that can be removed or for
classes that have another constructor that can be called.

The scope of `DelInitPass` can be limited by a `package_white_list` in the
[app's config file](config.md). Lacking a white list, `DelInitPass` works at
global scope.

`DelInitPass` should be run before
[`RemoveUnreachablePass`](#removeunreachablepass) (RMU) as `DelInitPass` cleans
constructors, enabling RMU to clean up more classes.

See related:
* [`LocalDcePass`](#localdcepass)
* [`RemoveUnreachablePass`](#removeunreachablepass)

## DelSuperPass

DelSuperPass eliminates subclass methods that invoke the superclass method and
trivially return.

`DelSuperPass` only optimizes virtual methods with the following
characteristics:
* The subclass method must match the name and signature of the superclass method
* The subclass method must only invoke the superclass method and either return
  `void` or the result of the callee.

`DelSuperPass` also fixes up references to the removed subclass methods, making
them refer to the superclass method instead. Though [Dalvik's
`invoke-virtual`](https://source.android.com/devices/tech/dalvik/dalvik-bytecode)
would automatically resolve to the correct superclass method, doing this reduces
the number of method references in the Dex file and saves on space.

## FinalInlinePassV2

`FinalInlinePassV2`, or an instance field's value after `<init>`, and inlines
the value in dex code. Note that this pass is separate from the `MethodInline`
and `SwitchInline` passes.

The `DX` tool often introduces verbose bytecode sequences to initialize static
fields in classes it generates. The `encoded_value` equivalents are much more
compact. This pass determines the values of static fields after `<clinit>` and
eliminates the redundant writes to the static field.

This pass applies to both final and non-final static fields. For final statics
it also inlines reads of the static field where possible, replacing them with
constant operations outside of `<clinit>`.

For instance fields, the pass calculates the field's value after `<init>` is
finished. It inlines reads of the instance field where possible.

Unlike a static field, if an instance field were changed outside of `<init>`, it
might have different values for different instances of the class. For classes
with multiple `<init>` the instance field values might differ based on the
constructor. This pass does not inline instance fields that are:

1. Modified outside of their class `<init>`.
2. In a class that have more than one constructor.
3. Accessed by reflection or native code anywhere in code.
4. Accessed in another method that is called inside of the constructor.

Note that this pass does not inline the `CharSequence` type for static or
instance fields because older Dalvik VMs cannot handle this class.

See related:
* [`MethodInlinePass`](#methodinlinepass)

## LocalDcePass

`LocalDcePass` removes dead instructions in a method. Code is considered to be
"dead' if it has no side-effects and does not change its output registers. Code
in a `catch` block is considered live for the duration of the `try`, as any
instruction in the `try` block is assumed to be able to throw. Methods annotated
with `@DoNotOptimize` are not considered for dead code elimination.

Dead code elimination (DCE) differs from RemoveUnreachable (RMU) in two ways:
first, RMU works from global roots (at the scope of Class/Method/Field) whereas
DCE works at the function scope. Second, DCE removes code that does not change
state, for example, a store to a memory address that is not read in the scope of
the block, whereas RMU removes code that is unreachable regardless of its effect
on state.

See related:
* [RemoveUnreachablePass](#removeunreachablepass)

## MethodDevirtualizationPass

`MethodDevirtualizationPass` converts virtual methods with single
implementation to static dmethods.

The [app's config file](config.md):

```
"MethodDevirtualizationPass" : {
  "staticize_vmethods_not_using_this" : true,
  "staticize_dmethods_not_using_this" : true
},
```

See related:
* [`AccessMarkingPass`](#accessmarkingpass)

## ObfuscatePass

`ObfuscatePass` pass obfuscates method and field names. `RenameClassesPassV2`
obfuscates class names.

See related:
* [`RenameClassesPassV2`](#renameclassespassv2)


## OptimizeEnumsPass

`OptimizeEnumsPass` does two things to make use of `Enum` classes more
efficient. It optimizes the use of `Enum` values in `switch` tables and replaces
some uses of `Enum` values with `Integer` singletons.

The `javac` compiler creates [Dalvik packed
switch](https://source.android.com/devices/tech/dalvik/dalvik-bytecode) tables
that contain a generated anonymous class. `OptimizeEnumPass` replaces these
packed `switch` statements with lookups based on the `Enum` ordinal itself. Note
that this optimization does not work with ProGuard obfuscation enabled. ProGuard
can rewrite `Enum` value names such that they no longer match the `Enum` class
name.

`OptimizeEnumsPass` also replaces some uses of an `Enum` with a boxed `Integer`
singleton and keeps the runtime behavior unchanged at the same time.

The pass does not guarantee to erase all the enums, perf sensitive code should
never use enums. An `Enum` is not optimizable if it is:
1. An abstract `Enum`.
2. Reflectively used.
3. Contains an instance field that is not a primitive.
4. Contains non-final instance fields.
5. Cast to any other types, like `java.lang.Object`, `java.lang.Enum`,
   `java.io.Serializable`, `java.lang.Comparable`


## OriginalNamePass

```
"OriginalNamePass" : {
  "hierarchy_roots" : [
    "Ljava/lang/Runnable;"
  ]
},
```

Redex renames classes for performance reasons. Renaming can result in different
class names in debug and release builds, which results in mismatches in logging.
Also, some system functions should not be renamed.

An alternative is to use `OriginalClassName.getSimpleName()` for logging.
`OriginalNamePass` is preferred as is does not significantly increase the APK
size.

## PeepholePass

Replace small code patterns with a more efficient pattern. The optimization
matches known patterns for replacement. It essentially performs a string search
of the code for known inefficient sequences and replaces them with more
efficient code. `PeepholePass` will not replace patterns that span a basic block
boundary. `PeepholePass` can remove no-op function calls such as redundant moves
and appends of null strings.

Peephole pass should be run early.

## ReBindRefsPass

Rebind references to their most abstract type.

The number of methods in a DEX file is limited to 64K. Method definitions (defs)
and references (refs) both count against this limit. The class scope in an
inheritance situation can create needless method refs. Calls based on the
subclassed methods create unnecessary method refs for the subclass. This is
especially true when calls are made through the implicit `this`.

For example, you have a class specialized on `<n>` with a method that calls
`Object.equals(Object)`. All of these calls create a ref `X<n>.equals(Object)`,
each of them counting against the 64K limit. Rebinding them lower in the
hierarchy reduces the number of unique refs.

```java
class X<n>
{
    public void foo<n>(Object o)
    {
        ...
        if (equals(o) {...}
        ...
    }
}
```

`ReBindRefsPass` rebinds all `invoke-virtual` to the base def of the virtual
scope. For `invoke-interface`, it rebinds to the first interface method def. The
optimization is only done as long as there is no change in method visibility: we
walk down the hiearchy as long as the method is public. `ReBindRefsPass`
drastically reduces the number of methods defined in DEX files.

## ReduceGotosPass

Reduces gotos in two ways:
1. When a conditional branch would fallthrough to a block that has multiple
   sources, and the branch target only one has one, invert condition and swap
   branch and goto target. This reduces the need for additional gotos and
   maximizes the fallthrough efficiency.
2. It replaces gotos that eventually simply return by return instructions.
   Return instructions tend to have a smaller encoding than goto instructions,
   and tend to compress better due to less entropy (no offset).

Example, inverting this conditional will eliminate a `goto`:
```
(const v2 0)

(if-eqz v0 :true)
(:back_jump_target)

(return v2)

(:true)
(const v2 1)
(goto :back_jump_target)
```

## RegAllocPass

`RegAllocPass` does register allocation: the process of allocating variables
into the available physical registers. The goal of register allocation is to
avoid "spilling", that is, moving values from registers into memory.

`RegAllocPass` uses a standard graph-coloring register allocator algorithm,
known as the Chaitin-Briggs algorithm.

## RemoveBuildersPass

Remove builder invocations. A trivial builder is one that:

* Doesn't escape the stack (`this` is never passed to a method not in this
  instance, stored in a field, or returned)
* Has no static methods
* Has no static fields

Unreferenced builders are left to be removed by RemoveUnreachablePass (RMU).

See related:
* [`RemoveUnreachablePass`](#removeunreachablepass)
* [`ResultPropagationPass`](#resultpropagationpass)

## RemoveEmptyClassesPass

`RemoveEmptyClassesPass` removes classes that contain no methods or fields.
Classes that are referenced by code or annotations are kept.

See related:
* [`StaticReloPassV2`](#staticrelopassv2)

## RemoveGotosPass

Remove unnecessary control flow edges. A merge of blocks `B` and `C` is done iff:
* `B` jumps to `C` unconditionally
* The only [predecessor] of `C` is `B`
* `B` and `C` both point to the same catch handler

## RemoveInterfacePass

The motivation of this pass is to remove a hierarchy of interfaces extending
each others. The removal of the interfaces simplifies the type system and
enables additional type system level optimizations.

We remove each interface by replacing each invoke-interface site with a
generated dispatch stub that models the interface call semantic at bytecode
level. After that we remove the existing references to them from the
implementors and remove them completely. We start at the leaf level of the
interface hierarchy. After removing the leaf level, we iteratively apply the
same transformation to the now newly formed leaf level again and again until all
interfaces are removed.

Note that this is a critical pass for optimizing GraphQL generated fragment
models. Aside from the fragment model classes themselves, the GraphQL tool chain
also generates a Java interface for each GraphQL fragment namely fragment
interface. The existence of these interfaces greatly complicates the type system
of the generated GraphQL fragment models making merging the underlying model
classes virtually impossible. The other interface removal optimizations like
`SingleImpl` and `RemoveUnreferencedInterface` can address this issue to some
extend. But they are not able to remove the majority of them.
`RemoveInterfacePass` is capable of removing most of the fragment interfaces at
the expense of producing the above mentioned dispatch stubs. Doing so before
Type Erasure paves the way for maximizing the code size reduction we can achieve
in Type Erasure.

## RemoveUnreachablePass

Starting from the roots, recursively mark the other elements that the roots
reference. Afterwards, it deletes all the unmarked elements. While doing the
marking, Redex doesn't attempt to figure out which basic blocks get executed in
each method; doing that for every single method would be too expensive.

More information about `RemoveUnreachablePass` is available in this [note on
Teaching Reachability Analysis about Dependency Injection].

See related:
* [`LocalDcePass`](#localdcepass)

## RemoveUnusedFieldsPass

It's pretty much in the name. A lot of these unread fields are actually
`javac`-generated fields for inner classes. Notably, this turns non-static inner
classes into static ones where possible.

This pass occasionally causes issues because the app may have been relying on an
unread field to stop the GC from deleting an object.

## RemoveUnusedArgsPass

Removes unused parameters. Currently only works on non-virtual methods and
virtual methods that are not part of some overriding inheritance hierarchy.

## RenameClassesPassV2

`RenameClassesPassV2` renames classes to shorter names such as "X.A1c", saving
in APK size, obfuscating the code, and ordering classes to optimize performance
of loading.

`RenameClassesPassV2` will not rename any class mentioned in resources, nor will
it rename anything in blacklisted either by direct class name or as part of
blacklisted package.

`RenameClassesPassV2` relies on the [app's config file](config.md), blacklisting
of the class or hierarchy, or use of reflection.

Logview and bug reports are configured to automatically undo this renaming.

See related:
* [`ObfuscatePass`](#obfuscatepass)

## ReorderInterfacesDeclPass

`ReorderInterfacesDeclPass` list for each class by how frquently the Interfaces
are called. The Interface list is searched linearly when an Interface is called,
so calling an Interface at the list will be faster. An alphabetical sort is used
for tie-breaks in number of incoming calls to preserve consistency across
Classes.

This pass could be improved by checking the number of incoming calls
dynamically.

## ResultPropagationPass

Refactor code, e.g.,

```java
Text.create(context)
    .clipToBounds(false)
    .text(myText)
```

to be as efficient as the less elegant equivalent version:

```java
Text.Builder b = Text.create(context);
b.clipToBounds(false)
b.text(myText)
```

See related:
* [RemoveBuildersPass](#removebuilderspass)

## ShortenSrcStringsPass

Replaces long filename strings with strings used elsewhere in the APK. This
munges the filename component of stack traces. Logview and bug reports
automatically reverse this for you.

## MethodInlinePass

For example, in this code, if `run` is inlined to `main` and the access of `bar`
throws, the stack trace in `main` will show a `NullPointerException` at the
dereference of `this` instead of a call to `run`.

```java
class Foo {
  private String bar;
  public void run() {
    System.out.println(bar);
  }
}

class Main {
  public static void main(String[] args) {
    Foo foo = null;
    foo.run();
  }
}
```

`MethodInlinePass` will not inline a constructor as the Android verifier checks
for a call to `<init>` before any access to the object.

`MethodInlinePass` cannot currently be run after [`InterDexPass`](#interdexpass).

See related:
* [`FinalInlinePassV2`](#finalinlinepassv2)

## SingleImplPass

Removes interfaces with only a single implementation. Any classes referring to
the interface will now refer to the implementation instead. This can cause minor
confusion in stack traces.

## StaticReloPassV2

`StaticReloPassV2` relocates static fields and methods that only have one
calling class to that class. It improves the performance and reduces the app
size.

Pass ordering dependencies:
* `StaticReloPassV2` should be run before `RemoveEmptyClassesPass` as it enables
  more classes to be optimized.

See related:
* [`RemoveEmptyClassesPass`](#removeemptyclassespass)
* [`StaticReloPass`](#staticrelopass)

## StringConcatenatorPass

Reduce string operations as well as reducing the number of strings that need
to be loaded.

Here's an example `<clinit>` method `StringConcatenationPass` will optimize:

```java
public static final String PREFIX = "foo";
public static final String CONCATENATED = PREFIX + "bar";
```

The output code should be equivalent to:

```java
public static final PREFIX = "foo";
public static final CONCATENATED = "foobar";
```

This is a targeted optimization that is only performed on static initializers
with many string concatenations.

## StripDebugInfoPass

`StripDebugInfoPass` removes debug information for instructions that will
never throw. As debug positions can correspond to multiple instructions, we need
to check that none of the instructions will throw. Also, Redex won't strip the
first piece of debug information in a function to preserve the accuracty of
sampling profiles and ANR stack traces.

The [app's config file](config.md) can direct `StripDebugInfoPass` removals at a
more granular level:

```
"StripDebugInfoPass" : {
  "drop_all_dbg_info" : "0",
  "drop_local_variables" : "1",
  "drop_line_numbers" : "0",
  "drop_src_files" : "0",
  "use_whitelist" : "0",
  "cls_whitelist" : [],
  "method_whitelist" : [],
  "drop_prologue_end" : "1",
  "drop_epilogue_begin" : "1",
  "drop_all_dbg_info_if_empty" : "1",
  "drop_synth_aggressive" : "0",
  "drop_line_numbers_preceeding_safe" : "1"
},
```

Pass ordering dependencies:
* `StripDebugInfoPass` should be run early as removal of the debug info should
  make other passes faster.
*  Inlining complicates the flow graph for debug info. `StripDebugInfoPass`
   should be run before any inlining passes, and will not optiimize if inlining
   has been performed.

## SynthPass

`SynthPass` removes synthetic methods introduced by `javac`. `javac` generates
these methods because while Java allows inner classes or nested classes, DEX
bytecode does not. Inner classes, like `class Delta` in this example, are
promoted to top-level classes in the DEX bytecode.

```java
public class Gamma {
    public Gamma(int v) {
        x = v;
    }
    private int x;

    public class Delta {
        public int doublex() {
            return 2*x;
        }
    }
}
```
`javac` generates a synthetic method that allows access to fields, methods, and
constructors in the promoted class. `SynthPass` removes these synthetic methods,
replacing them with a direct access to the field or call to the method or
constructor.

## TrackResourcesPass

An example config file entry:
```
"TrackResourcesPass" : {
  "classes_to_track" : [
    "Lcom/foo/R$drawable;",
    "Lcom/foo/R$string;",
    "Lcom/foo/R$plurals;",
  ],
  "tracked_fields_output": "coldstart_fields_in_R_classes.txt"
},
```

## TypeErasurePass

`TypeErasurePass` shrinks the size of code generated by some frameworks. These
tools produce large amounts of Java code for each component. The code generated
for different component types often shares the same structure, differing only by
the type of the component.

Type Erasure identifies pieces of generated code that have the same "shape".
Erasing the types that differ allows the pieces of generated code to be merged.

## UnreferencedInterfacesPass

`UnreferencedInterfacesPass` removes concrete Interfaces that are not
referenced anywhere in code except in `implements` clauses. Interfaces on
abstract classes are harder to track and are thus considered for optimization.