# Feasibility study: DMTN-352 Configuration proposals

Scope: the proposals in the DMTN-352 "Configuration" section --- Pydantic
configuration schema, client/server configuration separation, removal of the
`StorageClassFactory` singleton, and per-butler configuration supplied by
external packages via entry points.

Method: static analysis of `daf_butler` at `562029fef`, plus local checkouts of
`pipe_base`, `ctrl_mpexec`, `dax_obscore`, `daf_butler_migrate`,
`daf_butler_admin`.
No test suite was run (no EUPS environment configured); every claim below is
traceable to a file and line.
YAML parse timings were measured with stock PyYAML against the shipped config
files.

Status: DMTN-352 has since been revised to incorporate most of the findings
below --- the labelled cache, the `DatasetType` resolution problem, the
`conform_to` analogy, the composite `schema` parameter, the transfer-time
resolution rule, and an 8-week estimate.
This document is the supporting evidence and the detail that does not belong
in the technote.

## Summary

All the proposals are feasible.
Three of them are substantially further along than the original text suggested,
because the migration has been happening incrementally for some time.
One --- removing the `StorageClassFactory` singleton --- is harder, and its
difficulty is concentrated in a single question (how a `DatasetType` finds its
storage class definitions) that has a clean answer by analogy with dimensions.

The original 4-week estimate was too low.
7--10 weeks of engineering is realistic for the integrated package, excluding
cross-package coordination latency.
The revised 8-week figure in the technote sits inside that range.

## Current state

### The config system is smaller than it looks

Only 14 non-test modules in `daf_butler` consume a `ButlerConfig`:

```
_butler_config.py  _butler.py  _labeled_butler_factory.py  _quantum_backed.py
_rubin/file_datasets.py  _standalone_datastore.py  _storage_class.py
datastore/composites.py  direct_butler/_direct_butler.py
registry/_registry_factory.py  registry/sql_registry.py
remote_butler/_config.py  remote_butler/_factory.py  script/configDump.py
```

Downstream, direct consumption is almost nil.
`daf_butler_migrate/database.py:84` is the only non-test `ButlerConfig`
construction outside `daf_butler`; `pipe_base` references the type once as an
annotation (`quantum_provenance_graph.py:2071`).
`ctrl_mpexec` and `daf_butler_admin` do not touch it at all.

The claim that "the parts of Butler that accept a `ButlerConfig` are well
defined" holds up.

### Pydantic adoption is already well advanced

Three of the five config components are already Pydantic, or have a Pydantic
model behind a `ConfigSubset` shell:

- `DimensionConfig` --- the largest and most complex section --- has a full
  Pydantic model (`SerializedDimensionConfig`) with `to_simple`/`from_simple`
  round-tripping (`dimensions/_config.py:160-197`).
  `makeBuilder` already consumes the *validated* model, not the dict.
- ObsCore config is Pydantic end to end (`registry/obscore/_config.py`),
  embedded at `registry.managers.obscore.config`.
- The `RemoteButler` client section (`RemoteButlerConfigModel`) and the whole
  server deployment config (`ButlerServerConfig`, via `pydantic-settings`) are
  Pydantic (`remote_butler/_config.py`, `remote_butler/server/_config.py`).
- Individual storage class entries are already validated by
  `_StorageClassModel` (`_storage_class.py:128-137`).

31 non-test modules in `daf_butler` define Pydantic models.
The proposed pattern is not novel here; it is the direction the code has been
moving.

Not yet modelled: the top-level butler config keys, `RegistryConfig` outside
the obscore manager, `DatastoreConfig`, and `RepoTransferFormatConfig`.

### The discovery machinery is the actual complexity

`doc/lsst.daf.butler/configuring.rst` needs roughly 30 lines of prose to
describe how one config value is resolved.
Six independent mechanisms compose:

1. explicit `searchPaths` argument,
2. `$DAF_BUTLER_CONFIG_PATH`,
3. `defaultConfigFile` per `ConfigSubset` subclass,
4. a second defaults file discovered by importing `cls` and reading its
   `defaultConfigFile` (`_config.py:1281-1297`),
5. `containerKey` recursion for chained datastores (`_config.py:1310-1315`),
6. `!include` and `includeConfigs`, plus `<butlerRoot>` substitution.

