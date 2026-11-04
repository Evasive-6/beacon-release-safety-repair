# Beacon release review

## Pipeline audit
Gap / risk / repaired dependency:
- **Gap**: The `production` job in `.github/workflows/release.yml` did not define a dependency on the `validate` job.
- **Risk**: By default, GitHub Actions executes jobs concurrently. Without `needs: validate`, the production review job would run concurrently with or even after a failed test suite, potentially reviewing or promoting defective builds.
- **Repaired dependency**: Added `needs: validate` to the `production` job. The production stage is now strictly blocked if the `validate` job fails or is cancelled.

Production reviewer control to configure (not demonstrated locally):
- Declaring `environment: production` in workflow YAML assigns environment metadata only; it does NOT automatically enforce approvals.
- To enforce an approval gate, repository administrators must navigate to **GitHub Settings > Environments > production** and enable **Deployment protection rules > Required reviewers**, specifying authorized individuals or teams who must manually approve the deployment before GitHub Actions unpauses the job.

Secret handling and scope:
- Production secrets (such as `REGISTRY_TOKEN`) must be scoped strictly at the environment level (`production` environment secrets) or with least-privilege repository permissions, preventing access from untrusted branches or pre-validation jobs.
- Secrets must be passed explicitly into specific job steps via environment variables (`env: REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}`) and must never be echoed, printed in build logs, or exposed to unvetted third-party actions.

## Traceability
Why PATCH is appropriate:
- Under Semantic Versioning (SemVer 2.0.0), incrementing the PATCH component (`v2.3.0` -> `v2.3.1`) is designated for backward-compatible bug fixes. The candidate includes zero breaking API changes, no feature additions, and requires no database schema migrations.

Candidate image and annotated Git tag:
- Candidate image: `beacon-service:v2.3.1` (specified in `k8s/deployment.json`).
- Annotated Git tag: `v2.3.1` created via `git tag -a v2.3.1 -m "Release candidate v2.3.1"`.

Record tag -> code commit in PDF using git rev-parse commands:
- Resolving tag commit: `git rev-parse v2.3.1^{commit}`
- Resolving HEAD commit: `git rev-parse HEAD`
- Both commands yield the identical commit SHA on branch `fix/ca3-beacon`, demonstrating exact resolution to the final code commit.

Image-tag mutability limitation:
- Version tags (such as `:v2.3.1` or `:latest`) are mutable references in container registries that can be re-pushed or overwritten without trace.
- A cryptographic digest (`beacon-service@sha256:...`) provides an immutable, content-addressable identity. A Git tag confirms commit history but does not prove container image provenance without container image signing (e.g., Cosign/Sigstore).

## Incident recovery
Evidence and rollback vs forward decision:
- **Supplied fixture signals (training fixture, not observed live runtime telemetry)**: At 10:00 candidate `v2.3.1` was promoted while previous image `beacon-service:v2.3.0` was healthy; at 10:03, 35% of requests fail across all users with errors rising; two of three readiness checks fail; root cause is unknown; rollback compatibility confirmed with no database/schema migration.
- **Decision**: Immediate **Rollback** to known-good image `beacon-service:v2.3.0`.
- **Reason**: High widespread error rate (35%) and failing readiness probes indicate a severe outage with an unidentified cause. Forward-fixing under an active outage risks compounding defects, while rolling back to the confirmed compatible and healthy `v2.3.0` restores immediate service stability.

Known-good target:
- `beacon-service:v2.3.0` (as defined in `release-contract.json`).

Proposed kubectl command (not executed):
- `kubectl set image deployment/beacon beacon=beacon-service:v2.3.0`

Recovery checks after action:
1. Rollout status: `kubectl rollout status deployment/beacon` (ensure new replica set completes deployment).
2. Pod health: `kubectl get pods -l app=beacon` (verify pods transition to `Running` and pass readiness probes).
3. Telemetry/Metrics: Confirm request error rate returns to baseline (<0.1%) and latency normalizes.
4. Health probe: Verify health check endpoint returns 200 OK (`{"status": "ok", "version": "2.3.0"}`).

## Release note and decision
Change scope:
- Repaired pipeline ordering in `.github/workflows/release.yml` with `needs: validate`.
- Pinned traceable candidate image `beacon-service:v2.3.1` in `k8s/deployment.json` (replacing `:latest`).
- Corrected rollback target image to `beacon-service:v2.3.0` in `rollback.json`.

Local validation proof:
- `python -m unittest -v`: 1 test passed (`OK`).
- `python check_release.py all`: `pipeline: PASS`, `image: PASS`, `rollback: PASS`.

Open risk / owner / mitigation:
- **Open risk**: Container image reference uses a mutable tag rather than an immutable digest, and live GitHub Environment Required Reviewers are not yet configured on the remote repository.
- **Owner**: DevOps Lead / Release Engineering Team.
- **Mitigation**: Transition deployment manifest to pin image SHA256 digest (`beacon-service@sha256:...`) with Cosign signature validation, and configure Required Reviewers in GitHub Environment settings for `production`.

Current production decision and blocker:
- **Current production decision**: **NO-GO**.
- **Blocker**: Contract status is explicitly **Approval NOT REQUESTED** and candidate is **NOT DEPLOYED**. Local static contract check passing verifies file syntax only and does not validate image existence, live cluster readiness, or management approval.

Terraform vs Kubernetes responsibilities:
- Terraform is an Infrastructure-as-Code tool responsible for provisioning and lifecycle management of cloud and physical infrastructure (such as networks, compute instances, storage, and Kubernetes clusters), whereas Kubernetes is an orchestrator responsible for managing, scheduling, and running containerized application workloads and services on top of that provisioned infrastructure.
