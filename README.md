# Pactavium

> **Documentation preview:** This repository currently contains the project overview. Source code, supporting documentation, and release artifacts have not been published here yet. The examples below describe the existing implementation and will become runnable once those files are available.

**Versioned contracts. Verified integrations. Visible usage.**

Pactavium helps independently developed services agree on the data they exchange, verify that agreement before deployment, and make contract usage visible during operation.

It combines a language-neutral contract foundation, offline verification tools, an optional Python model library, a read-only contract catalog, consumer qualification tools, and a private observation service called **Observatorium**.

Validation can run locally without a catalog connection. Observation is explicit and optional: importing a model does not start a network client, send telemetry, or create a background worker.

<img width="1920" height="2079" alt="image" src="https://github.com/user-attachments/assets/e124f9b3-e666-46d9-8442-7714390e97cf" />


> **Status: pre-release, prepared for controlled deployment testing.**
> The current implementation has passed development-environment qualification. Deployment on the target platform, operational acceptance, and an independent real-world consumer pilot remain pending. There is no stable 1.0 release yet.

## Why Pactavium exists

Two services can exchange valid JSON and still disagree about its meaning. They may interpret a missing field differently, use different error formats, accept incompatible identifiers, or rely on different versions of the same contract.

Pactavium makes those agreements explicit and testable:

1. **Define the agreement:** describe structure, meaning, and examples of accepted and rejected data.
2. **Pin the agreement:** identify an exact release by version and manifest digest.
3. **Verify the integration:** test actual consumer data against the contract and its supported model library.
4. **Evaluate a change:** check sender and receiver compatibility separately.
5. **Observe adoption:** distinguish reported runtime use, CI results, catalog requests, and independent probes.

For example, a scheduling service and a worker can share definitions for operation IDs, deadlines, errors, and readiness. Before updating either service, its integration checks can detect that a previously accepted value would be rejected. During operation, Observatorium can show which contract a configured source reports using and whether that evidence is fresh, incomplete, or missing.

The application still owns its business logic, authorization decisions, delivery guarantees, and domain-specific data.

## Current components

Versions are independent. The Observatorium version is not the Core contract version.

| Component | Current baseline | Purpose |
| --- | --- | --- |
| Core contract | `1.0.0-rc.1` | Shared semantics, JSON Schemas, and conformance cases |
| CLI and Catalog | Original Core RC artifacts | Offline verification and a read-only HTTP interface |
| Python models | `1.0.0rc1` | Optional strict Pydantic v2 projection of the Core contract |
| Consumer integration kit | `1.0.0-rc.1` | Reference material, examples, artifact pins, and extension workflow |
| Consumer qualification | `0.2.0-candidate.2` | Repeatable consumer checks and directed contract comparisons |
| Observatorium | `0.1.0-candidate.10` | Private observation API, dashboard, adapters, and bounded storage |

This README describes the implementation reviewed on **17 October 2026**, with documentation prepared on **18 October 2026**. Public release downloads and registry distribution are separate from implementation readiness.

Pactavium is the public-facing project name. Existing binaries and immutable artifacts still use technical names such as `pactumctl`, `pactum-catalog`, and `pactum-core`. Those names are retained in the examples below; renaming the project does not alter previously verified bytes or contract identifiers.

## Contract foundation

The Core contract provides reusable technical building blocks:

| Area | Included concepts |
| --- | --- |
| Identity and references | Projects, services, instances, resources, and technical actors |
| Release identity | Digests, release references, artifact descriptors, and release manifests |
| Request context | Request, correlation, causation, operation, message, event, trace, and span identifiers |
| Time | Instants, durations, deadlines, expiry, and explicit clock-skew rules |
| Errors | Error envelopes, categories, field violations, and retry hints |
| Operations | Operation state, request fingerprints, reconciliation horizons, and cancellation |
| Health | Three-valued conditions and separate liveness/readiness concepts |
| Evidence | Evidence and attestation references, assessment states, and conformance/compatibility metadata |

Three sources jointly define a contract: semantic documentation, **JSON Schema Draft 2020-12**, and conformance cases. Language-specific models are derived consumers of that contract, not its source of truth.

Parsing and validation enforce explicit boundaries, including duplicate-key rejection, finite numbers, bounded depth and size, and distinctions between absent fields and `null`. Closed objects reject unexpected fields. Extensions use explicit namespaces and capability rules rather than silently accepting arbitrary additions.

All schema references resolve from the verified local bundle. Validation does not fetch remote schemas. The implementation uses a deliberately restricted schema profile; it is not a universal validator for every JSON Schema feature or an automatic contract generator for arbitrary APIs.

## Offline verification and release integrity

The CLI supports:

- Verifying and inspecting a contract release.
- Validating JSON instances against a pinned release and schema.
- Running the release's bound conformance cases.
- Comparing two independently pinned releases.
- Importing and verifying a local content-addressed catalog store.

