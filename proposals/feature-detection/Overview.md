# Feature Detection

## Introduction

WebAssembly engines today support different sets of experimental or prototype
features at different stages of standardization. It is necessary for engines to
implement such unstandardized features and for bleeding edge users to use these
features and provide feedback on them as part of the specification development
process. Engines are encouraged to ship these features by default once they are
far enough along in the standardization process, but it is inevitable that
different engines ship features at different times, if at all. As a result,
developers of WebAssembly modules want to produce artifacts supporting multiple
different feature sets.

Unlike in native contexts, WebAssembly's validation rules make it impossible to
instantiate a module that uses new features on an engine that does not support
them even if those new features are not used at runtime. Supporting multiple
feature sets today therefore means building multiple separate modules and
determining out of band which to serve to each user. In a web context, this
usually means performing feature detection by instantiating multiple hardcoded
small modules and dynamically choosing which module to fetch based on the
results. Non-web contexts do not have any standard mechanisms at all for
dynamically selecting one of multiple versions of a module.

Users may alternatively choose to target the smallest common subset of features
supported by the engines they wish to target. This is much simpler from the user
perspective but has the unfortunate effect of making new features underutilized,
slowing and demotivating their development. It would be beneficial to both users
and the wider WebAssembly ecosystem if were easier to use new WebAssembly
features in a backward-compatible manner, i.e. without running afoul of older
validation rules.

The goal of this proposal is to allow modules to be implemented differently
depending on the feature sets supported by the host engine in such a way that
the engine does not need to know anything about unsupported features in order to
validate the module. This will allow users to use the same module on engines
supporting different feature sets, letting them take advantage of new features
where possible and otherwise gracefully falling back to alternative
implementations.

Additional principles motivate the design of this proposal:

 - Modules should be able to support multiple overlapping feature sets with
   minimal code duplication or bloat.
 - It is an explicit non-goal to allow multiversioned modules to validate on
   engines that do not support this feature detection proposal. Although it
   would be possible to achieve this goal, it would incur extra complexity that
   would become technical specification debt in a future where all engines
   support this proposal anyway.
 - It is an explicit non-goal to support additional functionality that could in
   principle be supported by a more general feature detection proposal,
   e.g. optional imports.

## Design

There are two parts to this design. First, the binary format is extended to
allow many section types to appear multiple times. Second, a new type of
conditional section is introduced that can wrap any other section except
itself. During binary decoding, the contents of conditional sections are decoded
into a section only if the feature set provided by the hosts satisfies a
predicate encoded in the conditional section. Otherwise the conditional section
is skipped entirely during decoding. Together, these two extensions provide a
mechanism for providing alternate module contents for different feature sets
while minimizing duplication of common contents.

The following sections currently contain vectors, so the binary format is
extended to allow these sections to appear multiple times, with their vector
contents being concatenated in the order of their appearance.

 - Type section
 - Import section
 - Function section
 - Table section
 - Memory section
 - Global section
 - [Event section][Event] (introduced by exception handling)
 - Export section
 - Element section
 - Code section
 - Data section

The start section does not contain a vector, but the `start` component of the
module structure is updated to be a `vec(funcidx)` rather than a `funcidx`. The
vector is comprised of the individual start functions from each start section in
the binary module. The start functions are executed sequentially during
instantiation.

The other section that does not contain a vector is the [DataCount
section][DataCount], introduced in the bulk memory proposal. For the purposes of
validating the code section, the data count is taken to be the sum of the
contents of each individual DataCount section.

[Event]: https://github.com/WebAssembly/exception-handling/blob/master/proposals/Exceptions.md#event-section
[DataCount]: https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md#datacount-section

### Conditional Sections

New *conditional sections* are introduced to wrap other sections. Conditional
sections contain a *predicate* and binary *contents*. In order to maximize
future compatibility, the contents of a conditional section are skipped entirely
during binary decoding unless the features supplied by the host satisfy the
conditional section's predicate. If the predicate is satisfied, however, the
binary contents are decoded as a section. A conditional section is malformed if
its predicate is satisfied and it contains another conditional section.

Format of a conditional section:

| Field     | Type           |
|-----------|----------------|
| predicate | `feature_set*` |
| contents  | `u8*`          |

Format of a `feature_set`:

| Field    | Type       |
|----------|------------|
| features | `feature*` |

Format of a `feature`:

| Field   | Type           |
|---------|----------------|
| negated | u8             |
| name    | [`name`][name] |

Predicates are in disjunctive normal form, so they are satisfied if any of their
component `feature_set`s are satisfied. A `feature_set` is satisfied if all of
its component `feature`s are satisfied. A `feature` is satisfied if its
`negated` field is 0 and the host supplies the feature or if its `negated` field
is 1 and the host does not supply the feature. A `feature` is malformed if its
`negated` field has any value besides 0 or 1.

[name]: https://webassembly.github.io/spec/core/binary/values.html#names

## Example

Consider having a function `a` specialized for feature sets `{foo}` and `{}` and
a function `b` specialized for feature sets `{foo, bar}`, `{foo}`, and
`{}`. In C this might look like the following:

```C
__attribute__((target("foo")))
int a() {...}

__attribute__((target("default")))
int a() {...}

__attribute__((target("foo,bar")))
int b() {...}

__attribute__((target("foo")))
int b() {...}

__attribute__((target("default")))
int b() {...}
```

Then to minimize code duplication, each version of `a` should get its own
conditional section and each version of `b` should get its own conditional
section as well. The feature sets specified in the source are translated into
logical formulae by taking the conjunction of (1) its features and (2) the
negations of the logical formulae for all feature sets with higher
precedence. This lowering ensures that only one conditional section predicate
can be satisfied at a time for each function. This gives the following formulae
for function `a`:

 1. `foo`
 1. `true /\ ~foo`

And the following formulae for function `b`:

 1. `foo /\ bar`
 1. `foo /\ ~(foo /\ bar)`
 1. `true /\ ~(foo /\ ~(foo /\ bar)) /\ ~(foo /\ bar)`

Simplifying the latter formulae into disjunctive normal form gives the
following sequence of sections:

| section contents | predicate       |
|------------------|-----------------|
| unconditional    | n/a             |
| `a.foo`          | `(foo)`         |
| `a.mvp`          | `(~foo)`        |
| `b.foobar`       | `(foo /\ bar)`  |
| `b.foo`          | `(foo /\ ~bar)` |
| `b.mvp`          | `(~foo)`        |

## Open questions

 - Is there any reason to allow (or not allow) conditional data or element segments?
 - Should the proposal support gracefully degrading to MVP?
   - This would require the additional ability to override function bodies, not
     just conditionally include one or the other.
   - Also the ability to override segment types
   - Also conditional sections would have to be identified separately
   - Would be more immediately useful, not useful in long term
 - By what mechanism should features be identified? Should nonstandard
   extensions be allowed (e.g. for signaling the presence of system interfaces)?
 - How should feature names be standardized?
 - Is "feature detection" the best name for this feature? Should it be something
   like "conditional sections" instead?
 - How does this integrate with dynamic linking and relocation application?
 - How should conditional items be reflected in the text format?
 - Should feature sets be defined in their own section instead of repeated in
   each conditional section?
 - Should any steps be taken to prevent the module interface from changing
   conditionally?
