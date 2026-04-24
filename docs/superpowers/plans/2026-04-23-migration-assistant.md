# Migration Assistant Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the `migration-assistant` skill to `neubirdai/falconclaw-skills` — an end-to-end VMware → AWS migration executor for the Falcon TUI.

**Architecture:** Single skill directory at `skills/migration-assistant/` with an orchestrator `SKILL.md` (≤500 lines), eight reference markdown files loaded on demand, and two read-only discovery helper scripts sharing a common JSON output schema. Flat `references/` layout (Approach B per spec) with clean promotion path to nested layout when 3+ destinations exist.

**Tech Stack:** Markdown (`SKILL.md` + references); Bash (`probe-vcenter.sh`, requires `govc`, `jq`); Python 3.9+ (`parse-rvtools.py`, test harness; deps `openpyxl`, `jsonschema`); JSON Schema Draft 2020-12.

**Spec:** `docs/superpowers/specs/2026-04-23-migration-assistant-design.md` — keep open while executing.

**Branching:** Create a feature branch (`feat/migration-assistant`) before Task 1. Open a PR after Task 15.

---

## Conventions used in this plan

- **TDD applies to the scripts, not the markdown.** Scripts are code; markdown is content. For scripts, write the JSON schema + tests first, then the script to pass them. For markdown, write → validate structurally → commit.
- **"Validate" for markdown** means: (a) the skill's CI validator passes, (b) the file contains all required sections, (c) any embedded HCL passes `terraform fmt -check -`, any embedded YAML parses cleanly with `yq -r .`.
- **Every task ends with a commit.** Commit message convention: `<type>(migration-assistant): <subject>` where type is `feat` / `test` / `docs` / `chore`.
- **Exact content is embedded in the plan for `SKILL.md` and the scripts** (these files are where precision matters most). Reference files get outlines + required content + embedded examples; the executor writes narrative prose using the spec as reference.

---

## Prerequisites

- [ ] On `main` with clean working tree: `git status --short` outputs nothing.
- [ ] Python 3.9+ available: `python3 --version`.
- [ ] `jq` installed: `jq --version`.
- [ ] `node` + npm available (for local CI validator): `node --version`.
- [ ] Install Python deps for script tests: `pip install --user openpyxl jsonschema`.
- [ ] (Optional) `terraform` CLI installed for `terraform fmt -check` validation: `terraform version`. If missing, skip HCL-fmt checks; the CI itself does not run them.

---

## Task 1: Scaffold skill directory and stub SKILL.md

**Purpose:** Get CI green with minimal content, then fill in.

**Files:**
- Create: `skills/migration-assistant/SKILL.md`
- Create: `skills/migration-assistant/references/` (directory)
- Create: `skills/migration-assistant/scripts/` (directory)

### Steps

- [ ] **Step 1: Create feature branch**

```bash
git checkout main
git pull
git checkout -b feat/migration-assistant
```

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p skills/migration-assistant/references skills/migration-assistant/scripts
```

- [ ] **Step 3: Write stub `SKILL.md` with only frontmatter + body placeholder**

Write to `skills/migration-assistant/SKILL.md`:

```markdown
---
name: migration-assistant
description: "Executes VMware → AWS migrations end-to-end: capability-probed discovery (live vCenter, GitHub IaC, telemetry), drift detection, 6Rs rehost/replatform decisions, template conversion (VCF Automation, Aria blueprints, Terraform vSphere, OVF) to AWS equivalents, wave-based cutover runbooks, post-cutover validation, and monitoring setup. Falcon is read-only: the skill presents commands and copy-ready IaC bundles; the user executes them. Invoke when workloads are in VMware (vSphere/VCF) and the target is AWS."
tags: [migration, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-assistant
metadata: {"neubird":{"emoji":"🚚","requires":{"anyBins":["aws","govc"]}}}
---

# Migration Assistant (VMware → AWS)

_Body written in Task 5._
```

- [ ] **Step 4: Install node dep for CI validator (once per clone)**

```bash
cd .github/workflows
# Only if 'yaml' isn't already installed globally:
npm install --no-save yaml
cd -
```

- [ ] **Step 5: Run CI validator locally to confirm stub passes**

```bash
node --input-type=module -e '
import { readFileSync } from "fs";
import { parse as parseYaml } from "yaml";
const path = "skills/migration-assistant/SKILL.md";
const content = readFileSync(path, "utf-8");
const lines = content.split("\n").length;
if (lines > 500) { console.error(`FAIL: ${lines} > 500`); process.exit(1); }
const m = content.match(/^---\n([\s\S]*?)\n---/);
if (!m) { console.error("FAIL: no frontmatter"); process.exit(1); }
const fm = parseYaml(m[1]);
if (fm.name !== "migration-assistant") { console.error(`FAIL: name "${fm.name}"`); process.exit(1); }
if (!fm.description) { console.error("FAIL: missing description"); process.exit(1); }
console.log(`OK (${lines} lines)`);
'
```

Expected output: `OK (<N> lines)` where N is ~12.

- [ ] **Step 6: Commit**

```bash
git add skills/migration-assistant/
git commit -m "feat(migration-assistant): scaffold skill directory with stub SKILL.md"
```

---

## Task 2: JSON schema + fixtures for discovery script output

**Purpose:** Both `probe-vcenter.sh` and `parse-rvtools.py` emit the same JSON shape. Lock that contract first; scripts in Tasks 3–4 are written to pass it.

**Files:**
- Create: `skills/migration-assistant/scripts/schema.json`
- Create: `skills/migration-assistant/scripts/fixtures/sample-vcenter-output.json`
- Create: `skills/migration-assistant/scripts/fixtures/sample-rvtools.csv`
- Create: `skills/migration-assistant/scripts/fixtures/expected-rvtools-output.json`

### Steps

- [ ] **Step 1: Create fixtures directory**

```bash
mkdir -p skills/migration-assistant/scripts/fixtures
```

- [ ] **Step 2: Write `schema.json`**

Write to `skills/migration-assistant/scripts/schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/neubirdai/falconclaw-skills/migration-assistant/schema/v1",
  "title": "Migration Assistant Discovery Output v1",
  "type": "object",
  "required": ["schemaVersion", "source", "capturedAt", "vcenter", "clusters", "hosts", "vms", "networks", "datastores"],
  "properties": {
    "schemaVersion": { "const": "1" },
    "source": { "enum": ["govc", "rvtools"] },
    "capturedAt": { "type": "string", "format": "date-time" },
    "vcenter": {
      "type": "object",
      "required": ["url", "version"],
      "properties": {
        "url": { "type": "string" },
        "version": { "type": "string" }
      }
    },
    "clusters": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name"],
        "properties": { "name": { "type": "string" } }
      }
    },
    "hosts": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "cluster"],
        "properties": {
          "name": { "type": "string" },
          "cluster": { "type": "string" },
          "cpuMhz": { "type": "number" },
          "memoryMB": { "type": "number" }
        }
      }
    },
    "vms": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "vcpus", "memoryMB", "disks", "networks", "guestOs", "powerState"],
        "properties": {
          "id": { "type": "string" },
          "name": { "type": "string" },
          "vcpus": { "type": "integer", "minimum": 1 },
          "memoryMB": { "type": "integer", "minimum": 1 },
          "disks": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["sizeGB"],
              "properties": {
                "sizeGB": { "type": "number" },
                "datastore": { "type": "string" },
                "thin": { "type": "boolean" }
              }
            }
          },
          "networks": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["name"],
              "properties": {
                "name": { "type": "string" },
                "mac": { "type": "string" }
              }
            }
          },
          "guestOs": { "type": "string" },
          "tags": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["category", "value"],
              "properties": {
                "category": { "type": "string" },
                "value": { "type": "string" }
              }
            }
          },
          "powerState": { "enum": ["on", "off", "suspended", "unknown"] }
        }
      }
    },
    "networks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name"],
        "properties": {
          "name": { "type": "string" },
          "vlanId": { "type": "integer" }
        }
      }
    },
    "datastores": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name"],
        "properties": {
          "name": { "type": "string" },
          "capacityGB": { "type": "number" },
          "freeGB": { "type": "number" },
          "type": { "type": "string" }
        }
      }
    }
  }
}
```

- [ ] **Step 3: Write `fixtures/sample-vcenter-output.json`** (one VM, one host, one cluster — the canned `--dry-run` output for `probe-vcenter.sh`)

```json
{
  "schemaVersion": "1",
  "source": "govc",
  "capturedAt": "2026-04-23T15:00:00Z",
  "vcenter": { "url": "https://vcenter.example.com", "version": "8.0.2" },
  "clusters": [{ "name": "cluster-prod" }],
  "hosts": [
    { "name": "esx-01.example.com", "cluster": "cluster-prod", "cpuMhz": 2400, "memoryMB": 262144 }
  ],
  "vms": [
    {
      "id": "vm-1001",
      "name": "web-01",
      "vcpus": 4,
      "memoryMB": 8192,
      "disks": [{ "sizeGB": 80, "datastore": "ds-ssd-01", "thin": true }],
      "networks": [{ "name": "web-net", "mac": "00:50:56:aa:bb:cc" }],
      "guestOs": "ubuntu20_64Guest",
      "tags": [{ "category": "app", "value": "web" }],
      "powerState": "on"
    }
  ],
  "networks": [{ "name": "web-net", "vlanId": 100 }],
  "datastores": [{ "name": "ds-ssd-01", "capacityGB": 4096, "freeGB": 1024, "type": "VMFS" }]
}
```

- [ ] **Step 4: Write `fixtures/sample-rvtools.csv`** (minimal RVTools-style CSV — just the `vInfo` tab as CSV with a single VM)

```csv
VM,Powerstate,CPUs,Memory,Provisioned MiB,In Use MiB,Host,Cluster,Datacenter,OS according to the configuration file,Annotation,DNS Name,IP Address,Primary IP Address
web-01,poweredOn,4,8192,81920,40960,esx-01.example.com,cluster-prod,dc-east,ubuntu20_64Guest,,,10.0.1.10,10.0.1.10
```

Note: real RVTools exports have many more columns and multiple tabs. For v1 we normalize only `vInfo` (VM list) in the CSV path; the xlsx path in Task 3 handles the richer tab set.

- [ ] **Step 5: Write `fixtures/expected-rvtools-output.json`** (what `parse-rvtools.py fixtures/sample-rvtools.csv` should emit). The `capturedAt` field will vary by run, so the test in Task 3 sets it deterministically via `--now` flag.

```json
{
  "schemaVersion": "1",
  "source": "rvtools",
  "capturedAt": "2026-04-23T15:00:00Z",
  "vcenter": { "url": "unknown", "version": "unknown" },
  "clusters": [{ "name": "cluster-prod" }],
  "hosts": [{ "name": "esx-01.example.com", "cluster": "cluster-prod" }],
  "vms": [
    {
      "id": "web-01",
      "name": "web-01",
      "vcpus": 4,
      "memoryMB": 8192,
      "disks": [{ "sizeGB": 80 }],
      "networks": [],
      "guestOs": "ubuntu20_64Guest",
      "tags": [],
      "powerState": "on"
    }
  ],
  "networks": [],
  "datastores": []
}
```

- [ ] **Step 6: Commit**

```bash
git add skills/migration-assistant/scripts/schema.json skills/migration-assistant/scripts/fixtures/
git commit -m "feat(migration-assistant): add discovery output JSON schema and test fixtures"
```

---

## Task 3: `parse-rvtools.py` — TDD

**Purpose:** Convert RVTools exports into normalized JSON matching `schema.json`.

**Files:**
- Create: `skills/migration-assistant/scripts/parse-rvtools.py`
- Create: `skills/migration-assistant/scripts/test_parse_rvtools.py`

### Steps

- [ ] **Step 1: Write the failing test**

Write to `skills/migration-assistant/scripts/test_parse_rvtools.py`:

```python
"""Tests for parse-rvtools.py.

Run from repo root:
  python3 -m pytest skills/migration-assistant/scripts/test_parse_rvtools.py -v
Or directly:
  python3 skills/migration-assistant/scripts/test_parse_rvtools.py
"""
import json
import subprocess
import sys
import unittest
from pathlib import Path

