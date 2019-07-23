# Feature Detection

## Introduction

WebAssembly engines today support different sets of experimental or prototype
features at different stages of standardization. It is necessary for engines to
implement such unstandardized features and for bleeding edge users to use these
features and provide feedback on them as part of the specification development
process. Engines are encouraged to ship these features by default once they are
far enough along in the standardization process, but it is inevitable that
different engines ship features at different times, if at all. As a result,
developers of WebAssembly modules need to produce artifacts supporting multiple
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

The goal of this proposal is to allow functions to be implemented differently
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

A new *feature set* section is introduced that must occur before the type
section if it exists. The feature set section contains a vector of *N* feature
sets, each of which is itself a vector of feature strings. The index of the
first feature set for which all features are supported by the host engine is the
*active feature set index*. If no such index exists, the *active feature set
index* is instead *N*.

The format of the *feature set* section contents:

| Field        | Type           |
|--------------|----------------|
| feature sets | `feature_set*` |

The format of a `feature_set`:

| Field    | Type             |
|----------|------------------|
| features | [`name*`][names] |

[names]: https://webassembly.github.io/spec/core/binary/values.html#names

Another new type of section, the *feature-conditional section* is also
introduced. These sections contain extensions to previous sections such as the
type section or code section, but their contents are only considered to be part
of the module during validation and instantiation if the *active feature set
index* is in the sections' set of activating feature sets. Feature-conditional
sections must immediately follow the sections they are extending, and their contents are interpreted differently for each type of extended section.

The format of a *feature-conditional section*'s contents:

| Field                   | Type         |
|-------------------------|--------------|
| activating feature sets | `varuint32`* |
| contents                | varies       |

The following section types may be extended via feature-conditional sections:

 - Type section
 - Function section
 - Table section
 - Memory section
 - Global section
 - Code section
 - Name section

 Conveniently each of these sections is specified to be a vector of contents, so
 their corresponding feature-conditional sections' contents fields are vectors
 of the same format that are considered to be concatenated to the vector from
 the unconditional section when the feature-conditional section is active. For
 the name section the restriction that each subsection may occur at most once
 will have to be lifted as well.

 The export section may only use indices into the unconditional portions of the
 function, table, memory, and global sections.

 The following section types may *not* be extended, because doing so would
 change the module interface, would not be meaningful, or would be complex and
 have no obvious benefit:

 - Import section
 - Export section
 - Start section
 - Element section
 - Data count section
 - Data section

## Open questions

 - Is there any reason to allow (or not allow) conditional data or element segments?
 - Should the proposal support gracefully degrading to MVP?
   - This would require the additional ability to override function bodies, not just conditionally include one or the other.
   - Also the ability to override segment types
   - Also conditional sections would have to be identified separately
   - Would be more immediately useful, not useful in long term
 - By what mechanism should features be identified? Should nonstandard extensions be allowed (e.g. for signaling the presence of system interfaces)?
 - How should feature names be standardized?
 - Is "feature detection" the best name for this feature? Should it be something like "conditional sections" instead?
 - How does this integrate with dynamic linking and relocation application?
 - How should conditional items be reflected in the text format?
