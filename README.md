# ACMECORP Native LLM Inference Platform on BARE METAL #RKE2 demonstrator skill

## Download and start

[Download the complete Linux Bash project ZIP](acmecorp-llm-demonstrator.zip), extract it,
and open the extracted README.md before running the quickstart commands below.
This repository publishes the overview and ZIP; source files, runbooks and validation
records are inside the archive. Document links below download that complete archive;
the extracted README retains direct links to each included document.

## Introduction

### What this project does

This project provides a repeatable build kit for running a large language model (LLM) on
physical Supermicro servers with NVIDIA GPUs. It prepares a private inference service: an
application sends a prompt, and the locally hosted model generates a response. The proposed
lab configuration uses Qwen3.6-27B served by SGLang on a single GPU worker, with a separate
workload control-plane server and an existing independent Rancher management cluster.

The deliverable is an installable AI-assistant skill and a self-contained operator kit with
configuration templates, Kubernetes manifest generators, model-integrity checks, smoke tests,
architecture records and operating instructions. Operators supply verified site values; the
kit then produces files for review and controlled deployment. It does not provision physical
machines simply by being downloaded or built.

### Why it exists

A bare-metal inference service depends on more than a model and a GPU. Host images, firmware,
drivers, Kubernetes, model storage, access controls and recovery procedures must work together.
This project makes those dependencies explicit so platform engineers, security reviewers and
operations staff can evaluate the same design and reproduce the same build.

Its purpose is to reduce manual setup errors, configuration drift and uncertain ownership.
Version pins, model checksums, private networking, automated checks and documented handoff
steps make changes easier to inspect and troubleshoot. The current deliverable is a
single-tenant demonstrator with synthetic prompts; production readiness requires additional
hardware, security, availability and recovery evidence.

### Three sample use cases

These examples describe workloads to evaluate after lab deployment; the kit supplies the
inference platform, not finished business applications.

| Use case | Example input and outcome | Evaluation boundary |
|---|---|---|
| Operations runbook assistant | Supply a short, sanitized troubleshooting excerpt and request a checklist for a fictional incident. | Human engineer verifies each step; no automatic infrastructure changes or bundled RAG index. |
| Support-ticket drafting | Classify fabricated payment-support tickets and draft replies using approved sample policy text. | Synthetic data only; no customer/card data, transaction execution or autonomous decisions. |
| Model upgrade rehearsal | Run the same synthetic prompt set against an approved old and candidate model/runtime, comparing quality and latency. | Supplied smoke proves a narrow canary; a separate benchmark and rollback drill are required. |

### Text architecture diagram

```text
BUILD / REVIEW (operator workstation)
  Requirements + architecture decisions + pinned versions + verified site inputs
       -> Python renderers -> reviewed Kubernetes JSON + checksums
       -> human approval -> approved bootstrap / GitOps owner

MANAGEMENT (existing independent cluster)
  Rancher + Elemental / OS Manager
       | host registration, OS images and RKE2 provisioning
       v
WORKLOAD (physical Supermicro servers; no VM layer)
  +----------------------------+   +-----------------------------------------+
  | RKE2 server                |   | RKE2 GPU worker                         |
  | control plane + etcd       |   | NVIDIA host driver + GPU Operator / CDI |
  +----------------------------+   |                                         |
                                   | SGLang pod:                             |
  Private model artifact store     |   checksum init -> SGLang + Qwen        |
       -> staged read-only PVC --->|                       :30000            |
                                   |   metrics-only proxy  :30001            |
                                   +-----------------------------------------+
  Operator localhost tunnel ---------> :30000 inference + :30001 metrics
  Approved monitoring pods ----------> :30001/metrics only
       -> Prometheus or OTel metrics configuration -> private Grafana
  GPU Operator / DCGM ----------------> GPU metrics (integration required)
```

Argo CD owns downstream resources after bootstrap. Host lifecycle stays with
Rancher/Elemental. Monitoring configurations are extensions to deploy and validate;
they are not a preinstalled monitoring stack. NetworkPolicy denies ordinary pod access
to inference and model-pod egress. The authorized tunnel uses Kubernetes API permissions.

### BMAD / CodeSpec development strategy