SCRIPTS_DIR = Path(__file__).parent
FIXTURES = SCRIPTS_DIR / "fixtures"
SCHEMA = SCRIPTS_DIR / "schema.json"
SCRIPT = SCRIPTS_DIR / "parse-rvtools.py"


def run_script(*args):
    result = subprocess.run(
        [sys.executable, str(SCRIPT), *args],
        capture_output=True, text=True, check=False,
    )
    return result


class TestParseRvtools(unittest.TestCase):
    def test_csv_matches_expected_output(self):
        """CSV input produces JSON matching the committed fixture."""
        r = run_script(str(FIXTURES / "sample-rvtools.csv"), "--now", "2026-04-23T15:00:00Z")
        self.assertEqual(r.returncode, 0, msg=f"stderr: {r.stderr}")
        actual = json.loads(r.stdout)
        expected = json.loads((FIXTURES / "expected-rvtools-output.json").read_text())
        self.assertEqual(actual, expected)

    def test_output_conforms_to_schema(self):
        """Output validates against schema.json."""
        from jsonschema import validate
        r = run_script(str(FIXTURES / "sample-rvtools.csv"), "--now", "2026-04-23T15:00:00Z")
        self.assertEqual(r.returncode, 0)
        schema = json.loads(SCHEMA.read_text())
        validate(instance=json.loads(r.stdout), schema=schema)

    def test_missing_file_exits_nonzero(self):
        r = run_script("/nonexistent/path.csv")
        self.assertNotEqual(r.returncode, 0)
        self.assertIn("not found", r.stderr.lower())


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the test to confirm it fails**

```bash
python3 skills/migration-assistant/scripts/test_parse_rvtools.py
```

Expected: FAILS with `FileNotFoundError` or `No module named` because `parse-rvtools.py` doesn't exist yet.

- [ ] **Step 3: Write `parse-rvtools.py`**

Write to `skills/migration-assistant/scripts/parse-rvtools.py`:

```python
#!/usr/bin/env python3
"""Parse an RVTools export into the migration-assistant discovery v1 schema.

Accepts .xlsx or .csv. For CSV, expects an RVTools-style vInfo header.
For xlsx, reads vInfo / vCPU / vMemory / vDisk / vNetwork / vHost / vDatastore tabs
and merges them. Emits JSON to stdout conforming to scripts/schema.json.

Usage:
    parse-rvtools.py <path> [--now ISO8601]

The --now flag sets capturedAt deterministically (for tests); default is current UTC.
"""
import argparse
import csv
import json
import sys
from datetime import datetime, timezone
from pathlib import Path


SCHEMA_VERSION = "1"


def _now_iso() -> str:
    return datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")


def _power_state(v: str) -> str:
    v = (v or "").lower()
    if "on" in v: return "on"
    if "off" in v: return "off"
    if "susp" in v: return "suspended"
    return "unknown"


def _parse_int(v, default=0):
    try:
        return int(float(v))
    except (ValueError, TypeError):
        return default


def parse_csv(path: Path, now: str) -> dict:
    """Parse RVTools vInfo CSV into v1 schema."""
    vms = []
    hosts_seen = {}
    clusters_seen = set()

    with path.open(newline="") as f:
        reader = csv.DictReader(f)
        for row in reader:
            name = row.get("VM", "").strip()
            if not name:
                continue
            vcpus = _parse_int(row.get("CPUs"), 1)
            mem_mb = _parse_int(row.get("Memory"), 1)
            prov_mib = _parse_int(row.get("Provisioned MiB"), 0)
            size_gb = round(prov_mib / 1024, 2) if prov_mib else 0
            host = row.get("Host", "").strip()
            cluster = row.get("Cluster", "").strip()
            vms.append({
                "id": name,
                "name": name,
                "vcpus": max(vcpus, 1),
                "memoryMB": max(mem_mb, 1),
                "disks": [{"sizeGB": size_gb}] if size_gb else [],
                "networks": [],
                "guestOs": row.get("OS according to the configuration file", "").strip() or "unknown",
                "tags": [],
                "powerState": _power_state(row.get("Powerstate", "")),
            })
            if host:
                hosts_seen[host] = cluster
            if cluster:
                clusters_seen.add(cluster)

    return {
        "schemaVersion": SCHEMA_VERSION,
        "source": "rvtools",
        "capturedAt": now,
        "vcenter": {"url": "unknown", "version": "unknown"},
        "clusters": [{"name": c} for c in sorted(clusters_seen)],
        "hosts": [{"name": h, "cluster": c} for h, c in sorted(hosts_seen.items())],
        "vms": vms,
        "networks": [],
        "datastores": [],
    }


def parse_xlsx(path: Path, now: str) -> dict:
    """Parse full RVTools xlsx (multiple tabs) into v1 schema."""
    try:
        from openpyxl import load_workbook
    except ImportError:
        print("ERROR: openpyxl is required for xlsx input. Install with: pip install openpyxl", file=sys.stderr)
        sys.exit(2)

    wb = load_workbook(path, read_only=True, data_only=True)

    def rows(tab):
        if tab not in wb.sheetnames:
            return []
        ws = wb[tab]
        header = None
        for i, row in enumerate(ws.iter_rows(values_only=True)):
            if i == 0:
                header = [str(c).strip() if c is not None else "" for c in row]
            else:
                yield dict(zip(header, row))

    vms = {}
    hosts = {}
    clusters = set()
    networks = {}
    datastores = {}

    for r in rows("vInfo"):
        name = (r.get("VM") or "").strip()
        if not name:
            continue
        vms[name] = {
            "id": name,
            "name": name,
            "vcpus": _parse_int(r.get("CPUs"), 1),
            "memoryMB": _parse_int(r.get("Memory"), 1),
            "disks": [],
            "networks": [],
            "guestOs": (r.get("OS according to the configuration file") or "unknown").strip(),
            "tags": [],
            "powerState": _power_state(r.get("Powerstate") or ""),
        }
        host = (r.get("Host") or "").strip()
        cluster = (r.get("Cluster") or "").strip()
        if host:
            hosts[host] = cluster
        if cluster:
            clusters.add(cluster)

    for r in rows("vDisk"):
        vm = (r.get("VM") or "").strip()
        if vm not in vms:
            continue
        size_mib = _parse_int(r.get("Capacity MiB"), 0)
        vms[vm]["disks"].append({
            "sizeGB": round(size_mib / 1024, 2) if size_mib else 0,
            "datastore": (r.get("Datastore") or "").strip() or None,
            "thin": (r.get("Thin") or "").strip().lower() in ("true", "yes", "1"),
        })

    for r in rows("vNetwork"):
        vm = (r.get("VM") or "").strip()
        if vm not in vms:
            continue
        vms[vm]["networks"].append({
            "name": (r.get("Network") or r.get("Port Group") or "unknown").strip(),
            "mac": (r.get("Mac Address") or "").strip() or None,
        })

    for r in rows("vHost"):
        name = (r.get("Host") or "").strip()
        if not name:
            continue
        hosts[name] = (r.get("Cluster") or hosts.get(name, "")).strip()

    for r in rows("vDatastore"):
        name = (r.get("Name") or "").strip()
        if not name:
            continue
        datastores[name] = {
            "name": name,
            "capacityGB": round(_parse_int(r.get("Capacity MiB"), 0) / 1024, 2),
            "freeGB": round(_parse_int(r.get("Free MiB"), 0) / 1024, 2),
            "type": (r.get("Type") or "unknown").strip(),
        }

    for r in rows("dvPort"):
        name = (r.get("Port") or r.get("Name") or "").strip()
        if name and name not in networks:
            networks[name] = {"name": name}

    return {
        "schemaVersion": SCHEMA_VERSION,
        "source": "rvtools",
        "capturedAt": now,
        "vcenter": {"url": "unknown", "version": "unknown"},
        "clusters": [{"name": c} for c in sorted(clusters)],
        "hosts": [{"name": h, "cluster": c} for h, c in sorted(hosts.items())],
        "vms": list(vms.values()),
        "networks": list(networks.values()),
        "datastores": list(datastores.values()),
    }


def main():
    ap = argparse.ArgumentParser(description=__doc__.splitlines()[0])
    ap.add_argument("path", help="Path to RVTools export (.xlsx or .csv)")
    ap.add_argument("--now", default=None, help="Override capturedAt (ISO-8601 UTC) for deterministic output")
    args = ap.parse_args()

    p = Path(args.path)
    if not p.exists():
        print(f"ERROR: file not found: {p}", file=sys.stderr)
        sys.exit(1)

    now = args.now or _now_iso()

    if p.suffix.lower() == ".csv":
        out = parse_csv(p, now)
    elif p.suffix.lower() in (".xlsx", ".xlsm"):
        out = parse_xlsx(p, now)
    else:
        print(f"ERROR: unsupported extension: {p.suffix}", file=sys.stderr)
        sys.exit(1)

    json.dump(out, sys.stdout, indent=2, sort_keys=False)
    print()


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run the tests and confirm they pass**

```bash
python3 skills/migration-assistant/scripts/test_parse_rvtools.py -v
```

Expected: all three tests `PASS`.

If `test_csv_matches_expected_output` fails, check whether the normalized output shape has drifted from the fixture. Fix the script or regenerate the fixture (only if the script is correct).

- [ ] **Step 5: Make script executable**

```bash
chmod +x skills/migration-assistant/scripts/parse-rvtools.py
```

- [ ] **Step 6: Commit**

```bash
git add skills/migration-assistant/scripts/parse-rvtools.py skills/migration-assistant/scripts/test_parse_rvtools.py
git commit -m "feat(migration-assistant): add parse-rvtools.py with CSV+xlsx support and schema-validated tests"
```

---

## Task 4: `probe-vcenter.sh` — TDD

**Purpose:** Fixed read-only `govc` sweep that emits the same normalized JSON as `parse-rvtools.py`. Supports a `--dry-run` mode that emits the canned fixture (for tests and for users without vCenter access to sanity-check downstream logic).

**Files:**
- Create: `skills/migration-assistant/scripts/probe-vcenter.sh`
- Create: `skills/migration-assistant/scripts/test_probe_vcenter.py`

### Steps

- [ ] **Step 1: Write the failing test**

Write to `skills/migration-assistant/scripts/test_probe_vcenter.py`:

```python
"""Tests for probe-vcenter.sh.

Validates that --dry-run output conforms to schema.json and matches the fixture.
Live-mode testing requires a real vCenter; that's a manual test, documented in the skill.
"""
import json
import subprocess
import unittest
from pathlib import Path

