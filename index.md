# Enhancing the Portability of the Butler

```{abstract}
The LSST Science Pipelines Data Butler and associated middleware, has been developed almost entirely focused on the requirements associated with processing LSST data. Our main acknowledgment of the wider community was a deliberate decision early on to make the software installable standalone from PyPI, thereby theoretically enabling other observatories to use it. This has enabled SPHEREx to adopt the Butler but some early Butler design decisions do not make it as easy as it should be.

This document will explore ways in which we an adjust the Butler and other packages to make it more extensible and reusable.
```

## Introduction

The LSST Science Pipelines {cite:p}`PSTN-019` include a data abstraction layer known as the Butler {cite:p}`2022SPIE12189E..11J` that hides away from the user where files are stored, what file format they are stored in, and how to read and write those files.
The user can query the Butler registry for datasets and only interacts with Python representations of those datasets.
Additionally there is software to combine this registry with a data reduction pipeline definition and generate what is known as a "quantum graph" -- a graph including every processing step to be executed along with the input datasets and all the expected output datasets.
The Butler and graph middleware (`pipe_base`) have been adopted by a variety of places including the SPHEREx project {cite:p}`2026ApJS..285...67A`, Subaru's PFS, and WFST {cite:p}`2025arXiv250115018C`.
There are design decisions in the Butler that cause friction for external users where a Rubin user does not.
This document will discuss these problems and describe potential solutions.

## Configuration

The `daf_butler` repository includes configuration YAML that contains some values that either always need to be ignored by non-Rubin pipelines users or else confuse and contaminate their configurations.
When the configuration system was designed we built configuration overrides into the system via environment variable search paths or explicit paths given to the constructor.
The LSST Science Pipelines developers never needed to do this because everything they needed was in the defaults.
This leads to undoubted paper cuts for external users that should be fixed.

### Configuration Management

The core of the current configuration system was inherited from `daf_persistence` (the "gen2" Butler).
This can be summarized as a hierarchy of dictionaries with full tree merging as additional configurations are combined from the search paths.
There is no schema management and no config validation other than code failing when something unexpected is encountered.
A typo can not be detected and is ignored.
How different parts of the config system merge and what each part of the hierarchy means can be confusing in the absence of clear schema documentation.

It would be much easier to reason about a Butler configuration if the schema was defined as a Pydantic model.
Additionally, this would help to clarify server-side configuration versus client-side configuration --- currently the two are inter-mingled.
This might require that Butler configuration layout is reorganized in a backwards-incompatible manner but we would provide a migration pathway to go from existing `butler.yaml` configurations to the new form.
This could also provide an opportunity to remove some of the butler configuration merging facilities if combined with the proposal for simplifying configuration discovery.

### Configuration Discovery

We propose to replace the current configuration discovery system with a system likely based on Python entry points rather than environment variables and search paths.
We do not yet have a concrete design for this implementation but the core tenet is that the butler should be able to request configuration information from installed Python packages.
We need a mechanism for picking up storage class and datastore configurations, in particular compression options and file format overrides, in addition to the standard mappings of storage classes to formatter classes.

The current approach for external users is to add their storage class and datastore definitions to the `butler.yaml` file such that they are read when the Butler connection is configured.
This does work until you try to use a quantum-backed butler which is not using the main Butler configuration and so requires the environment variable to be set.
Additionally, when used in a client/server configuration it is deemed to be unsafe to allow data from an external network to directly trigger the import of Python code, and it is preferable for the storage class and formatter class definitions to come from locally-installed software.

A key requirement of such a reorganization is that the storage class definitions track the definitions used in data releases -- if DR1 has a different definition of a storage class to that in DR2 that configuration difference has to be preserved (this is also related to the singleton problem discussed later).
Formatter classes are stored in the data release itself so there is no per-data release configuration issue when reading a dataset so long as the mapping from that name to a Python class results in a class that can read that file.
We still need to consider how Butler repositories derived from a data release work and how butler-to-butler transfers work if each butler has differing definitions for a storage class or is using formatter labels that map to different formatter classes.

### Storage Class Definitions

Storage classes are a core concept of Butler and declare how a Python type should be associated with a `DatasetType` and how the data store should handle it.
By default Butler defines all the storage classes needed by the LSST Science Pipelines regardless of whether an external user is using the LSST Science Pipelines or not.
This has two problems: firstly defining many storage classes that are never used leads to performance slow downs (especially at start up) and, secondly and more seriously, since we can add new storage classes at any time it is possible for us to break users with a naming clash.

