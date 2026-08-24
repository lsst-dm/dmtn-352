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
This might require that Butler configuration layout is reorganized in a backwards-incompatible banner but we would provide a migration pathway to go from existing `butler.yaml` configurations to the new form.
This could also provide an opportunity to remove some of the butler configuration merging facilities if combined with the proposal for simplifying configuration discovery.

### Configuration Discovery

We propose to replace the current configuration discovery system with a system likely based on Python entry points rather than environment variables and search paths.
We do not yet have a concrete design for this implementation but the core tenet is that the butler should be able to request storage class definitions from installed Python packages.

The current approach for external users is to add their storage class definitions to the `butler.yaml` file such that they are read when the Butler connection is configured.
When used in a client/server configuration it is deemed to be unsafe to allow data from an external network to directly trigger the import of Python code, and it is preferable for the storage class definitions to come from locally-installed software.

A key requirement of such a reorganization is that the storage class definitions track the definitions used in data releases -- if DR1 has a different definition of a storage class to that in DR2 that configuration difference has to be preserved (this is also related to the singleton problem discussed later).

### Storage Class Definitions

Storage classes are a core concept of Butler and declare how a Python type should be associated with a `DatasetType` and how the data store should handle it.
By default Butler defines all the storage classes needed by the LSST Science Pipelines regardless of whether an external user is using the LSST Science Pipelines or not.
This has two problems: firstly defining many storage classes that are never used leads to performance slow downs (especially at start up) and, secondly and more seriously, since we can add new storage classes at any time it is possible for us to break users with a naming clash.

Furthermore, to simplify development in the beginning the `StorageClassFactory` was configured as a per-process singleton.
This is acceptable for a single butler connection or when connecting to multiple butlers with identical configurations, but unless care is taken it could lead to unforeseen errors in a shared multi-observatory archive where one user may want to make connections to multiple unrelated butler repositories in a single notebook.

## Schema Migrations

We provide some tooling for doing schema migrations in the form of `daf_butler_migrate`.
The tooling is designed to be able to handle dimension universes that are not named "daf_butler" but we have had no feedback as to whether anyone else is using the tooling.
It would be worth adding some tests that are explicitly not using the default universe name and also consider whether the package should be integrated into `daf_butler` itself.

## Dependency Management

The core of the Butler middleware is installable from PyPI but the instrument packages, subclasses of `obs_base`, are not because there is a core dependency on the `afw` package when representing camera geometry.
The new `lsst.images` package has its own camera geometry classes that depend solely on the AST library {cite:p}`2016A&C....15...33B`.
We should try to switch the `obs_base` package over to the new geometry specification.
This would likely then lead to some additional functionality requests from external users but that is to be expected.
Once `afw` is removed (it would still be an optional dependency for the legacy formatter I/O code) facilities such as calibration registration and raw data ingest would be available to the wider community.

## Proposed Enhancements

This section summarizes the proposed enhancements.

1. Replace the `StorageClassFactory` singleton with a per-butler singleton so that different butler repositories can not interfere with each other.
2. Replace the current configuration system with Pydantic models and improve configuration detection and overrides, separating server-side and client-side configuration.
   This would also involve moving the LSST Science Pipelines definitions out of `daf_butler`.
3. Remove the `afw` dependency from `obs_base` (apart from legacy formatters and assemblers).

## References

```{bibliography}
```