SCRIPTS_DIR = Path(__file__).parent
FIXTURES = SCRIPTS_DIR / "fixtures"
SCHEMA = SCRIPTS_DIR / "schema.json"
SCRIPT = SCRIPTS_DIR / "probe-vcenter.sh"


def run(*args):
    return subprocess.run(
        ["bash", str(SCRIPT), *args],
        capture_output=True, text=True, check=False,
    )


class TestProbeVcenter(unittest.TestCase):
    def test_dry_run_matches_fixture(self):
        r = run("--dry-run")
        self.assertEqual(r.returncode, 0, msg=f"stderr: {r.stderr}")
        actual = json.loads(r.stdout)
        expected = json.loads((FIXTURES / "sample-vcenter-output.json").read_text())
        self.assertEqual(actual, expected)

    def test_dry_run_validates_against_schema(self):
        from jsonschema import validate
        r = run("--dry-run")
        self.assertEqual(r.returncode, 0)
        schema = json.loads(SCHEMA.read_text())
        validate(instance=json.loads(r.stdout), schema=schema)

    def test_missing_env_vars_exits_nonzero(self):
        """Without --dry-run and without GOVC_URL, script should fail cleanly."""
        env = {"PATH": "/usr/bin:/bin"}  # Strip GOVC_* env vars
        r = subprocess.run(
            ["bash", str(SCRIPT)],
            env=env, capture_output=True, text=True, check=False,
        )
        self.assertNotEqual(r.returncode, 0)
        self.assertIn("GOVC_URL", r.stderr)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the test to confirm it fails**

```bash
python3 skills/migration-assistant/scripts/test_probe_vcenter.py
```

Expected: fails because `probe-vcenter.sh` doesn't exist.

- [ ] **Step 3: Write `probe-vcenter.sh`**

Write to `skills/migration-assistant/scripts/probe-vcenter.sh`:

```bash
#!/usr/bin/env bash
#
# probe-vcenter.sh — read-only vCenter inventory sweep for migration-assistant.
#
# Emits JSON to stdout conforming to scripts/schema.json (v1). Uses only
# read-only govc verbs: ls, vm.info, host.info, datastore.info, dvs.portgroup.info.
#
# Usage:
#   GOVC_URL=... GOVC_USERNAME=... GOVC_PASSWORD=... ./probe-vcenter.sh > out.json
#   ./probe-vcenter.sh --dry-run                # emits canned fixture
#
# Requires: govc, jq.
#
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

die() { echo "ERROR: $*" >&2; exit 1; }

if [[ "${1:-}" == "--dry-run" ]]; then
  cat "${SCRIPT_DIR}/fixtures/sample-vcenter-output.json"
  exit 0
fi

command -v govc >/dev/null 2>&1 || die "govc not found on PATH. Install from https://github.com/vmware/govmomi/releases"
command -v jq   >/dev/null 2>&1 || die "jq not found on PATH."

: "${GOVC_URL:?GOVC_URL must be set}"
: "${GOVC_USERNAME:?GOVC_USERNAME must be set}"
: "${GOVC_PASSWORD:?GOVC_PASSWORD must be set}"

NOW="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
VC_VERSION="$(govc about -json | jq -r '.About.Version // "unknown"')"

# Clusters
CLUSTERS_JSON="$(govc find / -type c 2>/dev/null \
  | awk -F/ '{print $NF}' \
  | jq -R -s -c 'split("\n") | map(select(length>0)) | map({name: .})')"

# Hosts (name + cluster)
HOSTS_JSON="$(govc find / -type h 2>/dev/null | while read -r host_path; do
  name="$(basename "$host_path")"
  # Cluster = 2nd-to-last component if present, else empty
  cluster="$(echo "$host_path" | awk -F/ 'NF>=3{print $(NF-1)}')"
  govc host.info -host "$host_path" -json 2>/dev/null | jq --arg n "$name" --arg c "$cluster" '
    {name: $n, cluster: $c,
     cpuMhz: (.HostSystems[0].Summary.Hardware.CpuMhz // 0),
     memoryMB: ((.HostSystems[0].Summary.Hardware.MemorySize // 0) / 1048576 | floor)}'
done | jq -s -c '.')"

# VMs
VMS_JSON="$(govc find / -type m 2>/dev/null | while read -r vm_path; do
  govc vm.info -vm.ipath "$vm_path" -json 2>/dev/null | jq '
    .VirtualMachines[0] as $vm
    | {
        id: ($vm.Self.Value // $vm.Config.InstanceUuid // $vm.Name),
        name: $vm.Name,
        vcpus: ($vm.Config.Hardware.NumCPU // 1),
        memoryMB: ($vm.Config.Hardware.MemoryMB // 1),
        disks: [ $vm.Config.Hardware.Device[]? | select(.DeviceInfo.Label | test("Hard disk"; "i")) | {
          sizeGB: ((.CapacityInKB // 0) / 1048576),
          datastore: (.Backing.Datastore.Value // null),
          thin: (.Backing.ThinProvisioned // false)
        }],
        networks: [ $vm.Config.Hardware.Device[]? | select(.MacAddress?) | {
          name: (.DeviceInfo.Summary // "unknown"),
          mac: .MacAddress
        }],
        guestOs: ($vm.Config.GuestId // "unknown"),
        tags: [],
        powerState: (
          if   ($vm.Runtime.PowerState == "poweredOn")  then "on"
          elif ($vm.Runtime.PowerState == "poweredOff") then "off"
          elif ($vm.Runtime.PowerState == "suspended")  then "suspended"
          else "unknown" end
        )
      }'
done | jq -s -c '.')"

# Networks
NETWORKS_JSON="$(govc find / -type n 2>/dev/null \
  | awk -F/ '{print $NF}' \
  | jq -R -s -c 'split("\n") | map(select(length>0)) | map({name: .})')"

# Datastores
DATASTORES_JSON="$(govc find / -type s 2>/dev/null | while read -r ds_path; do
  name="$(basename "$ds_path")"
  govc datastore.info -ds "$ds_path" -json 2>/dev/null | jq --arg n "$name" '
    .Datastores[0].Summary as $s
    | {name: $n,
       capacityGB: (($s.Capacity // 0) / 1073741824),
       freeGB:     (($s.FreeSpace // 0) / 1073741824),
       type:       ($s.Type // "unknown")}'
done | jq -s -c '.')"

jq -n \
  --arg ver "$VC_VERSION" \
  --arg url "$GOVC_URL" \
  --arg now "$NOW" \
  --argjson clusters   "$CLUSTERS_JSON" \
  --argjson hosts      "$HOSTS_JSON" \
  --argjson vms        "$VMS_JSON" \
  --argjson networks   "$NETWORKS_JSON" \
  --argjson datastores "$DATASTORES_JSON" \
  '{
    schemaVersion: "1",
    source:        "govc",
    capturedAt:    $now,
    vcenter:       {url: $url, version: $ver},
    clusters:      $clusters,
    hosts:         $hosts,
    vms:           $vms,
    networks:      $networks,
    datastores:    $datastores
  }'
```

- [ ] **Step 4: Make executable and run tests**

```bash
chmod +x skills/migration-assistant/scripts/probe-vcenter.sh
python3 skills/migration-assistant/scripts/test_probe_vcenter.py -v
```

