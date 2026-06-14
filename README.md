<p align="center">
  <img src="docs/plumber-audit.jpg" alt="Plumber audit of Brik pipelines">
</p>

<p align="center">
  <b>Brik pipelines, audited for compliance.</b><br>
  An independent scan of a Brik-generated GitLab pipeline with Plumber, run on the Briklab test infrastructure.<br>
  <i>Every control Brik owns comes back clean.</i>
</p>

<p align="center">
  <a href="https://getplumber.io/"><img src="https://img.shields.io/badge/scanner-Plumber%20v0.3.58-blue" alt="Plumber"></a>
  <a href="#results-2026-06-14"><img src="https://img.shields.io/badge/controls-10%2F12%20passing-yellow" alt="Controls"></a>
  <a href="#where-plumber-and-brik-see-different-layers"><img src="https://img.shields.io/badge/2%20findings-out%20of%20Brik's%20scope-lightgrey" alt="Findings"></a>
  <a href="#results-2026-06-14"><img src="https://img.shields.io/badge/audited-2026--06--14-lightgrey" alt="Audited"></a>
</p>

<p align="center">
  <a href="https://github.com/getbrik/brik#readme">Brik</a> -
  <a href="https://github.com/getbrik/briklab#readme">Briklab</a> -
  <a href="https://getplumber.io/">Plumber</a>
</p>

---

## What this is