The project follows a **specification-to-code-to-evidence** workflow. In BMAD/CodeSpec terms,
planning defines the intended outcome and architecture; implementation turns that agreed
contract into code; review and tests check whether the result meets it. This aligns with
[BMAD's planning and build approach](https://docs.bmad-method.org/) and the
[CodeSpec specification-driven workflow](https://github.com/LucivHuang/CodeSpec/blob/main/README.EN.md).
Here these names describe the method alignment: the repository contains requirements and
validation artifacts, but does not record execution of either framework's own tooling.

The strategy is visible in five steps:

1. **Define the outcome and constraints.** Requirements name the capability, accountable owner,
   implementation, test and evidence needed for acceptance.
2. **Record design decisions.** Architecture records establish the topology, lifecycle owners,
   model identity, private access boundary and known limitations before deployment.
3. **Implement the contract.** Python renderers convert reviewed site configuration into
   manifests; the buildspec requires validation before packaging. Missing critical inputs
   block preparation instead of being replaced with guessed values.
4. **Review and test the result.** Regression tests cover expected behavior and failure cases.
   Review findings become tracked fixes, such as release-identity checks and metrics isolation.
5. **Hand over with evidence.** Human reviewers inspect the artifacts and authorize site work.
   Live smoke, performance and recovery results establish acceptance; local tests alone do not.

Review the [requirements](acmecorp-llm-demonstrator.zip),
[design decisions](acmecorp-llm-demonstrator.zip)
and [validation record](acmecorp-llm-demonstrator.zip) to trace this process. TOGAF supplies the architecture
review structure; BMAD/CodeSpec alignment describes the development workflow.

### Technology stack

| Layer | Technology | Role in the project |
|---|---|---|
| Physical compute | Supermicro servers, NVIDIA GPU, local NVMe | Host execution and replaceable cache; sizing requires site validation |
| Host lifecycle | Elemental / OS Manager | Register hosts and manage approved OS images with compatible NVIDIA drivers |
| Cluster management | Rancher and bare-metal RKE2 | Provision and manage Kubernetes directly on physical hosts |
| GPU integration | NVIDIA GPU Operator and CDI | Make host GPUs available to workloads; this profile disables operator driver installation |
| Model serving | SGLang and pinned Qwen3.6-27B snapshot | Provide inference with bounded context and concurrency |
| Model storage | Approved private artifact store and existing read-only Kubernetes PVC | Preserve authoritative model files and mount the verified serving copy |
| Workload delivery | Git, Argo CD and Helm chart values | Define reviewed desired state; bootstrap and controller ownership remain explicit |
| Observability | Prometheus, metrics-only OpenTelemetry configuration, NVIDIA DCGM and private Grafana | Supply monitoring configuration extensions requiring site deployment and validation |
| Build and validation | Python 3.12+, Bash, JSON manifests, kubectl and optional schema tooling | Generate, inspect, test and package the operator kit |
| Governance | TOGAF records, release evidence and Day 0/1/2 runbooks | Support architecture review, operational acceptance and controlled change |

The [release baseline](acmecorp-llm-demonstrator.zip)
records component pins. Upstream stable releases still require a compatible, supported hardware
and software combination at the deployment site.

### How human reviewers should use it

Start with the purpose and scope above, then inspect the requirements, architecture decisions
and **Issues** section below. Confirm that the proposed topology and ownership match the lab.
Use the [operator quick start](acmecorp-llm-demonstrator.zip)
to generate a fresh kit, identify missing inputs and prepare a review bundle. Reviewers do not
need to install the assistant skill to read the documents or run the standalone kit.

Run the offline build checks, inspect the generated manifests and confirm image/model identity,
physical disk mapping, private access and rollback arrangements. The deployment owner then
follows Day 0 for host provisioning, Day 1 for inference deployment and validation, and Day 2
for operations and recovery. Keep management and workload contexts distinct. Complete the
[operations handoff](acmecorp-llm-demonstrator.zip)
only after the required site evidence exists. Commands and detailed procedures follow below.

## Deliverables

- [Skill entrypoint](acmecorp-llm-demonstrator.zip)
- [Build-kit README](acmecorp-llm-demonstrator.zip)
- [Day 0: Elemental and physical host provisioning](acmecorp-llm-demonstrator.zip)
- [Day 1: GPU, model, deployment and smoke](acmecorp-llm-demonstrator.zip)
- [Day 2: operations and recovery](acmecorp-llm-demonstrator.zip)
- [Architecture and TOGAF alignment](acmecorp-llm-demonstrator.zip)
- [Gap review](acmecorp-llm-demonstrator.zip)
- [Release evidence process](acmecorp-llm-demonstrator.zip)
- [Local validation and outstanding live checks](acmecorp-llm-demonstrator.zip)

## Stable release policy - checked 15 October 2026

Select the latest stable **patch within the site's supported combination** of OS, driver,
Rancher, RKE2 and GPU Operator. Release numbering alone cannot establish the most stable
combination for unspecified hardware. No nightly, preview or floating `latest` tag is used.

| Component | Upstream stable candidate | Change in this review |
|---|---|---|
| RKE2 | v1.37.1+rke2r1 | Retained |
| Rancher Community | v2.15.2 | Retained; not equivalent to Prime support entitlement |
| Elemental operator | v1.9.2 | Retained; verify the vendor chart channel separately |
| NVIDIA GPU Operator | v26.7.1 | Retained; host image owns the driver |
| SGLang | 0.5.21 | Retained |
| Argo CD | v3.5.5 | Updated from v3.5.4 |
| OTel Collector Contrib | v0.162.0 | Retained; optional metrics extension |
| Model | Qwen3.6-27B | Existing reviewed snapshot retained |

The model revision remains `6a9e13bd6fc8f0983b9b99948120bc37f49c13e9`. The proposed profile
is BF16, one 80 GiB+ GPU, 4096-token context and two active requests. This is sizing guidance,
not a measured capacity result. See the [official release sources and support boundaries](acmecorp-llm-demonstrator.zip)
and [machine-readable baseline](acmecorp-llm-demonstrator.zip).
The SUSE support matrix reviewed still resolves to Rancher Prime 2.14.4; the newer Community
candidate tuple is **not certified as a supported Prime installation**. Resolve the compatible
site tuple and record any candidate deviation before provisioning.

## Usage quick start

### Linux reference environment

Use Bash on a maintained **SUSE Linux Enterprise Server (SLES)** operator workstation.
Install Git, Bash and Python 3.12+ from approved, supported repositories; activate a virtual
environment if the system Python is older, without replacing the OS interpreter. Install
an approved kubectl version compatible with the target cluster. Run commands as an ordinary
user from the stated directory. No PowerShell is required.

The workstation OS is separate from the Elemental-managed server image. Select the server
OS/service pack, kernel and NVIDIA driver only after checking the
[SUSE RKE2 support matrix](https://www.suse.com/suse-rke2/support-matrix/all-supported-versions/)
and the Elemental and NVIDIA support requirements. SLES is a reference choice, not a claim
that every SLES version or this candidate stack is certified. Keep the seed image unresolved
until that combination is approved; do not install packages ad hoc on immutable hosts.

Bash examples use `python3`, quote paths and contexts, and stop on explicit prerequisite
failures. Run each block in order, stopping at any unexpected nonzero exit. `prepare --check`
returns 2 for incomplete inputs; `kubectl diff` returns 1 for an expected proposed change.

### 1. Review and validate locally

Start in this repository's root with Python 3.12+ and Bash. No GPU or cluster is
required for this step. The tests use temporary files and localhost HTTP servers; allow
loopback connections. Use a Python 3.12+ virtual environment so `python3` selects the approved interpreter.

```bash
python3 tools/build.py --check || { printf '%s\n' 'Local validation failed.' >&2; exit 1; }
KitRoot="$PWD/platform-build-kit"
python3 acmecorp-native-llm-inference-platform/scripts/scaffold.py --output "$KitRoot" || { printf '%s\n' 'Choose a new kit directory; existing work is preserved.' >&2; exit 1; }
python3 "$KitRoot/scripts/prepare.py" --check
```

The final command initially returns **exit 2**, listing unresolved site values. This is an
expected safety gate. Obtain image digests, GPU hostname, model PVC, support evidence,
SSH public key, host-specific install disk IDs and independent model-file hashes. Edit the
three JSON files in the generated kit's `config/` folder, not the reusable source templates.

### 2. Prepare a bundle for deployment review

```bash
python3 "$KitRoot/scripts/prepare.py" --check || { printf '%s\n' 'Resolve site configuration before rendering.' >&2; exit 1; }
python3 "$KitRoot/scripts/prepare.py" --output "$KitRoot/generated/review-001" || { printf '%s\n' 'Preparation failed; inspect the reported error.' >&2; exit 1; }
```

Read [Day 0](acmecorp-llm-demonstrator.zip)
for disk/host verification and management provisioning, then
[Day 1](acmecorp-llm-demonstrator.zip)
for GPU, model storage, server dry-run and reviewed GitOps deployment. Use distinct management
and workload contexts. The commands above only generate files; they do not deploy a cluster.

### 3. Run live smoke after an authorized deployment

In terminal A, from the repository root:

```bash
KitRoot="$(cd ./platform-build-kit && pwd)"
Namespace="$(python3 -c 'import json,sys; print(json.load(open(sys.argv[1]))["namespace"])' "$KitRoot/config/site.json")"
read -r -p 'Approved workload context: ' WorkloadContext
: "${WorkloadContext:?A value is required}"
kubectl --context "$WorkloadContext" -n "$Namespace" port-forward --address 127.0.0.1 service/sglang 30000:30000 30001:30001
```

Keep that tunnel running. In terminal B, also from the repository root:

```bash
KitRoot="$(cd ./platform-build-kit && pwd)"
read -r -p 'Same approved workload context: ' WorkloadContext
: "${WorkloadContext:?A value is required}"
python3 "$KitRoot/scripts/smoke.py" --context "$WorkloadContext" --site "$KitRoot/config/site.json" --endpoint http://127.0.0.1:30000 --metrics-endpoint http://127.0.0.1:30001 --output "$KitRoot/evidence/smoke-001.json" || { printf '%s\n' 'Smoke did not pass; inspect the named failed checks.' >&2; exit 1; }
```

Smoke performs cluster reads and two synthetic completions. It checks release/model identity,
checksum-init success, both serving containers, node/PVC placement, rollout, exact canary
responses and the **separate metrics proxy**. It does not prove latency SLOs or network-policy
isolation. Choose a new evidence filename each run. An existing file is never overwritten.
For approved remote endpoints, inference and metrics require separate explicit auth files;
credentials are never automatically forwarded from one endpoint to the other.

## Install the skill

Choose the skill destination recognized by your agent/project. Copy the **whole skill directory**,
including assets and references, or use the supplied copy helper:

```bash
read -r -p 'Full new destination directory for this skill: ' SkillDestination
: "${SkillDestination:?A value is required}"
python3 acmecorp-native-llm-inference-platform/scripts/install.py --destination "$SkillDestination"
```

The installer refuses existing destinations. For upgrades, review the diff and preserve the old
installation before selecting a new destination. It does not alter global agent configuration.
The .skill file is a ZIP with one skill-directory root; clients that do not import .skill directly
can extract it. Select the skill with your agent's supported invocation mechanism.

Example task: "Use this skill to generate the Elemental-managed ACMECORP DEV demonstrator kit.
Use synthetic prompts, preserve private endpoints, and generate reviewable files only."

## Build a distributable release

```bash
python3 tools/build.py --output release/my-reviewed-release || { printf '%s\n' 'Build failed.' >&2; exit 1; }
```

Choose a new output directory. `buildspec.json` requires validation before packaging and
references the version baseline; it is not an AWS CodeBuild specification. Outputs are the
source ZIP, installable .skill archive and CHECKSUMS.txt. The source ZIP includes its build
tools. Model weights, credentials, local evidence and generated projects are excluded.
Checksums establish integrity, not publisher authenticity.

## Caveats and acceptance boundary

- **No HA:** one control-plane server and one serving replica; upgrades and faults cause downtime.
- **Hardware remains unqualified:** verify Supermicro SKU, firmware, OS/kernel, NVIDIA driver,
  CUDA, CDI, VRAM, RAM and storage together. Null site inputs deliberately block rendering.
- **Private lab access only:** no enterprise gateway, SSO, tenant isolation or public endpoint is
  delivered. Monitoring must not receive inference access; audit all additive network policies.
- **Synthetic data only:** do not send PII, payment/card data or credentials. No financial actions
  or model training are implemented. Example use cases need human review and application work.
- **Operational prerequisites:** independent Rancher/Elemental management, approved model PVC,
  trusted registries, backups and restore procedures must exist. Model hashes are not provenance
  unless independently verified. Keep secrets outside Git and release archives.
- **Local tests are not live acceptance:** runtime non-root startup, GPU inference, quality,
  throughput, latency, privacy, network isolation, restore and rollback still require site evidence.
  TOGAF artifacts do not confer PCI/FIPS certification or organizational approval.
- **CLI change:** smoke now requires `--metrics-endpoint`; tunnel both 30000 and 30001. A passing
  inference-port metrics response can no longer hide a broken metrics sidecar.

See [validation results](acmecorp-llm-demonstrator.zip) and the
[operator quick start](acmecorp-llm-demonstrator.zip).

## Issues from the 9 October review

| Priority | Finding | Resolution in this revision | Remaining verification |
|---|---|---|---|
| P1 | Smoke could pass the wrong model release with the same runtime and alias. | Compare pod and deployment release/model/inventory annotations; require successful checksum init, expected node and PVC. Save expected identity in evidence. | Live release validation and independent endpoint/tunnel ownership verification. |
| P2 | Monitoring access to port 30000 also allowed inference. | Metrics-only sidecar on 30001; monitoring policy permits only that port and proxy forwards only exact GET /metrics. | Live negative network tests, including inference attempts from the monitoring pod. |
| P2 | A trickling stream could exceed the nominal timeout. | HTTP worker has a total deadline; incremental SSE stops at DONE and caps response bytes. | Actual gateway/runtime validation. |

All three have local regression coverage. Compatibility, model quality, performance,
restore and rollback remain open; local tests are not a bare-metal deployment acceptance.

## Review fixes - 15 October 2026

| Finding | Change | Verification |
|---|---|---|
| Metrics smoke bypassed the protected metrics listener. | Require a distinct metrics endpoint and separate credentials for remote metrics. | HTTP mock asserts port 30001; endpoint/credential rejection tests. |
| A partial container-status list could pass readiness. | Require both sglang and metrics-proxy containers and their ready runtime digests. | Missing and unready sidecar regression tests. |