Expected: all three tests PASS.

Notes on test_missing_env_vars_exits_nonzero: the test strips env vars. The script uses `set -u` + `${VAR:?msg}` which exits 1 with the error written to stderr — that's what the test asserts.

- [ ] **Step 5: Commit**

```bash
git add skills/migration-assistant/scripts/probe-vcenter.sh skills/migration-assistant/scripts/test_probe_vcenter.py
git commit -m "feat(migration-assistant): add probe-vcenter.sh with --dry-run and schema-validated tests"
```

---

## Task 5: SKILL.md orchestrator — full content

**Purpose:** Write the complete `SKILL.md` orchestrator. This is the only file Falcon's model sees at invocation time — precision matters most here.

**Files:**
- Modify: `skills/migration-assistant/SKILL.md` (replace stub body)

### Steps

- [ ] **Step 1: Replace the `SKILL.md` body with full orchestrator content**

Replace the entire file at `skills/migration-assistant/SKILL.md` with:

````markdown
---
name: migration-assistant
description: "Executes VMware → AWS migrations end-to-end: capability-probed discovery (live vCenter, GitHub IaC, telemetry), drift detection, 6Rs rehost/replatform decisions, template conversion (VCF Automation, Aria blueprints, Terraform vSphere, OVF) to AWS equivalents, wave-based cutover runbooks, post-cutover validation, and monitoring setup. Falcon is read-only: the skill presents commands and copy-ready IaC bundles; the user executes them. Invoke when workloads are in VMware (vSphere/VCF) and the target is AWS."
tags: [migration, vmware, aws, cloud]
homepage: https://github.com/neubirdai/falconclaw-skills/tree/main/skills/migration-assistant
metadata: {"neubird":{"emoji":"🚚","requires":{"anyBins":["aws","govc"]}}}
---

# Migration Assistant (VMware → AWS)

Executes end-to-end VMware → AWS migrations. Discovers the source environment, converts deployment templates to AWS-native IaC, drives wave-based cutovers, validates post-cutover health, and establishes baseline monitoring.

**Falcon is read-only.** This skill recommends commands, shell/python scripts to run, and copy-ready IaC bundles. The user executes them. Every "status check" and "validation" below uses read-only verbs (`describe-*`, `vm.info`, etc.); the skill never mutates state directly.

## Workflow Overview

```
Phase 0    Capability probe                    → Gate 1 (mode confirm)
Phase 0.5  Resume vs fresh + reconciliation
Phase 1    Discovery (live + IaC + telemetry)
Phase 2    Drift analysis                      → Gate 2 (inventory + drift)
Phase 3    6Rs decision per workload           → Gate 3 (per-workload)
Phase 4    Wave grouping + sequencing          → Gate 4 (plan)
Phase 5    Conversion (per wave)
Phase 6    Execution runbook (per wave)        → Gate 5 (per-wave cutover)
Phase 7    Validation + monitoring
Post       Decommission source VMs             → Gate 6 (per-wave decommission)
```

## Phase 0: Capability Probe (REQUIRED FIRST STEP)

Before answering any user question, run this probe. Emit the summary + declared operating mode, then wait at **Gate 1** for user approval.

### Probe commands

**AWS reachability:**

```bash
aws sts get-caller-identity
```

On success, record the account ID and active region.

**vCenter reachability** (only when `GOVC_URL` and `GOVC_USERNAME` are set):

```bash
govc about
```

On success, record vSphere version.

**IaC in cwd:**

```bash
find . -maxdepth 3 -type f \( -name "*.tf" -o -name "*.yaml" -o -name "*.yml" \) 2>/dev/null \
  | xargs grep -l -iE 'vmware|vcf|aria|blueprint|vsphere' 2>/dev/null \
  | head -20
```

For each detected file, determine its format (Terraform vsphere provider / VCF Automation YAML / Aria blueprint).

**Static exports:**

```bash
find . -maxdepth 2 -type f \
  \( -iname "RVTools*.xlsx" -o -iname "RVTools*.csv" \) 2>/dev/null
```

**Telemetry hints:**

```bash
env | grep -iE '^(VROPS_|ARIA_|PROMETHEUS_|DATADOG_|NEWRELIC_)' | cut -d= -f1
```

### Mode declaration

Based on probe outcomes, declare exactly one mode:

| Mode | Triggered when | Behavior |
|------|----------------|----------|
| `live+iac+telemetry` | live vCenter + IaC + telemetry all present | Authoritative sizing; drift detection first-class. |
| `live+iac` | live vCenter + IaC, no telemetry | Right-size from live config only. Flag missing telemetry as a risk at Gate 2. |
| `iac+exports` | no live vCenter; IaC + RVTools exports | Conversion proceeds from IaC; validate sizing against RVTools snapshots. Drift detection limited to IaC-vs-snapshot. |
| `advisory` | only AWS reachable, nothing on source side | Offer target-side advice only. Ask the user to provide `govc` credentials or an RVTools export before proceeding to Phase 1. |

Present the declared mode with a one-line rationale. **Stop at Gate 1 and wait for approval.**

## Phase 0.5: Resume vs Fresh Start

After Gate 1 clears, ask: "Is this a new migration, or are you resuming a previous one?"

### If resuming — structured prompt

Ask:

1. Which phase were you on? (discovery / 6Rs / wave planning / wave N in-flight / post-migration)
2. Which waves have been completed?
3. AWS account ID + region in use?
4. Source vCenter URL?
5. Last validated checkpoint (gate number + timestamp)?

Then **mandatorily reconcile** — do not trust the user's account alone. Run:

```bash
# Source side: live VM count
govc find / -type m | wc -l

# Destination side: EC2 count tagged by this migration
aws ec2 describe-instances --region <user-stated-region> \
  --filters "Name=tag:migration-source-vcenter,Values=<user-stated-vcenter>" \
  --query 'Reservations[].Instances[].InstanceId' --output text | wc -w
```

Compute delta vs. user-stated completed-wave counts. **If non-zero, stop and present divergence before suggesting any next step.**

### If fresh

Proceed to Phase 1.

## Phases 1–7

Each phase loads a specific reference. Read it, execute the phase, surface results at the phase's gate.

**Phase 1 — Discovery.** Build workload inventory from sources available in the declared mode. Load `references/discover-and-plan.md`.

**Phase 2 — Drift analysis.** Three-way reconcile live vs IaC vs telemetry. Load `references/drift-detection.md`. **Gate 2**: present inventory + drift report; user confirms scope and which drift is blocking.

**Phase 3 — 6Rs decision per workload.** Apply the VMware → AWS decision matrix in `references/discover-and-plan.md`. **Web-search current AWS service support matrices** (e.g. RDS engine versions, instance-family regional availability) before finalizing replatform picks. **Gate 3**: per-workload recommendation with rationale; accept overrides.

**Phase 4 — Wave grouping + sequencing.** Same reference. Group by dependency and risk tier (pilot → stateless → stateful → critical). **Gate 4**: confirm composition, sequence, cutover windows.

**Phase 5 — Conversion (per wave).** For each workload, identify template format (from Phase 1), load the matching `references/sources-*.md` + `references/destinations-aws.md`, emit a copy-ready AWS IaC bundle. Cross-check against Phase 2 drift; use telemetry-derived sizing.

**Phase 6 — Execution runbook (per wave).** Load `references/execute-and-verify.md`. Emit cutover runbook + paired rollback runbook + validation checklist. **Gate 5** (once per wave): user types "Execute Wave N now."

**Phase 7 — Validation + monitoring.** Same reference. Post-cutover health checks, before/after metrics comparison, baseline alarms + Config rules.

**Post — Decommission.** Only after validation passes + **72-hour minimum cooling period** + explicit user approval. **Gate 6** (per wave).

## Reference Guide

| Topic | Reference | Load when |
|-------|-----------|-----------|
| Discovery recipes, 6Rs matrix, wave grouping | `references/discover-and-plan.md` | Phases 1, 3, 4 |
| Three-way drift reconciliation | `references/drift-detection.md` | Phase 2 |
| VCF Automation templates | `references/sources-vcf-automation.md` | Phase 5, VCF format detected |
| Aria blueprints (vRA 8.x) | `references/sources-aria-blueprint.md` | Phase 5, Aria format detected |
| Terraform vSphere provider | `references/sources-terraform-vsphere.md` | Phase 5, TF format detected |
| OVF/OVA descriptors | `references/sources-ovf.md` | Phase 5, OVF format detected |
| AWS target selection | `references/destinations-aws.md` | Phases 3, 5 |
| Cutover / validation / monitoring | `references/execute-and-verify.md` | Phases 6, 7, Post |

## Scripts

Two read-only discovery helpers ship in `scripts/`. User runs them; Falcon reads the output back.

**`scripts/probe-vcenter.sh`** — fixed read-only `govc` sweep. Requires `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`, plus `govc` and `jq` on PATH. Supports `--dry-run` to emit a canned fixture.

```bash
bash ~/.config/neubird/skills/migration-assistant/scripts/probe-vcenter.sh > vcenter-inventory.json
```

**`scripts/parse-rvtools.py`** — normalizes RVTools `.xlsx` or CSV into the same JSON shape. Requires Python 3.9+; xlsx support needs `openpyxl` (`pip install openpyxl`).

```bash
python3 ~/.config/neubird/skills/migration-assistant/scripts/parse-rvtools.py path/to/RVTools.xlsx > vcenter-inventory.json
```

Both emit JSON conforming to `scripts/schema.json` (v1). Phase 1 discovery consumes this normalized file — downstream logic doesn't care which script produced it.

## Output Conventions

### Copy-ready IaC bundle

Every converted IaC snippet is fenced with a `# file: <destination-path>` header inside the block:

````
```hcl
# file: terraform/web-tier/main.tf
resource "aws_instance" "web_01" {
  ami           = "ami-..."
  instance_type = "t3.large"
  # ...
}
```
````

The user copies the whole block and drops it into their repo at the declared path.

### Runbook step format