A point-in-time compliance audit: Plumber v0.3.58 scanning the GitLab pipeline that
[Brik](https://github.com/getbrik/brik) generates, run against the
[Briklab](https://github.com/getbrik/briklab) end-to-end infrastructure. The question
is simple - does Brik's one-line `include:` produce a compliant pipeline, or does the
abstraction hide risk?

Ten of the twelve evaluated controls pass. The other two are not pipeline defects, and
one of the passes deserves an asterisk - and all three are the same story:
**Plumber reads the static pipeline file; Brik's substance lives at runtime**
([details](#where-plumber-and-brik-see-different-layers)).

> [!NOTE]
> A snapshot, not a live gate: Plumber v0.3.58 against `brik/node-deploy-signed` at
> commit `892dcc9`, 2026-06-14. The numbers come from [`reports/`](reports/); rerun to
> refresh (see [Reproduce the audit](#reproduce-the-audit)).

## What Plumber checks

Plumber is an open-source CLI compliance scanner for GitLab CI/CD pipelines. Its
controls map onto the supply-chain and hardening concerns Brik is built to enforce:
image pinning and authorized sources, branch protection, hardcoded jobs vs. reusable
includes, include versioning, debug trace, unsafe variable expansion, security-job
weakening, job-variable overrides, unverified script execution (`curl | bash`), and
Docker-in-Docker.

## Results (2026-06-14)

`brik/node-deploy-signed` is a Briklab E2E fixture: a deployable Node service running
the full Brik flow (CI plus signed-artifact CD) via `include:`. Plumber scores it
**83.3% compliant** - 10 of 12 evaluated controls at 100%.

| Control | Compliance | Note |
|---------|-----------|------|
| Container images must come from authorized sources | 🟢 100.0% | passes once `ghcr.io/getbrik/*` is trusted (see config) |
| Pipeline must not include hardcoded jobs | 🟢 100.0% | the `include:` model, Brik's core value |
| Includes must be up to date | 🟢 100.0% | |
| Includes must not use forbidden versions | 🟢 100.0% | |
| Pipeline must not enable debug trace | 🟢 100.0% | |
| Pipeline must not use unsafe variable expansion | 🟢 100.0% | |
| Security jobs must not be weakened | 🟢 100.0% | |
| Pipeline must not override job variables | 🟢 100.0% | |
| Pipeline must not use Docker-in-Docker | 🟢 100.0% | |
| Pipeline must not execute unverified scripts | 🟢 100.0% | passes, [with an asterisk](#where-plumber-and-brik-see-different-layers) |
| Container images must not use forbidden tags (pinned by digest) | 🔴 0.0% | [pinned at runtime](#where-plumber-and-brik-see-different-layers) |
| Branch must be protected | 🔴 0.0% | [infra's job, not Brik's](#where-plumber-and-brik-see-different-layers) |

Three further controls (required components, required templates, leaked secrets) are
disabled in this configuration and skipped.

## Where Plumber and Brik see different layers

All three results that are not a plain green come from one boundary: a static scan of
`.gitlab-ci.yml` cannot see what Brik resolves at runtime.

- **Image pinning** 🔴 *(false alarm)* - Brik never hardcodes an image path. `brik-init`
  resolves each runner class from
  [`runner_classes.yml`](https://github.com/getbrik/brik/blob/main/lib/registry/runner_classes.yml)
  and publishes `BRIK_IMG_*` via dotenv, so jobs run on `image: ${BRIK_IMG_STACK}`. The
  digest pin, when you want one, lives in that map - overridable via
  `BRIK_RUNNER_CLASSES_FILE`, or digest-locked in Briklab's `brik-images.lock.yaml` - one
  layer above the pipeline. Plumber sees an unresolved variable (and, in the `v0.1.0`
  template this fixture pins, a literal `:latest`) and flags it: correct for a static
  scan, blind to where Brik pins. The artifact Brik actually deploys is separately
  digest-gated (`require_digest` / `require_attestation`) in the CD flow.

- **Branch protection** 🔴 *(infra's job, not Brik's)* - whether `main` is protected is a
  GitLab project setting owned by the platform team. A stateless, portable pipeline tool
  has no business rewriting repo access policy. On Briklab the E2E projects are ephemeral
  fixtures created without protection, so this reflects the lab, not Brik.

- **Unverified scripts** 🟢 *(passes, with an asterisk)* - Plumber finds no inline
  `curl | bash` in the 29 script lines. True, but Brik's logic isn't in those lines: each
  job's `before_script` clones the Brik runtime
  (`git clone --depth 1 --branch ${BRIK_LIB_REF}`, default `v0.7.0`) and runs it, so the
  green says nothing about that code. Brik's mitigation: the runtime is cloned from a
  **signed, GitHub-verified release tag** of an adopter-controlled repo (not `curl | bash`
  from anywhere), and that repo is the ShellCheck-clean, 5100+ ShellSpec-tested codebase.
  The tag is signed but **not yet protected or immutable** at the source repo - closing
  that (tag protection + immutability, optionally verifying the signature at clone time)
  is the subject of an upcoming **Brik tag-hardening chantier**.

**Takeaway:** a static pipeline scan is necessary but not sufficient. It cannot reach the
runtime layer where Brik resolves images, pins its runtime, and enforces its CD gates -
which is why it both over-reports (image pinning) and under-reports (unverified scripts)
here.

## Reproduce the audit

The trust policy in [`.plumber.yaml`](.plumber.yaml) (schema v2.0) trusts Brik's own
registry - `ghcr.io/getbrik/*` and the `BRIK_IMG_*` runner-class variables - so the
authorized-sources control reflects Brik's real image provenance.

```bash
# Load GITLAB_PAT from the briklab .env, then expose it the way Plumber expects.
source ../briklab/.env
export GITLAB_TOKEN="$GITLAB_PAT"

plumber analyze \
  --gitlab-url http://gitlab.briklab.test:8929 \
  --project brik/node-deploy-signed \
  --output reports/node-deploy-signed.json
```

Plumber exits non-zero because compliance is below the default 100% threshold; the two
findings above account for the gap.

## Layout

```
plumber-audit/
  .plumber.yaml                 # Plumber trust policy (schema v2.0), tuned to Brik's registry
  reports/
    node-deploy-signed.json     # full report for this run (83.3%)
```

## Related projects

- [brik](https://github.com/getbrik/brik) - the portable CI/CD pipeline system whose output is audited here.
- [briklab](https://github.com/getbrik/briklab) - the local GitLab and Jenkins infrastructure this pipeline ran on.
- [Plumber](https://getplumber.io/) - the open-source GitLab CI/CD compliance scanner used for the audit.
