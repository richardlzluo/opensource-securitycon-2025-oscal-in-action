# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Demo content for the OpenSource SecurityCon 2025 talk "OSCAL in Action". It is a compliance-as-code pipeline: Gemara governance YAML → OSCAL Assessment Plan → OPA/Rego policy bundle → evidence (conftest results → OCSF → OpenTelemetry → Grafana Loki) → OSCAL Assessment Results / posture report. Single Go module (`github.com/jpower432/opensource-securitycon-2025-oscal-in-action`, Go 1.24) with three binaries under `cmd/`.

## Commands

```bash
go build ./...
go vet ./...
go test ./...                      # no Go tests exist yet

# Generate an OSCAL Assessment Plan from governance/ (prints JSON to stdout)
go run ./cmd/transformer-kit plan -t "GitHub Repository" -r 800-53 > assessment-plan.json

# Build the C2P plugin binary (CI also writes a c2p-opa-manifest.json next to it; see .github/actions/install-plugin)
go build -o c2p-plugins/opa-plugin ./cmd/plugin

# Export conftest JSON results as OCSF evidence to an OTel collector
go run ./cmd/conftest-exporter --endpoint localhost:4317 --path output.json

# Rego policies (run from the check's policy/ dir)
opa test .                                                    # all tests for a check
opa test s3_data_integrity_policy_test.rego s3_data_integrity_policy.rego
opa eval --data github_branch_protection.rego --input ../example.json 'data.main.deny'
```

The rest of the pipeline uses external tools that CI installs: `c2pcli` (`oscal2policy`, `result2oscal`, `oscal2posture`, all reading `configs/c2p-config.yaml`), `conftest` v0.62.0, and `snappy` (pulls GitHub branch rules into `input.json`).

## Architecture

The pipeline runs as three manual or push-triggered GitHub workflows, executed in this order:

1. **`govern.yml`** (runs on push to `checks/**`): `transformer-kit plan` → `c2pcli oscal2policy` (calls the plugin's `Generate`) → `conftest push` of `./policies/policy` to `ghcr.io/<org>/policy-bundle:dev`.
2. **`collect.yml`**: snappy snapshot → `conftest pull` + `conftest test` → `conftest-exporter` sends OCSF logs via `proofwatch` to an OTel collector service. The collector image is built from `Dockerfile` + `configs/collector.yaml` by `docker-publish.yml`.
3. **`report.yml`**: `c2pcli result2oscal` (calls the plugin's `GetResults`, which queries Loki) against the committed `compliance/assessment-plan.json`, then `oscal2posture` against `compliance/catalog.json` (NIST 800-53 rev5) to write the step summary.

Key components:
- **`cmd/transformer-kit`** (cobra): loads the Gemara Layer 2 catalog (`governance/catalogs/`), Layer 3 policy (`governance/policy.yaml`) and Layer 4 evaluation plan (`governance/plans/`), builds a component definition with `gemara2oscal`, and converts it to an Assessment Plan for the `-r` guidance reference. The reference must be one of the policy's `guidance-references`. After conversion it strips empty props and sets empty titles to `REPLACE_ME` so the output passes OSCAL schema validation.
- **`cmd/plugin`**: a compliance-to-policy-go PVP plugin served over hashicorp/go-plugin. `Generate` copies `checks/<check-id>/` into `policies/policy/` and can optionally build an OPA bundle. `GetResults` queries Loki for `policy_id="<id>"` and turns the log entries into OSCAL observations. Config keys are hyphenated (`policy-templates`, `loki-url`, …; see `server/config.go`). Credentials are read from the environment as `GRAFANA_CLOUD_LOKI_ENDPOINT`, `GRAFANA_CLOUD_INSTANCE_ID` and `GRAFANA_CLOUD_API_KEY`. `cmd/plugin/REAMDE.md` lists some keys and env var names that don't match the code; the code is correct.
- **`cmd/conftest-exporter`**: parses conftest JSON (`result.go`), converts each failure to OCSF, and logs it through proofwatch. Note that `main` currently hard-codes `newClient(endpoint, true, true)`, so the `--skip-tls`/`--skip-tls-verify` flags have no effect.

## The ID that links everything

A check's ID is used in four places and they must all match:
- the procedure `id` in `governance/plans/cnscc.yaml`
- the directory name under `checks/`
- the Rego `custom.short_name` annotation on each `deny` rule
- the `policy_id` / `policy.rule.id` label in Loki and in `configs/collector.yaml` transforms

To add a check, create `checks/<id>/policy/*.rego` and add a matching procedure to the plan.

`checks/github_branch_protection` (package `main`, rego.v1) is the check wired into the pipeline. `checks/s3_data_integrity` (package `terraform.aws.s3`, old-style `deny[msg]` syntax) is a standalone example and is not referenced in the plan.

Generated working dirs `c2p-plugins/`, `policies/`, `policy-results/`, `output.json` and `assessment-results.json` are gitignored.