Every command in a runbook is numbered `Wave N, Step M` and paired with an **Expected output** block:

````
**Wave 3, Step 5:** Verify ALB target group is healthy.

```bash
aws elbv2 describe-target-health --target-group-arn <arn>
```

**Expected output:** every target in `State: healthy`.
````

This lets the user verify each step independently and lets Falcon (on resume) reconcile actual vs expected.

## Constraints

### MUST DO

- **Run Phase 0 capability probe every session, before answering any other question.** Cache within the conversation; never skip.
- **Present phase results at every decision gate and wait for explicit user approval before advancing.** Six gates, non-negotiable.
- **Web-search current AWS best practices** for the specific service + region before finalizing a replatform recommendation. Service behavior, quotas, and regional availability change; training-data-only recommendations are a known failure mode.
- **Tag every recommended AWS resource** with `migration-wave`, `migration-source-vm-id`, and `migration-date` (ISO-8601 UTC).
- **Emit a rollback runbook for every wave, before the cutover runbook.**
- **Reconcile drift (live vs IaC vs telemetry) before recommending any sizing.** Per-source trust rules are in `references/drift-detection.md`.
- **Enforce a 72-hour minimum cooling period** between Gate 5 (cutover approval) and Gate 6 (decommission approval), unless the user explicitly overrides with stated rationale.

### MUST NOT DO

- **Execute any mutation directly.** Falcon is read-only, always. If a command would modify state, emit it for the user; never run it.
- **Skip the capability probe or resume reconciliation.**
- **Recommend deleting source VMs until validation passes AND the user explicitly approves decommission at Gate 6.**
- **Bundle unrelated workloads in one wave.** Atomic wave = atomic rollback.
- **Emit IaC with hardcoded secrets, IAM wildcards (`"*"`), or public `0.0.0.0/0` ingress** unless the user explicitly overrides with stated rationale.
- **Assume current AWS service behavior from training data.** Always web-search first for quotas / limits / deprecations.
````

- [ ] **Step 2: Run CI validator locally to confirm SKILL.md still passes**

```bash
node --input-type=module -e '
import { readFileSync } from "fs";
import { parse as parseYaml } from "yaml";
const path = "skills/migration-assistant/SKILL.md";
const content = readFileSync(path, "utf-8");
const lines = content.split("\n").length;
console.log(`Lines: ${lines} (cap 500)`);
if (lines > 500) { console.error(`FAIL: ${lines} > 500`); process.exit(1); }
const m = content.match(/^---\n([\s\S]*?)\n---/);
const fm = parseYaml(m[1]);
if (fm.name !== "migration-assistant") { console.error("FAIL: name mismatch"); process.exit(1); }
if (!fm.description) { console.error("FAIL: no description"); process.exit(1); }
console.log("OK");
'
```

Expected: `Lines: <N> (cap 500)` then `OK`. N should be in the ~260-290 range.

- [ ] **Step 3: Verify all required sections are present**

```bash
for section in \
  "# Migration Assistant (VMware → AWS)" \
  "## Workflow Overview" \
  "## Phase 0: Capability Probe (REQUIRED FIRST STEP)" \
  "## Phase 0.5: Resume vs Fresh Start" \
  "## Reference Guide" \
  "## Scripts" \
  "## Output Conventions" \
  "## Constraints"
do
  grep -qF "$section" skills/migration-assistant/SKILL.md || { echo "MISSING: $section"; exit 1; }
done
echo "All sections present."
```

Expected: `All sections present.`

- [ ] **Step 4: Commit**

```bash
git add skills/migration-assistant/SKILL.md
git commit -m "feat(migration-assistant): add SKILL.md orchestrator with 7-phase workflow and 6 gates"
```

---

## Task 6: `references/discover-and-plan.md`

**Purpose:** Drive Phases 1, 3, 4 — discovery recipes per mode, 6Rs decision matrix for VMware → AWS, wave grouping.

**File:** `skills/migration-assistant/references/discover-and-plan.md`

### Required sections (in this order)

1. **Purpose** (5–10 lines) — summarize what phases load this reference and what decisions it drives.

2. **Phase 1: Discovery recipes** (80–120 lines) — one sub-section per mode from Phase 0:
   - `live+iac+telemetry`: commands to run `probe-vcenter.sh`, parse IaC in cwd, pull telemetry (show `curl` snippets for Prometheus `http_api/query`, Datadog `api/v1/query`, vROps `suite-api/api/resources`).
   - `live+iac`: as above minus telemetry; note that sizing will be config-based.
   - `iac+exports`: instruct the user to drop the RVTools export into cwd; run `parse-rvtools.py`.
   - `advisory`: instruct user to install `govc` + set env vars OR provide RVTools export.
   - **Inventory normalization:** regardless of mode, end with a normalized list of workloads. Show the expected structure (JSON example).

3. **Phase 3: 6Rs decision matrix** (100–150 lines) — the centerpiece. Include this decision table verbatim:

   | If workload is... | AND... | Recommendation | Rationale |
   |-------------------|--------|----------------|-----------|
   | MSSQL Server | No license constraint preventing RDS | **Replatform → RDS SQL Server** | Managed backups, HA, patching |
   | MSSQL Server | License constraint (BYOL to dedicated host) | **Rehost → EC2 dedicated host** | License compliance |
   | MySQL/MariaDB | Standard workload | **Replatform → RDS MySQL or Aurora MySQL** | Managed; Aurora for read-heavy |
   | PostgreSQL | Standard workload | **Replatform → RDS PostgreSQL or Aurora PostgreSQL** | Managed; Aurora for read-heavy |
   | Oracle DB | Enterprise features, Oracle Cloud allowed | **Replatform → RDS for Oracle (BYOL)** | Managed where licensed |
   | Oracle DB | Hardcore Oracle-only features | **Rehost → EC2** | Feature compatibility |
   | NFS server | Standard POSIX access | **Replatform → EFS** | Managed NFS, multi-AZ |
   | NFS server | Windows-heavy, SMB required | **Replatform → FSx for Windows** | Native SMB |
   | NetApp ONTAP | Using advanced ONTAP features | **Replatform → FSx for NetApp ONTAP** | Feature parity |
   | Active Directory | Standalone, no forest trust complexity | **Replatform → AWS Managed Microsoft AD** | Managed domain controllers |
   | Active Directory | Complex trust topology | **Rehost → EC2 + Directory Service Connector** | Preserve topology during migration |
   | Exchange Server | Small team (<100 users) | **Replatform → WorkMail** | Managed mail |
   | Exchange Server | Large enterprise | **Rehost → EC2**, then future migration to M365 | Exchange on EC2 is complex; WorkMail may not fit |
   | SMB file server | — | **Replatform → FSx for Windows** | Native SMB |
   | Generic Linux app | Stateless, horizontally scalable | **Rehost → EC2 (ASG)** | Lift-and-shift; refactor later if desired |
   | Generic Linux app | Stateful, single-instance | **Rehost → EC2 + EBS** | Lift-and-shift |
   | Generic Windows app | — | **Rehost → EC2 Windows** | Lift-and-shift |
   | Custom appliance (vendor image) | — | **Rehost → EC2 via VM Import/Export** | Preserve appliance |
   | Workload with specialized hardware (GPU) | Modern GPU workload | **Rehost → EC2 `g5`/`p5` family** | Match hardware class |
   | Workload with specialized hardware (FC SAN) | — | **Rehost + re-architect storage** | FC SAN has no direct AWS equivalent; flag for user |

   After the table: add a "**Before finalizing any RDS pick:**" block telling the agent to web-search current RDS engine version support, regional availability, and quota limits for the user's target region. Provide a 3-line example search query.

4. **Phase 4: Wave grouping** (80–120 lines) —
   - **Dependency discovery:** for `live+iac+telemetry` mode, NSX flow logs or CloudWatch VPC-flow-analog on target. For other modes, use connection graphs inferred from telemetry (`netstat` during a probe window) or application-level knowledge from the user.
   - **Risk tiering:** pilot (1–2 low-risk workloads, observe end-to-end flow) → stateless tier → stateful tier → critical tier.
   - **Wave sizing:** default 5–10 workloads per wave; up to 20 for pure-stateless.
   - **Cutover windows:** weekdays-off-peak default; check application calendars for stateful workloads.
   - **Wave composition table template** (show the format Falcon emits at Gate 4).

5. **Handoff to Phase 5** (5 lines) — what output passes to the next reference (the `migration-plan` dict Falcon holds in memory).

### Steps

- [ ] **Step 1: Write the file per the outline above.** Embed the 6Rs decision matrix verbatim. Embed a small sample of the normalized inventory JSON (one VM entry).

- [ ] **Step 2: Validate markdown parses and required sections exist**