Furthermore, to simplify development in the beginning the `StorageClassFactory` was configured as a per-process singleton.
This is acceptable for a single butler connection or when connecting to multiple butlers with identical configurations, but unless care is taken it could lead to unforeseen errors in a shared multi-observatory archive where one user may want to make connections to multiple unrelated butler repositories in a single notebook but which have incompatible storage class definitions.
Given that we will likely encounter a bounded set of storage class definitions in a single process it makes sense to migrate from a singleton to something more akin to the global universe cache where multiple storage class factories are stored globally indexed by label and version so that a second Butler instance does not require a reload of a storage class definition that is already known.

Another downside of storage class definitions being available in the current way is that we never noticed that during graph execution an external user needs to inject their own storage class definitions into the environment to even be able to process data.
Since this is quantum-backed butler execution they do not have access to a hand-rolled butler configuration that included their own definitions since they are not using that butler, and the only option is to specify the storage class definitions separately using the `DAF_BUTLER_CONFIG_PATH`.

We need to consider our options regarding storage class discovery.
It is already the case that the butler server is designed never to be required to load a storage class definition since it does not have access to any of the science payload code.

It therefore would make sense for the storage class definitions to be extracted from `ButlerConfig`.

### Entry Points

One option to be considered is that we create a LSST data releases Python package that contains the configuration for every data release (storage classes and formatters) and the new butler configuration supports a configuration label that would be passed to an entry point which would then return the bespoke configuration for that butler.
The client code could then search for a registered entry point that understands that label and requests the storage class definitions for that version.
This would benefit all butler users since it could allow Rubin to have a different definition of storage classes for Data Preview 2 and Data Release 1 and would immediately solve the problem of external users having to keep track of definitions that ship in the core butler code or using code paths that we do not ourselves exercise.

### Complications and Effort Estimate

This is a substantial effort reorganizing how we configure a butler and is likely to be backwards incompatible given that we are required to not-only convert the configuration to Pydantic but also have to make a clean separation of server-side vs client-side configuration and introduce new items into the model to enable entry-point lookup.
A versioning key should be added to the configuration that could be used to handle legacy configurations smoothly.
Once we have separated the items out that tend to get the most edits, we can likely also support native JSON configuration to improve parsing performance at butler startup, with the caveat that entry point lookups will then cause additional overhead.

The `StorageClassFactory` work is only lightly coupled to the config reorganization if we adopt a labeled cache where the singleton is replaced by a new API that can return a new instance by name.
A similar approach could be used for handling formatter configurations.

One serious issue is that currently a `DatasetType` assumes that it can look up the storage class by using the singleton.
This class does not know which Butler it is attached to (and generally should not need to know) but some definition is required somewhere.
Additionally, `StoredFileInfo` and `InMemoryDatasetHandle` assume they can also access a storage class by name at any time but in many cases a Butler is available to them.
Pipeline construction also makes some assumptions concerning storage class availability.
This will need some design work and likely API changes, but might be mitigated by replacing the singleton with the proposed per-label cache so long as a `DatasetType` can know its own label to use similar to how it has to know its universe, although we have to be careful how we handle equality of `DatasetType`s for differing labels.
A butler-to-butler transfer would be required to resolve both storage classes, if the names differ or the labels differ, to determine if the Python type is compatible, but when it is transferred it would need to be modified to adopt the storage class factory of its target butler, similar to how the `conform_to` API ensures that universe differences are handled (the simplest approach may be to replace the `universe` parameter with a `schema` parameter that folds in the universe information and the configuration label rather than adding a second parameter to the constructor and `conform_to`).
Currently dataset type compatibility assumes that the names matching is sufficient, which is only true because of the singleton.
If we adopt this new namespace plus versioning approach for storage classes and formatters (for example "rubin-lsst" and "dp1") then we would ensure that quantum graphs also store this information such that at runtime the correct storage class definitions and formatter configuration can be loaded (pipeline execution requires formatter configuration for writes).

```{important} **Time estimate:**
Taking an existing software-derived schema and configuration system and turning it into a Pydantic schema with runtime extensions is potentially something that can be handled with LLM-assistance.
This is a well bounded problem in that the parts of Butler that accept a `ButlerConfig` are well defined and the config is only needed to configure a handful of classes at run time.

8 weeks.
```