Every one exists to let a value be supplied from somewhere other than the file
the user named.
If configuration instead arrives from installed packages through entry points,
all six collapse.

### `daf_butler` ships configuration for software it does not depend on

`configs/storageClasses.yaml` defines 120 storage classes.
Only 10 have a `pytype` that is a builtin or an `lsst.daf.butler` type.
The rest name `lsst.afw.*` (23), `lsst.ip.isr` (7), `lsst.pipe.tasks` (5),
`lsst.images` (14), and also `rail`, `qp`, `spectractor`, `healsparse`,
`lsst.atmospec`, `lsst.ts.observatory`, `lsst.meas.transiNet`,
`lsst.analysis.tools`, `lsst.scarlet.lite`.

`configs/datastores/formatters.yaml` has 94 entries; by package, 40 point at
`lsst.obs.*`, 11 at `lsst.images`, 6 at `lsst.meas.*`, 5 at `lsst.pipe.*`,
3 at `lsst.atmospec`.

Converters add a third axis of coupling (e.g. `Psf` converts from
`lsst.images.psfs.PointSpreadFunction`).

The large majority of the shipped configuration is unusable by anyone not
running the LSST Science Pipelines.

### Measured startup cost

Raw PyYAML parse of the default configs, best of 5:

| file | ms |
|---|---|
| `storageClasses.yaml` | 12.0 |
| `dimensions.yaml` | 8.8 |
| `datastores/formatters.yaml` | 4.1 |
| `registry.yaml` | 0.4 |
| `datastores/composites.yaml` | 0.3 |

About 26 ms per full parse, before any `Config` merging or `StorageClass`
derivation, and before the search path multiplies the reads.
Re-parsing the same content as JSON costs 0.03--0.04 ms --- roughly 300x
cheaper.