```bash
for section in \
  "# " \
  "## Phase 1: Discovery recipes" \
  "## Phase 3: 6Rs decision matrix" \
  "## Phase 4: Wave grouping"
do
  grep -qF "$section" skills/migration-assistant/references/discover-and-plan.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

Expected: `OK`.

- [ ] **Step 3: Commit**

```bash
git add skills/migration-assistant/references/discover-and-plan.md
git commit -m "docs(migration-assistant): add discover-and-plan reference with 6Rs matrix and wave grouping"
```

---

## Task 7: `references/execute-and-verify.md`

**Purpose:** Drive Phases 6, 7, and Post — per-wave cutover runbooks, validation, post-cutover monitoring, rollback, decommission.

**File:** `skills/migration-assistant/references/execute-and-verify.md`

### Required sections

1. **Purpose** (5–10 lines).

2. **Pre-cutover checklist** (30–50 lines) — must include:
   - DNS TTL reduced ≥1 hour before cutover (show `dig TXT` verification command).
   - Load balancer pre-configured with new targets but at 0% weight.
   - Rollback runbook tested end-to-end (dry run).
   - Backup/snapshot of source VMs taken.
   - Application owners acknowledged the cutover window.

3. **Cutover runbook templates — Rehost path** (60–80 lines) — via AWS MGN. Include numbered steps:
   - Install MGN agent on source VM (pre-wave).
   - Verify replication status: `aws mgn describe-source-servers --query ...`.
   - Launch test instance: `aws mgn start-test`.
   - Validate test instance health (see Validation section).
   - Launch cutover instance: `aws mgn start-cutover`.
   - Shift DNS/LB to new target.
   - **Expected output block** for every command.

4. **Cutover runbook templates — Replatform paths** (80–120 lines) — one sub-section per common replatform:
   - **RDS via DMS:** `aws dms create-replication-task` → monitor progress → cutover → `aws dms stop-replication-task`.
   - **EFS via DataSync:** create DataSync task → initial full → incremental → cutover.
   - **Managed AD:** `aws ds create-directory-connector` for migration, then promote.
   - Link to `references/destinations-aws.md` for target-side IaC.

5. **Validation runbook template** (50–80 lines) —
   - ALB target group health: `aws elbv2 describe-target-health`.
   - Application smoke tests: user-supplied endpoints, `curl -sS -o /dev/null -w "%{http_code}"`.
   - Metrics comparison: before/after p95 latency, 5xx rate, CPU.
   - Data parity (for DBs): row counts, checksums.
   - **Pass/fail criteria table** the user fills in.

6. **Post-cutover monitoring setup** (60–80 lines) — baseline:
   - CloudWatch alarms: `CPUUtilization >80% 5m`, `StatusCheckFailed >=1`, custom `mem_used_percent` via CW Agent.
   - RDS: `CPUUtilization`, `DatabaseConnections`, `FreeStorageSpace`, `ReadLatency`/`WriteLatency`.
   - ALB: `TargetResponseTime p99`, `HTTPCode_Target_5XX_Count`.
   - Systems Manager Inventory + Patch Compliance (association templates).
   - AWS Config rules (managed rules list: `required-tags`, `encrypted-volumes`, `ec2-imdsv2-check`).
   - Emit as Terraform module snippets (copy-ready bundle format).

7. **Rollback runbook template** (30–50 lines) — per cutover path:
   - Rehost rollback: revert DNS/LB, keep source VM online for 72h.
   - Replatform rollback: reverse DMS direction, switch reads back to source, halt cutover.
   - **Must be delivered before the cutover runbook per MUST DO rule.**

8. **Gate 6 decommission runbook** (30–50 lines) —
   - 72-hour minimum cooling period enforced (unless user-overridden).
   - Pre-decommission checklist: validation passed, logs archived, snapshot taken, last access >72h.
   - `govc vm.power -off=true` (user executes).
   - `govc vm.destroy` (user executes, per-VM confirmation).

### Steps

- [ ] **Step 1: Write the file per the outline.** Embed exact AWS CLI commands and Terraform snippets where shown.

- [ ] **Step 2: Structural validation**

```bash
for section in \
  "## Pre-cutover checklist" \
  "## Cutover runbook templates" \
  "## Validation runbook" \
  "## Post-cutover monitoring" \
  "## Rollback runbook" \
  "## Gate 6 decommission runbook"
do
  grep -qF "$section" skills/migration-assistant/references/execute-and-verify.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-assistant/references/execute-and-verify.md
git commit -m "docs(migration-assistant): add execute-and-verify reference with cutover/validation/monitoring/decommission runbooks"
```

---

## Task 8: `references/drift-detection.md`

**Purpose:** Cross-cutting — three-way reconciliation algorithm for live vs IaC vs telemetry.

**File:** `skills/migration-assistant/references/drift-detection.md`

### Required content

1. **Purpose** (5–10 lines).

2. **Three-way reconciliation algorithm** (40–60 lines). Embed pseudocode:

   ```
   for each workload W in union(live, iac, telemetry):
     record {
       presence:  in_live?, in_iac?, in_telemetry?
       sizing:    live.cpu, live.mem; iac.cpu, iac.mem; telemetry.p95_cpu, telemetry.p95_mem
       tagging:   live.tags, iac.tags
     }
     classify(W, record) → one of:
       stale_iac                (live ≠ iac in sizing or spec)
       untracked_in_iac         (in_live ∧ ¬in_iac)
       phantom_iac              (in_iac ∧ ¬in_live)
       over_provisioned         (live.cpu > telemetry.p95_cpu * 1.5)
       under_provisioned        (live.cpu == telemetry.p95_cpu, saturated)
       tag_drift                (live.tags ≠ iac.tags)
       ok                       (everything agrees)
   ```

3. **Trust-precedence rules table:**

   | Question | Authoritative source | Fallback | Rationale |
   |----------|----------------------|----------|-----------|
   | Does this resource exist? | live | IaC | IaC can lag deletions |
   | What shape is this resource? | live | IaC | Live is what runs |
   | What size should this resource be in AWS? | telemetry (p95) | live | Telemetry captures actual load |
   | What tags/names should this resource carry? | IaC | live | IaC encodes organizational intent |
   | What is its declared network topology? | IaC | live | Topology is declarative intent |

4. **Drift report output format** (20–40 lines) — show exact JSON schema the skill emits, plus an example human summary.

5. **User-decision rubric** (20–40 lines) — which classifications are **blocking** at Gate 2 vs **informational**:
   - Blocking: `stale_iac` (for replatform workloads), `phantom_iac`, `under_provisioned` (indicates the source is already struggling)
   - Informational: `over_provisioned`, `tag_drift`, `untracked_in_iac` (user decides case-by-case)

### Steps

- [ ] **Step 1: Write the file per the outline.** Embed the pseudocode and table verbatim.

- [ ] **Step 2: Structural validation**

```bash
for section in \
  "## Three-way reconciliation algorithm" \
  "## Trust-precedence rules" \
  "## Drift report output format" \
  "## User-decision rubric"
do
  grep -qF "$section" skills/migration-assistant/references/drift-detection.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-assistant/references/drift-detection.md
git commit -m "docs(migration-assistant): add drift-detection reference with 3-way reconciliation and trust rules"
```

---

## Task 9: `references/sources-vcf-automation.md`

**Purpose:** Parse and convert VCF Automation (Cloud Assembly) YAML templates.

**File:** `skills/migration-assistant/references/sources-vcf-automation.md`

### Required content

1. **Purpose** (5 lines).

2. **VCF Automation template anatomy** (20–30 lines) — `inputs`, `resources`, `outputs`. Show a minimal template example.

3. **Resource-type mapping matrix:**

   | VCF resource type | AWS Terraform equivalent | Notes |
   |-------------------|--------------------------|-------|
   | `Cloud.vSphere.Machine` | `aws_instance` + `aws_ebs_volume` | Map `flavor` via a flavor-table the skill builds from Phase 3 |
   | `Cloud.vSphere.Network` | (composed) `aws_vpc` + `aws_subnet` + `aws_security_group` | One VCF network typically maps to one subnet |
   | `Cloud.vSphere.Disk` | `aws_ebs_volume` + `aws_volume_attachment` | `storage_profile` → `type` (gp3 default) |
   | `Cloud.LoadBalancer` | `aws_lb` + `aws_lb_target_group` + `aws_lb_listener` | Preserve health-check paths |
   | `Cloud.SecurityGroup` | `aws_security_group` | Rules are 1:1 |
   | `Cloud.Volume` (NFS) | `aws_efs_file_system` + `aws_efs_mount_target` | See `destinations-aws.md` for sizing |

4. **Conversion example — VM + Disk + Network** (60–100 lines):
   - Embed a **before** VCF YAML (8-line example — 1 VM with 1 disk on 1 network).
   - Embed the **after** Terraform AWS (as a copy-ready bundle).
   - Annotate the mapping decisions with inline comments.

5. **Conversion example — Load balancer + security group** (40–60 lines) — before/after pair.

6. **Gotchas** (20–40 lines):
   - VCF-specific inputs (e.g., `Cloud.vSphere.Folder`, `Cloud.vSphere.ResourcePool`) — no AWS equivalent; drop or substitute with organizational tags.
   - Custom resource types (`CustomResource` from Aria extensibility) — require manual review; emit a TODO comment in the converted output.
   - `Cloud.Storage.Profile` — translate to EBS `type` + `iops` + `throughput`; see destination reference.

### Steps

- [ ] **Step 1: Write the file per the outline.** Include both conversion examples with complete, valid HCL.

- [ ] **Step 2: Extract HCL examples and validate with `terraform fmt -check`**

```bash
# Extract fenced ```hcl blocks and check they parse
python3 <<'PY'
import re, subprocess, sys, tempfile, os
content = open("skills/migration-assistant/references/sources-vcf-automation.md").read()
blocks = re.findall(r"```hcl\n(.*?)```", content, re.DOTALL)
if not blocks:
    print("No HCL blocks found"); sys.exit(1)
for i, b in enumerate(blocks):
    with tempfile.NamedTemporaryFile("w", suffix=".tf", delete=False) as f:
        # Strip the "# file: ..." header if present so fmt doesn't complain about the comment
        f.write(b)
        p = f.name
    try:
        r = subprocess.run(["terraform", "fmt", "-check", "-diff", p], capture_output=True, text=True)
        if r.returncode != 0:
            print(f"Block {i} failed fmt:\n{r.stdout}\n{r.stderr}"); sys.exit(1)
    finally:
        os.unlink(p)
