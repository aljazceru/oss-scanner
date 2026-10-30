# Enclava CAP

CAP is a Rust control plane for stateful confidential containers on Kubernetes,
using Kata Containers and AMD SEV-SNP. It includes an HTTP API, user CLI,
Kubernetes resource engine, in-TEE init and readiness processes, and native and
WebAssembly attestation verifiers. It handles deployment authorization, image
identity, workload ownership, encrypted persistent storage, and secret release.

## Security boundaries

Focus on the code in `crates/`, including shared cryptography and validation,
API handlers and database transitions, CLI verification, Kubernetes resource
construction, the appraiser, and in-guest initialization. Release/install scripts
and the browser verifier are also relevant attack surfaces.

Treat unauthenticated requests, authenticated tenants acting against other
tenants, deployment descriptors, image references and registry responses,
attestation evidence, certificates, signed metadata, and persisted state read
back across a trust boundary as adversarial inputs. Check signature verification,
identity and freshness binding, replay resistance, authorization, SSRF, parser
limits, filesystem handling, and concurrency around ownership and deployment.

Tenant authentication alone must not authorize another tenant's resources or
secrets. Workload images and platform artifacts must remain bound to their
intended digests and signer identities. Untrusted or incorrectly bound evidence
must not authorize secret release, storage unlock, or workload readiness.
Host-controlled storage and networking must not bypass guest verification.
Distinguish host denial of service from a confidentiality or integrity violation.

New workloads use password-only unlock; automatic owner-seed retrieval is
intentionally disabled. Assess legacy-mode migration against preservation of
existing encrypted data, not against a requirement to re-enable auto-unlock.
Deployment retries, cleanup, and recovery must not cross tenant boundaries or
silently destroy persistent state.

Development-only bypasses are not production vulnerabilities merely because they
exist. Check whether they remain reachable in release or `prod-strict` builds.
See `README.md`, `DEPLOYMENT.md`, and `crates/enclava-init/README.md` for deployment
assumptions. Bugs in CAP's integration with Kubernetes, Trustee/KBS, Sigstore, or
SEV-SNP are in scope; independent vulnerabilities in those external projects
should not be attributed to CAP.

## Offline build and tests

The image contains the source at `/src`, the Rust toolchain and fetched Cargo
dependencies, native build dependencies, compiled workspace binaries and test
targets, and a disposable PostgreSQL cluster. It is a development/test image,
not the production guest image. The `test` database role is a local fixture
superuser and must never be used against a deployed database.

Inside the scanner's offline shell, run:

```sh
pg_ctlcluster 15 main start
cargo test --locked --offline --workspace -- --test-threads=1
cargo test --locked --offline -p enclava-init --lib --bins --features prod-strict -- --test-threads=1
cargo test --locked --offline -p enclava-wait-exec --bins --features prod-strict -- --test-threads=1
```

Keep tests serial: fixtures share database fences and process-global environment
or init paths. Do not use workspace-wide `--all-features`; some combinations
intentionally conflict. Binaries are under `target/debug/`, including `enclava`
and `enclava-appraiser`. The native verifier is included in the workspace tests;
the browser/WASM build is not prebuilt in this image.

Two API integration tests currently require public DNS for `ghcr.io`:
`existing_app_storage_resize_is_refused_before_persistence` and
`rejected_signed_deployments_leave_existing_and_candidate_container_rows_unchanged`.
Without network access they fail at the registry destination guard before
reaching their expected validation errors. Do not disable the guard or enable
external networking to make these tests pass.

Real confidential-workload deployment, attestation, and encrypted-volume E2E
require hardware and platform services that this offline image does not provide.
Do not contact live deployments or use real credentials. Use isolated fixtures
for reproductions; state clearly when an exploit needs a hardware-backed check.

## Reports

Describe the attacker capabilities, affected trust boundary, required
configuration/build features, and concrete confidentiality, integrity, or
availability impact. Prefer a minimal offline reproducer and a focused regression
test or patch. Deduplicate findings with the same root cause. Distinguish a
verified exploit from a suspected path and identify any unavailable prerequisites.