Verification checks archive structure, safe paths, manifest identity, file membership, sizes, and content digests before interpreting contract payloads. A version string or a mutable alias alone is insufficient to identify the bytes a consumer uses.

A digest proves a match to expected bytes. It does not independently prove who published them: expected pins must come from a trusted distribution channel.

### Local verification example

These examples require a checkout containing the original RC artifacts, a supported Linux x86-64 environment, and trusted artifact pins. They do not download or install anything. Verify the CLI binary itself against the release lock before executing it.

From the repository root:

```sh
artifacts/bin/pactumctl release verify \
  --input artifacts/pactum-core-1.0.0-rc.1.zip \
  --expect sha256:17141d8ea361c028a92de35adb00af50158e0193a940997659dcb81fcd659362
```

For an existing `project.json` containing:

```json
{
  "authority": "https://example.org/",
  "id": "example-project"
}
```

Validate the project reference with:

```sh
artifacts/bin/pactumctl instance validate \
  --release artifacts/pactum-core-1.0.0-rc.1.zip \
  --expect sha256:17141d8ea361c028a92de35adb00af50158e0193a940997659dcb81fcd659362 \
  --schema 'resources/references.schema.json#/$defs/projectRef' \
  --input project.json
```

Successful instance validation returns an `accept` outcome. The schema fragment is important: this checks a project reference, not an arbitrary application document.

The integration kit supplies the full command reference and executable examples. Its standalone `VERIFY.py` checks the extracted kit using only Python's standard library. The six original RC artifacts are supplied separately; the kit is not a replacement for them.

## Optional Python models

The Python library provides strict Pydantic v2 models for the supported Core release. Its support metadata binds the supported release identity explicitly.

Model validation performs no file, network, or process I/O. Instrumentation belongs in a consumer-owned adapter and is never activated by importing a model.

The current qualification uses Pydantic `2.12.5`. The deployment materials include pinned dependencies for the documented Python profiles; this is not a claim of compatibility with every Python version or platform.

## Read-only Catalog