print(f"OK ({len(blocks)} HCL blocks pass fmt)")
PY
```

If `terraform` isn't installed, skip this step (documented in prereqs).

- [ ] **Step 3: Commit**

```bash
git add skills/migration-assistant/references/sources-vcf-automation.md
git commit -m "docs(migration-assistant): add VCF Automation template conversion reference"
```

---

## Task 10: `references/sources-aria-blueprint.md`

**Purpose:** Convert Aria Automation (vRA 8.x) blueprint YAML to AWS Terraform.

**File:** `skills/migration-assistant/references/sources-aria-blueprint.md`

### Required content

1. **Purpose** (5 lines).
2. **Aria blueprint anatomy** (20 lines) — `inputs`, `resources`, `outputs`, `dependencies`, `property groups`.
3. **Mapping matrix:**

   | Aria construct | AWS Terraform equivalent | Notes |
   |----------------|--------------------------|-------|
   | Property groups (reusable input sets) | Terraform `variable` files + `tfvars` | Group-per-file |
   | `Cloud.Machine` (cloud-agnostic) | `aws_instance` | Pick flavor from 6Rs matrix |
   | Input with `oneOf` enum | Terraform `variable` with `validation` block | Preserve choices |
   | Day-2 action hooks | No direct equivalent — use AWS Systems Manager Automation runbooks or Step Functions | Call out in gotchas |
   | Approval policies | No direct equivalent — use IAM SCPs + CI approval gates | Call out in gotchas |

4. **Conversion example — parameterized VM blueprint** (60–100 lines). Before (Aria YAML with inputs), after (Terraform with variables).
5. **Gotchas** (20–30 lines) — day-2 actions, approval policies, custom forms (no Terraform analog — document for manual follow-up).

### Steps

- [ ] **Step 1: Write per outline.**

- [ ] **Step 2: Structural validation**

```bash
for section in "## Aria blueprint anatomy" "## Mapping matrix" "## Conversion example" "## Gotchas"; do
  grep -qF "$section" skills/migration-assistant/references/sources-aria-blueprint.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit** (HCL fmt check for all reference files is consolidated in Task 14)

```bash
git add skills/migration-assistant/references/sources-aria-blueprint.md
git commit -m "docs(migration-assistant): add Aria Automation blueprint conversion reference"
```

---

## Task 11: `references/sources-terraform-vsphere.md`

**Purpose:** Convert `hashicorp/vsphere` provider code to `hashicorp/aws` provider code.

**File:** `skills/migration-assistant/references/sources-terraform-vsphere.md`

### Required content

1. **Purpose** (5 lines).
2. **Provider swap overview** (20 lines) — why "new TF state for AWS side" is strongly preferred over in-place provider swap (prevents destructive plan when old vsphere resources become unmanaged).

3. **Resource-type mapping matrix:**

   | vSphere resource | AWS equivalent | Notes |
   |------------------|----------------|-------|
   | `vsphere_virtual_machine` | `aws_instance` + `aws_ebs_volume` | One VM → one instance; attached disks → EBS volumes |
   | `vsphere_distributed_virtual_switch` | `aws_vpc` | High-level network boundary |
   | `vsphere_distributed_port_group` | `aws_subnet` + `aws_security_group` | Port group ≈ subnet; port-group security policy ≈ SG |
   | `vsphere_tag_category` + `vsphere_tag` | Tag key + value as resource `tags` map | Not a separate resource in AWS |
   | `vsphere_content_library` / `vsphere_content_library_item` | `aws_s3_bucket` + AMIs | Images live as AMIs; ancillary files in S3 |
   | `vsphere_folder` | (no equivalent) | Use resource tags or AWS Organizations OU |
   | `vsphere_host` / `vsphere_compute_cluster` | (no equivalent) | AWS manages capacity; drop |
   | `vsphere_datastore_cluster` | `aws_ebs_volume` type selection | Reflects storage profile, not cluster |
   | `vsphere_resource_pool` | Not applicable (remove); optionally represent as EC2 capacity reservation for critical workloads | |
   | `vsphere_nas_datastore` | `aws_efs_file_system` + mounts | NFS → EFS |
   | `vsphere_custom_attribute` | Tags | Same as categories |

4. **Conversion example — Complete VM** (~80 lines) — show a full `vsphere_virtual_machine` block and the equivalent `aws_instance` + `aws_ebs_volume` + `aws_volume_attachment` trio. Include outputs.

5. **Conversion example — Port-group + firewall** (~40 lines) — `vsphere_distributed_port_group` → `aws_subnet` + `aws_security_group`.

6. **State migration guidance** (30–50 lines):
   - Why prefer new TF state: avoids the destructive plan when vsphere provider drops resources.
   - Bootstrap snippet for new AWS module.
   - How to retire the old vsphere state (post-decommission): `terraform state rm` for each vsphere resource, then delete the old module directory.

7. **Provider replacement snippet** (~15 lines) — the new `required_providers` block for the AWS side.

8. **Gotchas** (20–40 lines):
   - vSphere has `wait_for_guest_ip_timeout` — AWS has no analog; document as "delete".
   - Disk ordering in AWS requires explicit `aws_volume_attachment` resources (device_name matters).
   - Nested ESXi / custom CPU flags — no AWS equivalent.

### Steps

- [ ] **Step 1: Write per outline.** The VM conversion example is the centerpiece — make it complete and copy-paste runnable.

- [ ] **Step 2: Commit** (HCL fmt check for all reference files is consolidated in Task 14)

```bash
git add skills/migration-assistant/references/sources-terraform-vsphere.md
git commit -m "docs(migration-assistant): add Terraform vSphere provider conversion reference"
```

---

## Task 12: `references/sources-ovf.md`

**Purpose:** Convert OVF/OVA descriptors to AMIs via AWS VM Import/Export.

**File:** `skills/migration-assistant/references/sources-ovf.md`

### Required content

1. **Purpose** (5 lines).
2. **OVF descriptor structure** (20 lines) — key sections: `VirtualSystem`, `DiskSection`, `NetworkSection`, `OperatingSystemSection`.
3. **Conversion command sequence** (30–50 lines):
   - Upload OVA to S3: `aws s3 cp <file>.ova s3://<bucket>/`
   - Create IAM role for VM Import (show trust policy + permissions policy as JSON blocks).
   - Run import: `aws ec2 import-image --description "<desc>" --disk-containers "<container-json>"`
   - Poll: `aws ec2 describe-import-image-tasks --import-task-ids <id>` — show the "Completed" signal.
   - Resulting AMI is referenced in the Terraform output; launch via `aws_instance`.
4. **Supported-OS matrix** (25–40 lines) — link to AWS docs, list top OSes with support status (Windows 2012/2016/2019/2022, RHEL 7+, Ubuntu 18+, CentOS 7+, etc.).
5. **Unsupported OS workarounds** (15–25 lines) — use AWS MGN instead (agent-based, broader OS support). Reference the MGN section in `execute-and-verify.md`.
6. **Gotchas** (15–30 lines):
   - VM Import does NOT support VMs with >1 network interface in one call.
   - UEFI boot requires specific flags.
   - VMs with encrypted disks fail — decrypt first.

### Steps

- [ ] **Step 1: Write per outline.** Embed exact `aws ec2 import-image` command with full `--disk-containers` JSON.

- [ ] **Step 2: Structural validation**

```bash
for section in "## OVF descriptor structure" "## Conversion command sequence" "## Supported-OS matrix" "## Unsupported OS workarounds" "## Gotchas"; do
  grep -qF "$section" skills/migration-assistant/references/sources-ovf.md \
    || { echo "MISSING: $section"; exit 1; }
done
echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add skills/migration-assistant/references/sources-ovf.md
git commit -m "docs(migration-assistant): add OVF/OVA conversion reference via AWS VM Import/Export"
```

---

## Task 13: `references/destinations-aws.md`

**Purpose:** AWS target selection matrix — which AWS service for which workload archetype.

**File:** `skills/migration-assistant/references/destinations-aws.md`

### Required content

1. **Purpose** (5 lines).

2. **Rehost target selection — EC2 instance families** (50–80 lines):
   - Compute-optimized (`c7i`/`c7a`): stateless web, batch compute.
   - Memory-optimized (`r7i`/`r7a`): SQL databases when rehosted.
   - General-purpose (`m7i`/`m7a`): mixed workloads.
   - Storage-optimized (`i4i`/`im4gn`): workloads with heavy local disk IOPS (Cassandra, etc.).
   - GPU (`g5`/`g6`): GPU workloads.
   - **Right-sizing formula:** `target_vcpu = ceil(p95_cpu_usage_percent * source_vcpu / 70)`; `target_mem_mb = max(p95_mem_mb * 1.2, source_mem_mb * 0.8)` — i.e. provision for p95 + 30% headroom, never drop below 80% of current source (to avoid right-sizing too aggressively).

3. **Replatform targets — Databases** (40–60 lines):
   - RDS engines (MySQL, PostgreSQL, MariaDB, SQL Server, Oracle) — show `aws_db_instance` Terraform snippet skeleton.
   - Aurora (MySQL/PostgreSQL) — show `aws_rds_cluster` + `aws_rds_cluster_instance` snippet.
   - DocumentDB for MongoDB-compatible workloads.
   - OpenSearch for Elasticsearch.
   - **Web-search anchor:** "Before committing to an engine version, search for `RDS <engine> <version> support status <region>` — deprecation notices move faster than training data."

4. **Replatform targets — Storage** (40–60 lines):
   - EFS for POSIX NFS.
   - FSx for Windows (SMB), NetApp ONTAP, OpenZFS, Lustre.
   - S3 for object/archive workloads.
   - Terraform snippet skeleton for each.

5. **Replatform targets — Identity / Messaging / Other** (40–60 lines):
   - Managed AD: `aws_directory_service_directory`.
   - IAM Identity Center: link to `aws_ssoadmin_*` resources.
   - MQ for ActiveMQ/RabbitMQ; MSK for Kafka; SQS/SNS for bespoke messaging.
   - Terraform snippets for each.

6. **Networking patterns** (50–80 lines):
   - VPC design: 3-tier or 2-tier subnet layout, multi-AZ.
   - Transit Gateway for multi-VPC + on-prem Direct Connect attachments.
   - Site-to-Site VPN for cutover connectivity to on-prem vCenter.
   - Route 53 Resolver for hybrid DNS.
   - Copy-ready VPC module snippet.

7. **Monitoring baseline** (20–40 lines) — CloudWatch, CW Agent, AWS Config. Reference `execute-and-verify.md` for complete alarm set.

8. **IaC conventions** (15–25 lines):
   - Prefer `hashicorp/aws` ≥ v5.x.
   - Module structure: `modules/ec2/`, `modules/rds/`, `modules/network/`.
   - State backend: S3 + DynamoDB locking (show a standard `terraform` block).
   - Tagging defaults via `default_tags`.