## Schema Migrations

We provide some tooling for doing schema migrations in the form of `daf_butler_migrate`.
The tooling is designed to be able to handle dimension universes that are not named "daf_butler" but we have had no feedback as to whether anyone else is using the tooling.
It would be worth adding some tests that are explicitly not using the default universe name and also consider whether the package should be integrated into `daf_butler` itself.

More importantly, the package should be modified to support Python entry points keyed by the dimension universe name so that the `butler migrate` command can find other migrations and support additional subcommands registered by external users.

```{important} **Time estimate:**
Less than 1 week.
```

## Dependency Management

The core of the Butler middleware is installable from PyPI but the instrument packages, subclasses of `obs_base`, are not because there is a core dependency on the `afw` package when representing camera geometry.
The new `lsst.images` package has its own camera geometry classes that depend solely on the AST library {cite:p}`2016A&C....15...33B`.
We should try to switch the `obs_base` package over to the new geometry specification (the camera geometry is required to work out the visit regions) so that external users have the option of defining `obs` packages of their own without adopting the full science pipelines codebase.
The package currently relies on generating geometries with `afw` and then converting them to the new format, but we would have to re-engineer how the YAML geometry specifications are read in and converted to the new form.
This would likely then lead to some additional functionality requests from external users but that is to be expected.
Once `afw` is removed (it would still be an optional dependency for the legacy formatter I/O code) facilities such as calibration registration and raw data ingest would be available to the wider community.

```{important} **Time estimate:**
4 weeks.
```

## Graph Building

Not all external users use BPS for managing their batch workflows.
This leads to some impedance mismatches as to what good defaults should be.
For example, when making a graph the default for the command-line tool is not to include the datastore records.
At Rubin we know that BPS is designed for include those records and we do not include them by default since they add unnecessary size to the graph when someone is doing anything other than using quantum backed butler for processing.

There has been a request to change the default so that we always include the datastore records.
This seems reasonable and we can ensure that `pipetask run` does not include them when it makes its own graph as currently configured.
One future improvement we would like to investigate is to change how `run` works such that it also uses the graph backed butler (make the graph, execute from the graph, register outputs in the main butler).
This would have the advantage of providing consistent execution at all scales but also solve the problem where we have had reports of external users sometimes trying to run large jobs in parallel with one `run` command and a SQLite butler over NFS.

```{important} **Time estimate:**
1 day to change the default.
4 weeks to change how `run` works to make it consistent for all users (BPS and non-BPS users).
```

## Sequential Processing

At Rubin we are either processing all the data at once for a data release where ordering of inputs does not matter during co-add production, or we are processing single frames (alert production).
This is not generally the way that other observatories operate.

### Incremental improvements

In many cases you have data arriving in real time from the observatory and you want to process it as it arrives so you get the best possible co-add or calibration dataset as fast as possible.
Butler is not set up to handle this because an output dataset is required to not have the same dataID as a dataset that is already present in that collection.
If observation N is combined with observation N+1 and stored as a co-add dataset type, then if you try to combine that co-add with observation N+2 you can not write it out without using a new collection or including some discriminator in the dataID.

This work does not directly benefit Rubin at this time, although it may be possible to integrate this into summit calibration pipelines to provide incremental feedback.

```{important} **Time estimate:**
This is currently thought of as a research topic since it would require analysis of different options including creation of a special dimension that corresponds to a UUIDv7.
Expect anything between 2 weeks and 1 month of work to demonstrate this new functionality.
```

### Persistence Correction

Persistence is a category of detector artifacts where the correction for one observation depends on the previous in. While weak persistence has been measured in LSSTCam's detectors, the effect is small enough to be ignored, as is usually the case for modern CCDs.
This is not true of most infrared detectors, and other projects (SPHEREx, PFS) that otherwise use the Rubin middleware have not been able to use it for this stage of their pipelines, because it requires the same dataset type to be used as both an input and output of a task.

Unlike incremental processing, persistence correction is already wholly compatible with the butler data model; only the quantum graph system would need to be updated.

```{important} **Time estimate:**
Signficant thought has already gone into this problem, and a solid design could probably be delivered with a week of focused effort.  Implementation for a production system ought to be doable with a month's focused effort, but since this could be disruptive to quantum graph file format, that effort might need to be spread out over a longer period to allow for a deprecation cycle.
```

## References

```{bibliography}
```