The Catalog serves a verified local store through eight routes:

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/v1/releases` | List releases |
| `POST` | `/api/v1/resolve` | Resolve a selector to a pinned release reference |
| `GET` | `/api/v1/releases/sha256/{hex}/manifest` | Retrieve exact manifest bytes |
| `GET` | `/api/v1/releases/sha256/{hex}/artifacts/{path...}` | Retrieve a manifest-bound artifact |
| `POST` | `/api/v1/validate` | Validate raw JSON against a specified release and schema |
| `POST` | `/api/v1/compatibility` | Compare two pinned releases |
| `GET` | `/health/live` | Check process liveness |
| `GET` | `/health/ready` | Check service readiness |

The POST routes perform queries or validation; they do not publish contracts. The versioned OpenAPI document defines their complete request and response formats.

An offline import verifies a complete snapshot before atomically selecting it. The running Catalog has no importer privileges and receives read-only access to its store. Failed reloads retain the last valid in-memory snapshot; a detected integrity failure in an actively served artifact makes the service unavailable.

Contract authoring and publication remain in version control and the release workflow. The Catalog is a derived view, not an independent authoring authority.

## Consumer qualification and compatibility

The qualification runner combines artifact verification and consumer-specific checks in one invocation. Its configuration binds a consumer identity, repository, commit, build label, contract digest, and **1–256 named positive and negative data cases**.

It checks the six original artifact pins before execution, runs the **146 bound CLI conformance cases**, tests actual consumer values through the validator and pinned Python models, checks SDK metadata, and verifies I/O-free validation after imports.

Results are written as schema-validated JSON and a readable Markdown report. The supplied CI template uses an already provisioned runner; it does not install infrastructure or modify consumer repositories.

| Result | Meaning |
| --- | --- |
| `passed` | All required checks completed successfully |
| `failed` | A concrete failing check was found |
| `blocked` / `not_checked` | Required inputs, identity checks, or delivery evidence are unavailable |

Offline checks can run without observation delivery. In the current runner, omitting delivery configuration leaves the overall qualification `blocked`, with the completed offline checks still reported individually. A complete pass also requires the configured checkout identity check and acknowledged mTLS delivery.

The commit check verifies the checkout's HEAD. A build label remains caller-supplied; it is not independent proof of every build artifact or a clean working tree.

### Directed contract changes

Compatibility checks distinguish:

- **Sender compatibility:** will values produced under the current contract still be accepted by the new one?
- **Receiver compatibility:** can the current consumer model accept the tested values allowed by the new contract?

The original checker separately reports source, wire, and semantic compatibility. Results may be `compatible`, `incompatible`, or `unknown`. Passing selected examples does not override an incompatible or unknown contract-level result.

The current consumer comparison is bound to the existing RC SDK. It does not claim support for arbitrary baseline SDKs, all possible input values, or the behavior of an entire application. A passing comparison is evidence for the tested scope, not a deployment authorization.

## Observatorium

Observatorium is a separate private Python service with a local-asset web interface. It keeps four kinds of evidence distinct:

| Evidence | What it can establish | Important boundary |
| --- | --- | --- |
| Catalog capture | Requests observed through the configured catalog proxy | Direct Catalog access bypasses capture; catalog traffic is partial coverage |
| Runtime reports | What a configured application adapter reports validating | Authenticated self-reporting, not independent proof of application behavior |
| CI reports | Checks associated with a reported commit, run, and contract | A successful CI run is not a runtime heartbeat |
| Probe reports | Explicit liveness, readiness, and store-verification observations | Independence requires separately administered probe credentials |

The dashboard provides project/environment filters, contract-pin and freshness information, detail views, JSON exports, storage occupancy, retention gaps, delivery failures, and technical service status. Expected sources remain visible even if they have never reported.

Freshness is based on observation time and configured cadence. Receiving an old report does not make it fresh. Missing or expired evidence is not presented as successful activity, and a disconnected browser does not retain an indefinitely green connection state.

### Access control and protocol

The private service requires **TLS 1.3 and mutual TLS**. An operator-controlled mapping binds each client certificate to a role and, where applicable, a fixed source. Report bodies and HTTP headers cannot choose another source identity, project, or environment.

- Readers can query and export, but cannot submit reports.
- Submitters can report only for their assigned source and cannot read other evidence.
- Catalog identities use the permitted proxy routes.

Core observation routes are `POST /observations/v1/reports` and the reader routes `GET /observations/v1/summary`, `/observations/v1/export`, and `/observations/v1/health`.

Reports contain bounded outcome metadata. The provided capture and validation integrations do not collect application payloads or request headers. Operators still control the sensitivity of configured identities and CI metadata.

Matching duplicate IDs and content receive a duplicate acknowledgement. Conflicting content, invalid sequence movement, retired epochs, and overlapping intervals are rejected. Acknowledgements follow the durable commit; connection failure alone does not establish whether a commit occurred.

### Explicit adapters and offline delivery

The validation-path adapter uses a bounded RAM queue and background persistence. A separate persistent spool supports ordered delivery, acknowledgement tracking, retry, and backoff. Delivery workers are explicitly started; they are not a side effect of model validation.

Instrumentation errors preserve the original validation result or exception in the tested wrapper paths. Queue depth, dropped observations, write errors, and retry state remain visible. A missing acknowledgement keeps the same report available for an identical retry.

RAM buffering can lose unflushed observations after abrupt termination. Disk failure can lose local spools. Arbitrary adapter code still executes inside the consumer process and is not sandboxed by the wrapper.

For higher event rates, a caller can explicitly report aggregate intervals of up to **1,000 outcomes** per report. This preserves reported totals for that interval, not individual-call history. There is no automatic sampling or claim of complete observation.

## Storage, retention, and recovery

Observatorium uses a bounded local JSON snapshot with exclusive locking, atomic replacement, and file/directory synchronization. It requires no database server or message broker.

| Limit or policy | Current behavior |
| --- | --- |
| Stored detail reports | At most 10,000 |
| State file | At most 32 MiB; configurable budget from 64 KiB |
| Detail age | Configurable from 60 seconds to 30 days |
| Capacity policy | `reject` or `prune-oldest`; existing configurations default to `reject` |
| Maintenance | At startup, during writes, and approximately every 60 seconds |
| Observation request body | At most 16 KiB |
| Configured sources / certificate mappings | At most 256 / 512 |
| Concurrent TLS connections | Default 64; configurable from 8 to 64 |
| Catalog persistent capture buffer | At most 128 reports / 2 MiB |

Age-based cleanup also applies with `reject`. Replay fences survive detail cleanup. Removed details remain disclosed as gaps; summary totals and exports use retained reports only. Loss counters are not reconstructed lifetime usage totals.

A single ordered writer groups up to 64 reports before an atomic commit. The storage commit remains serial, and snapshot work grows with the retained state. This is a bounded observation store, not an unlimited telemetry database. Temporary snapshots, migration originals, spools, installation generations, and backups require disk space beyond the 32 MiB state-file limit.

State-v1 migration preserves the original bytes before switching to State-v2. Older v1-only programs must not open the migrated store.

**Cold restore requires a trusted, current replay floor:** evidence of the latest acknowledged source positions. An old backup alone cannot establish what was acknowledged afterwards. Without that current evidence, restoring a non-empty backup into an empty store remains blocked. A replay floor prevents replay acceptance; it cannot recreate missing observations.

Backups must coordinate service state, capture state, configuration, release identity, and separate sender spools. The observer's backup command alone is not a complete operational backup. There is currently no approved older operational fallback for the first deployment: the fallback is to stop, preserve data, and repair.

## Deployment and platform scope

The prepared service deployment uses separate unprivileged identities for Catalog and Observatorium, systemd limits, a loopback-only Catalog, and an explicitly configured private observation endpoint. Certificate provisioning and network policy remain operator responsibilities.

The generation-based installer verifies package identity and configuration, requires stopped services for a package switch, and selects a complete generation atomically. An exact repeated preparation is a no-op. It does not create users, provision infrastructure, issue certificates, install operating-system packages, or automatically start services.

Current prepared binaries target **Linux x86-64**. Observatorium has been tested on **Ubuntu 24.04 / Python 3.12**; the **Ubuntu 26.04 / Python 3.14** path is prepared but still requires native qualification. Local state requires Linux filesystem locking, atomic rename, and file/directory synchronization; NFS and SMB state directories are outside the deployment contract.

The existing installer is bound to a specific target identity and platform contract. It is not yet a general-purpose installer for arbitrary hosts. Public package-registry availability, an unrestricted one-command installation, and support for other operating systems are not claimed.

For the current candidate, [deployment readiness](docs/deployment/READINESS.md), [operations](docs/deployment/OPERATIONS.md), and [platform requirements](docs/deployment/PLATFORM.md) take precedence over older deployment examples. Use the units under `deploy/readiness/` for this candidate.

## Testing and known limits

The latest recorded qualification includes:

- **228 tests and 36 subtests** in the final application/consumer qualification suite.
- Separate Core and original consumer-kit regression checks.
- **20 browser test groups** against an actual local mTLS backend, including filters, focus, mobile layout, exports, outages, retention, and malicious text handling.
- Accessibility checks with 25 passes and no reported violations or incomplete checks in the tested views; this is not a complete accessibility certification.
- A retention fixture with **30,000 accepted reports**, **10,000 retained details**, and **20,000 removed details**, followed by restart, restore, and replay checks.
- Process-interruption and constrained out-of-memory recovery tests on copied stores.
- Three byte-identical builds of each current candidate archive.

These scopes overlap and should not be added into a single test-total claim. Results apply to the recorded environments and versions.

The latest audit corrected adapter exception handling, invalid timestamps, shutdown bounds, HTTP header limits, reflected HTTP errors, per-source connection admission, batch delivery deadlines, and a qualification check that had modified its own evidence.

Remaining limits include:

- Native installation, reboot, backup/restore, effective service isolation, and load acceptance on the target platform are pending.
- Durable-write p95 measurements remain variable and require target-storage measurement.
- Network loss, bounded buffers, retention, and abrupt termination can create observation gaps.
- Authenticated sources can submit incorrect claims about their own behavior.
- In-process adapters, privileged operator compromise, arbitrary network overload, and power-loss behavior are not covered by a universal safety guarantee.
- Socket deadlines do not forcibly interrupt every DNS resolver or local filesystem stall.
- An independent consumer pilot and stable-release acceptance remain outstanding.

See the [security and regression audit](docs/deployment/ADAPTER_SECURITY_AUDIT_2026-10-17.md) and [long-term qualification](docs/deployment/LONG_TERM_QUALIFICATION_2026-10-17.md) for methods, results, and their boundaries.

## Repository guide

| Path | Contents |
| --- | --- |
| `contracts/core/` | Core semantics, schemas, and conformance material |
| `contracts/catalog/` | Catalog API contract |
| `cmd/` | Go CLI and Catalog entry points |
| `internal/` | Go verification, storage, compatibility, and service implementation |
| `python/` | Optional Python model package |
| `consumer/` | Integration kit, references, examples, and tests |
| `consumer/qualification/` | Qualification runner, contract comparison, result schema, and CI template |
| `observatorium/` | Observation service, adapters, interface, tests, and candidate packages |
| `deploy/readiness/` | Current service units, dependency pins, and deployment support |
| `docs/deployment/` | Platform, operations, acceptance, security, and qualification documentation |
| `artifacts/` | Original pinned RC artifacts in the development baseline |

Start with the [consumer guide](consumer/README.md) for integration and the [qualification guide](consumer/qualification/README.md) for consumer-specific checks. Historical documents retain their original version scope; the current deployment documents govern current operations.

## Contributing

Useful contributions include reproducible integration failures, realistic positive and negative consumer cases, clearer onboarding, and independently measured operational behavior.

Contract changes must keep semantics, schemas, and conformance cases consistent. Preserve published artifact identities and add new versions instead of silently changing pinned releases. Include the relevant component version, contract digest, environment, and minimal reproduction when reporting a bug; exclude credentials and private application data.

## License

The current source baseline declares **Apache License 2.0**. See [LICENSE](LICENSE). Bundled dependencies retain their respective licenses.