Recent history (`77a7eff80` "Reuse storage class definitions derived from an
identical config", `b16f67fd9` "Speed up Config merging with a dict fast
path", `27713369c` "Use a dedicated copy routine") shows this cost has already
been attacked from the optimisation side; the proposals here remove the cost
rather than amortise it.

## Feasibility by proposal

### 1. Pydantic configuration schema --- feasible, low risk

The V2-plus-translator strategy works.
`DimensionConfig` is a live proof: a Pydantic model behind a legacy loader,
with lossless conversion both ways.

Two parts of the schema need care.

**The datastore section is a recursive, open, discriminated union.**
`cls` selects the datastore type, each type contributes its own defaults file,
and `ChainedDatastore` nests a list of datastore configs under
`containerKey = "datastores"` (`datastores/chainedDatastore.py:113`).
Third parties can define their own `Datastore` subclasses, so the union cannot
be closed at the `daf_butler` level.
This needs a registry where a `Datastore` subclass declares its config model,
resolved at validation time.
It is the most intricate modelling task in the work, and it cannot be started
until the client/server split of the section is settled, since that determines
which keys the model has to carry at all.

**Lookup sections cannot be fully validated at load time.**
`processLookupConfigs` (`_config_support.py:238`) accepts keys that are dataset
type names, storage class names, `dim1+dim2` dimension expressions, or
`field<value>` dataId qualifiers, with values that are either a string or a
nested mapping.
Validating the dimension-expression keys requires a `DimensionUniverse`, which
is not known until the registry is open.
Validation must therefore be two-phase: structural at load, semantic once the
universe is bound.
So "a typo can not be detected" is only partly fixed --- typos in formatter and
template lookup keys remain uncatchable at load time.

`Butler.makeRepo` (`_butler.py:540-607`) will change shape.
It currently writes a deliberately *partial* config that depends on runtime
defaults merging, with a `standalone` flag to expand everything.
Under V2 the natural output is always a complete validated model and
`standalone` loses its meaning --- a simplification, but a behaviour change for
anyone who hand-edits `butler.yaml` afterwards.

### 2. Client/server separation --- feasible, small, do it early

The line is not drawn anywhere today, but the code is close to it, because
storage class resolution is already lazy almost everywhere:

- `DatasetType.__init__` stores a name and resolves only on the `.storageClass`
  property (`_dataset_type.py:220-226, 429-440`).
- `DatasetType.__eq__` explicitly avoids forcing resolution
  (`_dataset_type.py:290-310`), and `storageClass_name` exists so callers can
  avoid it.
- `DatasetType.__reduce__` pickles the storage class *name*
  (`_dataset_type.py:815-829`).
- No handler under `remote_butler/server/` mentions storage classes at all.

Two server-reachable violations found, one small and one structural.

**The small one, which turned out not to be a defect.**
`registry/datasets/byDimensions/_manager.py:422` does
`dataset_type.storageClass.name`, forcing a factory lookup to get back a name
it already holds --- but the adjacent comment says this is deliberate ("Force
the storage class to be loaded to ensure it exists and there is no typo in the
name"), and there is no register endpoint among the server's external routes,
so `registerDatasetType` is not server-reachable.
Left as is.

**The structural one: every configuration lookup forces resolution.**
`DatasetType._lookupNames()` builds its `LookupKey` tuple by calling
`self.storageClass._lookupNames()` and, for components,
`self.parentStorageClass._lookupNames()` (`_dataset_type.py:704-706`).
Both force the storage class to be resolved.

But `StorageClass._lookupNames()` is a one-liner returning
`(LookupKey(name=self.name),)` (`_storage_class.py:372-382`) --- it needs only
the *name*, which `DatasetType` already holds as `_storageClassName` and
`_parentStorageClassName`.

Every configuration lookup in the datastore goes through this path: templates
(`datastore/file_templates.py:267`), formatters
(`_formatter.py:1941, :1997`), composites (`datastore/composites.py:143`),
cache decisions (`datastore/cache_manager.py:728`), constraints
(`datastore/constraints.py:124`), and `DatasetRef._lookupNames` delegates to it
(`_dataset_ref.py:659`).

So *any* config lookup on a dataset type or ref currently imports science
payload code to obtain a string it already has.
This is the blocker for server-side template evaluation under a signed-upload
design, and fixing it is nearly free: construct the two `LookupKey`s from the
stored names instead of resolving.
It is also a put-path performance win, since it stops forcing `pytype` imports
on every write.

**Status: fixed on `tickets/DM-56109`** (`57e3e4c98`), along with the unreachable
`StorageClassFactory` `config` parameter (`43b0e39b6`).

The work is a sweep of registry and server paths, plus a regression guard.
The guard matters more than the sweep: a server test that installs a factory
raising on `getStorageClass`, so the invariant is enforced rather than
documented.
Half a week, worth doing first because it pins the invariant everything else
assumes.

Note that this invariant is *not* the same as the line along which the
datastore configuration has to be split for the entry-point work.
That split is by ownership (per-butler versus per-data-release); this one is by
execution.
They are orthogonal, and conflating them puts `templates` and `datastore.root`
on the wrong side.
See "Splitting the datastore configuration" below.

### 3. Entry-point configuration from external packages --- feasible, design partly open

`daf_butler` already has two working entry-point mechanisms of the right shape:

- `butler.obscore_factory`, keyed by **dimension universe namespace**
  (`registry/obscore/_records.py:359-371`): looks up a plugin by label, loads
  it, validates the returned type, raises if absent.
  Structurally this is the "configuration label -> entry point -> bespoke
  configuration" design, already in production.
- `cls.entryPoint` for CLI plugin commands (`cli/butler.py:339`), plus the
  older `DAF_BUTLER_PLUGINS` manifest it supersedes.

The mechanism is proven.
What remains open is the provider interface: what a provider returns, how
providers compose, and what happens on conflict.

**Versioning belongs in the key, not alongside it.**
DR1 and DR2 holding different definitions of the same storage class means the
lookup key is `(namespace, version)`, not `namespace`.
`DimensionUniverse` already does exactly this: cached on `(version, namespace)`
rather than being a true singleton
(`dimensions/_universe.py:98, 141, 194`).
That is the model to copy, and it is a better target than "remove the
singleton" --- what is wanted is a *keyed* registry, not no registry.

### 4. Removing the `StorageClassFactory` singleton --- feasible, but this is the hard one

Nine non-test construction sites in `daf_butler`:

```
direct_butler/_direct_butler.py:222   remote_butler/_remote_butler.py:162
registry/sql_registry.py:254          datastore/_datastore.py:420
_quantum_backed.py:340                queries/_query.py:816
_dataset_type.py:440, :463            datastore/stored_file_info.py:273
```

The first six are easy: all are objects that have, or could have, a
butler-scoped context to hold a factory reference.
The last three are value objects that hold only a storage class name and have
no butler reference.
`DatasetType` is the hard one; the other two are not, and the distinction
matters for scheduling.

**`StoredFileInfo` is easy.**
Exactly two callers of `.storageClass`, both in
`datastores/file_datastore/get.py` (lines 219, 490), and every construction
site is inside a datastore or a remote-butler get path
(`fileDatastore.py:748, :1113, :474, :554, :2248, :3198`;
`remote_butler/_get.py:111`; `remote_butler/_remote_file_transfer_source.py:91`).
`Datastore` already holds `self.storageClassFactory`
(`datastore/_datastore.py:420`).
Make that per-butler, delete the property, route the two callers through the
datastore.

**`InMemoryDatasetHandle` is nearly easy.**
No non-test construction site lacks a butler in scope
(`pipe_base/caching_limited_butler.py:129, :174`;
`pipe_base/pipeline_graph/_pipeline_graph.py:1888, :1915`).
It is a signature change rather than a design problem.
The one real subtlety is the fallback in `_getStorageClass`
(`pipe_base/_dataset_handle.py:254`): when no name is given it reverse-looks-up
by Python type via `findStorageClass` (`_storage_class.py:843`).
That has no name to key on, so it genuinely needs the correct *set* and cannot
fall back to a minimal default.

**Two complications outside `daf_butler`.**

- `pipe_base/tests/mocks/_storage_class.py:662-672` monkeypatches
  `StorageClassFactory.getStorageClass` at class level.
  The whole pipeline-mocking strategy used across Science Pipelines CI depends
  on there being exactly one global resolution point.
  De-singletonizing breaks it and it needs redesigning in step.
- `pipe_base/_dataset_handle.py:168, :254` calls `StorageClassFactory()` from a
  package that cannot see the butler config.

**Minor cleanup found on the way.**
`StorageClassFactory.__init__` documents and accepts a `config` parameter, but
the `Singleton` metaclass (`lsst.utils.classes.Singleton.__call__`) takes no
arguments, so `StorageClassFactory(config)` raises `TypeError`.
Verified by reproducing the metaclass standalone.
The parameter is unreachable and the docstring is wrong.

## How a `DatasetType` knows its label

This is the crux of the singleton work, and the answer is already in the code:
**`DatasetType` solves this exact problem for dimensions today.**
It should carry a handle to a resolved factory, not a label string.

### The existing precedent

`DatasetType.__init__` requires a `DimensionUniverse` and raises if it cannot
derive one from `dimensions` (`_dataset_type.py:213`).
The universe rides along inside `DimensionGroup`, which exposes it via
`__getnewargs__` (`dimensions/_group.py:248`).
`DimensionUniverse.__reduce__` pickles `(version, namespace)`
(`dimensions/_universe.py:539`) and unpickling *fails loudly* if that universe
is not registered in the receiving process
(`dimensions/_universe.py:528-530`).
`DatasetType.from_simple` already demands universe-or-registry
(`_dataset_type.py:786`).

That is a globally-keyed, immutable, cheap-to-pickle context handle carried by
a value object across pickling and server round-trips --- exactly what a
storage class label needs to be.

### It must be context, not identity

`DimensionGroup.__eq__` compares **only `names`**
(`dimensions/_group.py:345-347`).
Two `DimensionGroup`s from different universes with the same dimension names
compare *equal*.
The universe handle rides on the value object and is deliberately excluded from
equality.

The storage class label must be excluded the same way.
If it enters `DatasetType.__eq__`, butler-to-butler transfer breaks
immediately: every dataset type compares unequal across repositories even when
the definitions are byte-identical.
Identity stays `(name, dimensions, storage class name)` --- which is what
`_equal_ignoring_storage_class` already compares
(`_dataset_type.py:280-288`).

### Where the label enters

Five construction paths; four have an injection point that already exists for
the universe.

| Path | Context available |
|---|---|
| `SqlRegistry.getDatasetType` | registry holds the butler config, hence the label |
| Server deserialization | `from_simple(universe=, registry=)` already mandatory |
| Pipeline connections | `PipelineGraph.resolve(registry=, dimensions=)` (`pipe_base/pipeline_graph/_pipeline_graph.py:568`) injects the universe today |
| QG deserialization | graph already threads a universe through `pipeline_graph/io.py` |
| Hand-constructed in scripts and tests | **nothing** |

Only the last is a genuine gap, and the extraction work supplies the answer.
After storage classes move out, `daf_butler`'s legitimate residue is about 22
definitions: `int`, `StructuredDataDict`, `StructuredDataList`, the
Arrow/Parquet/Astropy family it ships formatters for (`ArrowTable`,
`ArrowAstropy`, `DataFrame`, `ArrowNumpy` and their schema variants),
`PropertySet`, `PropertyList`, `NumpyArray`, `Thumbnail`, `ButlerLogRecords`,
`Timespan`.
An unlabelled `DatasetType` resolving against that core set fails loudly for
anything science-specific, which is the right failure mode.

### One composite handle, not two parameters

Universe and storage class label are both repository-scoped, obtained from the
same place, and needed at the same sites.
Threading one composite context object --- universe plus storage class factory,
globally cached on the pair --- is one plumbing change rather than two.
`universe` survives as a property for back-compat;
`PipelineGraph.resolve(dimensions=...)` becomes `resolve(schema=...)`.

The cost is a wide but mechanical refactor of every `universe=` parameter
across `daf_butler`, `pipe_base` and `ctrl_mpexec`.
Doing it once is cheaper than adding a second independent parameter to the same
several hundred call sites.

### `conform_to` is the precedent for adoption at transfer

`DatasetType.conform_to(universe)` (`_dataset_type.py:346-397`) rebuilds a
dataset type against a target universe, raising `InconsistentUniverseError` on
namespace mismatch or incompatible dimensions.
`transfer_from` already calls it:
`refs_by_type[source_type.conform_to(self.dimensions)] = refs`
(`direct_butler/_direct_butler.py:2229`).
`DatasetRef.conform_to` (`_dataset_ref.py:831`) wraps it.

The storage-class version is the same operation on a second axis, at a line
that already exists.
If the composite `schema` handle is adopted, `conform_to(schema)` covers both
axes in one call.

## The transfer path has two guards that will fail open

This is the sharpest correctness finding, and it is not obvious from reading
the proposal.

**Guard 1: the name short-circuit.**
`DatasetType.is_compatible_with` returns `True` without resolving either
storage class when the names match (`_dataset_type.py:336-338`):

```python
# If the storage class names match then they are compatible.
if self._storageClassName == other._storageClassName:
    return True
```

This is sound **only because of the singleton**.
One global factory means one name can only ever denote one definition.
It is an unobvious correctness dependency on precisely the thing being removed.
Once definitions are per-label, same-name-different-definition is exactly the
case the scheme exists to catch, and this short-circuit passes it silently.
The rule after the change: resolve whenever the names differ **or** the labels
differ.

**Guard 2: the equality pre-check.**
`transfer_from` guards the compatibility check with
`if target_dataset_type != datasetType:`
(`direct_butler/_direct_butler.py:2314`).
Since the label must stay out of `__eq__` (above), cross-label dataset types
compare equal and the check is skipped before the short-circuit is even
reached.

Both guards fail open, from different directions.
"Keep the label out of equality" and "transfer still checks compatibility" only
coexist if the transfer path gains an explicit label-aware check.

**Ordering constraint.**
`conform_to` runs at line 2229; the compatibility checks at 2296 and 2316.
A label `conform_to` preserves the storage class *name* and swaps the label, so
if it runs first the later check compares target-label against target-label
with equal names and passes unconditionally.
Compatibility must be established **before** adoption, not after.

## The quantum graph: embed versus label

The serialized pipeline graph currently embeds the **entire dimension universe
configuration** (`pipe_base/pipeline_graph/io.py:674, :713`), not a key, and
rebuilds the universe from it on load (`io.py:743-745`).

Storage classes must use a label instead, because their definitions name Python
types that have to come from locally-installed software --- which is the whole
point of the extraction.
This is not an inconsistency: dimension configs are pure data, storage class
definitions are import instructions.
Worth stating explicitly so the obvious "why not embed those too?" question is
answered in advance.

Related: formatter *names* are stored per-dataset in the datastore records
(`StoredFileInfo.formatter`), so reads need no formatter configuration.
Writes do, which is why the QG must carry the label.

## A real bug found while tracing the QBB complaint

The technote says external users must fall back on `DAF_BUTLER_CONFIG_PATH`
for quantum-backed butler execution.
The mechanism is actually plumbed --- and then dropped.

`QuantumBackedButler.initialize` and `.from_predicted` both accept
`search_paths` and pass it to `ButlerConfig`
(`_quantum_backed.py:163, :226, :328`).
`--config-search-path` is a documented CLI option on both `pre-exec-init-qbb`
and `run-qbb` (`ctrl_mpexec/cli/opt/options.py:652`).

`pre_exec_init_qbb.py:97` forwards it correctly, through
`PredictedQuantumGraph.make_init_qbb` to `from_predicted(search_paths=...)`
(`pipe_base/quantum_graph/_predicted.py:1270`).

`run_qbb.py` does not.
`_QBBFactory` stores `config_search_path` (line 238) and carries it through
`__reduce__`/`_unpickle` for the worker processes (lines 262, 279), but
`__call__` --- the method that actually builds the butler --- never passes it
to `QuantumBackedButler.initialize` (lines 245-250).

Net effect: `--config-search-path` works for init-output writing and is
silently ignored for quantum execution, which is exactly the asymmetry that
forces users to the environment variable.
A small, independent fix worth making regardless of the larger redesign.

**Status: fixed on `ctrl_mpexec` `tickets/DM-56109`** (`d89d2a4`).

## Notes on framing

Most of these are now reflected in the technote; retained as the reasoning
behind them.

**The Pydantic work is only partly off the critical path.**
The singleton chain is: entry-point provisioning -> storage classes leave
`ButlerConfig` -> singleton becomes a keyed registry.
Modelling the *registry*, top-level and `repo_transfer_formats` sections
neither helps nor hinders that chain, and can start immediately.

The *datastore* section cannot.
It has to be split along the client/server line before it can be modelled, so
its schema depends on a decision that belongs to the entry-point work.
See "Splitting the datastore configuration" below.
The gating item is the split *decision* --- which keys move --- not the
provider implementation, so the dependency costs days rather than weeks.

**"A typo can not be detected" will only be partly fixed.**
See the two-phase validation constraint above.

**The `DatasetType` resolution-context question determines the size of the
work**, and is worth its own paragraph rather than a clause.

## Splitting the datastore configuration

The section splits on **ownership**, not on execution: is this setting a
property of *this butler*, or of *the data release*?
That is the axis that decides where a key lives.

Client-versus-server is a separate, orthogonal invariant --- the server must
never need to import science payload code --- and the two axes do not line up.
`datastore.root` and `templates` are read by client-side code but are per-butler;
`registry.db` is server-side and per-butler; `formatters` is client-side and
per-release.
An earlier draft of this document collapsed the two, which put `templates` on
the wrong side.

### Per-butler --- stays in `butler.yaml`

`datastore.root`, `templates`, `cls`, `records.table`, the `ChainedDatastore`
`datastores` list, and the registry `db` and `managers` keys.

`FileDatastore.setConfigRoot` already freezes most of these explicitly
(`datastores/fileDatastore.py:226-232`):

```python
toUpdate={"root": root},
toCopy=("cls", ("records", "table")),
```

and `ChainedDatastore.setConfigRoot` extends the same treatment to each child,
computing a per-child `newroot` (`datastores/chainedDatastore.py:167-180`).

`templates` joins them: it fixes the on-disk layout, so two clients writing to
one repository must agree on it regardless of what software they have
installed.

### Both root and templates are read server-side --- and it does not matter

`datastore.root` is read on the server in the `RemoteButler` get path:
`location.uri` derives from it and is pre-signed before the path is relativized
for the client (`server/handlers/_file_info.py:38-71`).

`templates` are *not* read server-side today, but only because writes through
the server do not exist: `RemoteButler.put` raises `NotImplementedError`
(`remote_butler/_remote_butler.py:244-256`) and there is no upload endpoint
among the fifteen external routes.
`templates.getTemplate(ref)` is called only on the write, ingest and transfer
paths (`datastores/fileDatastore.py:839, :1250, :3115`), all of which currently
run in the process that owns the repository.

Under a signed-upload design the server must compute the destination path
before it can sign a URL for it, so templates become server-read.
There is no `prepare_put_for_external_client` counterpart to
`prepare_get_for_external_client` (`datastores/fileDatastore.py:2304`) yet;
that is where it would live.

The useful observation is that **neither item changes side under the ownership
axis**.
Both are per-butler whether or not the server reads them, so both stay in
`butler.yaml` under either reading.
The two keys whose execution side was hardest to call are exactly the two the
ownership axis places without ambiguity --- which is the strongest argument
that it is the right axis to split on.

### Per-data-release --- moves to the provider

`storageClasses`, `formatters` (including the nested `write_recipes`),
`composites`, and `cached.cacheable`.

Each of these keys on, or resolves to, science payload code, and each changes
with the pipeline rather than with the repository:
`composites.disassembled` is keyed by storage class or dataset type name,
`cached.cacheable` by storage class name
(`configs/datastores/fileDatastore.yaml:38-40`), `formatters` by storage class
or dataset type name with a Python class as the value.

`write_recipes` needs no special treatment: it is keyed by formatter class name
and, by design, is write-side only.
Files are self-describing, so reading one never consults a recipe --- the
recipe selects compression at write time and nothing more.
It therefore travels with the formatters as ordinary per-release configuration.

### What external users already do confirms the cut

The canonical override config in the test suite
(`tests/config/basic/posixDatastore.yaml`) is:

```yaml
datastore:
  cls: ...
  root: <butlerRoot>/butler_test_repository
  templates: !include templates.yaml
  formatters: !include formatters.yaml
  composites: !include composites.yaml
```

with `storageClasses: !include storageClasses.yaml` alongside in `butler.yaml`.

The read path agrees at runtime.
The server ships back a location plus the recorded formatter *name*
(`server/handlers/_file_info.py:53`); the client resolves the formatter class
and the storage class definition locally and reads the file itself
(`remote_butler/_get.py:67-111`).
The server never needs a formatter class or a storage class definition ---
which is the client/server invariant, satisfied independently of where the
configuration lives.

### Consequence: two Pydantic models, not one

The Pydantic work does not shrink, it partitions --- a `butler.yaml` model for
the per-butler settings, and a provider-payload model for what the entry point
returns.
Both need schemas, and the provider payload arguably needs the stricter one,
since it is the part third parties will author.

### Good news: formatters have no singleton problem

`FormatterFactory` is already constructed per-`FileDatastore`
(`datastores/fileDatastore.py:323`), not as a singleton.
The formatter axis therefore has the configuration-*source* problem but not the
shared-global-state problem.
Only storage classes need the keyed-cache treatment.

## Developer overrides without an environment variable

Removing `DAF_BUTLER_CONFIG_PATH` raises the question of how a developer tweaks
a definition that now lives in a released package.
The intended answer --- a local EUPS setup of the release package, e.g.
`rubin-releases` --- works, and the mechanism is worth recording because it is
not obvious.

EUPS packages are not pip-installed into `site-packages`; the table file does
`envPrepend(PYTHONPATH, ${PRODUCT_DIR}/python)` (`ups/daf_butler.table`).
But the build does emit standard distribution metadata into that directory ---
`python/lsst_daf_butler.dist-info/` exists in a plain checkout --- and
`importlib.metadata` scans `sys.path` entries for `*.dist-info`.

Verified empirically: inserting a checkout's `python/` directory onto
`sys.path` is sufficient for `importlib.metadata.distributions()` to discover
`lsst-daf-butler` with its EUPS version string, with no install step.

So `setup -r .` on a local clone puts that clone's metadata ahead of the
stack's on `PYTHONPATH`, and its entry points are discoverable.
The developer override story needs no new machinery.

### One risk to test before relying on it

The same experiment surfaced **two** `lsst-daf-butler` distributions at
different versions simultaneously --- the locally inserted one and the ambient
stack one.
EUPS local setups shadow by path precedence but do not remove the shadowed
distribution, so `entry_points(group=...)` will return entries from both.

The existing lookup idiom collapses that with a dict comprehension:

```python
plugins = {p.name: p for p in entry_points(group="butler.obscore_factory")}
```

(`registry/obscore/_records.py:359`).
A later entry overwrites an earlier one, so if `entry_points` yields in
`sys.path` order the winner is the *lowest*-priority distribution --- the
opposite of what a developer override needs.

I have not confirmed the iteration order for duplicate distributions, so this
is a risk to test rather than a established defect.
It should be settled with an explicit test (local setup shadowing a stack
package, asserting the local provider wins) before the provider system depends
on it.

### Precedent: the CLI loader kept both mechanisms

`LoaderCLI` supports a `pluginEnvVar` manifest list *and* an `entryPoint`
group, merged together (`cli/butler.py:256-264, 338-353`), with
`ButlerCLI.pluginEnvVar = "DAF_BUTLER_PLUGINS"` and
`entryPoint = "butler.cli"` both live (`cli/butler.py:406-407`).

Worth deciding deliberately whether the configuration provider follows suit or
commits to entry points alone.
Keeping an env-var escape hatch would re-introduce, under a new name, the
mechanism this work exists to remove; but the CLI loader is evidence that the
pressure to add one is real.

## Effort

4 weeks is achievable for the Pydantic schema and translator in isolation --- a
well-bounded transformation with a working precedent in the same repository,
genuinely suited to LLM assistance.
It is not achievable for the integrated package.

Assuming one experienced developer with LLM assistance:

| Item | Estimate |
|---|---|
| Phase 0 quick wins (run_qbb fix, `_lookupNames` de-resolution, server sweep + guard, dead parameter) | 3 days |
| Datastore per-butler / per-release split decision | 3 days |
| Entry-point provider design + prototype | 1--1.5 weeks |
| Pydantic ButlerConfigV2 + V1 translator + version key | 2 weeks |
| Move storage classes and formatters out of `daf_butler` | 1--2 weeks engineering |
| Replace singleton with keyed registry, incl. the `schema` handle refactor | 2--3 weeks |
| Downstream fixes (`pipe_base` mocks, `InMemoryDatasetHandle`, `ctrl_mpexec`) | 1 week |
| Migration tooling, deprecation, docs | 1 week |
| **Total** | **7--10 weeks** |

Not included: coordination latency with the packages whose types appear in
`storageClasses.yaml`.
That is at least a dozen packages, several outside DM's direct control, and it
is likely to dominate calendar time even though it is small in engineering
terms.

The technote's revised 8-week figure sits inside this range.

## Recommendation

Proceed, with the work split and re-sequenced.

1. **Do Phase 0 now.**
   The `run_qbb` search-path fix, making `DatasetType._lookupNames` work from
   stored names instead of resolved storage classes, the `.storageClass_name`
   sweep with a server guard, and deleting the unreachable `config` parameter
   are days of work, are independently valuable, and pin the client/server
   invariant that everything else assumes.
   The `_lookupNames` change is also a prerequisite for ever evaluating file
   templates server-side.

2. **Settle the per-butler / per-release split of the datastore section first
   --- it is the cheap gate.**
   Deciding which keys move is days of work, and it unblocks both the
   `DatastoreConfig` Pydantic model and the provider payload schema.
   Do not start either model before it.
   Split on ownership, not on who executes the code; the client/server
   invariant is separate and is already satisfied.

3. **Then settle the provider interface.**
   What a provider returns, how providers compose, and what happens on
   conflict.
   The `DatasetType` resolution question is now answered by analogy with
   dimensions; this one is not.

4. **Adopt the composite `schema` handle rather than a second parameter.**
   Universe and storage class label are both repository-scoped and always
   travel together; one plumbing change is cheaper than two across the same
   call sites.

5. **Treat the transfer path as a correctness fix, not a port.**
   Both existing guards (the `!=` pre-check and the name short-circuit in
   `is_compatible_with`) fail open once labels exist, and the compatibility
   check must precede `conform_to` adoption.
   This deserves dedicated tests: two labels, same storage class name,
   incompatible definitions.

6. **Run the registry and top-level Pydantic work in parallel with the provider design.**
   Those sections are genuinely independent and can start immediately with
   known scope.
   The datastore model is the exception and waits on item 2.

7. **Target a keyed registry, not the absence of one.**
   Follow `DimensionUniverse`'s `(version, namespace)` caching; it solves the
   multi-repository problem without paying for per-instance resolution
   everywhere.