9. **Web-search anchors** (10–20 lines, scattered) — explicit reminders embedded at the replatform matrices telling the agent to search before finalizing:
   - RDS engine versions and regional availability.
   - EC2 instance-type quotas per region.
   - FSx deployment options (current GA features change).

### Steps

- [ ] **Step 1: Write per outline.** Each replatform section gets at least one complete copy-ready Terraform snippet.

- [ ] **Step 2: Commit** (HCL fmt check for all reference files is consolidated in Task 14)

```bash
git add skills/migration-assistant/references/destinations-aws.md
git commit -m "docs(migration-assistant): add AWS destinations reference with service matrix and networking patterns"
```

---

## Task 14: Final validation — full CI simulation + example fmt check

**Purpose:** Before opening the PR, run the full validation that CI will run, plus the optional `terraform fmt` checks across all reference files.

**Files:** None to create; validation only.

### Steps

- [ ] **Step 1: Run full CI validator locally** (mirrors `.github/workflows/validate.yml`)

```bash
node --input-type=module -e '
import { readdirSync, readFileSync } from "fs";
import { join } from "path";
import { parse as parseYaml } from "yaml";

const SKILLS_DIR = "skills";
const NAME_RE = /^[a-z0-9][a-z0-9-]{0,63}$/;
const MAX_LINES = 500;
let errors = 0;
function fail(s, m){ console.error(`❌ ${s}: ${m}`); errors++; }

const dirs = readdirSync(SKILLS_DIR, { withFileTypes: true })
  .filter(d => d.isDirectory()).map(d => d.name);

for (const dir of dirs) {
  if (!NAME_RE.test(dir)) fail(dir, "bad name");
  const p = join(SKILLS_DIR, dir, "SKILL.md");
  let content;
  try { content = readFileSync(p, "utf-8"); }
  catch { fail(dir, "missing SKILL.md"); continue; }
  const lines = content.split("\n").length;
  if (lines > MAX_LINES) fail(dir, `${lines} lines > ${MAX_LINES}`);
  const m = content.match(/^---\n([\s\S]*?)\n---/);
  if (!m) { fail(dir, "no frontmatter"); continue; }
  let fm; try { fm = parseYaml(m[1]); } catch(e){ fail(dir, "bad yaml"); continue; }
  if (!fm.name) fail(dir, "no name");
  if (!fm.description) fail(dir, "no description");
  if (fm.name && fm.name !== dir) fail(dir, `name/dir mismatch`);
}
if (errors) { console.error(`\n${errors} error(s)`); process.exit(1); }
console.log(`✅ ${dirs.length} skills validated`);
'
```

Expected: `✅ 17 skills validated` (16 existing + migration-assistant).

- [ ] **Step 2: Run all script tests**

```bash
python3 skills/migration-assistant/scripts/test_parse_rvtools.py -v
python3 skills/migration-assistant/scripts/test_probe_vcenter.py -v
```

Expected: all tests PASS.

- [ ] **Step 3: Validate every reference file's sections exist**

```bash
set -e
SKILL=skills/migration-assistant
declare -A req=(
  [references/discover-and-plan.md]="## Phase 1: Discovery recipes|## Phase 3: 6Rs decision matrix|## Phase 4: Wave grouping"
  [references/execute-and-verify.md]="## Pre-cutover checklist|## Cutover runbook templates|## Validation runbook|## Post-cutover monitoring|## Rollback runbook|## Gate 6 decommission runbook"
  [references/drift-detection.md]="## Three-way reconciliation algorithm|## Trust-precedence rules|## Drift report output format|## User-decision rubric"
  [references/sources-vcf-automation.md]="## Resource-type mapping matrix|## Conversion example|## Gotchas"
  [references/sources-aria-blueprint.md]="## Mapping matrix|## Conversion example|## Gotchas"
  [references/sources-terraform-vsphere.md]="## Resource-type mapping matrix|## Conversion example|## State migration guidance|## Gotchas"
  [references/sources-ovf.md]="## OVF descriptor structure|## Conversion command sequence|## Supported-OS matrix|## Gotchas"
  [references/destinations-aws.md]="## Rehost target selection|## Replatform targets|## Networking patterns|## Monitoring baseline|## IaC conventions"
)
for f in "${!req[@]}"; do
  IFS='|' read -ra sections <<< "${req[$f]}"
  for s in "${sections[@]}"; do
    grep -qF "$s" "$SKILL/$f" || { echo "❌ $f missing: $s"; exit 1; }
  done
  echo "✅ $f"
done
echo "All reference sections present."
```

- [ ] **Step 4: (Optional) Run HCL fmt across every reference file that contains HCL examples**

```bash
if command -v terraform >/dev/null 2>&1; then
  python3 <<'PY'
import re, subprocess, tempfile, os, glob, sys
failed = 0
for f in glob.glob("skills/migration-assistant/references/*.md"):
    content = open(f).read()
    blocks = re.findall(r"```hcl\n(.*?)```", content, re.DOTALL)
    for i, b in enumerate(blocks):
        with tempfile.NamedTemporaryFile("w", suffix=".tf", delete=False) as tf:
            tf.write(b); p = tf.name
        try:
            r = subprocess.run(["terraform","fmt","-check",p], capture_output=True, text=True)
            if r.returncode != 0:
                print(f"❌ {f} block {i} fails fmt"); failed += 1
        finally:
            os.unlink(p)
print(f"HCL fmt: {failed} failures" if failed else "✅ All HCL examples fmt-clean")
sys.exit(1 if failed else 0)
PY
else
  echo "terraform CLI not installed — skipping HCL fmt check"
fi
```

- [ ] **Step 5: Full-skill line budget summary**

```bash
wc -l skills/migration-assistant/SKILL.md skills/migration-assistant/references/*.md skills/migration-assistant/scripts/*.py skills/migration-assistant/scripts/*.sh
```

Expected: `SKILL.md` ≤ 500 lines; total skill content ≥ 2000 lines across references.

- [ ] **Step 6: Commit nothing** — this task is validation only. If any step fails, go back to the offending task to fix, then re-run Task 14.

---

## Task 15: Open pull request

**Purpose:** Push the branch and open the PR for review.

### Steps

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feat/migration-assistant
```

- [ ] **Step 2: Open PR using `gh`**

```bash
gh pr create --title "feat: migration-assistant skill (VMware → AWS)" --body "$(cat <<'EOF'
## Summary

- Adds `skills/migration-assistant/` — an end-to-end VMware → AWS migration executor for Falcon.
- Orchestrator `SKILL.md` drives a 7-phase workflow with 6 explicit user-approval gates at every mutation boundary.
- 8 reference files loaded on demand: discovery + 6Rs + wave grouping, drift detection, 4 source-format converters (VCF Automation, Aria, Terraform vSphere, OVF), AWS destinations, execute/verify/monitor/decommission.
- 2 read-only helper scripts (`probe-vcenter.sh`, `parse-rvtools.py`) sharing a common JSON output schema (`scripts/schema.json`).
- Falcon is read-only: the skill recommends commands and copy-ready IaC bundles; the user always executes.

**Spec:** `docs/superpowers/specs/2026-04-23-migration-assistant-design.md`
**Plan:** `docs/superpowers/plans/2026-04-23-migration-assistant.md`

## Test plan

- [ ] CI `validate.yml` passes on this branch.
- [ ] Local script tests pass: `python3 skills/migration-assistant/scripts/test_*.py`.
- [ ] `SKILL.md` ≤ 500 lines.
- [ ] Manual walkthrough in Falcon: skill activates on VMware → AWS questions, runs Phase 0 probe, stops at Gate 1.
- [ ] Sample conversion from a VCF Automation template produces Terraform that passes `terraform validate` when dropped into an empty module.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 3: Record the PR URL in the plan**

The PR URL is printed by `gh pr create`. Paste it into the PR field below for future reference.

**PR URL:** _(paste after creating)_

---

## Appendix: Spec coverage audit

For each spec section, the task(s) that implement it:

| Spec section | Implemented by |
|--------------|----------------|
| Architecture / directory layout | Task 1 |
| Frontmatter | Task 1, refined in Task 5 |
| SKILL.md orchestrator: Phase 0 probe | Task 5 |
| SKILL.md: Phase 0.5 resume protocol | Task 5 |
| SKILL.md: Phases 1–7 brief + decision gates | Task 5 |
| SKILL.md: Reference Guide table | Task 5 |
| SKILL.md: Scripts pointer | Task 5 |
| SKILL.md: Output conventions | Task 5 |
| SKILL.md: MUST DO / MUST NOT DO | Task 5 |
| `discover-and-plan.md` | Task 6 |
| `execute-and-verify.md` | Task 7 |
| `drift-detection.md` | Task 8 |
| `sources-vcf-automation.md` | Task 9 |
| `sources-aria-blueprint.md` | Task 10 |
| `sources-terraform-vsphere.md` | Task 11 |
| `sources-ovf.md` | Task 12 |
| `destinations-aws.md` | Task 13 |
| `scripts/schema.json` (JSON Schema v1) | Task 2 |
| `scripts/fixtures/` | Task 2 |
| `scripts/probe-vcenter.sh` + TDD | Task 4 |
| `scripts/parse-rvtools.py` + TDD | Task 3 |
| Success criteria: CI validation | Task 14 (manual run), CI on push |
| Success criteria: scripts schema conformance | Tasks 3, 4 (automated in unit tests) |
| Success criteria: HCL examples pass fmt | Task 14 (optional step) |
| Success criteria: manual Falcon walkthrough | Post-merge, not gated in plan |

**Deferred (per spec "Open items"):**
- Install spec in metadata for `govc` / Python deps — v2.
- `user-invocable` flag — keeping default (true).
- Additional telemetry sources beyond vROps/Aria/Prometheus/Datadog/NewRelic — add when an engagement surfaces them.
- `openpyxl` install hint — Task 3 embeds this as a script-level `ImportError` message with pip install instructions.
