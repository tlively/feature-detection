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

 - Module import and export interfaces, including names and types, should not be
   allowed to be conditional on supported features. If they were, modules would
   be much harder to reason about because their interface with their host would
   be context dependent. Note that this restriction prevents some potentially
   useful uses of feature detection, such as importing or exporting a shared
   memory only on engines that support threads, or exposing SIMD or multivalue
   types in exported function signatures only on engines that support those
   respective features.
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

New *conditional subsections* are introduced in a number of existing section
types. Each conditional subsection contains a sequence of activating feature
sets followed by contents. The conditional subsections' contents are encoded as
a vector of uninterpreted bytes, but if a conditional subsection is *active* its
contents are reinterpreted to be of the same format as its parent section's
unconditional (MVP) contents. If those contents are a vector, the vectors in the
contents field of active conditional subsections are appended to the
unconditional vector. Otherwise the contents of activated conditional
subsections replace the unconditional contents.

Format of a conditional subsection:

| Field                   | Type           |
|-------------------------|----------------|
| activating feature sets | `feature_set*` |
| contents                | `u8*`          |

Format of a `feature_set`

| Field    | Type             |
|----------|------------------|
| features | [`name*`][names] |

[names]: https://webassembly.github.io/spec/core/binary/values.html#names

The following sections are extended with optional conditional subsections that
append their content vectors to the unconditional content vectors:

 - Type section
 - Function section
 - Table section
 - Memory section
 - Global section
 - Element section
 - Code section
 - Data section
 - Name section

The following sections are extended with optional conditional subsections whose
contents replace the unconditional section contents (TODO: what if there are
multiple active conditional subsections?):

 - Start section
 - Data count section

The following sections are explicitly excluded from having conditional
subsections because they define the module's external interface, which should
not depend on the host context.

 - Import section
 - Export section

Multiple feature sets may be subsets of the available feature set, but only one
may be the *active feature set*. This is the first occurring feature set in the
module that is a subset of the available feature set. Any conditional subsection
whose *activating feature sets* field contains the active feature set is itself
active.

## Example

Consider having a function `a` specialized for feature sets `{foo}` and `{}` and
a function `b` specialized for feature sets `{foo, bar}`, `{foo}`, and
`{}`. Then to minimize code duplication, each version of `a` should get its own
conditional subsection and each version of `b` should get its own conditional
subsection as well, giving the following ordering.

| subsection    | activating feature sets |
|---------------|-------------------------|
| unconditional | n/a                     |
| `a.foo`       | `{foo, bar}, {foo}`     |
| `a.mvp`       | `{}`                    |
| `b.foobar`    | `{foo, bar}`            |
| `b.foo`       | `{foo}`                 |
| `b.mvp`       | `{}`                    |

On a host supporting features `foo` and `bar`, all three features sets are
subsets of the available features, but `{foo, bar}` is the active feature set
because it occurs first. The active conditional subsections are `a.foo` and
`b.foobar`.

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
