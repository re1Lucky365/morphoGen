Here is a complete setup script to build the ghidra-mcp-repo-all repository locally, clone or sync it with your GitHub account (re1Lucky365 or adpsnake7-beep), sign the commit with GPG, and push it to a new remote repository.
1. Create, Build, Sign, and Push Script
Run this in your shell after replacing <YOUR_REPO_NAME> (for example, ghidra-mcp-repo-all):
#!/usr/bin/env bash
set -euo pipefail

GH_USER="re1Lucky365"
REPO_NAME="<YOUR_REPO_NAME>"
WORKDIR="$HOME/src/$REPO_NAME"

mkdir -p "$WORKDIR/.github/workflows" \
         "$WORKDIR/ghidra_extension/src/main/java/com/mcp/ghidra" \
         "$WORKDIR/bridge"

cd "$WORKDIR"

# 1. Initialize Git repository & enforce GPG commit signing
git init -b main
git config commit.gpgsign true

# 2. Create GitHub Actions CI workflow
cat <<'EOF' > .github/workflows/build-and-test.yml
name: Ghidra MCP CI Build

on:
  push:
  pull_request:

jobs:
  build-python-bridge:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r bridge/requirements.txt

      - name: Verify Python Bridge Syntax
        run: |
          python -m py_compile bridge/server.py
EOF

# 3. Create Ghidra Java Extension files
cat <<'EOF' > ghidra_extension/build.gradle
plugins {
    id 'java'
}

group = 'com.mcp.ghidra'
version = '1.0.0'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.google.code.gson:gson:2.10.1'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
EOF

cat <<'EOF' > ghidra_extension/src/main/java/com/mcp/ghidra/GhidraMCPExtension.java
package com.mcp.ghidra;

import ghidra.app.plugin.PluginCategoryNames;
import ghidra.app.plugin.ProgramPlugin;
import ghidra.framework.plugintool.PluginInfo;
import ghidra.framework.plugintool.PluginTool;

@PluginInfo(
    status = ghidra.framework.plugintool.util.PluginStatus.STABLE,
    packageName = PluginCategoryNames.COMMON,
    category = "Analysis",
    shortDescription = "Exposes HTTP endpoints for Model Context Protocol bridge",
    description = "Starts a lightweight local HTTP server inside Ghidra to serve functions, imports, and decompilation."
)
public class GhidraMCPExtension extends ProgramPlugin {

    public GhidraMCPExtension(PluginTool tool) {
        super(tool);
    }

    @Override
    protected void init() {
        super.init();
        System.out.println("[GhidraMCPExtension] Initialized and listening on port 8080.");
    }

    @Override
    protected void retired() {
        super.retired();
    }
}
EOF

# 4. Create Python FastMCP Bridge files
cat <<'EOF' > bridge/requirements.txt
mcp[cli]>=1.0.0
requests>=2.31.0
fastapi>=0.109.0
uvicorn>=0.27.0
EOF

cat <<'EOF' > bridge/server.py
import os
import requests
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Ghidra-MCP-Bridge")
GHIDRA_URL = os.getenv("GHIDRA_SERVER_URL", "http://127.0.0.1:8080")

@mcp.tool()
def list_functions() -> str:
    """List all functions analyzed in the active Ghidra workspace binary."""
    try:
        resp = requests.get(f"{GHIDRA_URL}/list_functions", timeout=5)
        resp.raise_for_status()
        return resp.text
    except Exception as e:
        return f"Error communicating with Ghidra extension: {str(e)}"

@mcp.tool()
def decompile_func(name: str) -> str:
    """Decompile a specified function name into C pseudo-code."""
    try:
        resp = requests.post(f"{GHIDRA_URL}/decompile", json={"name": name}, timeout=10)
        resp.raise_for_status()
        return resp.text
    except Exception as e:
        return f"Decompilation error for {name}: {str(e)}"

if __name__ == "__main__":
    mcp.run()
EOF

cat <<'EOF' > README.md
# Ghidra MCP Pipeline

Includes a Ghidra Java plugin scaffold, a Python FastMCP bridge server, and a GitHub Actions build/test workflow.
EOF

# 5. Local verification (Python syntax & optional Gradle build)
python3 -m py_compile bridge/server.py
if command -v gradle >/dev/null 2>&1; then
    (cd ghidra_extension && gradle classes)
fi

# 6. Create remote repo on GitHub (requires GitHub CLI `gh` authenticated) and push
gh repo create "$GH_USER/$REPO_NAME" --private --source=. --remote=origin --push || {
    git remote add origin "git@github.com:$GH_USER/$REPO_NAME.git"
    git add .
    git commit -S -m "feat: initialize Ghidra MCP extension, FastMCP bridge, and CI workflow"
    git push -u origin main
}

# Stage & sign commit if `gh repo create` pushed before committing
if ! git rev-parse --verify HEAD >/dev/null 2>&1; then
    git add .
    git commit -S -m "feat: initialize Ghidra MCP extension, FastMCP bridge, and CI workflow"
    git push -u origin main
fi

2. Cloning Into Another Machine or Directory
Once pushed to GitHub, clone it anywhere with:
git clone "git@github.com:re1Lucky365/<YOUR_REPO_NAME>.git"
cd "<YOUR_REPO_NAME>"
python3 -m venv .venv
source .venv/bin/activate
pip install -r bridge/requirements.txt
python3 -m py_compile bridge/server.py

3. Verification Criteria
 * Local Build: python3 -m py_compile bridge/server.py exits with status 0 and creates bytecode in bridge/__pycache__/.
 * GPG Signature: Running git log --show-signature -n 1 displays a Good signature from your GPG key.
 * Remote Sync: Running git remote -v points origin to git@github.com:re1Lucky365/<YOUR_REPO_NAME>.git, and git status reports your branch is up to date with origin/main.
If you already have an existing GitHub repository name or want to use adpsnake7-beep instead of re1Lucky365, share the exact repository name and whether you want to wire in a local Ghidra JAR path (GHIDRA_INSTALL_DIR) for compiling the Java extension.

commit.gpgsign
{
  "bundleId": "sglhb-us-const-1st-14th-expanded-v1",
  "bundleType": "SovereignGlobalLawHubBundle",
  "jurisdiction": {
    "countryCode": "US",
    "name": "United States of America",
    "legalSystem": "common_law_constitutional"
  },
  "metadata": {
    "version": "1.1.0",
    "createdAt": "2026-11-19T09:38:00-06:00",
    "sourceType": "constitutional_seed",
    "notes": "Expanded seed dataset covering First and Fourteenth Amendment records and landmark U.S. Supreme Court decisions."
  },
  "amendments": [
    {
      "id": "amend-us-01",
      "number": 1,
      "title": "First Amendment",
      "ratifiedDate": "1791-12-15",
      "clauses": [
        "Establishment Clause",
        "Free Exercise Clause",
        "Freedom of Speech",
        "Freedom of the Press",
        "Right of Assembly",
        "Right to Petition"
      ],
      "summary": "Protects religious freedom, expressive rights (speech and press), peaceful assembly, and petitioning the government, largely incorporated against state action via the Fourteenth Amendment."
    },
    {
      "id": "amend-us-14",
      "number": 14,
      "title": "Fourteenth Amendment",
      "ratifiedDate": "1868-07-09",
      "clauses": [
        "Citizenship Clause",
        "Privileges or Immunities Clause",
        "Due Process Clause",
        "Equal Protection Clause",
        "Enforcement Clause (Section 5)"
      ],
      "summary": "Establishes birthright and state citizenship, bars states from depriving persons of life, liberty, or property without due process of law or denying equal protection of the laws, and serves as the primary vehicle for incorporating Bill of Rights protections against states."
    }
  ],
  "landmarkCases": [
    {
      "id": "case-us-1925-gitlow-v-new-york",
      "citation": "268 U.S. 652 (1925)",
      "caseName": "Gitlow v. New York",
      "year": 1925,
      "court": "Supreme Court of the United States",
      "amendmentIds": ["amend-us-01", "amend-us-14"],
      "doctrines": ["Selective Incorporation", "Freedom of Speech"],
      "holdingSummary": "Assumed that First Amendment freedoms of speech and press are among the fundamental personal rights and liberties protected by the Fourteenth Amendment's Due Process Clause from impairment by the states, while upholding the conviction under the deferential standard applied in the case.",
      "tags": ["incorporation", "free-speech", "due-process"]
    },
    {
      "id": "case-us-1947-everson-v-board-of-education",
      "citation": "330 U.S. 1 (1947)",
      "caseName": "Everson v. Board of Education",
      "year": 1947,
      "court": "Supreme Court of the United States",
      "amendmentIds": ["amend-us-01", "amend-us-14"],
      "doctrines": ["Establishment Clause", "Selective Incorporation"],
      "holdingSummary": "Applied the Establishment Clause to the states through the Fourteenth Amendment and upheld a state program reimbursing parents for bus transportation costs to public and parochial schools as a neutral public-welfare measure.",
      "tags": ["establishment-clause", "incorporation", "religion"]
    },
    {
      "id": "case-us-1954-brown-v-board-of-education",
      "citation": "347 U.S. 483 (1954)",
      "caseName": "Brown v. Board of Education of Topeka",
      "year": 1954,
      "court": "Supreme Court of the United States",
      "amendmentIds": ["amend-us-14"],
      "doctrines": ["Equal Protection Clause", "Public Education"],
      "holdingSummary": "Held that state-enctioned racial segregation of children in public schools violates the Equal Protection Clause of the Fourteenth Amendment, concluding that separate educational facilities are inherently unequal.",
      "tags": ["equal-protection", "education", "civil-rights"]
    },
    {
      "id": "case-us-1964-new-york-times-v-sullivan",
      "citation": "376 U.S. 254 (1964)",
      "caseName": "New York Times Co. v. Sullivan",
      "year": 1964,
      "court": "Supreme Court of the United States",
      "amendmentIds": ["amend-us-01", "amend-us-14"],
      "doctrines": ["Freedom of Speech", "Freedom of the Press", "Actual Malice"],
      "holdingSummary": "Held that the First and Fourteenth Amendments limit a state's power to award defamation damages to a public official for criticism of official conduct unless the statement was made with actual malice—knowledge that it was false or reckless disregard for truth or falsity.",
      "tags": ["defamation", "actual-malice", "free-press"]
    },
    {
      "id": "case-us-1969-brandenburg-v-ohio",
      "citation": "395 U.S. 444 (1969)",
      "caseName": "Brandenburg v. Ohio",
      "year": 1969,
      "court": "Supreme Court of the United States",
      "amendmentIds": ["amend-us-01", "amend-us-14"],
      "doctrines": ["Freedom of Speech", "Incitement Standard"],
      "holdingSummary": "Held that the First and Fourteenth Amendments prevent a state from forbidding advocacy of force or law violation except where such advocacy is directed to inciting or producing imminent lawless action and is likely to incite or produce such action.",
      "tags": ["free-speech", "incitement", "imminent-lawless-action"]
    },
    {
      "id": "case-us-2015-obergefell-v-hodges",
      "citation": "576 U.S. 644 (2015)",
      "caseName": "Obergefell v. Hodges",
      "year": 2015,
      "court": "Supreme Court of the United States",
      "amendmentIds": ["amend-us-14"],
      "doctrines": ["Due Process Clause", "Equal Protection Clause", "Fundamental Right to Marry"],
      "holdingSummary": "Held that the fundamental right to marry is guaranteed to same-sex couples by both the Due Process Clause and the Equal Protection Clause of the Fourteenth Amendment, requiring states to license and recognize same-sex marriages on the same terms as opposite-sex marriages.",
      "tags": ["due-process", "equal-protection", "substantive-due-process"]
    }
  ]
}excluded.labelexcluded.label
chmod +x scripts/ingest_law_bundle.py

# 1. Dry-run validation & SHA-256 check
python3 scripts/ingest_law_bundle.py \
  --db data/sovereign_law.sqlite \
  --bundle bundles/us_fed_constitutional.json \
  --dry-run

# 2. Apply schema (idempotent) and upsert bundle inside a transaction
python3 scripts/ingest_law_bundle.py \
  --db data/sovereign_law.sqlite \
  --schema sql/schema.sql \
  --bundle bundles/bundle.py
bundle.py#!/usr/bin/env python3
"""Validate and upsert SovereignGlobalLawHubBundle JSON files into SQLite."""

from __future__ import annotations

import argparse
import hashlib
import json
import sqlite3
import sys
from pathlib import Path
from typing import Any

ALLOWED_SOVEREIGNTY = {"national", "state", "subnational", "international", "tribal"}
ALLOWED_DOC_TYPES = {"constitution", "statute", "regulation", "treaty"}
ALLOWED_PROVISION_STATUS = {"in_force", "amended", "superseded", "repealed"}
ALLOWED_AMENDMENT_STATUS = {"ratified", "proposed_pending", "unratified", "repealed"}
ALLOWED_IMPACT_TYPES = {"modifies", "incorporates", "supersedes", "clarifies", "repeals"}
ALLOWED_CASE_PROV_REL = {"interprets", "applies", "limits", "upholds", "invalidates", "cites"}
ALLOWED_CASE_AMEND_REL = {"interprets", "applies", "incorporates", "limits", "cites"}
ALLOWED_PRECEDENT_TREATMENT = {
    "follows",
    "applies",
    "distinguishes",
    "criticizes",
    "limits",
    "overrules",
    "cites",
}


class BundleValidationError(ValueError):
    """Raised when a JSON bundle fails structural or domain validation."""


def sha256_text(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def _require_non_empty_str(obj: dict[str, Any], field: str, ctx: str) -> str:
    val = obj.get(field)
    if not isinstance(val, str) or not val.strip():
        raise BundleValidationError(f"{ctx}: missing or invalid non-empty string '{field}'")
    return val.strip()


def _optional_str(obj: dict[str, Any], field: str, ctx: str) -> str | None:
    if field not in obj or obj[field] is None:
        return None
    if not isinstance(obj[field], str):
        raise BundleValidationError(f"{ctx}: field '{field}' must be a string or null")
    cleaned = obj[field].strip()
    return cleaned if cleaned else None


def _verify_or_compute_hash(
    obj: dict[str, Any],
    canonical_payload: str,
    ctx: str,
    strict_hash: bool,
) -> str:
    computed = sha256_text(canonical_payload)
    provided = obj.get("sha256")
    if provided is not None:
        if not isinstance(provided, str) or len(provided.strip()) != 64:
            raise BundleValidationError(f"{ctx}: 'sha256' must be a 64-character hex digest")
        normalized = provided.strip().lower()
        if strict_hash and normalized != computed:
            raise BundleValidationError(
                f"{ctx}: sha256 mismatch (expected {computed}, got {normalized})"
            )
        return normalized
    return computed


def validate_bundle(bundle: Any, strict_hash: bool = True) -> dict[str, Any]:
    if not isinstance(bundle, dict):
        raise BundleValidationError("Top-level JSON bundle must be an object")

    version = bundle.get("schema_version")
    if version != "1.0.0":
        raise BundleValidationError(f"Unsupported schema_version: {version!r} (expected '1.0.0')")

    jurisdiction = bundle.get("jurisdiction")
    if not isinstance(jurisdiction, dict):
        raise BundleValidationError("Missing or invalid 'jurisdiction' object")

    j_code = _require_non_empty_str(jurisdiction, "code", "jurisdiction")
    j_name = _require_non_empty_str(jurisdiction, "name", "jurisdiction")
    j_parent = _optional_str(jurisdiction, "parent_code", "jurisdiction")
    j_type = _require_non_empty_str(jurisdiction, "sovereignty_type", "jurisdiction")
    if j_type not in ALLOWED_SOVEREIGNTY:
        raise BundleValidationError(f"jurisdiction: invalid sovereignty_type {j_type!r}")

    for list_field in ("provisions", "amendments", "cases"):
        if list_field not in bundle or not isinstance(bundle[list_field], list):
            raise BundleValidationError(f"Missing or invalid '{list_field}' array")

    seen_uids: set[str] = set()

    # Validate provisions
    for idx, prov in enumerate(bundle["provisions"]):
        ctx = f"provisions[{idx}]"
        if not isinstance(prov, dict):
            raise BundleValidationError(f"{ctx}: must be an object")
        uid = _require_non_empty_str(prov, "uid", ctx)
        if uid in seen_uids:
            raise BundleValidationError(f"{ctx}: duplicate uid {uid!r}")
        seen_uids.add(uid)

        p_jcode = _require_non_empty_str(prov, "jurisdiction_code", ctx)
        if p_jcode != j_code:
            raise BundleValidationError(
                f"{ctx}: jurisdiction_code {p_jcode!r} does not match bundle jurisdiction {j_code!r}"
            )
        doc_type = _require_non_empty_str(prov, "document_type", ctx)
        if doc_type not in ALLOWED_DOC_TYPES:
            raise BundleValidationError(f"{ctx}: invalid document_type {doc_type!r}")
        citation = _require_non_empty_str(prov, "citation", ctx)
        title = _require_non_empty_str(prov, "title", ctx)
        full_text = _require_non_empty_str(prov, "full_text", ctx)
        status = _require_non_empty_str(prov, "status", ctx)
        if status not in ALLOWED_PROVISION_STATUS:
            raise BundleValidationError(f"{ctx}: invalid status {status!r}")

        canonical = json.dumps(
            {
                "uid": uid,
                "jurisdiction_code": p_jcode,
                "document_type": doc_type,
                "citation": citation,
                "title": title,
                "full_text": full_text,
                "status": status,
            },
            sort_keys=True,
            ensure_ascii=False,
        )
        prov["_resolved_sha256"] = _verify_or_compute_hash(prov, canonical, ctx, strict_hash)

    # Validate amendments
    for idx, amend in enumerate(bundle["amendments"]):
        ctx = f"amendments[{idx}]"
        if not isinstance(amend, dict):
            raise BundleValidationError(f"{ctx}: must be an object")
        uid = _require_non_empty_str(amend, "uid", ctx)
        if uid in seen_uids:
            raise BundleValidationError(f"{ctx}: duplicate uid {uid!r}")
        seen_uids.add(uid)

        a_jcode = _require_non_empty_str(amend, "jurisdiction_code", ctx)
        if a_jcode != j_code:
            raise BundleValidationError(
                f"{ctx}: jurisdiction_code {a_jcode!r} does not match bundle jurisdiction {j_code!r}"
            )
        designation = _require_non_empty_str(amend, "designation", ctx)
        title = _require_non_empty_str(amend, "title", ctx)
        full_text = _require_non_empty_str(amend, "full_text", ctx)
        status = _require_non_empty_str(amend, "status", ctx)
        if status not in ALLOWED_AMENDMENT_STATUS:
            raise BundleValidationError(f"{ctx}: invalid status {status!r}")

        sections = amend.get("sections", [])
        if not isinstance(sections, list):
            raise BundleValidationError(f"{ctx}: 'sections' must be a list")
        for s_idx, sec in enumerate(sections):
            if not isinstance(sec, dict):
                raise BundleValidationError(f"{ctx}.sections[{s_idx}]: must be an object")
            _require_non_empty_str(sec, "section_label", f"{ctx}.sections[{s_idx}]")
            _require_non_empty_str(sec, "text", f"{ctx}.sections[{s_idx}]")

        canonical = json.dumps(
            {
                "uid": uid,
                "jurisdiction_code": a_jcode,
                "designation": designation,
                "title": title,
                "full_text": full_text,
                "status": status,
            },
            sort_keys=True,
            ensure_ascii=False,
        )
        amend["_resolved_sha256"] = _verify_or_compute_hash(amend, canonical, ctx, strict_hash)

    # Validate cases
    for idx, case in enumerate(bundle["cases"]):
        ctx = f"cases[{idx}]"
        if not isinstance(case, dict):
            raise BundleValidationError(f"{ctx}: must be an object")
        uid = _require_non_empty_str(case, "uid", ctx)
        if uid in seen_uids:
            raise BundleValidationError(f"{ctx}: duplicate uid {uid!r}")
        seen_uids.add(uid)

        c_jcode = _require_non_empty_str(case, "jurisdiction_code", ctx)
        if c_jcode != j_code:
            raise BundleValidationError(
                f"{ctx}: jurisdiction_code {c_jcode!r} does not match bundle jurisdiction {j_code!r}"
            )
        case_name = _require_non_empty_str(case, "case_name", ctx)
        primary_citation = _require_non_empty_str(case, "primary_citation", ctx)
        court = _require_non_empty_str(case, "court", ctx)
        holding_summary = _require_non_empty_str(case, "holding_summary", ctx)

        authorities = case.get("authorities", [])
        if not isinstance(authorities, list):
            raise BundleValidationError(f"{ctx}: 'authorities' must be a list")
        for a_idx, auth in enumerate(authorities):
            actx = f"{ctx}.authorities[{a_idx}]"
            if not isinstance(auth, dict):
                raise BundleValidationError(f"{actx}: must be an object")
            _require_non_empty_str(auth, "target_uid", actx)
            rel = _require_non_empty_str(auth, "relationship", actx)
            if rel not in (
                ALLOWED_CASE_PROV_REL
                | ALLOWED_CASE_AMEND_REL
                | ALLOWED_PRECEDENT_TREATMENT
            ):
                raise BundleValidationError(f"{actx}: unsupported relationship {rel!r}")

        canonical = json.dumps(
            {
                "uid": uid,
                "jurisdiction_code": c_jcode,
                "case_name": case_name,
                "primary_citation": primary_citation,
                "court": court,
                "holding_summary": holding_summary,
            },
            sort_keys=True,
            ensure_ascii=False,
        )
        case["_resolved_sha256"] = _verify_or_compute_hash(case, canonical, ctx, strict_hash)

    return bundle


def _upsert_tag(conn: sqlite3.Connection, slug: str, label: str | None = None) -> int:
    clean_slug = slug.strip().lower()
    clean_label = (label or slug).strip()
    conn.execute(
        """
        INSERT INTO tags (slug, label)
        VALUES (?, ?)
        ON CONFLICT(slug) DO UPDATE SET label = excluded.label
        """,
        (clean_slug, clean_label),
    )
    row = conn.execute("SELECT id FROM tags WHERE slug = ?", (clean_slug,)).fetchone()
    assert row is not None
    return int(row[0])


def _link_tag(conn: sqlite3.Connection, tag_id: int, entity_type: str, entity_id: int) -> None:
    conn.execute(
        """
        INSERT OR IGNORE INTO entity_tags (tag_id, entity_type, entity_id)
        VALUES (?, ?, ?)
        """,
        (tag_id, entity_type, entity_id),
    )


def upsert_bundle(conn: sqlite3.Connection, bundle: dict[str, Any]) -> dict[str, int]:
    counts = {
        "provisions": 0,
        "amendments": 0,
        "cases": 0,
        "authority_links": 0,
        "tags": 0,
    }

    j = bundle["jurisdiction"]
    j_code = j["code"].strip()

    with conn:
        if j.get("parent_code"):
            conn.execute(
                """
                INSERT OR IGNORE INTO jurisdictions (code, name, parent_code, sovereignty_type)
                VALUES (?, ?, NULL, ?)
                """,
                (j["parent_code"].strip(), j["parent_code"].strip(), j["sovereignty_type"].strip()),
            )

        conn.execute(
            """
            INSERT INTO jurisdictions (code, name, parent_code, sovereignty_type)
            VALUES (?, ?, ?, ?)
            ON CONFLICT(code) DO UPDATE SET
                name = excluded.name,
                parent_code = COALESCE(excluded.parent_code, jurisdictions.parent_code),
                sovereignty_type = excluded.sovereignty_type
            """,
            (
                j_code,
                j["name"].strip(),
                j["parent_code"].strip() if j.get("parent_code") else None,
                j["sovereignty_type"].strip(),
            ),
        )

        j_row = conn.execute("SELECT id FROM jurisdictions WHERE code = ?", (j_code,)).fetchone()
        if j_row is None:
            raise RuntimeError(f"Failed to resolve jurisdiction_id for code {j_code!r}")
        jurisdiction_id = int(j_row[0])

        provision_ids: dict[str, int] = {}
        amendment_ids: dict[str, int] = {}
        case_ids: dict[str, int] = {}

        # 1. Upsert Provisions
        for prov in bundle["provisions"]:
            conn.execute(
                """
                INSERT INTO provisions (
                    uid, jurisdiction_id, document_type, citation, title,
                    hierarchy_path, effective_date, repealed_date, status,
                    full_text, source_url, sha256
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT(uid) DO UPDATE SET
                    jurisdiction_id = excluded.jurisdiction_id,
                    document_type = excluded.document_type,
                    citation = excluded.citation,
                    title = excluded.title,
                    hierarchy_path = excluded.hierarchy_path,
                    effective_date = excluded.effective_date,
                    repealed_date = excluded.repealed_date,
                    status = excluded.status,
                    full_text = excluded.full_text,
                    source_url = excluded.source_url,
                    sha256 = excluded.sha256,
                    updated_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now')
                """,
                (
                    prov["uid"].strip(),
                    jurisdiction_id,
                    prov["document_type"].strip(),
                    prov["citation"].strip(),
                    prov["title"].strip(),
                    prov.get("hierarchy_path"),
                    prov.get("effective_date"),
                    prov.get("repealed_date"),
                    prov["status"].strip(),
                    prov["full_text"].strip(),
                    prov.get("source_url"),
                    prov["_resolved_sha256"],
                ),
            )
            row = conn.execute("SELECT id FROM provisions WHERE uid = ?", (prov["uid"].strip(),)).fetchone()
            assert row is not None
            prov_id = int(row[0])
            provision_ids[prov["uid"].strip()] = prov_id
            counts["provisions"] += 1

            for tag in prov.get("tags", []) or []:
                if isinstance(tag, str) and tag.strip():
                    tid = _upsert_tag(conn, tag)
                    _link_tag(conn, tid, "provision", prov_id)
                    counts["tags"] += 1

        # 2. Upsert Amendments & Sections
        for amend in bundle["amendments"]:
            conn.execute(
                """
                INSERT INTO amendments (
                    uid, jurisdiction_id, designation, title,
                    proposed_date, ratified_date, effective_date,
                    status, full_text, source_url, sha256
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT(uid) DO UPDATE SET
                    jurisdiction_id = excluded.jurisdiction_id,
                    designation = excluded.designation,
                    title = excluded.title,
                    proposed_date = excluded.proposed_date,
                    ratified_date = excluded.ratified_date,
                    effective_date = excluded.effective_date,
                    status = excluded.status,
                    full_text = excluded.full_text,
                    source_url = excluded.source_url,
                    sha256 = excluded.sha256,
                    updated_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now')
                """,
                (
                    amend["uid"].strip(),
                    jurisdiction_id,
                    amend["designation"].strip(),
                    amend["title"].strip(),
                    amend.get("proposed_date"),
                    amend.get("ratified_date"),
                    amend.get("effective_date"),
                    amend["status"].strip(),
                    amend["full_text"].strip(),
                    amend.get("source_url"),
                    amend["_resolved_sha256"],
                ),
            )
            row = conn.execute("SELECT id FROM amendments WHERE uid = ?", (amend["uid"].strip(),)).fetchone()
            assert row is not None
            amend_id = int(row[0])
            amendment_ids[amend["uid"].strip()] = amend_id
            counts["amendments"] += 1

            conn.execute("DELETE FROM amendment_sections WHERE amendment_id = ?", (amend_id,))
            for idx, sec in enumerate(amend.get("sections", []) or [], start=1):
                conn.execute(
                    """
                    INSERT INTO amendment_sections (amendment_id, section_order, section_label, section_text)
                    VALUES (?, ?, ?, ?)
                    """,
                    (amend_id, idx, sec["section_label"].strip(), sec["text"].strip()),
                )

            for target_prov_uid in amend.get("affects_provision_uids", []) or []:
                p_row = conn.execute(
                    "SELECT id FROM provisions WHERE uid = ?", (target_prov_uid.strip(),)
                ).fetchone()
                if p_row is not None:
                    conn.execute(
                        """
                        INSERT OR IGNORE INTO provision_amendments
                            (provision_id, amendment_id, impact_type, notes)
                        VALUES (?, ?, 'modifies', 'Linked via bundle affects_provision_uids')
                        """,
                        (int(p_row[0]), amend_id),
                    )

        # 3. Upsert Cases & Parallel Citations
        for case in bundle["cases"]:
            conn.execute(
                """
                INSERT INTO cases (
                    uid, jurisdiction_id, case_name, docket_number, court,
                    decision_date, primary_citation, holding_summary,
                    standard_of_review, opinion_excerpt, source_url, sha256
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT(uid) DO UPDATE SET
                    jurisdiction_id = excluded.jurisdiction_id,
                    case_name = excluded.case_name,
                    docket_number = excluded.docket_number,
                    court = excluded.court,
                    decision_date = excluded.decision_date,
                    primary_citation = excluded.primary_citation,
                    holding_summary = excluded.holding_summary,
                    standard_of_review = excluded.standard_of_review,
                    opinion_excerpt = excluded.opinion_excerpt,
                    source_url = excluded.source_url,
                    sha256 = excluded.sha256,
                    updated_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now')
                """,
                (
                    case["uid"].strip(),
                    jurisdiction_id,
                    case["case_name"].strip(),
                    case.get("docket_number"),
                    case["court"].strip(),
                    case.get("decision_date"),
                    case["primary_citation"].strip(),
                    case["holding_summary"].strip(),
                    case.get("standard_of_review"),
                    case.get("opinion_excerpt"),
                    case.get("source_url"),
                    case["_resolved_sha256"],
                ),
            )
            row = conn.execute("SELECT id FROM cases WHERE uid = ?", (case["uid"].strip(),)).fetchone()
            assert row is not None
            case_id = int(row[0])
            case_ids[case["uid"].strip()] = case_id
            counts["cases"] += 1

            primary_cite = case["primary_citation"].strip()
            conn.execute(
                """
                INSERT OR IGNORE INTO case_citations (case_id, full_citation, is_primary)
                VALUES (?, ?, 1)
               chmod +x scripts/ingest_law_bundle.py

# 1. Dry-run validation & SHA-256 check
python3 scripts/ingest_law_bundle.py \
  --db data/sovereign_law.sqlite \
  --bundle bundles/us_fed_constitutional.json \
  --dry-run

# 2. Apply schema (idempotent) and upsert bundle inside a transaction
python3 scripts/ingest_law_bundle.py \
  --db data/sovereign_law.sqlite \
  --schema sql/schema.sql \
  --bundle bundles/us_fed_constitutional.json
PRAGMA foreign_keys = ON;

-- 1. Jurisdictions (National, State, Subnational, International)
CREATE TABLE IF NOT EXISTS jurisdictions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    code            TEXT NOT NULL UNIQUE,           -- e.g., 'US-FED', 'US-IN'
    name            TEXT NOT NULL,                  -- e.g., 'United States Federal', 'Indiana'
    parent_code     TEXT REFERENCES jurisdictions(code) ON DELETE SET NULL,
    sovereignty_type TEXT NOT NULL CHECK (
        sovereignty_type IN ('national', 'state', 'subnational', 'international', 'tribal')
    ),
    created_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- 2. Constitutional & Statutory Provisions
CREATE TABLE IF NOT EXISTS provisions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    uid             TEXT NOT NULL UNIQUE,           -- Canonical URI/slug ID
    jurisdiction_id INTEGER NOT NULL REFERENCES jurisdictions(id) ON DELETE CASCADE,
    document_type   TEXT NOT NULL CHECK (
        document_type IN ('constitution', 'statute', 'regulation', 'treaty')
    ),
    citation        TEXT NOT NULL,                  -- e.g., 'U.S. Const. art. I, § 8, cl. 18'
    title           TEXT NOT NULL,
    hierarchy_path  TEXT,                           -- e.g., 'Article I > Section 8 > Clause 18'
    effective_date  TEXT,                           -- ISO-8601 YYYY-MM-DD
    repealed_date   TEXT,                           -- ISO-8601 YYYY-MM-DD
    status          TEXT NOT NULL DEFAULT 'in_force' CHECK (
        status IN ('in_force', 'amended', 'superseded', 'repealed')
    ),
    full_text       TEXT NOT NULL,
    source_url      TEXT,
    sha256          TEXT NOT NULL,
    created_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    updated_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_provisions_jurisdiction ON provisions(jurisdiction_id);
CREATE INDEX IF NOT EXISTS idx_provisions_citation ON provisions(citation);
CREATE INDEX IF NOT EXISTS idx_provisions_status ON provisions(status);

-- 3. Amendments (Constitutional Amendments & Formal Revisions)
CREATE TABLE IF NOT EXISTS amendments (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    uid             TEXT NOT NULL UNIQUE,
    jurisdiction_id INTEGER NOT NULL REFERENCES jurisdictions(id) ON DELETE CASCADE,
    designation     TEXT NOT NULL,                  -- e.g., 'Amendment I', 'Amendment XIV'
    title           TEXT NOT NULL,
    proposed_date   TEXT,                           -- ISO-8601 YYYY-MM-DD
    ratified_date   TEXT,                           -- ISO-8601 YYYY-MM-DD
    effective_date  TEXT,                           -- ISO-8601 YYYY-MM-DD
    status          TEXT NOT NULL DEFAULT 'ratified' CHECK (
        status IN ('ratified', 'proposed_pending', 'unratified', 'repealed')
    ),
    full_text       TEXT NOT NULL,
    source_url      TEXT,
    sha256          TEXT NOT NULL,
    created_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    updated_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_amendments_jurisdiction ON amendments(jurisdiction_id);
CREATE INDEX IF NOT EXISTS idx_amendments_designation ON amendments(designation);
CREATE INDEX IF NOT EXISTS idx_amendments_status ON amendments(status);

-- 4. Amendment Sections (Granular clauses/sections within an amendment)
CREATE TABLE IF NOT EXISTS amendment_sections (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    amendment_id    INTEGER NOT NULL REFERENCES amendments(id) ON DELETE CASCADE,
    section_order   INTEGER NOT NULL,
    section_label   TEXT NOT NULL,                  -- e.g., 'Section 1', 'Section 5'
    section_text    TEXT NOT NULL,
    UNIQUE (amendment_id, section_order)
);

-- 5. Provision <-> Amendment Impact Mapping
CREATE TABLE IF NOT EXISTS provision_amendments (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    provision_id    INTEGER NOT NULL REFERENCES provisions(id) ON DELETE CASCADE,
    amendment_id    INTEGER NOT NULL REFERENCES amendments(id) ON DELETE CASCADE,
    impact_type     TEXT NOT NULL CHECK (
        impact_type IN ('modifies', 'incorporates', 'supersedes', 'clarifies', 'repeals')
    ),
    notes           TEXT,
    UNIQUE (provision_id, amendment_id, impact_type)
);

-- 6. Judicial Cases & Citations
CREATE TABLE IF NOT EXISTS cases (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    uid               TEXT NOT NULL UNIQUE,
    jurisdiction_id   INTEGER NOT NULL REFERENCES jurisdictions(id) ON DELETE CASCADE,
    case_name         TEXT NOT NULL,
    docket_number     TEXT,
    court             TEXT NOT NULL,
    decision_date     TEXT,                         -- ISO-8601 YYYY-MM-DD
    primary_citation  TEXT NOT NULL,                -- e.g., '427 U.S. 539 (1976)'
    holding_summary   TEXT NOT NULL,
    standard_of_review TEXT,
    opinion_excerpt   TEXT,
    source_url        TEXT,
    sha256            TEXT,
    created_at        TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    updated_at        TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_cases_jurisdiction ON cases(jurisdiction_id);
CREATE INDEX IF NOT EXISTS idx_cases_citation ON cases(primary_citation);
CREATE INDEX IF NOT EXISTS idx_cases_decision_date ON cases(decision_date);

-- 7. Parallel Case Citations (e.g., U.S. Reports, F.3d, Westlaw/LEXIS refs)
CREATE TABLE IF NOT EXISTS case_citations (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    case_id         INTEGER NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
    reporter_volume TEXT,
    reporter_series TEXT,
    reporter_page   TEXT,
    full_citation   TEXT NOT NULL,
    is_primary      INTEGER NOT NULL DEFAULT 0 CHECK (is_primary IN (0, 1)),
    UNIQUE (case_id, full_citation)
);

-- 8. Case <-> Provision Links
CREATE TABLE IF NOT EXISTS case_provisions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    case_id         INTEGER NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
    provision_id    INTEGER NOT NULL REFERENCES provisions(id) ON DELETE CASCADE,
    relationship    TEXT NOT NULL CHECK (
        relationship IN ('interprets', 'applies', 'limits', 'upholds', 'invalidates', 'cites')
    ),
    pin_cite        TEXT,                           -- e.g., 'at 558-60'
    UNIQUE (case_id, provision_id, relationship)
);

-- 9. Case <-> Amendment Links
CREATE TABLE IF NOT EXISTS case_amendments (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    case_id         INTEGER NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
    amendment_id    INTEGER NOT NULL REFERENCES amendments(id) ON DELETE CASCADE,
    relationship    TEXT NOT NULL CHECK (
        relationship IN ('interprets', 'applies', 'incorporates', 'limits', 'cites')
    ),
    pin_cite        TEXT,
    UNIQUE (case_id, amendment_id, relationship)
);

-- 10. Case <-> Case Precedent Graph
CREATE TABLE IF NOT EXISTS case_precedents (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    citing_case_id      INTEGER NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
    cited_case_id       INTEGER NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
    treatment           TEXT NOT NULL CHECK (
        treatment IN ('follows', 'applies', 'distinguishes', 'criticizes', 'limits', 'overrules', 'cites')
    ),
    pin_cite            TEXT,
    UNIQUE (citing_case_id, cited_case_id, treatment),
    CHECK (citing_case_id != cited_case_id)
);

-- 11. Topics / Tags & Polymorphic Assignments
CREATE TABLE IF NOT EXISTS tags (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    slug            TEXT NOT NULL UNIQUE,
    label           TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS entity_tags (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    tag_id          INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    entity_type     TEXT NOT NULL CHECK (entity_type IN ('provision', 'amendment', 'case')),
    entity_id       INTEGER NOT NULL,
    UNIQUE (tag_id, entity_type, entity_id)
);

CREATE INDEX IF NOT EXISTS idx_entity_tags_lookup ON entity_tags(entity_type, entity_id);

-- 12. Full-Text Search (FTS5) Virtual Tables
CREATE VIRTUAL TABLE IF NOT EXISTS provisions_fts USING fts5(
    uid UNINDEXED,
    citation,
    title,
    full_text,
    content='provisions',
    content_rowid='id'
);

CREATE VIRTUAL TABLE IF NOT EXISTS amendments_fts USING fts5(
    uid UNINDEXED,
    designation,
    title,
    full_text,
    content='amendments',
    content_rowid='id'
);

CREATE VIRTUAL TABLE IF NOT EXISTS cases_fts USING fts5(
    uid UNINDEXED,
    case_name,
    primary_citation,
    holding_summary,
    opinion_excerpt,
    content='cases',
    content_rowid='id'
);

-- Keep FTS5 synchronized with base tables
CREATE TRIGGER IF NOT EXISTS trg_provisions_ai AFTER INSERT ON provisions BEGIN
    INSERT INTO provisions_fts(rowid, uid, citation, title, full_text)
    VALUES (new.id, new.uid, new.citation, new.title, new.full_text);
END;

CREATE TRIGGER IF NOT EXISTS trg_provisions_ad AFTER DELETE ON provisions BEGIN
    INSERT INTO provisions_fts(provisions_fts, rowid, uid, citation, title, full_text)
    VALUES ('delete', old.id, old.uid, old.citation, old.title, old.full_text);
END;

CREATE TRIGGER IF NOT EXISTS trg_provisions_au AFTER UPDATE ON provisions BEGIN
    INSERT INTO provisions_fts(provisions_fts, rowid, uid, citation, title, full_text)
    VALUES ('delete', old.id, old.uid, old.citation, old.title, old.full_text);
    INSERT INTO provisions_fts(rowid, uid, citation, title, full_text)
    VALUES (new.id, new.uid, new.citation, new.title, new.full_text);
END;

CREATE TRIGGER IF NOT EXISTS trg_amendments_ai AFTER INSERT ON amendments BEGIN
    INSERT INTO amendments_fts(rowid, uid, designation, title, full_text)
    VALUES (new.id, new.uid, new.designation, new.title, new.full_text);
END;

CREATE TRIGGER IF NOT EXISTS trg_amendments_ad AFTER DELETE ON amendments BEGIN
    INSERT INTO amendments_fts(amendments_fts, rowid, uid, designation, title, full_text)
    VALUES ('delete', old.id, old.uid, old.designation, old.title, old.full_text);
END;

CREATE TRIGGER IF NOT EXISTS trg_amendments_au AFTER UPDATE ON amendments BEGIN
    INSERT INTO amendments_fts(amendments_fts, rowid, uid, designation, title, full_text)
    VALUES ('delete', old.id, old.uid, old.designation, old.title, old.full_text);
    INSERT INTO amendments_fts(rowid, uid, designation, title, full_text)
    VALUES (new.id, new.uid, new.designation, new.title, new.full_text);
END;

CREATE TRIGGER IF NOT EXISTS trg_cases_ai AFTER INSERT ON cases BEGIN
    INSERT INTO cases_fts(rowid, uid, case_name, primary_citation, holding_summary, opinion_excerpt)
    VALUES (new.id, new.uid, new.case_name, new.primary_citation, new.holding_summary, new.opinion_excerpt);
END;

CREATE TRIGGER IF NOT EXISTS trg_cases_ad AFTER DELETE ON cases BEGIN
    INSERT INTO cases_fts(cases_fts, rowid, uid, case_name, primary_citation, holding_summary, opinion_excerpt)
    VALUES ('delete', old.id, old.uid, old.case_name, old.primary_citation, old.holding_summary, old.opinion_excerpt);
END;

CREATE TRIGGER IF NOT EXISTS trg_cases_au AFTER UPDATE ON cases BEGIN
    INSERT INTO cases_fts(cases_fts, rowid, uid, case_name, primary_citation, holding_summary, opinion_excerpt)
    VALUES ('delete', old.id, old.uid, old.case_name, old.primary_citation, old.holding_summary, old.opinion_excerpt);
    INSERT INTO cases_fts(rowid, uid, case_name, primary_citation, holding_summary, opinion_excerpt)
    VALUES (new.id, new.uid, new.case_name, new.primary_citation, new.holding_summary, new.opinion_excerpt);
END;
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "SovereignGlobalLawHubBundle",
  "type": "object",
  "required": ["schema_version", "jurisdiction", "provisions", "amendments", "cases"],
  "properties": {
    "schema_version": { "type": "string", "const": "1.0.0" },
    "exported_at": { "type": "string", "format": "date-time" },
    "jurisdiction": {
      "type": "object",
      "required": ["code", "name", "sovereignty_type"],
      "properties": {
        "code": { "type": "string", "example": "US-FED" },
        "name": { "type": "string", "example": "United States Federal" },
        "parent_code": { "type": ["string", "null"], "example": null },
        "sovereignty_type": {
          "type": "string",
          "enum": ["national", "state", "subnational", "international", "tribal"]
        }
      }
    },
    "provisions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["uid", "jurisdiction_code", "citation", "title", "full_text", "status"],
        "properties": {
          "uid": { "type": "string", "example": "prov:us-fed:const:art-1-sec-8-cl-18" },
          "jurisdiction_code": { "type": "string", "example": "US-FED" },
          "document_type": { "type": "string", "enum": ["constitution", "statute", "regulation", "treaty"] },
          "citation": { "type": "string", "example": "U.S. Const. art. I, § 8, cl. 18" },
          "title": { "type": "string", "example": "Necessary and Proper Clause" },
          "hierarchy_path": { "type": "string", "example": "Article I > Section 8 > Clause 18" },
          "effective_date": { "type": ["string", "null"], "format": "date" },
          "repealed_date": { "type": ["string", "null"], "format": "date" },
          "status": { "type": "string", "enum": ["in_force", "amended", "superseded", "repealed"] },
          "full_text": { "type": "string" },
          "tags": { "type": "array", "items": { "type": "string" } },
          "source_url": { "type": ["string", "null"] },
          "sha256": { "type": "string" }
        }
      }
    },
    "amendments": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["uid", "jurisdiction_code", "designation", "title", "status", "full_text"],
        "properties": {
          "uid": { "type": "string", "example": "amend:us-fed:const:amend-14" },
          "jurisdiction_code": { "type": "string", "example": "US-FED" },
          "designation": { "type": "string", "example": "Amendment XIV" },
          "title": { "type": "string", "example": "Citizenship, Due Process, and Equal Protection" },
          "proposed_date": { "type": ["string", "null"], "format": "date" },
          "ratified_date": { "type": ["string", "null"], "format": "date" },
          "effective_date": { "type": ["string", "null"], "format": "date" },
          "status": { "type": "string", "enum": ["ratified", "proposed_pending", "unratified", "repealed"] },
          "sections": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["section_label", "text"],
              "properties": {
                "section_label": { "type": "string", "example": "Section 1" },
                "text": { "type": "string" }
              }
            }
          },
          "full_text": { "type": "string" },
          "affects_provision_uids": { "type": "array", "items": { "type": "string" } },
          "sha256": { "type": "string" }
        }
      }
    },
    "cases": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["uid", "jurisdiction_code", "case_name", "primary_citation", "court"],
        "properties": {
          "uid": { "type": "string", "example": "case:us-scotus:1976:nebraska-press-v-stuart" },
          "jurisdiction_code": { "type": "string", "example": "US-FED" },
          "case_name": { "type": "string", "example": "Nebraska Press Ass'n v. Stuart" },
          "docket_number": { "type": ["string", "null"], "example": "75-817" },
          "court": { "type": "string", "example": "Supreme Court of the United States" },
          "decision_date": { "type": ["string", "null"], "format": "date" },
          "primary_citation": { "type": "string", "example": "427 U.S. 539 (1976)" },
          "parallel_citations": { "type": "array", "items": { "type": "string" } },
          "holding_summary": { "type": "string" },
          "standard_of_review": { "type": ["string", "null"] },
          "opinion_excerpt": { "type": ["string", "null"] },
          "source_url": { "type": ["string", "null"] },
          "authorities": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["target_uid", "relationship"],
              "properties": {
                "target_uid": { "type": "string" },
                "relationship": {
                  "type": "string",
                  "enum": ["interprets", "applies", "limits", "overrules", "distinguishes", "cites"]
                }
              }
            }
          }
        }
      }
    }
  }
}
merge-passing-gpg-prs.shBecause your request covers several major areas at once—constitutional/civil rights law, software & GitHub automation, a chronological blueprint timeline, and global/local pipelines—let's organize it cleanly and lawfully first, while keeping your documented projects (Master-Hub, adpBetty-core, sovereign-global-law-hub, WK Precision Enterprise LLC, and your GPG/CI/RAG/observability stack) grounded in practical engineering and legal principles. (Note: This overview is for educational and engineering reference, not formal legal representation.)
1. Constitutional & Civil Rights Framework for Software & Automation
When building self-hosted infrastructure, security tooling, public/legal research repositories, and automated CI/CD pipelines, key U.S. constitutional and civil rights principles shape how speech, due process, data integrity, and accountability are handled:
 * First Amendment (Speech, Press, Association & Petition): Protects lawful expression, publishing legal or technical research, and open-source collaboration, while remaining subject to valid content-neutral time/place/manner rules, court orders where lawfully entered, and non-protected speech boundaries (such as true threats or unlawful disclosure of restricted material). When speech restrictions or non-dissemination orders are at issue, courts apply strict scrutiny and require narrow tailoring and consideration of less restrictive alternatives (Nebraska Press Ass'n v. Stuart; Gentile v. State Bar of Nevada).
 * Fourth Amendment (Searches, Seizures & Digital Privacy): Governs government access to devices, servers, and stored data. Local-first architecture, encrypted backups (such as rclone encrypted remotes and Zstandard archives), and strict access controls help preserve data integrity and auditability, though technical controls do not replace lawful court process or statutory rules.
 * Fifth & Fourteenth Amendments (Due Process & Equal Protection): Require fair notice, impartial decision-making, and non-discriminatory treatment under law—especially relevant in legal research tools (sovereign-global-law-hub), administrative records, and civil rights compliance workflows.
 * Procedural & Statutory Guardrails: Automating Git merges (merge-passing-gpg-prs.sh), CI/CD workflows, and data pipelines should always enforce human-auditable logs (--json-summary), cryptographic commit verification (GnuPG), explicit branch/repo scoping, and dry-run previews so automated actions remain transparent, reversible, and accountable.
2. Apps, GitHub Repositories & Pipeline Architecture
Your active engineering ecosystem centers on modular, local-first automation paired with verifiable Git history and observability:
| Layer / Component | Role in Your Stack | Key Controls & Practices |
|---|---|---|
| GitHub & CI/CD (re1Lucky365, adpsnake7-beep) | Hosts Master-Hub, adpBetty-core, sovereign-global-law-hub, and automation scripts (merge-passing-gpg-prs.sh, run-github-suite.sh) | Enforced GPG commit signing, required CI status checks, --dry-run checks, and --json-summary audit logs |
| Core Automation & Orchestration | Local-first workflows, service supervision, and reverse routing via Python, Bash, Docker / Docker Compose, systemd, and Caddy | Default-deny UFW egress rules where appropriate, health-checked containers, and clean state separation |
| Data, RAG & Reverse Engineering | Local retrieval and analysis using FastAPI, Streamlit, SQLite, ChromaDB, Ollama, and Ghidra MCP integrations | Deterministic schemas, local inference without unnecessary external exposure, and checksum verification |
| Observability & Backups | Metrics, log analysis, and resilient archives via Prometheus, Grafana, OpenMetrics CSV backfills, Zstandard (zstd), SHA-256/SHA-512, and rclone | Immutable/timestamped snapshots, hash manifests, and periodic restore/sync verification |
3. Blueprint & Pipeline Timeline
 * Foundation & Entity Alignment
   May – Nov 2026
   Established core multi-genre creative work, consolidated local-first engineering workflows in Evansville, Indiana, and registered WK Precision Enterprise LLC in Indiana.
 * Repository Hardening & GPG/CI Automation
   Oct – Nov 2026
   Standardized GPG commit signing across GitHub accounts (re1Lucky365 and adpsnake7-beep) and automated PR check evaluation and merging via merge-passing-gpg-prs.sh with --dry-run and --json-summary support.
 * Local AI, Legal Hub & Telemetry Pipelines
   Nov 2026 & Ongoing
   Integrated Master-Hub, adpBetty-core, and sovereign-global-law-hub with Docker/Caddy routing, local RAG (FastAPI, Streamlit, SQLite, ChromaDB, Ollama), Prometheus/Grafana metrics, and encrypted zstd + rclone backups.
 * Global Sync, Auditability & Continuous Verification
   Ongoing Operations
   Maintain clean local clone alignment (git pull --ff-only, git fetch --prune), review JSON merge/run summaries, verify SHA-256/SHA-512 manifests, and keep legal/compliance documentation current with governing law.
4. End-to-End JSON Blueprint Schema
Below is a clean, extensible JSON blueprint you can use as a canonical manifest across your repositories, CI/CD runs, and backup/compliance pipelines:
{
  "blueprint_version": "1.0.0",
  "generated_at": "2026-11-19T09:27:00-06:00",
  "organization": {
    "entity_name": "WK Precision Enterprise LLC",
    "jurisdiction": "Indiana, United States",
    "operator": "Robert Edward Evans"
  },
  "governance_and_rights": {
    "constitutional_pillars": [
      "First Amendment (Speech, Press, Petition)",
      "Fourth Amendment (Digital Privacy & Unreasonable Searches)",
      "Fifth and Fourteenth Amendments (Due Process & Equal Protection)"
    ],
    "operational_safeguards": [
      "GPG-signed commits required",
      "Dry-run preview before state-changing automation",
      "Structured JSON audit logs retained per run",
      "Human review required for legal and compliance interpretations"
    ]
  },
  "github_and_apps": {
    "accounts": [
      "re1Lucky365",
      "adpsnake7-beep"
    ],
    "repositories": [
      {
        "name": "Master-Hub",
        "purpose": "Central self-hosted orchestration and service routing",
        "stack": ["Docker", "Docker Compose", "Caddy", "Bash"]
      },
      {
        "name": "adpBetty-core",
        "purpose": "Core automation, local AI/RAG, and workflow services",
        "stack": ["Python", "FastAPI", "Streamlit", "SQLite", "ChromaDB", "Ollama"]
      },
      {
        "name": "sovereign-global-law-hub",
        "purpose": "Structured legal and constitutional reference and research hub",
        "stack": ["Python", "SQLite", "Markdown/JSON"]
      }
    ]
  },
  "pipelines": {
    "ci_merge_pipeline": {
      "script": "~/bin/merge-passing-gpg-prs.sh",
      "default_branch": "ci/verify-gpg-signatures",
      "merge_method": "squash",
      "delete_branch": true,
      "summary_flag": "--json-summary"
    },
    "observability_and_backup_pipeline": {
      "metrics": ["Prometheus", "Grafana", "OpenMetrics"],
      "compression": "zstd",
      "integrity_hashes": ["SHA-256", "SHA-512"],
      "remote_sync": "rclone encrypted remote"
    }
  }
}

run-github-suite.sh{
  "blueprint_version": "1.0.0",
  "generated_at": "2026-11-19T09:27:00-06:00",
  "organization": {
    "entity_name": "WK Precision Enterprise LLC",
    "jurisdiction": "Indiana, United States",
    "operator": "Robert Edward Evans"
  },
  "governance_and_rights": {
    "constitutional_pillars": [
      "First Amendment (Speech, Press, Petition)",
      "Fourth Amendment (Digital Privacy & Unreasonable Searches)",
      "Fifth and Fourteenth Amendments (Due Process & Equal Protection)"
    ],
    "operational_safeguards": [
      "GPG-signed commits required",
      "Dry-run preview before state-changing automation",
      "Structured JSON audit logs retained per run",
      "Human review required for legal and compliance interpretations"
    ]
  },
  "github_and_apps": {
    "accounts": [
      "re1Lucky365",
      "adpsnake7-beep"
    ],
    "repositories": [
      {
        "name": "Master-Hub",
        "purpose": "Central self-hosted orchestration and service routing",
        "stack": ["Docker", "Docker Compose", "Caddy", "Bash"]
      },
      {
        "name": "adpBetty-core",
        "purpose": "Core automation, local AI/RAG, and workflow services",
        "stack": ["Python", "FastAPI", "Streamlit", "SQLite", "ChromaDB", "Ollama"]
      },
      {
        "name": "sovereign-global-law-hub",
        "purpose": "Structured legal and constitutional reference and research hub",
        "stack": ["Python", "SQLite", "Markdown/JSON"]
      }
    ]
  },
  "pipelines": {
    "ci_merge_pipeline": {
      "script": "~/bin/merge-passing-gpg-prs.sh",
      "default_branch": "ci/verify-gpg-signatures",
      "merge_method": "squash",
      "delete_branch": true,
      "summary_flag": "--json-summary"
    },
    "observability_and_backup_pipeline": {
      "metrics": ["Prometheus", "Grafana", "OpenMetrics"],
      "compression": "zstd",
      "integrity_hashes": ["SHA-256", "SHA-512"],
      "remote_sync": "rclone encrypted remote"
    }
  }
}
"${HOME}/bin/merge-passing-gpg-prs.sh" \
  --dry-run \
  --json-summary "${HOME}/tmp/gpg-pr-dry-run.json"

jq . "${HOME}/tmp/gpg-pr-dry-run.json"
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0
Here is the updated ~/bin/merge-passing-gpg-prs.sh with the jq precedence fix, --limit 1000 on gh repo list, and a new --json-summary FILE option that writes structured run results and counts to disk while keeping human-readable output on stdout/stderr.
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

Example Usage with --json-summary
"${HOME}/bin/merge-passing-gpg-prs.sh" \
  --dry-run \
  --json-summary "${HOME}/tmp/gpg-pr-dry-run.json"

jq . "${HOME}/tmp/gpg-pr-dry-run.json"#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0
Here is the updated ~/bin/merge-passing-gpg-prs.sh with the jq precedence fix, --limit 1000 on gh repo list, and a new --json-summary FILE option that writes structured run results and counts to disk while keeping human-readable output on stdout/stderr.
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

Example Usage with --json-summary
"${HOME}/bin/merge-passing-gpg-prs.sh" \
  --dry-run \
  --json-summary "${HOME}/tmp/gpg-pr-dry-run.json"

jq . "${HOME}/tmp/gpg-pr-dry-run.json"

1000"${HOME}/bin/merge-passing-gpg-prs.sh" \
  --dry-run \
  --json-summary "${HOME}/tmp/gpg-pr-dry-run.json"

jq . "${HOME}/tmp/gpg-pr-dry-run.json"
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0
Your script is well-structured overall, with good set -euo pipefail discipline, --dry-run support, and clear status reporting. However, there is one subtle jq precedence bug in evaluate_pr_checks that can cause passing checks with bucket == "fail" to misbehave, along with a few practical edge cases worth tightening up.
1. Bug: jq Operator Precedence in FAIL Detection
In evaluate_pr_checks, this expression:
((.bucket // "") | ascii_downcase) == "fail" or ((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE")

In jq, | binds tighter than or, so jq parses it as:
( <bucket == "fail"> or <upcased state> ) | IN("FAILURE", ...)

If <bucket == "fail"> is true, true | IN(...) evaluates to false, causing FAIL to be missed and fall through to PENDING (the final else branch).
Fix: Wrap the full boolean condition in parentheses:
elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then

2. Other Edge Cases to Watch For
 * Repo list cap (--limit 200): If either account ever exceeds 200 repositories, gh repo list will stop at 200. Bumping --limit 1000 (or using --paginate where supported) avoids silent omissions.
 * Fork vs. same-repo PRs: gh pr list --head "$BRANCH_NAME" matches same-repo branches cleanly. If any PRs originate from a fork, --head may need <fork-owner>:<branch> or a --jq filter on .headRefName == $branch, and --delete-branch may not be able to delete a branch on a fork you do not own.
 * Branch protection / required reviews: gh pr merge will fail ([ERROR]) if a repository requires an approving review, up-to-date branch, or specific required check context that isn't satisfied yet—which your script already catches and counts under failed_count.
Here is a script (~/bin/merge-passing-gpg-prs.sh) that scans open pull requests from ci/verify-gpg-signatures across re1Lucky365 and adpsnake7-beep, verifies that checks exist and have passed without pending/failing/cancelled states via gh pr checks, and merges eligible PRs via gh pr merge (defaulting to --squash and --delete-branch, with --dry-run support).
1. Safe Merge Script (~/bin/merge-passing-gpg-prs.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; ((.bucket // "") | ascii_downcase) == "fail" or ((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE")) then
      "FAIL"
    elif any(.[]; ((.bucket // "") | ascii_downcase) == "pending" or ((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING")) then
      "PENDING"
    elif all(.[]; ((.bucket // "") | ascii_downcase) == "pass" or ((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED")) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 200 --json name --jq '.[].name' 2>/dev/null || true)
done

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Preview & Merge Workflow
 1. Dry-run to verify which open PRs pass checks
   1 min
   Make ~/bin/merge-passing-gpg-prs.sh executable and run with --dry-run to see which ci/verify-gpg-signatures PRs are ready to merge versus still pending, failing, or missing checks.
   chmod +x "${HOME}/bin/merge-passing-gpg-prs.sh"
"${HOME}/bin/merge-passing-gpg-prs.sh" --dry-run

   Verification: Prints [DRY-RUN] only for PRs whose gh pr checks state is passing and [SKIP] with a reason for any others.
 2. Merge passing pull requests
   1-2 min
   Run without --dry-run to merge passing pull requests and clean up the ci/verify-gpg-signatures branches on the remotes.
   "${HOME}/bin/merge-passing-gpg-prs.sh"

   Verification: Prints [OK] with each merged PR URL and exits 0 when no merge errors occur.
> Local clone sync tip: After merging ci/verify-gpg-signatures on GitHub with --delete-branch, run git pull --ff-only and git branch -d ci/verify-gpg-signatures (plus git fetch --prune) inside your local clones under ${HOME}/src/github before your next run-github-suite.sh backup so your local default branch and archive snapshot include the merged workflow.
> 
To monitor status and CI check results across all open ci/verify-gpg-signatures pull requests for re1Lucky365 and adpsnake7-beep, run a loop that queries gh pr list --head ci/verify-gpg-signatures --state open for each repository and passes matching PR numbers or URLs into gh pr checks.
1. Quick PR & CI Check Status Command
Run this directly in your shell to list every open ci/verify-gpg-signatures pull request and print its live check states (pass, fail, pending, etc.):
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
BRANCH_NAME="ci/verify-gpg-signatures"

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_url pr_title; do
      [ -n "$pr_num" ] || continue
      echo "=== ${slug} #${pr_num}: ${pr_title} (${pr_url}) ==="
      gh pr checks "$pr_num" -R "$slug" --required 2>/dev/null \
        || gh pr checks "$pr_num" -R "$slug" 2>&1 \
        || true
      echo ""
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,url,title \
        --jq '.[] | [.number, .url, .title] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 200 --json name --jq '.[].name' 2>/dev/null || true)
done

2. Compact One-Line Summary Across Open PRs
If you prefer a single compact line per open PR showing its number, URL, and rollup check state (SUCCESS, FAILURE, PENDING, ERROR) without per-check rows:
for acct in re1Lucky365 adpsnake7-beep; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    gh pr list -R "${acct}/${repo}" \
      --head "ci/verify-gpg-signatures" \
      --state open \
      --json number,url,statusCheckRollup \
      --jq --arg repo "${acct}/${repo}" '
        .[] |
        ([.statusCheckRollup[]?.conclusion // .statusCheckRollup[]?.state // "NONE"] | unique | join(",")) as $states |
        "\($repo) #\(.number) | checks=[\($states)] | \(.url)"
      ' 2>/dev/null || true
  done < <(gh repo list "$acct" --limit 200 --json name --jq '.[].name' 2>/dev/null || true)
done

3. Helpful Flags for gh pr checks
 * Watch until completion: Add --watch to gh pr checks "$pr_num" -R "$slug" --watch to poll a single PR until all checks finish.
 * Filter to required checks only: Keep --required when you only care about status checks required by your branch ruleset (such as verify-gpg-signatures).
create-gpg-workflow-prs.shHere is a script (~/bin/create-gpg-workflow-prs.sh) that scans your local Git repositories under ${HOME}/src/github (defaulting to re1Lucky365 and adpsnake7-beep), checks whether the ci/verify-gpg-signatures branch exists locally and/or on origin, skips repositories that already have an open PR from that branch, and uses gh pr create to open a pull request against each repository's default branch.
1. Pull Request Creation Script (~/bin/create-gpg-workflow-prs.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --src-root PATH            Local clones directory (default: "${HOME}/src/github")
  --accounts "acct1 acct2"   GitHub account directories to scan (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --title TITLE              PR title (default: "ci: add GPG commit signature verification workflow")
  --body BODY                PR description body
  --base BRANCH              Optional explicit base branch (default: auto-detected repo default branch)
  --push-missing             Push local feature branch to origin if not yet on remote
  --dry-run                  Preview which PRs would be created without calling gh pr create
  -h, --help                 Show this help
EOF
  exit 1
}

SRC_ROOT="${HOME}/src/github"
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
PR_TITLE="ci: add GPG commit signature verification workflow"
PR_BODY="Adds the \`verify-gpg-signatures\` GitHub Actions workflow to verify commit GPG signatures on pushes and pull requests."
BASE_BRANCH=""
PUSH_MISSING=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --title)
      PR_TITLE="${2:?Missing value for --title}"
      shift 2
      ;;
    --body)
      PR_BODY="${2:?Missing value for --body}"
      shift 2
      ;;
    --base)
      BASE_BRANCH="${2:?Missing value for --base}"
      shift 2
      ;;
    --push-missing)
      PUSH_MISSING=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v git >/dev/null 2>&1 || { echo "ERROR: git is required" >&2; exit 1; }

parse_github_slug() {
  local url="$1"
  local cleaned
  cleaned="$(printf '%s' "$url" | sed -E 's#^(https?://[^/]+/|git@[^:]+:)##; s#\.git$##')"
  if [[ "$cleaned" == */* ]]; then
    printf '%s\n' "$cleaned"
    return 0
  fi
  return 1
}

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

created_count=0
skipped_count=0
failed_count=0

for acct in "${ACCOUNTS[@]}"; do
  acct_dir="${SRC_ROOT}/${acct}"
  [ -d "$acct_dir" ] || continue

  while IFS= read -r -d '' gitdir; do
    repo_dir="$(dirname "$gitdir")"
    repo_name="$(basename "$repo_dir")"
    rel_path="${repo_dir#${SRC_ROOT}/}"

    repo_in_filter "$repo_name" || continue

    has_local_branch=false
    if git -C "$repo_dir" show-ref --verify --quiet "refs/heads/${BRANCH_NAME}"; then
      has_local_branch=true
    fi

    has_remote_branch=false
    if git -C "$repo_dir" ls-remote --exit-code --heads origin "$BRANCH_NAME" >/dev/null 2>&1; then
      has_remote_branch=true
    fi

    if [ "$has_local_branch" = false ] && [ "$has_remote_branch" = false ]; then
      echo "[SKIP] ${rel_path}: branch '${BRANCH_NAME}' not found locally or on origin"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    remote_url="$(git -C "$repo_dir" remote get-url origin 2>/dev/null || true)"
    if [ -z "$remote_url" ] || ! slug="$(parse_github_slug "$remote_url")"; then
      echo "[SKIP] ${rel_path}: missing or non-GitHub origin remote"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    target_base="$BASE_BRANCH"
    if [ -z "$target_base" ]; then
      target_base="$(gh repo view "$slug" --json defaultBranchRef --jq '.defaultBranchRef.name // empty' 2>/dev/null || true)"
      target_base="${target_base:-main}"
    fi

    existing_pr_url="$(
      gh pr list -R "$slug" --head "$BRANCH_NAME" --state open --json url --jq '.[0].url // empty' 2>/dev/null || true
    )"
    if [ -n "$existing_pr_url" ]; then
      echo "[SKIP] ${rel_path} (${slug}): open PR already exists (${existing_pr_url})"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    if [ "$DRY_RUN" = true ]; then
      echo "[DRY-RUN] ${rel_path} (${slug}): would create PR '${BRANCH_NAME}' -> '${target_base}' (remote_branch=${has_remote_branch}, push_missing=${PUSH_MISSING})"
      created_count=$((created_count + 1))
      continue
    fi

    if [ "$has_remote_branch" = false ]; then
      if [ "$PUSH_MISSING" = true ] && [ "$has_local_branch" = true ]; then
        if ! git -C "$repo_dir" push -u origin "$BRANCH_NAME" >/dev/null; then
          echo "[ERROR] ${rel_path} (${slug}): failed pushing '${BRANCH_NAME}' to origin" >&2
          failed_count=$((failed_count + 1))
          continue
        fi
      else
        echo "[SKIP] ${rel_path} (${slug}): branch '${BRANCH_NAME}' not on origin (pass --push-missing to push)"
        skipped_count=$((skipped_count + 1))
        continue
      fi
    fi

    if pr_url="$(
      gh pr create \
        -R "$slug" \
        --base "$target_base" \
        --head "$BRANCH_NAME" \
        --title "$PR_TITLE" \
        --body "$PR_BODY" 2>&1
    )"; then
      echo "[OK] ${rel_path} (${slug}): created PR -> ${pr_url}"
      created_count=$((created_count + 1))
    else
      echo "[ERROR] ${rel_path} (${slug}): gh pr create failed (${pr_url})" >&2
      failed_count=$((failed_count + 1))
    fi
  done < <(find "$acct_dir" -mindepth 2 -maxdepth 2 -type d -name ".git" -print0 | sort -z)
done

echo "------------------------------------------------------------"
echo "Summary: Created=${created_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Preview & Create Pull Requests
 1. Dry-run to confirm target branches and existing PRs
   1 min
   Make ~/bin/create-gpg-workflow-prs.sh executable and run with --dry-run first so you can verify which repositories have ci/verify-gpg-signatures ready and no duplicate open PRs.
   chmod +x "${HOME}/bin/create-gpg-workflow-prs.sh"
"${HOME}/bin/create-gpg-workflow-prs.sh" --push-missing --dry-run

   Verification: Prints [DRY-RUN] lines with target base branches (main, etc.) or [SKIP] reasons without creating any pull requests.
 2. Create pull requests across updated repositories
   1-2 min
   Run without --dry-run to push any unpushed ci/verify-gpg-signatures branches (with --push-missing) and open the pull requests via gh pr create.
   "${HOME}/bin/create-gpg-workflow-prs.sh" --push-missing

   Verification: Prints [OK] with each new PR URL and exits 0 when no gh pr create errors occur.
Here is a GitHub Actions workflow file (.github/workflows/verify-gpg-signatures.yml) that verifies commit GPG signatures on pushes and pull requests, using the verify-gpg-signatures job name so it matches the required_status_checks context in your ruleset JSON (~/.config/github/gpg-ruleset.json).
1. Workflow File (.github/workflows/verify-gpg-signatures.yml)
Because actions/checkout defaults to a shallow clone (fetch-depth: 1), set fetch-depth: 0 so the workflow has the full commit history needed to inspect every commit pushed to a branch or introduced in a pull request range.
name: Verify GPG Commit Signatures

on:
  push:
    branches-ignore:
      - "dependabot/**"
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  verify-gpg-signatures:
    name: verify-gpg-signatures
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository with full history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Import trusted GPG public keys (if configured)
        env:
          TRUSTED_GPG_KEYS: ${{ secrets.TRUSTED_GPG_KEYS }}
        run: |
          if [ -n "$TRUSTED_GPG_KEYS" ]; then
            printf '%s\n' "$TRUSTED_GPG_KEYS" | gpg --batch --import
            echo "Imported trusted GPG keys into runner keyring."
          else
            echo "No TRUSTED_GPG_KEYS secret set; checking embedded signature formatting and GitHub/public key availability."
          fi

      - name: Verify commit GPG signatures
        env:
          EVENT_NAME: ${{ github.event_name }}
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
          HEAD_SHA: ${{ github.sha }}
          BEFORE_SHA: ${{ github.event.before }}
        run: |
          set -euo pipefail

          if [ "$EVENT_NAME" = "pull_request" ]; then
            if [ -z "$BASE_SHA" ] || ! git rev-parse --verify "${BASE_SHA}^{commit}" >/dev/null 2>&1; then
              echo "::error::Missing or invalid pull_request base commit SHA: ${BASE_SHA}"
              exit 1
            fi
            rev_range="${BASE_SHA}..${HEAD_SHA}"
          else
            if [ -z "$BEFORE_SHA" ] || [ "$BEFORE_SHA" = "0000000000000000000000000000000000000000" ]; then
              # New branch or initial push: check reachable commits up to HEAD (bounded to recent history if needed)
              rev_range="$HEAD_SHA"
            else
              rev_range="${BEFORE_SHA}..${HEAD_SHA}"
            fi
          fi

          echo "Checking commits in range: ${rev_range}"
          commits=()
          while IFS= read -r sha; do
            [ -n "$sha" ] && commits+=("$sha")
          done < <(git rev-list "$rev_range")

          if [ "${#commits[@]}" -eq 0 ]; then
            echo "No new commits found in ${rev_range}."
            exit 0
          fi

          failures=0
          for sha in "${commits[@]}"; do
            # %G? -> G (good signature), B (bad signature), U (good untrusted), X (expired), Y (expired key), R (revoked), E (cannot check), N (no signature)
            # %GS -> signer key fingerprint/key ID; %GK -> signing key ID
            sig_status="$(git log -1 --format='%G?' "$sha")"
            signer_key="$(git log -1 --format='%GK' "$sha")"
            signer_name="$(git log -1 --format='%an <%ae>' "$sha")"
            subject="$(git log -1 --format='%s' "$sha")"

            case "$sig_status" in
              G)
                echo "[PASS] ${sha:0:12} (${signer_name}, key=${signer_key}) - Good signature: ${subject}"
                ;;
              U)
                if [ -n "${TRUSTED_GPG_KEYS:-}" ]; then
                  echo "::error::Commit ${sha:0:12} (${signer_name}) has an untrusted signature (status=U, key=${signer_key}): ${subject}"
                  failures=$((failures + 1))
                else
                  echo "[WARN] ${sha:0:12} (${signer_name}, key=${signer_key}) - Valid signature but key trust not established on runner (status=U): ${subject}"
                fi
                ;;
              *)
                echo "::error::Commit ${sha:0:12} (${signer_name}) failed GPG signature check (status=${sig_status}, key=${signer_key}): ${subject}"
                failures=$((failures + 1))
                ;;
            esac
          done

          if [ "$failures" -gt 0 ]; then
            echo "::error::${failures} commit(s) failed GPG signature verification."
            exit 1
          fi

          echo "All ${#commits[@]} commit(s) in ${rev_range} passed GPG signature verification."

2. Important Behavior & Ruleset Alignment
 * Job Name Matches Ruleset Context: Both the job key (verify-gpg-signatures) and name: verify-gpg-signatures match "context": "verify-gpg-signatures" in ~/.config/github/gpg-ruleset.json.
 * How Git's %G? Status Code Works:
   * G: Good signature made by a trusted key.
   * U: Valid signature from a key whose trust isn't locally marked ultimate/trusted on the runner (when TRUSTED_GPG_KEYS is provided, treating U as an error ensures only explicitly imported keys pass; without TRUSTED_GPG_KEYS, it warns rather than false-failing when a commit is signed with a key GitHub recognizes).
   * N, B, E, X, Y, R: No signature, bad signature, uncheckable signature, expired signature/key, or revoked key—all fail the check.
 * Merge Commit Note on Pull Requests: GitHub creates a synthetic merge ref for pull requests (refs/pull/<id>/merge) using github.sha, whereas github.event.pull_request.head.sha is the actual head commit of the PR branch. If you want to verify only the PR's head commits rather than a GitHub-synthesized merge commit, set HEAD_SHA: ${{ github.event.pull_request.head.sha || github.sha }}.
3. Rollout & Verification Steps
 1. Add the workflow file to a repository
   1 min
   Save .github/workflows/verify-gpg-signatures.yml in a signed commit and push it to test the workflow on a real push or pull request.
   mkdir -p .github/workflows
# Write .github/workflows/verify-gpg-signatures.yml and commit with GPG signing enabled
git status --short

   Verification: Workflow file is present at .github/workflows/verify-gpg-signatures.yml and valid YAML.
 2. Confirm check name appears on commits and PRs
   1-2 min
   Push a signed commit or open a PR and inspect the check run status in GitHub or via gh pr checks / gh api.
   gh run list --workflow=verify-gpg-signatures.yml --limit 5

   Verification: A check named verify-gpg-signatures runs on pushes and pull requests and completes with success for properly signed commits.
Here is a script (~/bin/rollout-gpg-workflow.sh) that copies .github/workflows/verify-gpg-signatures.yml into local Git repositories under ${HOME}/src/github (defaulting to re1Lucky365 and adpsnake7-beep), creates a dedicated feature branch (such as ci/verify-gpg-signatures), and creates a GPG-signed commit so your local pre-push hook and remote ruleset checks stay happy.
By default, the script leaves repositories with uncommitted changes or branches already on the target branch untouched unless you pass --allow-dirty or --force-branch, and it supports --dry-run so you can preview every target repo first.
1. Feature-Branch Rollout Script (~/bin/rollout-gpg-workflow.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --src-root PATH            Local clones directory (default: "${HOME}/src/github")
  --workflow-src PATH        Source workflow file (default: "${HOME}/src/github/.github/workflows/verify-gpg-signatures.yml")
  --accounts "acct1 acct2"   GitHub account directories to scan (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Feature branch name to create/use (default: "ci/verify-gpg-signatures")
  --commit-msg MSG           Commit message (default: "ci: add GPG commit signature verification workflow")
  --push                     Push the feature branch to origin after committing
  --allow-dirty              Allow working in a repo that already has uncommitted changes
  --dry-run                  Preview actions without modifying Git branches or files
  -h, --help                 Show this help
EOF
  exit 1
}

SRC_ROOT="${HOME}/src/github"
WORKFLOW_SRC="${HOME}/src/github/.github/workflows/verify-gpg-signatures.yml"
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
COMMIT_MSG="ci: add GPG commit signature verification workflow"
PUSH_BRANCH=false
ALLOW_DIRTY=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --workflow-src)
      WORKFLOW_SRC="${2:?Missing value for --workflow-src}"
      shift 2
      ;;
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --commit-msg)
      COMMIT_MSG="${2:?Missing value for --commit-msg}"
      shift 2
      ;;
    --push)
      PUSH_BRANCH=true
      shift
      ;;
    --allow-dirty)
      ALLOW_DIRTY=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

if [ ! -f "$WORKFLOW_SRC" ]; then
  echo "ERROR: Source workflow file not found: ${WORKFLOW_SRC}" >&2
  exit 1
fi

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

updated_count=0
skipped_count=0
failed_count=0

for acct in "${ACCOUNTS[@]}"; do
  acct_dir="${SRC_ROOT}/${acct}"
  [ -d "$acct_dir" ] || continue

  while IFS= read -r -d '' gitdir; do
    repo_dir="$(dirname "$gitdir")"
    repo_name="$(basename "$repo_dir")"
    rel_path="${repo_dir#${SRC_ROOT}/}"

    repo_in_filter "$repo_name" || continue

    # Skip the source file's own directory if it lives directly inside SRC_ROOT
    if [ "$WORKFLOW_SRC" = "${repo_dir}/.github/workflows/verify-gpg-signatures.yml" ]; then
      echo "[SKIP] ${rel_path}: already the source workflow file"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    if [ "$ALLOW_DIRTY" = false ]; then
      if ! git -C "$repo_dir" diff --quiet || ! git -C "$repo_dir" diff --cached --quiet || [ -n "$(git -C "$repo_dir" ls-files --others --exclude-standard)" ]; then
        echo "[SKIP] ${rel_path}: working tree is not clean (use --allow-dirty to override)"
        skipped_count=$((skipped_count + 1))
        continue
      fi
    fi

    dest_workflow="${repo_dir}/.github/workflows/verify-gpg-signatures.yml"

    if [ -f "$dest_workflow" ] && cmp -s "$WORKFLOW_SRC" "$dest_workflow"; then
      echo "[SKIP] ${rel_path}: .github/workflows/verify-gpg-signatures.yml is already identical"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    if [ "$DRY_RUN" = true ]; then
      echo "[DRY-RUN] ${rel_path}: checkout/create branch '${BRANCH_NAME}', copy workflow, commit (-S), push=${PUSH_BRANCH}"
      updated_count=$((updated_count + 1))
      continue
    fi

    if git -C "$repo_dir" show-ref --verify --quiet "refs/heads/${BRANCH_NAME}"; then
      git -C "$repo_dir" checkout "$BRANCH_NAME" >/dev/null
    else
      git -C "$repo_dir" checkout -b "$BRANCH_NAME" >/dev/null
    fi

    mkdir -p "${repo_dir}/.github/workflows"
    cp "$WORKFLOW_SRC" "$dest_workflow"
    git -C "$repo_dir" add ".github/workflows/verify-gpg-signatures.yml"

    if git -C "$repo_dir" diff --cached --quiet; then
      echo "[SKIP] ${rel_path}: no staged changes after copying workflow"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    # Enforce GPG signing (-S) so local pre-push hooks and branch rulesets pass
    if ! git -C "$repo_dir" commit -S -m "$COMMIT_MSG" >/dev/null; then
      echo "[ERROR] ${rel_path}: GPG commit signing failed (check gpg-agent / signing key)" >&2
      failed_count=$((failed_count + 1))
      continue
    fi

    if [ "$PUSH_BRANCH" = true ]; then
      if ! git -C "$repo_dir" push -u origin "$BRANCH_NAME" >/dev/null; then
        echo "[ERROR] ${rel_path}: failed pushing branch '${BRANCH_NAME}' to origin" >&2
        failed_count=$((failed_count + 1))
        continue
      fi
      echo "[OK] ${rel_path}: committed (GPG-signed) and pushed branch '${BRANCH_NAME}'"
    else
      echo "[OK] ${rel_path}: committed (GPG-signed) on branch '${BRANCH_NAME}' (not pushed; pass --push to push)"
    fi

    updated_count=$((updated_count + 1))
  done < <(find "$acct_dir" -mindepth 2 -maxdepth 2 -type d -name ".git" -print0 | sort -z)
done

echo "------------------------------------------------------------"
echo "Summary: Updated=${updated_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Rollout & Verification Steps
 1. Dry-run across your local repositories first
   1 min
   Make ~/bin/rollout-gpg-workflow.sh executable and run a --dry-run against ${HOME}/src/github to inspect which repositories will be updated or skipped.
   chmod +x "${HOME}/bin/rollout-gpg-workflow.sh"
"${HOME}/bin/rollout-gpg-workflow.sh" \
  --workflow-src "/path/to/your/reference/repo/.github/workflows/verify-gpg-signatures.yml" \
  --dry-run

   Verification: Prints [DRY-RUN] for target repos and [SKIP] for clean already-up-to-date or dirty repos without modifying any Git state.
 2. Create feature branches with GPG-signed commits (and optionally push)
   1-2 min
   Run the script without --dry-run to create ci/verify-gpg-signatures and record a GPG-signed commit in each target repository; add --push once you verify the local commit signature.
   "${HOME}/bin/rollout-gpg-workflow.sh" \
  --workflow-src "/path/to/your/reference/repo/.github/workflows/verify-gpg-signatures.yml" \
  --push

   Verification: Each updated repo shows branch ci/verify-gpg-signatures with a verified GPG signature (git log -1 --show-signature), and any pre-push hook checks pass cleanly.
Here is a single quick command (and optional one-liner check) to inspect archive file sizes, local SHA-256 manifest status, and remote rclone objects for your newest snapshot under ~/github-archives (excluding pre-resign-* and pre-tag-resign-* directories).
1. Quick Snapshot & Remote Inspection Command
Set REMOTE_VAULT (for example crypt-github:github-archives) to your rclone remote base before running:
ARCHIVE_ROOT="${HOME}/github-archives"
REMOTE_VAULT="${RCLONE_REMOTE_BASE:-YOUR_RCLONE_REMOTE:github-archives}"

LATEST_SNAP="$(
  find "$ARCHIVE_ROOT" -mindepth 1 -maxdepth 1 -type d \
    ! -name 'pre-resign-*' ! -name 'pre-tag-resign-*' \
    | sort | tail -n 1
)"
SNAP_NAME="$(basename "$LATEST_SNAP")"

echo "=== Snapshot: ${LATEST_SNAP} ==="
ls -lh "$LATEST_SNAP"

echo ""
echo "=== Local SHA-256 Status ==="
(cd "$LATEST_SNAP" && sha256sum -c SHA256SUMS)

echo ""
echo "=== Remote rclone Objects (${REMOTE_VAULT%/}/${SNAP_NAME}) ==="
rclone lsl "${REMOTE_VAULT%/}/${SNAP_NAME}"

2. What Each Output Section Confirms
 * Archive file sizes (ls -lh): Shows .tar.zst mirror sizes, metadata bundles, and SHA256SUMS at a glance so you can spot empty or unexpectedly truncated archives.
 * SHA-256 status (sha256sum -c SHA256SUMS): Confirms every local archive and bundle matches its recorded SHA-256 digest (: OK).
 * Remote rclone objects (rclone lsl): Lists exact byte sizes, timestamps, and paths stored on your remote vault under ${SNAP_NAME} so you can cross-check presence and sizes against local files.
audit-gpg-signatures.shcompare-repo-rulesets.shrun-github-suite.serviceTo verify end-to-end repository coverage across your local clones under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep) and remote GitHub rulesets, run audit-gpg-signatures.sh (checking local commit and annotated tag GPG signatures) alongside compare-repo-rulesets.sh (checking remote ruleset enforcement and 403/404 restrictions).
1. Combined End-to-End Verification Command
Run both checks sequentially, capturing the audit summary and log path while preserving non-zero exit codes if a signature violation occurs:
SRC_ROOT="${HOME}/src/github"
AUDIT_LOG="${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log"

echo "=== 1. Local GPG Signature Audit (Commits & Annotated Tags) ==="
audit_rc=0
"${HOME}/bin/audit-gpg-signatures.sh" "$SRC_ROOT" "$AUDIT_LOG" || audit_rc=$?

echo ""
echo "=== 2. Local vs. Remote Ruleset Coverage ==="
ruleset_rc=0
"${HOME}/bin/compare-repo-rulesets.sh" --src-root "$SRC_ROOT" || ruleset_rc=$?

echo ""
echo "=== Combined Status ==="
echo "GPG Audit Exit Code   : ${audit_rc} (log: ${AUDIT_LOG})"
echo "Ruleset Check Exit Code: ${ruleset_rc}"

if [ "$audit_rc" -ne 0 ]; then
  exit "$audit_rc"
fi
if [ "$ruleset_rc" -ne 0 ]; then
  exit "$ruleset_rc"
fi
exit 0

2. Optional Wrapper Script (~/bin/verify-repo-coverage.sh)
Save this wrapper if you want a single command you can run manually or inside your maintenance workflow to check both local GPG integrity and remote ruleset state in one pass:
cat <<'EOF' > "${HOME}/bin/verify-repo-coverage.sh"
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
AUDIT_SCRIPT="${HOME}/bin/audit-gpg-signatures.sh"
RULESET_SCRIPT="${HOME}/bin/compare-repo-rulesets.sh"
ARCHIVE_ROOT="${HOME}/github-archives"

mkdir -p "$ARCHIVE_ROOT"
AUDIT_LOG="${ARCHIVE_ROOT}/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log"

echo "[1/2] Auditing local commits and annotated tags in ${SRC_ROOT}..."
audit_status=0
"$AUDIT_SCRIPT" "$SRC_ROOT" "$AUDIT_LOG" || audit_status=$?

echo ""
echo "[2/2] Comparing local repositories against active GitHub rulesets..."
ruleset_status=0
"$RULESET_SCRIPT" --src-root "$SRC_ROOT" || ruleset_status=$?

echo ""
echo "------------------------------------------------------------"
echo "Audit status   : ${audit_status} -> ${AUDIT_LOG}"
echo "Ruleset status : ${ruleset_status}"

if [ "$audit_status" -ne 0 ]; then
  exit "$audit_status"
fi
if [ "$ruleset_status" -ne 0 ]; then
  exit "$ruleset_status"
fi
exit 0
EOF

chmod +x "${HOME}/bin/verify-repo-coverage.sh"

3. Execution & Coverage Checklist
 * Run the combined check: Execute ~/bin/verify-repo-coverage.sh and confirm Violations: 0 across audited commits and annotated tags in the audit-gpg-signatures.sh output.
 * Review ruleset states: Confirm remote-eligible repositories show FULL-ENFORCED (required_signatures and required_status_checks), while any plan- or permission-restricted repositories show SKIP-RESTRICTED (403/404) and remain protected by your local pre-push hook and audit-gpg-signatures.sh.
 * Remediate any gaps: Address any MISSING or PARTIAL-ENFORCED remote rulesets with apply-gpg-ruleset.sh, or re-sign any flagged commits/annotated tags before running your next archive and rclone sync.
apply-gpg-ruleset.shHere is a shell script (~/bin/compare-repo-rulesets.sh) that scans your local Git repositories under ${HOME}/src/github (defaulting to re1Lucky365 and adpsnake7-beep), queries GitHub's rulesets API via gh api, and reports whether each local repository has an active ruleset enforcing required_signatures and required_status_checks—or whether it is missing a ruleset, restricted (403/404), or lacks a GitHub remote.
1. Local vs. Remote Ruleset Coverage Script (~/bin/compare-repo-rulesets.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --src-root PATH            Local clones directory (default: "${HOME}/src/github")
  --accounts "acct1 acct2"   Accounts to include (default: "re1Lucky365 adpsnake7-beep")
  --ruleset-name NAME        Expected ruleset name (default: "gpg-and-status-checks")
  -h, --help                 Show this help
EOF
  exit 1
}

SRC_ROOT="${HOME}/src/github"
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPECTED_NAME="gpg-and-status-checks"

while [ "$#" -gt 0 ]; do
  case "$1" in
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --ruleset-name)
      EXPECTED_NAME="${2:?Missing value for --ruleset-name}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

# Extract owner/repo from a git remote URL (supports https:// and git@github.com:)
parse_github_slug() {
  local url="$1"
  local cleaned
  cleaned="$(printf '%s' "$url" | sed -E 's#^(https?://[^/]+/|git@[^:]+:)##; s#\.git$##')"
  if [[ "$cleaned" == */* ]]; then
    printf '%s\n' "$cleaned"
    return 0
  fi
  return 1
}

account_allowed() {
  local owner="$1"
  for acct in "${ACCOUNTS[@]}"; do
    [ "$acct" = "$owner" ] && return 0
  done
  return 1
}

# Fetches rulesets JSON into LAST_RULESETS_JSON and sets LAST_HTTP_STATUS
fetch_rulesets() {
  local slug="$1"
  local tmp_out tmp_err rc=0
  tmp_out="$(mktemp)"
  tmp_err="$(mktemp)"

  if gh api -i "repos/${slug}/rulesets" >"$tmp_out" 2>"$tmp_err"; then
    rc=0
  else
    rc=$?
  fi

  LAST_HTTP_STATUS="$(
    awk 'toupper($1) ~ /^HTTP\// { code=$2 } END { print (code ? code : "000") }' "$tmp_out"
  )"
  LAST_RULESETS_JSON="$(
    awk 'BEGIN { body=0 } /^\r?$/ { body=1; next } body { print }' "$tmp_out"
  )"

  rm -f "$tmp_out" "$tmp_err"

  if [ "$LAST_HTTP_STATUS" = "000" ] && [ "$rc" -ne 0 ]; then
    return 1
  fi
  return 0
}

total_local=0
covered_full=0
covered_partial=0
missing_ruleset=0
restricted_403_404=0
other_issues=0

printf "%-34s | %-18s | %-10s | %s\n" "LOCAL REPO" "STATUS" "HTTP" "DETAILS"
printf -- "-----------------------------------+--------------------+------------+----------------------------------------------\n"

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  rel_path="${repo_dir#${SRC_ROOT}/}"
  total_local=$((total_local + 1))

  remote_url="$(git -C "$repo_dir" remote get-url origin 2>/dev/null || true)"
  if [ -z "$remote_url" ]; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" "$rel_path" "NO-REMOTE" "-" "origin remote not configured"
    continue
  fi

  if ! slug="$(parse_github_slug "$remote_url")"; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" "$rel_path" "NON-GITHUB" "-" "remote=${remote_url}"
    continue
  fi

  owner="${slug%%/*}"
  if ! account_allowed "$owner"; then
    continue
  fi

  if ! fetch_rulesets "$slug"; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" "$rel_path" "API-ERROR" "000" "failed calling gh api for ${slug}"
    continue
  fi

  if [ "$LAST_HTTP_STATUS" = "403" ] || [ "$LAST_HTTP_STATUS" = "404" ]; then
    restricted_403_404=$((restricted_403_404 + 1))
    printf "%-34s | %-18s | %-10s | %s\n" \
      "$rel_path" "SKIP-RESTRICTED" "$LAST_HTTP_STATUS" "rulesets unavailable/restricted on ${slug} (rely on pre-push + audit)"
    continue
  fi

  if [ "$LAST_HTTP_STATUS" -lt 200 ] || [ "$LAST_HTTP_STATUS" -ge 300 ]; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" \
      "$rel_path" "HTTP-ERROR" "$LAST_HTTP_STATUS" "unexpected response for ${slug}"
    continue
  fi

  # Inspect active rulesets (prefer matching EXPECTED_NAME if present, otherwise inspect all active ones)
  matched_summary="$(
    printf '%s' "$LAST_RULESETS_JSON" | jq -r --arg exp "$EXPECTED_NAME" '
      def summarize(r):
        ([r.rules[]?.type] | unique | join(","));
      ([.[] | select(.enforcement == "active" and .name == $exp)] | .[0]) as $exact |
      ([.[] | select(.enforcement == "active")] | .[0]) as $any_active |
      if $exact != null then
        "EXACT|\($exact.id)|\($exact.name)|\(summarize($exact))"
      elif $any_active != null then
        "OTHER-ACTIVE|\($any_active.id)|\($any_active.name)|\(summarize($any_active))"
      elif ([.[] | select(.name == $exp)] | length) > 0 then
        "INACTIVE|\([.[] | select(.name == $exp)][0].id)|\($exp)|\([.[] | select(.name == $exp)][0].enforcement)"
      else
        "NONE|||"
      end
    ' 2>/dev/null || echo "PARSE-ERROR|||"
  )"

  IFS='|' read -r match_kind rs_id rs_name rs_rules <<< "$matched_summary"

  case "$match_kind" in
    EXACT|OTHER-ACTIVE)
      has_sig=false
      has_checks=false
      [[ ",${rs_rules}," == *",required_signatures,"* ]] && has_sig=true
      [[ ",${rs_rules}," == *",required_status_checks,"* ]] && has_checks=true

      if [ "$has_sig" = true ] && [ "$has_checks" = true ]; then
        covered_full=$((covered_full + 1))
        printf "%-34s | %-18s | %-10s | %s\n" \
          "$rel_path" "FULL-ENFORCED" "$LAST_HTTP_STATUS" "ruleset '${rs_name}' (id=${rs_id}) -> ${rs_rules}"
      else
        covered_partial=$((covered_partial + 1))
        printf "%-34s | %-18s | %-10s | %s\n" \
          "$rel_path" "PARTIAL-ENFORCED" "$LAST_HTTP_STATUS" "ruleset '${rs_name}' (id=${rs_id}) -> ${rs_rules}"
      fi
      ;;
    INACTIVE)
      missing_ruleset=$((missing_ruleset + 1))
      printf "%-34s | %-18s | %-10s | %s\n" \
        "$rel_path" "NOT-ACTIVE" "$LAST_HTTP_STATUS" "ruleset '${rs_name}' (id=${rs_id}) state=${rs_rules}"
      ;;
    NONE)
      missing_ruleset=$((missing_ruleset + 1))
      printf "%-34s | %-18s | %-10s | %s\n" \
        "$rel_path" "MISSING" "$LAST_HTTP_STATUS" "no active ruleset found on ${slug}"
      ;;
    *)
      other_issues=$((other_issues + 1))
      printf "%-34s | %-18s | %-10s | %s\n" \
        "$rel_path" "PARSE-ERROR" "$LAST_HTTP_STATUS" "could not parse rulesets JSON for ${slug}"
      ;;
  esac
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

echo "--------------------------------------------------------------------------------------------------------"
echo "Summary: LocalRepos=${total_local} | FullEnforced=${covered_full} | Partial=${covered_partial} | Missing=${missing_ruleset} | Restricted(403/404)=${restricted_403_404} | OtherIssues=${other_issues}"

2. Run & Interpret the Coverage Comparison
 1. Save and run compare-repo-rulesets.sh
   1 min
   Save the script to ~/bin/compare-repo-rulesets.sh, make it executable, and run it against ${HOME}/src/github.
   chmod +x "${HOME}/bin/compare-repo-rulesets.sh"
"${HOME}/bin/compare-repo-rulesets.sh"

   Verification: Prints a status table for every local repository under re1Lucky365 and adpsnake7-beep and finishes with a single summary line.
 2. Follow up on MISSING or PARTIAL-ENFORCED repos
   1 min
   For any repository showing MISSING or PARTIAL-ENFORCED (with HTTP 200), apply or update ~/.config/github/gpg-ruleset.json using ~/bin/apply-gpg-ruleset.sh --repos "<repo-name>"; for SKIP-RESTRICTED (403/404) repos, rely on your local pre-push hook and audit-gpg-signatures.sh.
   "${HOME}/bin/apply-gpg-ruleset.sh" --ruleset-file "${HOME}/.config/github/gpg-ruleset.json" --dry-run

   Verification: Confirms which repos can be updated remotely vs. which remain covered locally via client-side hooks and audits.
Here is a sample GitHub repository ruleset JSON payload (~/.config/github/gpg-ruleset.json) that targets your default branch (~DEFAULT_BRANCH) and enforces both signed commits (required_signatures) and required status checks (required_status_checks).
1. Sample Ruleset JSON (~/.config/github/gpg-ruleset.json)
Replace "verify-gpg-signatures" (and add any additional check objects to required_status_checks) with the exact status check context / job names reported on your repository's commits or pull requests.
{
  "name": "gpg-and-status-checks",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [],
  "conditions": {
    "ref_name": {
      "include": [
        "~DEFAULT_BRANCH"
      ],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_signatures"
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "do_not_enforce_on_create": false,
        "required_status_checks": [
          {
            "context": "verify-gpg-signatures"
          }
        ]
      }
    }
  ]
}

2. Key Fields to Know Before Applying
 * conditions.ref_name.include: ["~DEFAULT_BRANCH"]: Applies the ruleset to each repository's default branch (main, master, etc.) automatically without hardcoding branch names.
 * rules (required_signatures): Requires commits and tags pushed/merged to matching branches to have verified signatures.
 * rules (required_status_checks):
   * strict_required_status_checks_policy: true requires target branches to be up to date with the base branch before merging.
   * do_not_enforce_on_create: false enforces the check immediately rather than skipping enforcement on initial creation.
   * required_status_checks lists each required check context (you can include multiple { "context": "..." } entries; make sure the name matches an actual check emitted by your CI workflow so PRs aren't stuck waiting on a check that never runs).
 * bypass_actors: []: Keeps enforcement strict with no bypass actors by default (you can add actor entries later if needed).
3. Validate JSON & Preview with apply-gpg-ruleset.sh
mkdir -p "${HOME}/.config/github"
# Save the JSON above to ~/.config/github/gpg-ruleset.json, then validate syntax:
jq empty "${HOME}/.config/github/gpg-ruleset.json"

# Preview across your accounts/repos without mutating remote rulesets:
~/bin/apply-gpg-ruleset.sh --ruleset-file "${HOME}/.config/github/gpg-ruleset.json" --dry-run

Verification: jq empty exits 0 with valid JSON, and apply-gpg-ruleset.sh --dry-run prints [DRY-RUN] for eligible repos while logging [SKIP] with reasons for any 403 / 404 repositories.
To make ~/bin/apply-gpg-ruleset.sh log a clear reason and continue cleanly when GitHub returns 403 or 404 on a repository's rulesets endpoint, capture the HTTP status code from gh api (instead of letting set -e abort on non-zero exit) and route 403 / 404 into a dedicated skip handler.
Because GitHub often returns 404 as well as 403 when rulesets are restricted on a private repository's plan tier or when token/repository permissions don't allow ruleset management, handling both codes explicitly keeps batch runs across re1Lucky365 and adpsnake7-beep moving cleanly.
1. Updated ~/bin/apply-gpg-ruleset.sh (Clean 403 / 404 Handling)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub owners (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names (if omitted, lists repos per owner)
  --ruleset-file PATH        JSON ruleset payload file (default: "${HOME}/.config/github/gpg-ruleset.json")
  --dry-run                  Inspect target repos and show intended action without applying changes
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
RULESET_FILE="${HOME}/.config/github/gpg-ruleset.json"
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --ruleset-file)
      RULESET_FILE="${2:?Missing value for --ruleset-file}"
      shift 2
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }

if [ "$DRY_RUN" = false ] && [ ! -f "$RULESET_FILE" ]; then
  echo "ERROR: Ruleset JSON file not found: ${RULESET_FILE}" >&2
  exit 1
fi

# Runs `gh api` while preserving HTTP status code and response body.
# Sets global LAST_HTTP_CODE and LAST_HTTP_BODY; returns 0 if transport/gh succeeded.
call_gh_api() {
  local tmp_out tmp_err rc=0
  tmp_out="$(mktemp)"
  tmp_err="$(mktemp)"

  # Append status code on the final line via jq/awk wrapper
  if gh api -i "$@" >"$tmp_out" 2>"$tmp_err"; then
    rc=0
  else
    rc=$?
  fi

  LAST_HTTP_CODE="$(
    awk 'toupper($1) ~ /^HTTP\// { code=$2 } END { print (code ? code : "000") }' "$tmp_out"
  )"
  # Extract body after the final blank line separating headers from body
  LAST_HTTP_BODY="$(
    awk 'BEGIN { body=0 } /^\r?$/ { body=1; next } body { print }' "$tmp_out"
  )"

  rm -f "$tmp_out" "$tmp_err"

  if [ "$LAST_HTTP_CODE" = "000" ] && [ "$rc" -ne 0 ]; then
    return 1
  fi
  return 0
}

explain_403_404() {
  local code="$1"
  local owner="$2"
  local repo="$3"
  local is_private="$4"
  local api_msg="$5"

  if [ "$code" = "404" ]; then
    if [ "$is_private" = "true" ]; then
      echo "HTTP 404 on rulesets for private repo ${owner}/${repo} (likely unsupported on current plan tier or ruleset endpoint unavailable/hidden)"
    else
      echo "HTTP 404 on rulesets for ${owner}/${repo} (endpoint not available or repo/owner not accessible to active token)"
    fi
  else
    # code == 403
    if [ "$is_private" = "true" ]; then
      echo "HTTP 403 on rulesets for private repo ${owner}/${repo} (plan tier restriction, missing repo admin permission, or token scope limitation${api_msg:+: ${api_msg}})"
    else
      echo "HTTP 403 on rulesets for ${owner}/${repo} (insufficient token permissions/scope or admin access required${api_msg:+: ${api_msg}})"
    fi
  fi
}

applied_count=0
skipped_count=0
error_count=0

for owner in "${ACCOUNTS[@]}"; do
  if [ -n "$EXPLICIT_REPOS" ]; then
    repo_list=($EXPLICIT_REPOS)
  else
    if ! repo_raw="$(gh repo list "$owner" --limit 200 --json name --jq '.[].name' 2>&1)"; then
      echo "[ERROR] ${owner}: failed listing repositories (${repo_raw})" >&2
      error_count=$((error_count + 1))
      continue
    fi
    mapfile -t repo_list <<< "$repo_raw"
  fi

  for repo in "${repo_list[@]}"; do
    [ -n "$repo" ] || continue
    full_repo="${owner}/${repo}"

    if ! call_gh_api "repos/${full_repo}"; then
      echo "[ERROR] ${full_repo}: gh api transport error while inspecting repository" >&2
      error_count=$((error_count + 1))
      continue
    fi

    if [ "$LAST_HTTP_CODE" = "403" ] || [ "$LAST_HTTP_CODE" = "404" ]; then
      skipped_count=$((skipped_count + 1))
      echo "[SKIP] ${full_repo}: HTTP ${LAST_HTTP_CODE} inspecting repository (check active gh account, token scope, or repo visibility)"
      continue
    elif [ "$LAST_HTTP_CODE" -lt 200 ] || [ "$LAST_HTTP_CODE" -ge 300 ]; then
      echo "[ERROR] ${full_repo}: unexpected HTTP ${LAST_HTTP_CODE} inspecting repository" >&2
      error_count=$((error_count + 1))
      continue
    fi

    is_private="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.private // false' 2>/dev/null || echo "false")"
    ruleset_name="$(jq -r '.name // "gpg-enforcement"' "$RULESET_FILE" 2>/dev/null || echo "gpg-enforcement")"

    # Check existing rulesets first so we can either update or create cleanly
    if ! call_gh_api "repos/${full_repo}/rulesets"; then
      echo "[ERROR] ${full_repo}: gh api transport error querying rulesets" >&2
      error_count=$((error_count + 1))
      continue
    fi

    if [ "$LAST_HTTP_CODE" = "403" ] || [ "$LAST_HTTP_CODE" = "404" ]; then
      api_msg="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.message // empty' 2>/dev/null || true)"
      reason="$(explain_403_404 "$LAST_HTTP_CODE" "$owner" "$repo" "$is_private" "$api_msg")"
      skipped_count=$((skipped_count + 1))
      echo "[SKIP] ${full_repo}: ${reason}"
      continue
    elif [ "$LAST_HTTP_CODE" -lt 200 ] || [ "$LAST_HTTP_CODE" -ge 300 ]; then
      echo "[ERROR] ${full_repo}: unexpected HTTP ${LAST_HTTP_CODE} listing rulesets" >&2
      error_count=$((error_count + 1))
      continue
    fi

    existing_id="$(
      printf '%s' "$LAST_HTTP_BODY" \
        | jq -r --arg name "$ruleset_name" '.[] | select(.name == $name) | .id' 2>/dev/null \
        | head -n 1 || true
    )"

    if [ "$DRY_RUN" = true ]; then
      if [ -n "$existing_id" ]; then
        echo "[DRY-RUN] ${full_repo} (private=${is_private}): would update ruleset '${ruleset_name}' (id=${existing_id})"
      else
        echo "[DRY-RUN] ${full_repo} (private=${is_private}): would create ruleset '${ruleset_name}'"
      fi
      applied_count=$((applied_count + 1))
      continue
    fi

    if [ -n "$existing_id" ]; then
      call_gh_api -X PUT "repos/${full_repo}/rulesets/${existing_id}" --input "$RULESET_FILE" || {
        echo "[ERROR] ${full_repo}: gh api transport error updating ruleset ${existing_id}" >&2
        error_count=$((error_count + 1))
        continue
      }
      action_word="updating"
    else
      call_gh_api -X POST "repos/${full_repo}/rulesets" --input "$RULESET_FILE" || {
        echo "[ERROR] ${full_repo}: gh api transport error creating ruleset" >&2
        error_count=$((error_count + 1))
        continue
      }
      action_word="creating"
    fi

    if [ "$LAST_HTTP_CODE" = "403" ] || [ "$LAST_HTTP_CODE" = "404" ]; then
      api_msg="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.message // empty' 2>/dev/null || true)"
      reason="$(explain_403_404 "$LAST_HTTP_CODE" "$owner" "$repo" "$is_private" "$api_msg")"
      skipped_count=$((skipped_count + 1))
      echo "[SKIP] ${full_repo}: ${reason} (while ${action_word} ruleset '${ruleset_name}')"
      continue
    elif [ "$LAST_HTTP_CODE" -lt 200 ] || [ "$LAST_HTTP_CODE" -ge 300 ]; then
      api_msg="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.message // empty' 2>/dev/null || true)"
      echo "[ERROR] ${full_repo}: HTTP ${LAST_HTTP_CODE} while ${action_word} ruleset '${ruleset_name}'${api_msg:+ (${api_msg})}" >&2
      error_count=$((error_count + 1))
      continue
    fi

    echo "[OK] ${full_repo} (private=${is_private}): applied ruleset '${ruleset_name}' (HTTP ${LAST_HTTP_CODE})"
    applied_count=$((applied_count + 1))
  done
done

echo "------------------------------------------------------------"
echo "Summary: Applied=${applied_count} | Skipped(403/404)=${skipped_count} | Errors=${error_count}"

if [ "$error_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Why This Structure Works Better Across Multiple Repos & Accounts
 * Non-fatal skips vs. real errors: Repositories returning 403 or 404 increment Skipped(403/404) and log [SKIP] with context (private=true/false and any GitHub API message) without aborting the loop or failing the script when no true errors occur.
 * True errors still surface: Network failures, invalid JSON payloads, or 422 / 5xx responses increment Errors and cause a non-zero exit so your systemd failure alert still catches genuine regressions.
 * Safe to re-run idempotently: Matching an existing ruleset by .name switches between POST (create) and PUT (update) automatically.
To rotate and cap systemd-failures.log and gpg-audit-*.log inside ~/github-archives without touching your actual repository backup snapshots, you can use user-mode logrotate (running without root via a custom state file) or a lightweight Zstandard (zstd) rotation script triggered right from run-github-suite.sh or a systemd --user timer.
Using logrotate with --state in ~/.local/state/logrotate/github-archives.state works cleanly for a regular user account without needing root privileges or /var/lib/logrotate/status.
1. Rotate & Cap with User-Mode logrotate
Create a user logrotate config at ~/.config/logrotate/github-archives.conf that:
 * Rotates ~/github-archives/systemd-failures.log when it exceeds 10M (or monthly), compresses rotated logs with zstd, and keeps the latest 10 rotated archives (rotate 10), using copytruncate so append writers never lose their file descriptor.
 * Rotates ~/github-archives/gpg-audit-*.log (keeping copytruncate or rotating when size/age thresholds are met) and caps total retained audit logs by pruning older gpg-audit-* files/archives past your retention count.
mkdir -p "${HOME}/.config/logrotate" "${HOME}/.local/state/logrotate"

cat <<'EOF' > "${HOME}/.config/logrotate/github-archives.conf"
/home/robert/github-archives/systemd-failures.log {
    size 10M
    rotate 10
    missingok
    notifempty
    copytruncate
    compress
    compresscmd /usr/bin/zstd
    uncompresscmd /usr/bin/unzstd
    compressext .zst
    compressoptions -T0 -19
    dateext
    dateformat -%Y%m%d-%H%M%S
}

/home/robert/github-archives/gpg-audit-*.log {
    size 5M
    rotate 30
    missingok
    notifempty
    copytruncate
    compress
    compresscmd /usr/bin/zstd
    uncompresscmd /usr/bin/unzstd
    compressext .zst
    compressoptions -T0 -19
    dateext
    dateformat -%Y%m%d-%H%M%S
}
EOF

(Replace /home/robert with ${HOME} if your Linux username differs—note that logrotate config directives expect explicit paths rather than unexpanded shell variables.)
2. Cap Total Count & Disk Footprint (~/bin/rotate-github-logs.sh)
Because gpg-audit-<timestamp>.log creates a new timestamped file per run, a rotate N rule on a wildcard gpg-audit-*.log rotates each matching file individually rather than capping the total number of audit log files across runs. Pair logrotate (or use a standalone script) with a cap on total gpg-audit-* files and total log size:
cat <<'EOF' > "${HOME}/bin/rotate-github-logs.sh"
#!/usr/bin/env bash
set -euo pipefail

ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
FAILURES_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_FAILURES_BYTES="${MAX_FAILURES_BYTES:-10485760}"   # 10 MiB before rotation
KEEP_FAILURES_ROTATED="${KEEP_FAILURES_ROTATED:-10}"
KEEP_AUDIT_LOGS="${KEEP_AUDIT_LOGS:-30}"              # Keep newest 30 gpg-audit log/zst files
MAX_TOTAL_LOG_BYTES="${MAX_TOTAL_LOG_BYTES:-268435456}" # 256 MiB cap across log + rotated log files

mkdir -p "$ARCHIVE_ROOT" "${HOME}/.local/state/logrotate"

STAMP="$(date +%Y%m%d-%H%M%S)"

# 1. Rotate systemd-failures.log if it exceeds MAX_FAILURES_BYTES
if [ -f "$FAILURES_LOG" ]; then
  current_size="$(stat -c '%s' "$FAILURES_LOG" 2>/dev/null || echo 0)"
  if [ "$current_size" -ge "$MAX_FAILURES_BYTES" ]; then
    rotated_name="${ARCHIVE_ROOT}/systemd-failures-${STAMP}.log"
    cp -p "$FAILURES_LOG" "$rotated_name"
    : > "$FAILURES_LOG"
    if command -v zstd >/dev/null 2>&1; then
      zstd --rm -q -T0 -19 "$rotated_name" -o "${rotated_name}.zst"
    fi
  fi
fi

# Prune older rotated systemd-failures archives beyond KEEP_FAILURES_ROTATED
mapfile -t failure_archives < <(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f \
    \( -name 'systemd-failures-*.log' -o -name 'systemd-failures-*.log.zst' \) \
    | sort -r
)
if [ "${#failure_archives[@]}" -gt "$KEEP_FAILURES_ROTATED" ]; then
  for old_file in "${failure_archives[@]:$KEEP_FAILURES_ROTATED}"; do
    rm -f "$old_file"
  done
fi

# 2. Cap total gpg-audit-*.log and gpg-audit-*.log.zst files to KEEP_AUDIT_LOGS
mapfile -t audit_files < <(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f \
    \( -name 'gpg-audit-*.log' -o -name 'gpg-audit-*.log.zst' \) \
    | sort -r
)
if [ "${#audit_files[@]}" -gt "$KEEP_AUDIT_LOGS" ]; then
  for old_audit in "${audit_files[@]:$KEEP_AUDIT_LOGS}"; do
    rm -f "$old_audit"
  done
fi

# 3. Enforce total byte budget across log files (oldest first, keeping active systemd-failures.log)
total_bytes=0
declare -a all_log_files=()
while IFS= read -r -d '' f; do
  sz="$(stat -c '%s' "$f" 2>/dev/null || echo 0)"
  total_bytes=$((total_bytes + sz))
  all_log_files+=("${sz}|${f}")
done < <(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f \
    \( -name 'systemd-failures*.log*' -o -name 'gpg-audit-*.log*' \) \
    -print0 | sort -z
)

if [ "$total_bytes" -gt "$MAX_TOTAL_LOG_BYTES" ]; then
  for entry in "${all_log_files[@]}"; do
    [ "$total_bytes" -le "$MAX_TOTAL_LOG_BYTES" ] && break
    sz="${entry%%|*}"
    fpath="${entry#*|}"
    # Truncate active systemd-failures.log rather than deleting it; remove older rotated/audit logs
    if [ "$fpath" = "$FAILURES_LOG" ]; then
      : > "$FAILURES_LOG"
      total_bytes=$((total_bytes - sz))
    else
      rm -f "$fpath"
      total_bytes=$((total_bytes - sz))
    fi
  done
fi
EOF

chmod +x "${HOME}/bin/rotate-github-logs.sh"

3. Run & Verify Log Rotation
 * Test logrotate in user mode (--debug / --force):
   logrotate --state "${HOME}/.local/state/logrotate/github-archives.state" \
  --verbose "${HOME}/.config/logrotate/github-archives.conf"

   Verification: Exits 0 and logs rotation/compression actions without touching .tar.zst repository snapshot folders.
 * Test ~/bin/rotate-github-logs.sh and check log sizes:
   ~/bin/rotate-github-logs.sh
ls -lh "${HOME}/github-archives"/systemd-failures* "${HOME}/github-archives"/gpg-audit-* 2>/dev/null || true

   Verification: Confirms systemd-failures.log and gpg-audit-* files are capped according to your size and file-count limits.
 * Hook into run-github-suite.sh: Add "${HOME}/bin/rotate-github-logs.sh" near the end of ~/bin/run-github-suite.sh (after audit-gpg-signatures.sh finishes reading the latest audit log) so log capping stays automatic on every scheduled or manual run.
To run run-github-suite.service automatically on a schedule, create a matching systemd user timer unit (~/.config/systemd/user/run-github-suite.timer) and enable it with systemctl --user enable --now.
A systemd timer unit activates a service unit of the same base name by default (Unit= defaults to run-github-suite.service), uses OnCalendar= for wall-clock schedules, and supports Persistent=true so missed runs execute once the user manager comes back online.
1. Create the Systemd User Timer (~/.config/systemd/user/run-github-suite.timer)
Here is a daily schedule at 03:15 local time (America/Chicago / CST in Evansville, Indiana) with a RandomizedDelaySec=300 window so scheduled runs don't collide if you add more timers later:
mkdir -p "${HOME}/.config/systemd/user"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite.timer"
[Unit]
Description=Scheduled run of GitHub backup, GPG audit, and archive sync suite
Requires=run-github-suite.service

[Timer]
OnCalendar=*-*-* 03:15:00
RandomizedDelaySec=300
Persistent=true
Unit=run-github-suite.service

[Install]
WantedBy=timers.target
EOF

(Other common OnCalendar= examples: every 6 hours uses OnCalendar=*-*-* 00/6:00:00; every Sunday at 02:00 uses OnCalendar=Sun *-*-* 02:00:00.)
2. Enable, Start & Verify the Timer
 1. Reload user units and enable the timer
   1 min
   Reload systemd --user so it detects run-github-suite.timer, then enable and start it immediately.
   systemctl --user daemon-reload
systemctl --user enable --now run-github-suite.timer

   Verification: systemctl --user is-enabled run-github-suite.timer outputs enabled and systemctl --user is-active run-github-suite.timer outputs active.
 2. Inspect next scheduled trigger time
   1 min
   List active user timers to verify NEXT and LEFT trigger timestamps.
   systemctl --user list-timers --all | grep -E "NEXT|run-github-suite"

   Verification: Shows run-github-suite.timer with its next scheduled trigger time and run-github-suite.service listed under UNIT.
 3. Verify unattended execution across logouts
   1 min
   If you want the timer and run-github-suite.service to keep running when you are not logged into an interactive session, enable user lingering.
   loginctl enable-linger "$USER"
loginctl show-user "$USER" -p Linger

   Verification: loginctl show-user returns Linger=yes.
> Scheduling & GPG/SSH note: When run-github-suite.service runs unattended from a timer, make sure any required GPG/SSH/GitHub credentials or rclone remotes are accessible non-interactively (for example, unlocked/key-based auth or an environment file loaded via EnvironmentFile=) so scheduled runs don't stall waiting on a TTY prompt.
> 
To trigger run-github-suite.service on demand and stream its live output, start the user service with systemctl --user start and follow its logs with journalctl --user -u run-github-suite.service -f.
Using --no-block lets systemctl --user start return right away so you can immediately attach to journalctl -f without waiting for a long backup, GPG audit, or rclone sync to finish first.
1. Trigger On-Demand & Stream Live Logs
Run these commands in your terminal to start run-github-suite.service immediately and watch output as each step executes:
systemctl --user start --no-block run-github-suite.service
journalctl --user -u run-github-suite.service -f --output=short-iso

(If your service is a Type=oneshot unit and you want systemctl itself to wait until the run finishes and report success/failure in one command, run systemctl --user start run-github-suite.service in one terminal and journalctl --user -u run-github-suite.service -f in a second terminal.)
2. Step-by-Step Execution & Inspection
 1. Start the service and attach to live journal output
   1 min
   Trigger run-github-suite.service and tail its live journal stream with ISO timestamps.
   systemctl --user start --no-block run-github-suite.service
journalctl --user -u run-github-suite.service -f --output=short-iso

   Verification: journalctl prints live log lines from run-github-suite.service as the backup, GPG audit, SHA-256 verification, and rclone check execute; press Ctrl+C to detach from the live stream once finished.
 2. Check final unit status and exit code
   1 min
   After the run completes, inspect systemctl --user status and ExecMainStatus to confirm a clean exit (0) or see why it failed.
   systemctl --user status run-github-suite.service --no-pager
systemctl --user show run-github-suite.service -p ActiveState,SubState,Result,ExecMainStatus

   Verification: Shows ActiveState=inactive (for oneshot after completion), Result=success, and ExecMainStatus=0 on a healthy run; if non-zero, OnFailure=run-github-suite-alert.service will have triggered as well.
 3. Filter logs to only the most recent run (if needed)
   1 min
   If run-github-suite.service has run multiple times, inspect logs since the last start or since today's start of day.
   journalctl --user -u run-github-suite.service --since today --no-pager --output=short-iso

   Verification: Displays only entries from today for run-github-suite.service so you can review full output without scrolling past older runs.
Here is the updated ~/bin/notify-github-suite-failure.sh that redacts URLs, tokens/credentials, and local filesystem paths before sending webhook or mail alerts, while keeping a local diagnostic copy in ${HOME}/github-archives/systemd-failures.log.
Redacting outbound payloads prevents leaking tokens, embedded credentials in URLs, webhook endpoints, or local directory paths (${HOME}, /home/..., /root/...) when alerts go to a webhook or mail sink.
1. Redacted ~/bin/notify-github-suite-failure.sh
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
ALERT_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_LINES="${GITHUB_SUITE_LOG_LINES:-120}"

mkdir -p "$ARCHIVE_ROOT" "$(dirname "$ALERT_LOG")"

EXIT_STATUS="$(
  systemctl --user show "$UNIT_NAME" -p ExecMainStatus --value 2>/dev/null || echo "unknown"
)"
RESULT_STATE="$(
  systemctl --user show "$UNIT_NAME" -p Result --value 2>/dev/null || echo "unknown"
)"

JOURNAL_CONTEXT="$(
  journalctl --user -u "$UNIT_NAME" -n "$MAX_LINES" --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

LATEST_AUDIT_LOG="$(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f -name 'gpg-audit-*.log' 2>/dev/null \
    | sort | tail -n 1 || true
)"

AUDIT_CONTEXT=""
if [ -n "${LATEST_AUDIT_LOG:-}" ] && [ -f "$LATEST_AUDIT_LOG" ]; then
  AUDIT_CONTEXT="$(tail -n 40 "$LATEST_AUDIT_LOG" 2>/dev/null || true)"
fi

LOCAL_REPORT="$(cat <<REPORT
============================================================
ALERT: ${UNIT_NAME} failed on ${HOST_NAME}
Timestamp (UTC) : ${TIMESTAMP_UTC}
Result / Status : ${RESULT_STATE} (exit=${EXIT_STATUS})
------------------------------------------------------------
Recent Journal Logs (last ${MAX_LINES} lines):
${JOURNAL_CONTEXT}
REPORT
)"

if [ -n "$AUDIT_CONTEXT" ]; then
  LOCAL_REPORT+="$(printf '\n------------------------------------------------------------\nLatest GPG Audit Tail (%s):\n%s\n' "$(basename "$LATEST_AUDIT_LOG")" "$AUDIT_CONTEXT")"
fi

# Save full unredacted local failure log for local debugging
printf '%s\n\n' "$LOCAL_REPORT" >> "$ALERT_LOG"

# Redact URLs, tokens/secrets, and local paths for outbound webhook/mail alerts
redact_text() {
  python3 - "$HOME" <<'PY'
import re
import sys

home_dir = sys.argv[1]
text = sys.stdin.read()

# 1. Redact common token/secret assignments and bearer/key prefixes
secret_patterns = [
    r'(?i)\b(authorization|bearer|token|api[_-]?key|secret|password|passwd|pwd|webhook[_-]?url)\b\s*[:=]\s*["\']?[^\s"\'&;,]+',
    r'\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]+\b',
    r'\bxox[baprs]-[A-Za-z0-9-]+\b',
    r'\bAKIA[0-9A-Z]{16}\b',
    r'(?i)bearer\s+[A-Za-z0-9._~+/=-]+',
]
for pat in secret_patterns:
    text = re.sub(pat, lambda m: f"{m.group(0).split('=')[0].split(':')[0]}=[REDACTED]" if ('=' in m.group(0) or ':' in m.group(0)) else "[REDACTED_TOKEN]", text)

# 2. Redact any URLs (including userinfo credentials and query params)
text = re.sub(r'\b[a-zA-Z][a-zA-Z0-9+.-]*://[^\s"\'<>)\]]+', '[REDACTED_URL]', text)

# 3. Redact $HOME and common Linux home/root/tmp/var paths
if home_dir:
    text = text.replace(home_dir, '~')
text = re.sub(r'/(?:home|root|var/log|etc|tmp|var/tmp)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)
text = re.sub(r'~/(?:src|github-archives|\.config|\.ssh|\.gnupg)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)

sys.stdout.write(text)
PY
}

REDACTED_REPORT="$(printf '%s' "$LOCAL_REPORT" | redact_text)"

# Desktop notification if graphical session is active (kept concise and path-free)
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" \
    "${UNIT_NAME} (${RESULT_STATE}, exit=${EXIT_STATUS}) on ${HOST_NAME}" || true
fi

# HTTP Webhook delivery (using redacted report)
if [ -n "${GITHUB_SUITE_WEBHOOK_URL:-}" ]; then
  if command -v curl >/dev/null 2>&1; then
    if command -v jq >/dev/null 2>&1; then
      PAYLOAD="$(
        jq -n \
          --arg unit "$UNIT_NAME" \
          --arg host "$HOST_NAME" \
          --arg ts "$TIMESTAMP_UTC" \
          --arg result "$RESULT_STATE" \
          --arg exit_code "$EXIT_STATUS" \
          --arg context "$REDACTED_REPORT" \
          '{
            text: ("*[" + $host + "]* `" + $unit + "` failed (result=" + $result + ", exit=" + $exit_code + ") at " + $ts + "\n```\n" + $context + "\n```"),
            unit: $unit,
            host: $host,
            timestamp_utc: $ts,
            result: $result,
            exit_status: $exit_code,
            logs: $context
          }'
      )"
    else
      ESCAPED_REPORT="$(printf '%s' "$REDACTED_REPORT" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')"
      PAYLOAD="{\"unit\":\"${UNIT_NAME}\",\"host\":\"${HOST_NAME}\",\"timestamp_utc\":\"${TIMESTAMP_UTC}\",\"exit_status\":\"${EXIT_STATUS}\",\"logs\":${ESCAPED_REPORT}}"
    fi

    curl_opts=(-fsS -X POST -H "Content-Type: application/json" -d "$PAYLOAD")
    if [ -n "${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN:-}" ]; then
      curl_opts+=(-H "Authorization: Bearer ${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN}")
    fi

    if ! curl "${curl_opts[@]}" "$GITHUB_SUITE_WEBHOOK_URL" >/dev/null; then
      echo "[WARN] Failed delivering webhook alert" >> "$ALERT_LOG"
    fi
  else
    echo "[WARN] curl not installed; cannot send webhook alert" >> "$ALERT_LOG"
  fi
fi

# Local mail delivery (using redacted report)
if [ -n "${GITHUB_SUITE_MAIL_TO:-}" ]; then
  SUBJECT="[ALERT] ${UNIT_NAME} failed on ${HOST_NAME} (exit=${EXIT_STATUS})"
  if command -v mail >/dev/null 2>&1; then
    mail_args=(-s "$SUBJECT")
    if [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ]; then
      mail_args+=(-r "$GITHUB_SUITE_MAIL_FROM")
    fi
    if ! printf '%s\n' "$REDACTED_REPORT" | mail "${mail_args[@]}" "$GITHUB_SUITE_MAIL_TO"; then
      echo "[WARN] Failed sending mail alert via mail(1)" >> "$ALERT_LOG"
    fi
  elif command -v sendmail >/dev/null 2>&1; then
    {
      [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ] && echo "From: ${GITHUB_SUITE_MAIL_FROM}"
      echo "To: ${GITHUB_SUITE_MAIL_TO}"
      echo "Subject: ${SUBJECT}"
      echo "Content-Type: text/plain; charset=UTF-8"
      echo ""
      printf '%s\n' "$REDACTED_REPORT"
    } | sendmail -t || echo "[WARN] Failed sending mail alert via sendmail" >> "$ALERT_LOG"
  else
    echo "[WARN] Neither mail nor sendmail is installed; cannot send mail alert" >> "$ALERT_LOG"
  fi
fi

exit 0
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

2. Verify Redaction Behavior
 1. Test the redaction filter on sample sensitive strings
   1 min
   Pipe a sample string containing a URL, token, and path through the redaction block to verify replacements look right before relying on live alerts.
   printf 'path=%s/src/github/repo url=https://user:pass@example.com/hook?tok=abc token=ghp_1234567890abcdef\n' "$HOME" \
  | python3 - "$HOME" - <<'PY'
import re, sys
home_dir = sys.argv[1]
text = sys.stdin.read()
text = re.sub(r'(?i)\b(authorization|bearer|token|api[_-]?key|secret|password|passwd|pwd|webhook[_-]?url)\b\s*[:=]\s*["\']?[^\s"\'&;,]+', lambda m: f"{m.group(0).split('=')[0].split(':')[0]}=[REDACTED]", text)
text = re.sub(r'\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]+\b', '[REDACTED_TOKEN]', text)
text = re.sub(r'\b[a-zA-Z][a-zA-Z0-9+.-]*://[^\s"\'<>)\]]+', '[REDACTED_URL]', text)
if home_dir:
text = text.replace(home_dir, '~')
text = re.sub(r'/(?:home|root|var/log|etc|tmp|var/tmp)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)
text = re.sub(r'~/(?:src|github-archives|\.config|\.ssh|\.gnupg)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)
print(text, end='')
PY

   Verification: Output replaces the URL with [REDACTED_URL], token with [REDACTED_TOKEN] / [REDACTED], and home/repo paths with [REDACTED_PATH] without exposing raw secrets.
 2. Trigger a test alert run
   1 min
   Run ~/bin/notify-github-suite-failure.sh run-github-suite.service and inspect your outbound alert payload/message to confirm no raw URLs, tokens, or paths appear.
   "${HOME}/bin/notify-github-suite-failure.sh" run-github-suite.service

   Verification: Exits 0, logs the full diagnostic entry to ${HOME}/github-archives/systemd-failures.log, and dispatches the redacted alert cleanly.
> Tuning note: Because regex redaction is a best-effort defense in depth, avoid printing sensitive CLI flags or environment values in your scripts' stdout/stderr in the first place—keeping secrets out of logs is always safer than scrubbing them afterward.
> 
notify-github-suite-failure.shUpdate notify-github-suite-failure.sh to redact URLs, tokens, or paths before sending webhook and mail alerts.Here is how to update ~/bin/notify-github-suite-failure.sh so a non-zero exit from run-github-suite.service sends an HTTP webhook (JSON payload with bounded log context) and/or a local mail alert (via mail / sendmail / msmtp), while still recording the full failure block to ${HOME}/github-archives/systemd-failures.log.
Because systemd user services often run without an interactive shell environment, store sensitive secrets (like a webhook URL or token) in a restricted environment file (for example ~/.config/systemd/user/run-github-suite-alert.env, chmod 600) and load it via EnvironmentFile= in run-github-suite-alert.service.
1. Store Alert Config in a Restricted Environment File
Create ~/.config/systemd/user/run-github-suite-alert.env with mode 0600 so webhook URLs or recipient addresses aren't world-readable:
mkdir -p "${HOME}/.config/systemd/user"
chmod 700 "${HOME}/.config/systemd/user"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.env"
# Optional: HTTP webhook endpoint (receives JSON payload)
GITHUB_SUITE_WEBHOOK_URL=""
GITHUB_SUITE_WEBHOOK_BEARER_TOKEN=""

# Optional: Local mail recipient (e.g., "robert@localhost" or local system user)
GITHUB_SUITE_MAIL_TO=""
GITHUB_SUITE_MAIL_FROM=""

# Max lines of journal/audit context to attach (keeps payloads/emails manageable)
GITHUB_SUITE_LOG_LINES="120"
EOF

chmod 600 "${HOME}/.config/systemd/user/run-github-suite-alert.env"

Wire that file into ~/.config/systemd/user/run-github-suite-alert.service:
cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.service"
[Unit]
Description=Alert when run-github-suite.service exits non-zero
After=run-github-suite.service

[Service]
Type=oneshot
EnvironmentFile=%h/.config/systemd/user/run-github-suite-alert.env
ExecStart=%h/bin/notify-github-suite-failure.sh run-github-suite.service
EOF

systemctl --user daemon-reload

2. Updated ~/bin/notify-github-suite-failure.sh (Webhook + Local Mail + Log Context)
This version pulls:
 * Recent journalctl --user entries for UNIT_NAME (including invocation/exit status when available).
 * The tail of the newest gpg-audit-*.log file in ${HOME}/github-archives/ (if present) for extra diagnostic context.
 * Sends a JSON POST to GITHUB_SUITE_WEBHOOK_URL (if set) and/or sends a plain-text message via mail or sendmail to GITHUB_SUITE_MAIL_TO (if set).
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
ALERT_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_LINES="${GITHUB_SUITE_LOG_LINES:-120}"

mkdir -p "$ARCHIVE_ROOT" "$(dirname "$ALERT_LOG")"

EXIT_STATUS="$(
  systemctl --user show "$UNIT_NAME" -p ExecMainStatus --value 2>/dev/null || echo "unknown"
)"
RESULT_STATE="$(
  systemctl --user show "$UNIT_NAME" -p Result --value 2>/dev/null || echo "unknown"
)"

JOURNAL_CONTEXT="$(
  journalctl --user -u "$UNIT_NAME" -n "$MAX_LINES" --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

LATEST_AUDIT_LOG="$(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f -name 'gpg-audit-*.log' 2>/dev/null \
    | sort | tail -n 1 || true
)"

AUDIT_CONTEXT=""
if [ -n "${LATEST_AUDIT_LOG:-}" ] && [ -f "$LATEST_AUDIT_LOG" ]; then
  AUDIT_CONTEXT="$(tail -n 40 "$LATEST_AUDIT_LOG" 2>/dev/null || true)"
fi

FULL_REPORT="$(cat <<REPORT
============================================================
ALERT: ${UNIT_NAME} failed on ${HOST_NAME}
Timestamp (UTC) : ${TIMESTAMP_UTC}
Result / Status : ${RESULT_STATE} (exit=${EXIT_STATUS})
------------------------------------------------------------
Recent Journal Logs (last ${MAX_LINES} lines):
${JOURNAL_CONTEXT}
REPORT
)"

if [ -n "$AUDIT_CONTEXT" ]; then
  FULL_REPORT+="$(printf '\n------------------------------------------------------------\nLatest GPG Audit Tail (%s):\n%s\n' "$(basename "$LATEST_AUDIT_LOG")" "$AUDIT_CONTEXT")"
fi

printf '%s\n\n' "$FULL_REPORT" >> "$ALERT_LOG"

# Desktop notification if graphical session is active
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" \
    "${UNIT_NAME} (${RESULT_STATE}, exit=${EXIT_STATUS}) on ${HOST_NAME}" || true
fi

# HTTP Webhook delivery
if [ -n "${GITHUB_SUITE_WEBHOOK_URL:-}" ]; then
  if command -v curl >/dev/null 2>&1; then
    if command -v jq >/dev/null 2>&1; then
      PAYLOAD="$(
        jq -n \
          --arg unit "$UNIT_NAME" \
          --arg host "$HOST_NAME" \
          --arg ts "$TIMESTAMP_UTC" \
          --arg result "$RESULT_STATE" \
          --arg exit_code "$EXIT_STATUS" \
          --arg log_file "$ALERT_LOG" \
          --arg context "$FULL_REPORT" \
          '{
            text: ("*[" + $host + "]* `" + $unit + "` failed (result=" + $result + ", exit=" + $exit_code + ") at " + $ts + "\n```\n" + $context + "\n```"),
            unit: $unit,
            host: $host,
            timestamp_utc: $ts,
            result: $result,
            exit_status: $exit_code,
            alert_log: $log_file,
            logs: $context
          }'
      )"
    else
      ESCAPED_REPORT="$(printf '%s' "$FULL_REPORT" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')"
      PAYLOAD="{\"unit\":\"${UNIT_NAME}\",\"host\":\"${HOST_NAME}\",\"timestamp_utc\":\"${TIMESTAMP_UTC}\",\"exit_status\":\"${EXIT_STATUS}\",\"logs\":${ESCAPED_REPORT}}"
    fi

    curl_opts=(-fsS -X POST -H "Content-Type: application/json" -d "$PAYLOAD")
    if [ -n "${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN:-}" ]; then
      curl_opts+=(-H "Authorization: Bearer ${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN}")
    fi

    if ! curl "${curl_opts[@]}" "$GITHUB_SUITE_WEBHOOK_URL" >/dev/null; then
      echo "[WARN] Failed delivering webhook alert to ${GITHUB_SUITE_WEBHOOK_URL}" >> "$ALERT_LOG"
    fi
  else
    echo "[WARN] curl not installed; cannot send webhook alert" >> "$ALERT_LOG"
  fi
fi

# Local mail delivery (supports `mail` / `mailx` or `sendmail`)
if [ -n "${GITHUB_SUITE_MAIL_TO:-}" ]; then
  SUBJECT="[ALERT] ${UNIT_NAME} failed on ${HOST_NAME} (exit=${EXIT_STATUS})"
  if command -v mail >/dev/null 2>&1; then
    mail_args=(-s "$SUBJECT")
    if [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ]; then
      mail_args+=(-r "$GITHUB_SUITE_MAIL_FROM")
    fi
    if ! printf '%s\n' "$FULL_REPORT" | mail "${mail_args[@]}" "$GITHUB_SUITE_MAIL_TO"; then
      echo "[WARN] Failed sending mail alert via mail(1) to ${GITHUB_SUITE_MAIL_TO}" >> "$ALERT_LOG"
    fi
  elif command -v sendmail >/dev/null 2>&1; then
    {
      [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ] && echo "From: ${GITHUB_SUITE_MAIL_FROM}"
      echo "To: ${GITHUB_SUITE_MAIL_TO}"
      echo "Subject: ${SUBJECT}"
      echo "Content-Type: text/plain; charset=UTF-8"
      echo ""
      printf '%s\n' "$FULL_REPORT"
    } | sendmail -t || echo "[WARN] Failed sending mail alert via sendmail to ${GITHUB_SUITE_MAIL_TO}" >> "$ALERT_LOG"
  else
    echo "[WARN] Neither mail nor sendmail is installed; cannot send mail alert" >> "$ALERT_LOG"
  fi
fi

exit 0
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

3. Verify Webhook & Local Mail Delivery
 1. Test the alert script directly with your configured env vars
   1 min
   Load ~/.config/systemd/user/run-github-suite-alert.env and invoke the script once to confirm webhook and/or mail delivery without waiting for a real failure.
   set -a
source "${HOME}/.config/systemd/user/run-github-suite-alert.env"
set +a
"${HOME}/bin/notify-github-suite-failure.sh" run-github-suite.service

   Verification: Your webhook endpoint receives the JSON payload, your local mail spool/inbox gets the alert message, and ${HOME}/github-archives/systemd-failures.log has no [WARN] lines.
 2. Check local mail queue if using local MTA
   1 min
   If you configured a local mail address (GITHUB_SUITE_MAIL_TO), check local mail delivery (mail or postqueue -p / journal logs for your MTA).
   tail -n 15 "${HOME}/github-archives/systemd-failures.log"

   Verification: Confirms the alert block was written cleanly and no transport warnings were logged.
> Payload & security note: Because FULL_REPORT includes recent journal lines and GPG audit output, avoid sending full raw logs over unencrypted HTTP or to shared webhook channels unless your environment and destination are trusted.
> 
To trigger an alert whenever run-github-suite.service exits non-zero (or fails), add OnFailure=run-github-suite-failure@%i.service (or a non-template OnFailure=run-github-suite-alert.service) to the [Unit] section of your systemd user service and define a companion oneshot unit under ~/.config/systemd/user/.
Because run-github-suite.service runs in your user systemd manager (systemctl --user), the OnFailure= target unit must also live in ~/.config/systemd/user/ so the user manager can resolve and start it.
1. Add OnFailure= to run-github-suite.service
Edit ~/.config/systemd/user/run-github-suite.service and include OnFailure= under [Unit] (using a drop-in override file is even cleaner so you don't overwrite your base unit):
mkdir -p "${HOME}/.config/systemd/user/run-github-suite.service.d"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite.service.d/on-failure.conf"
[Unit]
OnFailure=run-github-suite-alert.service
EOF

(If you run a template service like run-github-suite@.service, use OnFailure=run-github-suite-alert@%i.service in that unit instead so %i passes the instance name.)
2. Create the Alert Handler Script & OnFailure User Unit
Create a local alert script (~/bin/notify-github-suite-failure.sh) that captures recent journalctl --user lines for run-github-suite.service and dispatches whatever alert mechanism you prefer (for example notify-send for desktop notifications, a webhook/email/script, or appending to a dedicated failure log).
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
ALERT_LOG="${HOME}/github-archives/systemd-failures.log"
mkdir -p "$(dirname "$ALERT_LOG")"

RECENT_LOGS="$(
  journalctl --user -u "$UNIT_NAME" -n 25 --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

SUMMARY="FAILURE in ${UNIT_NAME} on ${HOST_NAME} at $(date -u +"%Y-%m-%dT%H:%M:%SZ")"

{
  echo "============================================================"
  echo "$SUMMARY"
  echo "------------------------------------------------------------"
  printf '%s\n' "$RECENT_LOGS"
  echo ""
} >> "$ALERT_LOG"

# Desktop notification if running inside a graphical session
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" "${UNIT_NAME} exited non-zero on ${HOST_NAME}. See ${ALERT_LOG}" || true
fi

# Optional webhook example (uncomment and set WEBHOOK_URL if desired):
# if [ -n "${WEBHOOK_URL:-}" ] && command -v curl >/dev/null 2>&1; then
#   curl -fsS -X POST -H "Content-Type: application/json" \
#     -d "$(jq -n --arg text "$SUMMARY" --arg logs "$RECENT_LOGS" '{text: ($text + "\n```\n" + $logs + "\n```")}')" \
#     "$WEBHOOK_URL" || true
# fi
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

Now create ~/.config/systemd/user/run-github-suite-alert.service:
cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.service"
[Unit]
Description=Alert when run-github-suite.service exits non-zero
After=run-github-suite.service

[Service]
Type=oneshot
ExecStart=%h/bin/notify-github-suite-failure.sh run-github-suite.service
EOF

3. Reload Systemd & Test Non-Zero Exit Handling
 1. Reload the systemd user daemon
   1 min
   Reload user unit definitions so run-github-suite.service picks up the OnFailure= drop-in and run-github-suite-alert.service is registered.
   systemctl --user daemon-reload
systemctl --user cat run-github-suite.service | grep -n "OnFailure"

   Verification: grep prints OnFailure=run-github-suite-alert.service.
 2. Verify failure trigger behavior
   1 min
   Check systemctl --user status run-github-suite-alert.service after a failed run (or test your alert script directly with ~/bin/notify-github-suite-failure.sh run-github-suite.service).
   ~/bin/notify-github-suite-failure.sh run-github-suite.service
tail -n 20 "${HOME}/github-archives/systemd-failures.log"
systemctl --user status run-github-suite-alert.service --no-pager || true

   Verification: ${HOME}/github-archives/systemd-failures.log records the timestamped failure block and recent journal lines for run-github-suite.service.
 3. Ensure user services run even when logged out (if needed)
   1 min
   If run-github-suite.timer runs unattended while you are logged out of an interactive session, verify lingering is enabled for your user account (loginctl show-user "$USER" -p Linger).
   loginctl show-user "$USER" -p Linger

   Verification: Shows Linger=yes if background user units are enabled across logouts (enable with sudo loginctl enable-linger "$USER" if needed).
To add automated SHA-256 checksum verification and rclone check validation directly into ~/bin/run-github-suite.sh, wire a dedicated verify_snapshot_integrity() function right after your backup/sync steps so a corrupted archive or remote mismatch fails the suite run loudly instead of going unnoticed.
1. Drop-In Verification Function & run-github-suite.sh Integration
Set RCLONE_REMOTE_BASE (for example crypt-github:github-archives) and SNAP_DIR to the snapshot directory created during that run. If your SHA256SUMS file uses relative paths inside the snapshot folder, run sha256sum -c inside SNAP_DIR; if it contains paths relative to ARCHIVE_ROOT, run sha256sum -c from ARCHIVE_ROOT.
verify_snapshot_integrity() {
  local snap_dir="$1"
  local archive_root="$2"
  local rclone_remote_base="${3:-}"

  if [ ! -d "$snap_dir" ]; then
    echo "[ERROR] Snapshot directory not found: ${snap_dir}" >&2
    return 1
  fi

  local sums_file="${snap_dir}/SHA256SUMS"
  if [ ! -f "$sums_file" ]; then
    echo "[ERROR] SHA256SUMS manifest missing in: ${snap_dir}" >&2
    return 1
  fi

  echo "[VERIFY] Checking local SHA-256 manifest (${sums_file})..."
  if (cd "$snap_dir" && sha256sum --status -c "$sums_file"); then
    echo "[PASS] Local SHA-256 checksums verified for $(basename "$snap_dir")"
  elif (cd "$archive_root" && sha256sum --status -c "$sums_file"); then
    echo "[PASS] Local SHA-256 checksums verified from ${archive_root}"
  else
    echo "[FAIL] SHA-256 checksum mismatch in ${snap_dir}" >&2
    return 1
  fi

  if [ -n "$rclone_remote_base" ]; then
    command -v rclone >/dev/null 2>&1 || {
      echo "[ERROR] rclone is required when RCLONE_REMOTE_BASE is set" >&2
      return 1
    }

    local snap_name
    snap_name="$(basename "$snap_dir")"
    local remote_target="${rclone_remote_base%/}/${snap_name}"

    echo "[VERIFY] Running rclone check against ${remote_target}..."
    if rclone check "$snap_dir" "$remote_target" --one-way --checkers=8; then
      echo "[PASS] Remote vault matches local snapshot (${snap_name})"
    else
      echo "[FAIL] rclone check detected differences between ${snap_dir} and ${remote_target}" >&2
      return 1
    fi
  else
    echo "[SKIP] RCLONE_REMOTE_BASE not configured; skipped remote rclone check"
  fi
}

2. Example ~/bin/run-github-suite.sh Flow
Here is how the verification step fits cleanly after backup, metadata export, GPG audit, and rclone sync:
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${SRC_ROOT:-${HOME}/src/github}"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
RCLONE_REMOTE_BASE="${RCLONE_REMOTE_BASE:-}" # e.g., "crypt-github:github-archives"

"${HOME}/bin/backup-github.sh"
"${HOME}/bin/export-github-metadata.sh"
"${HOME}/bin/audit-gpg-signatures.sh" "$SRC_ROOT"

LATEST_SNAP="$(
  find "$ARCHIVE_ROOT" -mindepth 1 -maxdepth 1 -type d \
    ! -name 'pre-resign-*' ! -name 'pre-tag-resign-*' \
    | sort | tail -n 1
)"

if [ -n "$RCLONE_REMOTE_BASE" ] && [ -n "$LATEST_SNAP" ]; then
  SNAP_NAME="$(basename "$LATEST_SNAP")"
  rclone copy "$LATEST_SNAP" "${RCLONE_REMOTE_BASE%/}/${SNAP_NAME}" \
    --transfers=4 --checkers=8 --fast-list
fi

verify_snapshot_integrity "$LATEST_SNAP" "$ARCHIVE_ROOT" "$RCLONE_REMOTE_BASE"

3. Rollout & Verification Steps
 1. Add verify_snapshot_integrity to run-github-suite.sh
   1 min
   Paste verify_snapshot_integrity into ~/bin/run-github-suite.sh, set RCLONE_REMOTE_BASE in your environment or script config, and ensure ~/bin/run-github-suite.sh is executable (chmod +x ~/bin/run-github-suite.sh).
   Verification: Running bash -n ~/bin/run-github-suite.sh exits 0 with no syntax errors.
 2. Run the suite with RCLONE_REMOTE_BASE set
   1-3 min
   Execute ~/bin/run-github-suite.sh with your remote target configured so both local checksums and remote vault objects are verified in one pass.
   RCLONE_REMOTE_BASE="YOUR_RCLONE_REMOTE:github-archives" ~/bin/run-github-suite.sh

   Verification: Output prints [PASS] Local SHA-256 checksums verified... and [PASS] Remote vault matches local snapshot... and exits with status 0.
 3. Confirm failure behavior on a mismatched test file
   1 min
   Temporarily test on a non-production copy or inspect exit status 1 behavior when a file fails sha256sum -c or rclone check so systemd alerts/logs catch integrity failures immediately.
   Verification: Any checksum or remote mismatch exits 1 before the run is treated as healthy.
> Crypt remotes & rclone check note: On crypt remotes, rclone check uses size and modification time/hash checks supported by the backend (--one-way verifies every file in snap_dir exists and matches on the remote without complaining about extra remote files).
> 
backup-github.shresign-annotated-tags.shHere is a script (~/bin/resign-annotated-tags.sh) that scans repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep), identifies annotated tags (objecttype == tag) that lack a valid signature (git verify-tag --raw), backs up a compressed bare mirror first, re-signs each unsigned/invalid annotated tag onto its original target object while preserving its tagger message and date, and optionally force-pushes updated tag refs with --force-with-lease --force-if-includes.
Lightweight tags (objecttype = commit or tree) are left untouched because they are plain pointers rather than tag objects with OpenPGP signatures.
1. Batch Annotated Tag Re-Signer (~/bin/resign-annotated-tags.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub accounts (default: "re1Lucky365 adpsnake7-beep")
  --src-root PATH            Local working clones root (default: "${HOME}/src/github")
  --key-id ID                GPG signing key ID (default: git config user.signingkey)
  --repos "a b"              Optional space-separated repo names to restrict to
  --push                     Force-with-lease push updated tags to origin
  --dry-run                  Audit and list unsigned/invalid annotated tags without modifying anything
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"
KEY_ID="$(git config --get user.signingkey || true)"
FILTER_REPOS=""
DO_PUSH=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --key-id)
      KEY_ID="${2:?Missing value for --key-id}"
      shift 2
      ;;
    --repos)
      FILTER_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --push)
      DO_PUSH=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v git >/dev/null 2>&1 || { echo "git is required" >&2; exit 1; }
command -v gpg >/dev/null 2>&1 || { echo "gpg is required" >&2; exit 1; }

if [ -z "$KEY_ID" ]; then
  echo "ERROR: No GPG signing key specified (--key-id or git config user.signingkey)." >&2
  exit 1
fi

if ! gpg --list-secret-keys --keyid-format=long "$KEY_ID" >/dev/null 2>&1; then
  echo "ERROR: Secret key not found in local GPG keyring: $KEY_ID" >&2
  exit 1
fi

export GPG_TTY="${GPG_TTY:-$(tty 2>/dev/null || true)}"

STAMP="$(date +%Y-%m-%d_%H%M%S)"
BACKUP_ROOT="${HOME}/github-archives/pre-tag-resign-${STAMP}"
WORK_TMP="$(mktemp -d)"
trap 'rm -rf "$WORK_TMP"' EXIT

repo_matches_filter() {
  local name="$1"
  [ -z "$FILTER_REPOS" ] && return 0
  for r in $FILTER_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

tag_needs_signing() {
  local repo_dir="$1"
  local tag_sha="$2"
  local raw_out rc=0

  raw_out="$(git -C "$repo_dir" verify-tag --raw "$tag_sha" 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    return 0
  fi
  if ! printf '%s\n' "$raw_out" | grep -qE '^\[GNUPG:\] (GOODSIG|VALIDSIG)'; then
    return 0
  fi
  return 1
}

repos_scanned=0
repos_updated=0
tags_resigned=0
failed_repos=0

echo "Signing key ID  : ${KEY_ID}"
echo "Push to origin  : ${DO_PUSH}"
echo "Dry-run mode    : ${DRY_RUN}"
echo "------------------------------------------------------------"

for acct in "${ACCOUNTS[@]}"; do
  acct_src="${SRC_ROOT}/${acct}"
  acct_bak="${BACKUP_ROOT}/${acct}"

  if [ ! -d "$acct_src" ]; then
    echo "[SKIP] Account directory missing: ${acct_src}"
    continue
  fi

  for repo_dir in "${acct_src}"/*; do
    [ -d "${repo_dir}/.git" ] || continue
    repo_name="$(basename "$repo_dir")"
    repo_matches_filter "$repo_name" || continue

    repos_scanned=$((repos_scanned + 1))

    target_tags=()
    while IFS=$'\t' read -r tag_name tag_sha obj_type; do
      [ -n "$tag_name" ] || continue
      if [ "$obj_type" = "tag" ] && tag_needs_signing "$repo_dir" "$tag_sha"; then
        target_tags+=("${tag_name}:${tag_sha}")
      fi
    done < <(git -C "$repo_dir" for-each-ref --format="%(refname:short)%09%(objectname)%09%(objecttype)" refs/tags)

    if [ "${#target_tags[@]}" -eq 0 ]; then
      echo "[PASS] ${acct}/${repo_name} (no unsigned/invalid annotated tags)"
      continue
    fi

    echo "[TARGET] ${acct}/${repo_name}: ${#target_tags[@]} annotated tag(s) to re-sign"
    for entry in "${target_tags[@]}"; do
      echo "         - ${entry%%:*} (${entry##*:})"
    done

    if [ "$DRY_RUN" = true ]; then
      tags_resigned=$((tags_resigned + ${#target_tags[@]}))
      continue
    fi

    # 1. Back up repository mirror before rewriting tag objects
    mkdir -p "$acct_bak"
    mirror_dir="${WORK_TMP}/mirror_${acct}_${repo_name}.git"
    git clone --mirror "$repo_dir" "$mirror_dir"
    tar -cf - -C "$(dirname "$mirror_dir")" "$(basename "$mirror_dir")" \
      | zstd -T0 -19 -o "${acct_bak}/${repo_name}.tar.zst"

    # 2. Re-sign each unsigned/invalid annotated tag while preserving message & tagger date
    repo_fail=false
    resigned_names=()

    for entry in "${target_tags[@]}"; do
      tag_name="${entry%%:*}"
      tag_target="$(git -C "$repo_dir" rev-parse "${tag_name}^{}")"
      tag_msg="$(git -C "$repo_dir" for-each-ref --format="%(contents)" "refs/tags/${tag_name}")"
      tag_date="$(git -C "$repo_dir" for-each-ref --format="%(taggerdate:iso8601)" "refs/tags/${tag_name}")"

      # Strip trailing single newline added by %(contents) if original message didn't end with one
      if ! GIT_COMMITTER_DATE="$tag_date" git -C "$repo_dir" \
        tag -f -s -u "$KEY_ID" -m "$tag_msg" "$tag_name" "$tag_target"; then
        echo "[FAIL] ${acct}/${repo_name}: failed re-signing tag ${tag_name}" >&2
        repo_fail=true
        break
      fi

      # Verify tag signature immediately after signing
      if tag_needs_signing "$repo_dir" "refs/tags/${tag_name}"; then
        echo "[FAIL] ${acct}/${repo_name}: tag ${tag_name} still failed verification after signing" >&2
        repo_fail=true
        break
      fi

      resigned_names+=("$tag_name")
      tags_resigned=$((tags_resigned + 1))
    done

    if [ "$repo_fail" = true ]; then
      failed_repos=$((failed_repos + 1))
      continue
    fi

    # 3. Optionally push updated tag refs with --force-with-lease --force-if-includes
    if [ "$DO_PUSH" = true ]; then
      push_args=(push --force-with-lease --force-if-includes origin)
      for tname in "${resigned_names[@]}"; do
        push_args+=("refs/tags/${tname}:refs/tags/${tname}")
      done
      if ! git -C "$repo_dir" "${push_args[@]}"; then
        echo "[FAIL] ${acct}/${repo_name}: failed pushing re-signed tags to origin" >&2
        failed_repos=$((failed_repos + 1))
        continue
      fi
    fi

    echo "[OK] ${acct}/${repo_name}: re-signed ${#resigned_names[@]} tag(s)"
    repos_updated=$((repos_updated + 1))
  done
done

echo "------------------------------------------------------------"
echo "Summary: ReposScanned=${repos_scanned} | ReposUpdated=${repos_updated} | TagsResigned=${tags_resigned} | FailedRepos=${failed_repos}"
if [ -d "$BACKUP_ROOT" ]; then
  echo "Pre-tag-resign backups saved in: ${BACKUP_ROOT}"
fi

if [ "$failed_repos" -gt 0 ]; then
  exit 1
fi

2. Install, Preview & Run
 1. Save, make executable, and run a dry-run preview
   1 min
   Write the script to ~/bin/resign-annotated-tags.sh, make it executable, and run --dry-run first to list every annotated tag that needs re-signing across re1Lucky365 and adpsnake7-beep.
   chmod +x ~/bin/resign-annotated-tags.sh
~/bin/resign-annotated-tags.sh --dry-run

   Verification: Shows [TARGET] lines for repositories with unsigned/invalid annotated tags and [PASS] for clean ones without modifying any Git objects or remotes.
 2. Re-sign locally and push with --force-with-lease
   1-3 min
   Run the script with --push (or run without --push first if you want to inspect git verify-tag locally before pushing).
   ~/bin/resign-annotated-tags.sh --push

   Verification: Outputs [OK] for updated repositories, saves .tar.zst pre-rewrite mirrors under ~/github-archives/pre-tag-resign-<timestamp>/, and exits with status 0.
 3. Confirm zero tag or commit violations in your audit script
   1 min
   Run your updated ~/bin/audit-gpg-signatures.sh to verify that both commits and annotated tags pass with zero violations.
   ~/bin/audit-gpg-signatures.sh "${HOME}/src/github"

   Verification: Reports Violations: 0 and exits with status 0.
> Important notes:
>  * Existing releases & downstream consumers: Re-signing an annotated tag replaces the tag object's SHA on the remote when pushed with --force-with-lease. Anyone who previously fetched or pinned that tag SHA will see an updated tag object.
>  * Rollback: If you ever need to restore a repository's pre-run tags and refs, extract its .tar.zst mirror from ~/github-archives/pre-tag-resign-<timestamp>/ and fetch/push refs back as needed.
> 
user.signingkey0123456789Here is the updated ~/bin/audit-gpg-signatures.sh that audits both commits and annotated tags across your repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep).
Lightweight tags (objecttype = commit or tree) do not carry tag-object signatures of their own, so the script inspects refs/tags/* objects of type tag using git verify-tag --raw, checks for GOODSIG / VALIDSIG status and optional ALLOWED_KEY_IDS allowlisting, and counts any unsigned, bad, expired, revoked, or missing-key tags toward Violations.
1. Updated ~/bin/audit-gpg-signatures.sh (Commits + Annotated Tags)
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_FILE="${2:-${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log}"
mkdir -p "$(dirname "$REPORT_FILE")"

# Optional: populate with one or more 16-char or full GPG key IDs to enforce allowlisting
# Example: ALLOWED_KEY_IDS=("0123456789ABCDEF")
ALLOWED_KEY_IDS=()

total_repos=0
total_commits=0
total_tags=0
failed_items=0
SEP=$'\x1f'

is_key_allowed() {
  local key_id="${1:-}"
  if [ "${#ALLOWED_KEY_IDS[@]}" -eq 0 ]; then
    return 0
  fi

  local matched=false
  for allowed in "${ALLOWED_KEY_IDS[@]}"; do
    [ -n "$allowed" ] || continue
    if [ -n "$key_id" ] && [[ "$key_id" == *"$allowed" ]]; then
      matched=true
      break
    fi
  done

  [ "$matched" = true ]
}

verify_annotated_tag_item() {
  local repo_dir="$1"
  local repo_name="$2"
  local tag_name="$3"
  local tag_sha="$4"

  total_tags=$((total_tags + 1))

  local raw_out rc=0
  raw_out="$(git -C "$repo_dir" verify-tag --raw "$tag_sha" 2>&1)" || rc=$?

  local key_id=""
  key_id="$(
    printf '%s\n' "$raw_out" \
      | awk '/\[GNUPG:\] (VALIDSIG|GOODSIG)/ {
          if ($2 == "VALIDSIG") print $11;
          else if ($2 == "GOODSIG") print $3;
        }' | head -n 1
  )"

  local reason=""
  if [ "$rc" -ne 0 ]; then
    if printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] BADSIG'; then
      reason="BAD tag signature (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] REVKEYSIG'; then
      reason="Revoked key on tag (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPKEYSIG'; then
      reason="Expired key on tag (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPSIG'; then
      reason="Expired tag signature (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] ERRSIG'; then
      reason="Missing public key on tag (${key_id:-unknown})"
    else
      reason="Unsigned annotated tag"
    fi
  elif ! printf '%s\n' "$raw_out" | grep -qE '^\[GNUPG:\] (GOODSIG|VALIDSIG)'; then
    reason="Unsigned annotated tag"
  elif ! is_key_allowed "$key_id"; then
    reason="Unauthorized tag key (${key_id:-unknown})"
  fi

  if [ -n "$reason" ]; then
    failed_items=$((failed_items + 1))
    repo_failures=$((repo_failures + 1))
    printf "[FAIL] %-35s | tag:%-18s | %-34s | %s\n" \
      "$repo_name" "$tag_name" "$reason" "${tag_sha:0:10}" | tee -a "$REPORT_FILE"
  fi
}

{
  echo "GPG Signature Audit (Commits & Annotated Tags) - $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  echo "Scanning root: ${SRC_ROOT}"
  echo "------------------------------------------------------------"
} | tee "$REPORT_FILE"

if [ ! -d "$SRC_ROOT" ]; then
  echo "ERROR: Source directory does not exist: ${SRC_ROOT}" >&2
  exit 1
fi

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  repo_name="${repo_dir#${SRC_ROOT}/}"
  total_repos=$((total_repos + 1))

  if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
    echo "[SKIP] ${repo_name} (empty repository)" | tee -a "$REPORT_FILE"
    continue
  fi

  repo_failures=0

  # 1. Audit commits across all refs
  while IFS="$SEP" read -r hash status key_id email subject; do
    total_commits=$((total_commits + 1))
    is_valid=true
    reason=""

    case "$status" in
      G|U)
        if ! is_key_allowed "$key_id"; then
          is_valid=false
          reason="Unauthorized key (${key_id})"
        fi
        ;;
      N) is_valid=false; reason="Unsigned commit" ;;
      B) is_valid=false; reason="BAD signature" ;;
      X) is_valid=false; reason="Signature expired" ;;
      Y) is_valid=false; reason="Expired key (${key_id})" ;;
      R) is_valid=false; reason="Revoked key (${key_id})" ;;
      E) is_valid=false; reason="Missing public key (${key_id})" ;;
      *) is_valid=false; reason="Unknown status (${status})" ;;
    esac

    if [ "$is_valid" = false ]; then
      failed_items=$((failed_items + 1))
      repo_failures=$((repo_failures + 1))
      printf "[FAIL] %-35s | %-22s | %-34s | %s (%s)\n" \
        "$repo_name" "${hash:0:10}" "$reason" "$subject" "$email" | tee -a "$REPORT_FILE"
    fi
  done < <(git -C "$repo_dir" log --all --format="%H%x1f%G?%x1f%GK%x1f%ae%x1f%s")

  # 2. Audit annotated tags (objecttype == tag)
  while IFS=$'\t' read -r tag_name tag_sha obj_type; do
    [ -n "$tag_name" ] || continue
    if [ "$obj_type" = "tag" ]; then
      verify_annotated_tag_item "$repo_dir" "$repo_name" "$tag_name" "$tag_sha"
    fi
  done < <(git -C "$repo_dir" for-each-ref --format="%(refname:short)%09%(objectname)%09%(objecttype)" refs/tags)

  if [ "$repo_failures" -eq 0 ]; then
    echo "[PASS] ${repo_name}" | tee -a "$REPORT_FILE"
  fi
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

{
  echo "------------------------------------------------------------"
  echo "Repos audited: ${total_repos} | Commits checked: ${total_commits} | Annotated tags checked: ${total_tags} | Violations: ${failed_items}"
  echo "Full log saved to: ${REPORT_FILE}"
} | tee -a "$REPORT_FILE"

if [ "$failed_items" -gt 0 ]; then
  exit 1
fi

2. Install & Verify
 1. Save and run the updated audit script
   1 min
   Write the script to ~/bin/audit-gpg-signatures.sh, mark it executable, and run it against ${HOME}/src/github.
   chmod +x ~/bin/audit-gpg-signatures.sh
~/bin/audit-gpg-signatures.sh "${HOME}/src/github"

   Verification: The summary line displays Commits checked, Annotated tags checked, and Violations: 0, exiting with status 0.
 2. Check the log output for any tag or commit failures
   1 min
   Inspect the generated log file in ${HOME}/github-archives/ to verify that both commits and annotated tags are covered.
   LATEST_LOG="$(ls -t "${HOME}/github-archives"/gpg-audit-*.log | head -n 1)"
tail -n 10 "$LATEST_LOG"

   Verification: The output shows Annotated tags checked and Violations: 0 with no [FAIL] entries.
Here is the updated ~/.config/git/hooks/pre-push hook that checks both outgoing commits and pushed annotated tags (refs/tags/...).
For annotated tags, Git stores a tag object (objecttype = tag) that can carry its own OpenPGP signature (gpgsig / gpgsig-sha256). We verify the tag object signature with git verify-tag --raw "$local_sha" (parsing GOODSIG / VALIDSIG / BADSIG / EXPSIG / EXPKEYSIG / REVKEYSIG / ERRSIG and checking GPG_PRE_PUSH_ALLOWED_KEYS if set), while leaving lightweight tags (objecttype = commit or tree) to commit verification.
1. Updated ~/.config/git/hooks/pre-push (Commits + Annotated Tags)
mkdir -p "${HOME}/.config/git/hooks"

cat <<'EOF' > "${HOME}/.config/git/hooks/pre-push"
#!/usr/bin/env bash
set -euo pipefail

# Optional: comma-separated list of allowed 16-char or full GPG key IDs
# Example: GPG_PRE_PUSH_ALLOWED_KEYS="0123456789ABCDEF,FEDCBA9876543210"
ALLOWED_KEY_IDS="${GPG_PRE_PUSH_ALLOWED_KEYS:-}"

ZERO_SHA="0000000000000000000000000000000000000000"
SEP=$'\x1f'
violations=0
checked_commits=0
checked_tags=0

is_key_allowed() {
  local key_id="${1:-}"
  if [ -z "$ALLOWED_KEY_IDS" ]; then
    return 0
  fi

  local matched=false
  IFS=',' read -r -a allowed_list <<< "$ALLOWED_KEY_IDS"
  for allowed in "${allowed_list[@]}"; do
    local allowed_clean
    allowed_clean="$(printf '%s' "$allowed" | xargs)"
    [ -z "$allowed_clean" ] && continue
    if [ -n "$key_id" ] && [[ "$key_id" == *"$allowed_clean" ]]; then
      matched=true
      break
    fi
  done

  [ "$matched" = true ]
}

verify_annotated_tag() {
  local tag_ref="$1"
  local tag_sha="$2"
  local tag_name="${tag_ref#refs/tags/}"

  checked_tags=$((checked_tags + 1))

  local raw_out rc=0
  raw_out="$(git verify-tag --raw "$tag_sha" 2>&1)" || rc=$?

  # Extract signer key ID from VALIDSIG or GOODSIG line if present
  local key_id=""
  key_id="$(
    printf '%s\n' "$raw_out" \
      | awk '/\[GNUPG:\] (VALIDSIG|GOODSIG)/ {
          if ($2 == "VALIDSIG") print $11;
          else if ($2 == "GOODSIG") print $3;
        }' | head -n 1
  )"

  local reason=""
  if [ "$rc" -ne 0 ]; then
    if printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] BADSIG'; then
      reason="bad GPG signature on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] REVKEYSIG'; then
      reason="revoked signing key (${key_id:-unknown}) on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPKEYSIG'; then
      reason="expired signing key (${key_id:-unknown}) on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPSIG'; then
      reason="expired signature on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] ERRSIG'; then
      reason="missing public key or GPG verification error (${key_id:-unknown})"
    else
      reason="unsigned or unverified annotated tag"
    fi
  elif ! printf '%s\n' "$raw_out" | grep -qE '^\[GNUPG:\] (GOODSIG|VALIDSIG)'; then
    reason="unsigned or unverified annotated tag"
  elif ! is_key_allowed "$key_id"; then
    reason="tag signing key ${key_id:-unknown} not in allowed list"
  fi

  if [ -n "$reason" ]; then
    violations=$((violations + 1))
    echo "PRE-PUSH BLOCKED [${tag_ref}]: tag ${tag_name} (${tag_sha:0:10}) -> ${reason}" >&2
  fi
}

refs_input="$(cat)"
[ -n "$refs_input" ] || exit 0

while read -r local_ref local_sha remote_ref remote_sha; do
  # Skip branch/tag deletions
  if [ "$local_sha" = "$ZERO_SHA" ]; then
    continue
  fi

  # 1. If pushing a tag ref, verify annotated tag signatures
  if [[ "$local_ref" == refs/tags/* ]]; then
    obj_type="$(git cat-file -t "$local_sha" 2>/dev/null || echo "unknown")"
    if [ "$obj_type" = "tag" ]; then
      verify_annotated_tag "$local_ref" "$local_sha"
    fi
  fi

  # 2. Verify outgoing commits not yet present on remote
  if [ "$remote_sha" = "$ZERO_SHA" ]; then
    commits="$(git rev-list "$local_sha" --not --remotes 2>/dev/null || git rev-list "$local_sha")"
  else
    commits="$(git rev-list "${remote_sha}..${local_sha}" 2>/dev/null || true)"
  fi

  [ -n "$commits" ] || continue

  while IFS= read -r sha; do
    [ -n "$sha" ] || continue
    checked_commits=$((checked_commits + 1))

    IFS="$SEP" read -r status key_id subject < <(
      git log -n 1 --format="%H%x1f%G?%x1f%GK%x1f%s" "$sha" | cut -d"$SEP" -f2-
    )

    valid=false
    reason=""

    case "$status" in
      G|U) valid=true ;;
      N) reason="unsigned commit" ;;
      B) reason="bad GPG signature" ;;
      X) reason="signature expired" ;;
      Y) reason="expired signing key (${key_id})" ;;
      R) reason="revoked signing key (${key_id})" ;;
      E) reason="missing public key (${key_id})" ;;
      *) reason="unverified signature status (${status})" ;;
    esac

    if [ "$valid" = true ] && ! is_key_allowed "$key_id"; then
      valid=false
      reason="commit key ${key_id} not in allowed list"
    fi

    if [ "$valid" = false ]; then
      violations=$((violations + 1))
      echo "PRE-PUSH BLOCKED [${local_ref}]: commit ${sha:0:10} (${subject}) -> ${reason}" >&2
    fi
  done <<<"$commits"
done <<<"$refs_input"

if [ "$violations" -gt 0 ]; then
  echo "" >&2
  echo "Push rejected: ${violations} signature violation(s) across ${checked_commits} commit(s) and ${checked_tags} annotated tag(s)." >&2
  echo "Sign commits/tags (e.g., git tag -f -s ... or git rebase -S) or bypass only intentionally with --no-verify." >&2
  exit 1
fi

exit 0
EOF

chmod +x "${HOME}/.config/git/hooks/pre-push"

2. Quick Test for Annotated Tags
You can verify how the hook behaves on a local test tag before pushing real tags:
 1. Create a temporary unsigned annotated tag and dry-run push
   1 min
   Create an annotated tag without -s on a test repo and run git push --dry-run with that tag ref.
   REPO_DIR="${HOME}/src/github/re1Lucky365/YOUR_REPO"
git -C "$REPO_DIR" tag -a -m "test unsigned tag" test-unsigned-tag HEAD
git -C "$REPO_DIR" push --dry-run origin refs/tags/test-unsigned-tag || true
git -C "$REPO_DIR" tag -d test-unsigned-tag

   Verification: The dry-run push exits non-zero and prints PRE-PUSH BLOCKED [refs/tags/test-unsigned-tag]: ... unsigned or unverified annotated tag, then the temporary local tag is deleted cleanly.
 2. Test with a signed annotated tag
   1 min
   Create a signed annotated tag (git tag -s) and run git push --dry-run to confirm it passes.
   git -C "$REPO_DIR" tag -s -m "test signed tag" test-signed-tag HEAD
git -C "$REPO_DIR" push --dry-run origin refs/tags/test-signed-tag
git -C "$REPO_DIR" tag -d test-signed-tag

   Verification: git push --dry-run exits 0 with no PRE-PUSH BLOCKED output.
> Note: Since your hook is symlinked into .git/hooks/pre-push across your repositories under ${HOME}/src/github, updating ~/.config/git/hooks/pre-push automatically applies the annotated tag check to all linked repos without re-linking.
> 
A local Git pre-push hook runs right before git push sends commits or tags to a remote, letting you block unsigned or invalid-signature commits on private repositories—even on plan tiers where server-side branch rulesets aren't available.
Because pre-push hooks are client-side, the cleanest setup is a shared hook template (for example under ~/.config/git/hooks/pre-push) that you can install across all your cloned repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep), either per-repo or via core.hooksPath.
1. Create the Shared pre-push Hook Script
Git passes <remote-name> <remote-url> on command line arguments and <local-ref> <local-sha> <remote-ref> <remote-sha> on stdin for each ref being pushed. For new branches or tags where <remote-sha> is all zeros (0000...), we inspect commits reachable from <local-sha> that aren't already in any remote ref (--not --remotes).
Create ~/.config/git/hooks/pre-push:
mkdir -p "${HOME}/.config/git/hooks"

cat <<'EOF' > "${HOME}/.config/git/hooks/pre-push"
#!/usr/bin/env bash
set -euo pipefail

# Optional: set to a comma-separated list of allowed 16-char GPG key IDs
# Example: ALLOWED_KEY_IDS="0123456789ABCDEF,FEDCBA9876543210"
ALLOWED_KEY_IDS="${GPG_PRE_PUSH_ALLOWED_KEYS:-}"

ZERO_SHA="0000000000000000000000000000000000000000"
SEP=$'\x1f'
violations=0
checked=0

# Read pushed refs from stdin
refs_input="$(cat)"
[ -n "$refs_input" ] || exit 0

while read -r local_ref local_sha remote_ref remote_sha; do
  # Skip branch/tag deletions
  if [ "$local_sha" = "$ZERO_SHA" ]; then
    continue
  fi

  # Determine commit range not yet on remote
  if [ "$remote_sha" = "$ZERO_SHA" ]; then
    range_spec="${local_sha}"
    commits="$(git rev-list "$range_spec" --not --remotes 2>/dev/null || git rev-list "$range_spec")"
  else
    range_spec="${remote_sha}..${local_sha}"
    commits="$(git rev-list "$range_spec" 2>/dev/null || true)"
  fi

  [ -n "$commits" ] || continue

  while IFS= read -r sha; do
    [ -n "$sha" ] || continue
    checked=$((checked + 1))

    IFS="$SEP" read -r status key_id subject < <(
      git log -n 1 --format="%H%x1f%G?%x1f%GK%x1f%s" "$sha" | cut -d"$SEP" -f2-
    )

    valid=false
    reason=""

    case "$status" in
      G|U) valid=true ;;
      N) reason="unsigned commit" ;;
      B) reason="bad GPG signature" ;;
      X) reason="signature expired" ;;
      Y) reason="expired signing key (${key_id})" ;;
      R) reason="revoked signing key (${key_id})" ;;
      E) reason="missing public key (${key_id})" ;;
      *) reason="unverified signature status (${status})" ;;
    esac

    if [ "$valid" = true ] && [ -n "$ALLOWED_KEY_IDS" ]; then
      matched=false
      IFS=',' read -r -a allowed_list <<< "$ALLOWED_KEY_IDS"
      for allowed in "${allowed_list[@]}"; do
        allowed_clean="$(printf '%s' "$allowed" | xargs)"
        [ -z "$allowed_clean" ] && continue
        if [[ "$key_id" == *"$allowed_clean" ]]; then
          matched=true
          break
        fi
      done
      if [ "$matched" = false ]; then
        valid=false
        reason="key ${key_id} not in allowed list"
      fi
    fi

    if [ "$valid" = false ]; then
      violations=$((violations + 1))
      echo "PRE-PUSH BLOCKED [${local_ref}]: commit ${sha:0:10} (${subject}) -> ${reason}" >&2
    fi
  done <<<"$commits"
done <<<"$refs_input"

if [ "$violations" -gt 0 ]; then
  echo "" >&2
  echo "Push rejected: ${violations} of ${checked} outgoing commit(s) failed GPG signature verification." >&2
  echo "Re-sign commits (e.g., git rebase -S or resign-github-repos.sh) or run with --no-verify only if intentional." >&2
  exit 1
fi

exit 0
EOF

chmod +x "${HOME}/.config/git/hooks/pre-push"

2. Install the Hook Across Your Repositories
You can either install the hook into .git/hooks/pre-push for each repository under ${HOME}/src/github (useful if you also use other repo-specific hooks) or point Git's core.hooksPath to ~/.config/git/hooks.
 1. Link the pre-push hook into all repos under ~/src/github
   1 min
   Symlink ~/.config/git/hooks/pre-push into every repository's .git/hooks/pre-push across re1Lucky365 and adpsnake7-beep.
   HOOK_SRC="${HOME}/.config/git/hooks/pre-push"
SRC_ROOT="${HOME}/src/github"

while IFS= read -r -d '' gitdir; do
  hooks_dir="${gitdir}/hooks"
  mkdir -p "$hooks_dir"
  ln -sf "$HOOK_SRC" "${hooks_dir}/pre-push"
  echo "Linked pre-push hook -> ${gitdir}/hooks/pre-push"
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

   Verification: Running ls -la ~/src/github/re1Lucky365/YOUR_REPO/.git/hooks/pre-push shows a symlink pointing to ~/.config/git/hooks/pre-push with executable permissions (x).
 2. Test against a simulated push before pushing real changes
   1 min
   Dry-run a push (git push --dry-run) on a branch with signed commits to confirm it passes cleanly.
   REPO_DIR="${HOME}/src/github/re1Lucky365/YOUR_REPO"
git -C "$REPO_DIR" push --dry-run

   Verification: The push completes without any PRE-PUSH BLOCKED message; if an outgoing commit is unsigned, git push exits non-zero and prints the commit hash and failure reason.
> Important caveats:
>  * Local-only protection: Because pre-push hooks live on the local machine, any commit pushed from another machine, via the GitHub web editor, or with git push --no-verify bypasses the local hook. Pair it with your audit script and CI check for stronger coverage.
>  * Tags: The script above inspects commits (git rev-list). If you push annotated tags pointing to unsigned commits, those underlying commits are still checked when reachable from pushed refs.
> 
For private repositories, repository rulesets require a paid GitHub plan tier—specifically GitHub Pro for personal accounts, or GitHub Team / GitHub Enterprise for organization-owned repositories. On GitHub Free (both personal and free organizations), rulesets and branch protection rules apply to public repositories rather than private ones, which commonly results in a 403 or 404 when attempting to create or manage rulesets on a private repository.
Plan Requirements for Rulesets on Private Repositories
| Repository Owner Type | Free Tier Behavior | Paid Tier Required for Private Repo Rulesets |
|---|---|---|
| Personal Account (re1Lucky365, adpsnake7-beep) | Rulesets are available on public repositories; private repos lack ruleset enforcement | GitHub Pro |
| Organization Account | Rulesets are available on public repositories; private repos lack ruleset enforcement | GitHub Team or GitHub Enterprise |
What You Can Do If the Private Repo Is on a Free Tier
If a private repository is on a free plan and you don't want to upgrade the account or organization tier right away, you still have practical alternatives:
 * Run your local & CI checks without rulesets: Keep .github/workflows/verify-gpg-signatures.yml running on pushes and pull requests so unsigned or invalid-key commits fail the workflow check visibly, and keep running ~/bin/audit-gpg-signatures.sh locally during your sync/backup runs.
 * Use local Git hooks (pre-push): Add a client-side pre-push hook in your local clones that runs a signature check before pushing branches or tags (while keeping in mind client-side hooks can be bypassed locally if someone doesn't install them).
 * Upgrade only when server-side blocking on private branches is required: Upgrading the personal account to GitHub Pro (or the owning organization to GitHub Team / Enterprise) unlocks native branch/tag rulesets and required status checks on private repositories.
My gh api rulesets call returned a 403 or 404 error. What permissions or plan requirements should I check?Here is a Bash script (~/bin/apply-gpg-ruleset.sh) that loops over repositories in re1Lucky365 and adpsnake7-beep using gh api, checks whether a ruleset named Require Signed Commits & GPG Audit already exists (updating it via PATCH or creating it via POST), and supports --dry-run and fork/archived filtering.
1. Ruleset Rollout Script (~/bin/apply-gpg-ruleset.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub accounts (default: "re1Lucky365 adpsnake7-beep")
  --include-forks            Include forked repositories (default: skip forks)
  --include-archived         Include archived repositories (default: skip archived)
  --dry-run                  Preview target repositories and actions without modifying GitHub
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
INCLUDE_FORKS=false
INCLUDE_ARCHIVED=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --include-forks)
      INCLUDE_FORKS=true
      shift
      ;;
    --include-archived)
      INCLUDE_ARCHIVED=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown option: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "jq is required" >&2; exit 1; }

RULESET_NAME="Require Signed Commits & GPG Audit"

PAYLOAD_FILE="$(mktemp)"
trap 'rm -f "$PAYLOAD_FILE"' EXIT

cat <<'EOF' > "$PAYLOAD_FILE"
{
  "name": "Require Signed Commits & GPG Audit",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_signatures"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          {
            "context": "Audit Commit GPG Signatures"
          }
        ]
      }
    }
  ]
}
EOF

applied=0
updated=0
skipped=0
failed=0

for acct in "${ACCOUNTS[@]}"; do
  echo "============================================================"
  echo "Account: ${acct}"
  echo "============================================================"

  repos_json="$(gh repo list "$acct" --limit 1000 --json name,isFork,isArchived)"

  while IFS=$'\t' read -r repo is_fork is_archived; do
    [ -n "$repo" ] || continue

    if [ "$INCLUDE_FORKS" = false ] && [ "$is_fork" = "true" ]; then
      echo "[SKIP] ${acct}/${repo} (fork)"
      skipped=$((skipped + 1))
      continue
    fi

    if [ "$INCLUDE_ARCHIVED" = false ] && [ "$is_archived" = "true" ]; then
      echo "[SKIP] ${acct}/${repo} (archived)"
      skipped=$((skipped + 1))
      continue
    fi

    existing_id="$(
      gh api "repos/${acct}/${repo}/rulesets" \
        --jq ".[] | select(.name == \"${RULESET_NAME}\") | .id" 2>/dev/null | head -n 1 || true
    )"

    if [ "$DRY_RUN" = true ]; then
      if [ -n "$existing_id" ]; then
        echo "[DRY-RUN] Would update ruleset #${existing_id} on ${acct}/${repo}"
      else
        echo "[DRY-RUN] Would create ruleset on ${acct}/${repo}"
      fi
      continue
    fi

    if [ -n "$existing_id" ]; then
      if gh api --method PUT "repos/${acct}/${repo}/rulesets/${existing_id}" --input "$PAYLOAD_FILE" >/dev/null; then
        echo "[UPDATED] ${acct}/${repo} (ruleset #${existing_id})"
        updated=$((updated + 1))
      else
        echo "[FAIL] ${acct}/${repo} (failed updating ruleset #${existing_id})" >&2
        failed=$((failed + 1))
      fi
    else
      if gh api --method POST "repos/${acct}/${repo}/rulesets" --input "$PAYLOAD_FILE" >/dev/null; then
        echo "[CREATED] ${acct}/${repo}"
        applied=$((applied + 1))
      else
        echo "[FAIL] ${acct}/${repo} (failed creating ruleset—check repo visibility/plan/permissions)" >&2
        failed=$((failed + 1))
      fi
    fi
  done < <(printf '%s' "$repos_json" | jq -r '.[] | [.name, (.isFork|tostring), (.isArchived|tostring)] | @tsv')
done

echo "------------------------------------------------------------"
echo "Summary: Created=${applied} | Updated=${updated} | Skipped=${skipped} | Failed=${failed}"

if [ "$failed" -gt 0 ]; then
  exit 1
fi

2. Run & Verify
 1. Save, make executable, and preview with --dry-run
   1 min
   Save the script to ~/bin/apply-gpg-ruleset.sh, set execute permissions, and run a dry run first so you see which repositories will be touched.
   chmod +x ~/bin/apply-gpg-ruleset.sh
~/bin/apply-gpg-ruleset.sh --dry-run

   Verification: Lists [DRY-RUN] lines for non-fork, non-archived repositories across re1Lucky365 and adpsnake7-beep without modifying GitHub.
 2. Apply the ruleset across repositories
   1-3 min
   Run the script without --dry-run to create or update the ruleset on each repository.
   ~/bin/apply-gpg-ruleset.sh

   Verification: Reports [CREATED] or [UPDATED] for target repositories and finishes with Failed: 0.
 3. Confirm ruleset enforcement on a sample repository
   1 min
   Spot-check a repository ruleset via gh api to verify enforcement is "active" and both required_signatures and required_status_checks (Audit Commit GPG Signatures) are present.
   gh api "repos/re1Lucky365/YOUR_REPO/rulesets" --jq '.[] | {id, name, enforcement, rules: [.rules[].type]}'

   Verification: Output shows "enforcement": "active" and rules including "required_signatures" and "required_status_checks".
> Account & plan note: Ensure your active gh auth status token has admin/ruleset permissions on both re1Lucky365 and adpsnake7-beep, and keep in mind that GitHub repository ruleset availability/features on private repositories depend on the repository's GitHub plan tier.
> 
To enforce both GitHub's native signed commits rule and your custom Audit Commit GPG Signatures GitHub Actions status check on a repository (such as those under re1Lucky365 or adpsnake7-beep), configure a GitHub Repository Ruleset targeting your default/main branch.
Combining both gives you two layers of enforcement:
 * GitHub's native required_signatures rule rejects commits/pushes that lack a valid signature recognized by GitHub.
 * Your Audit Commit GPG Signatures status check runs your exact GPG keyring and ALLOWED_GPG_KEY_IDS policy against the commit range.
1. Configure via GitHub CLI (gh api)
Replace OWNER (re1Lucky365 or adpsnake7-beep) and REPO with your repository name. This creates an active branch ruleset on ~DEFAULT_BRANCH requiring signed commits, requiring the Audit Commit GPG Signatures check, and blocking force pushes (non_fast_forward).
OWNER="re1Lucky365"
REPO="YOUR_REPO"

gh api --method POST "repos/${OWNER}/${REPO}/rulesets" --input - <<'EOF'
{
  "name": "Require Signed Commits & GPG Audit",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_signatures"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          {
            "context": "Audit Commit GPG Signatures"
          }
        ]
      }
    }
  ]
}
EOF

(Note: If you ever need to intentionally force-push a re-signed history to a protected branch, temporarily set "enforcement": "disabled" or "evaluate" first, push with --force-with-lease, and set "enforcement": "active" back on.)
2. Configure via the GitHub Web UI
 1. Open Repository Rulesets
   1 min
   Go to your repository on GitHub, click Settings -> Rules -> Rulesets, and select New branch ruleset.
 2. Set Ruleset Name, Status & Target Branches
   1 min
   Name the ruleset Require Signed Commits & GPG Audit, set Enforcement status to Active, and under Target branches click Add target -> Include default branch (~DEFAULT_BRANCH).
 3. Enable Signed Commits and Required Status Check
   2 min
   Under Rules, check Require signed commits and Block force pushes. Then check Require status checks to pass before merging, click Add checks, search for and select Audit Commit GPG Signatures, and click Create.
3. Verify the Ruleset & Check Context Name
 * List active rulesets:
   gh api "repos/${OWNER}/${REPO}/rulesets" --jq '.[] | {id, name, enforcement}'

   Verification: Your ruleset appears with "enforcement": "active".
 * Confirm exact check context match: In .github/workflows/verify-gpg-signatures.yml, the job name (Audit Commit GPG Signatures) must match the context string in required_status_checks character-for-character so GitHub doesn't wait indefinitely for a missing check name.
build-and-audit-gpg.shHi Robert—here is a ready-to-build, self-contained GPG Signature Verification & Audit Builder (~/bin/build-and-audit-gpg.sh) tailored to your repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep).
Because I don't have direct shell access to your local Linux workstation, this script writes and enables ~/bin/audit-gpg-signatures.sh for you on your machine and immediately runs the audit across all cloned repositories, reporting Violations: 0 or printing any offending commits.
1. Create and Run the Self-Contained Builder (~/bin/build-and-audit-gpg.sh)
cat <<'EOF' > "${HOME}/bin/build-and-audit-gpg.sh"
#!/usr/bin/env bash
set -euo pipefail

AUDIT_SCRIPT="${HOME}/bin/audit-gpg-signatures.sh"
SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_DIR="${HOME}/github-archives"
STAMP="$(date +%Y-%m-%d_%H%M%S)"
REPORT_FILE="${REPORT_DIR}/gpg-audit-${STAMP}.log"

mkdir -p "${HOME}/bin" "${REPORT_DIR}"

# Optional: set one or more 16-character GPG key IDs to enforce exact key allowlisting
# Example: ALLOWED_KEY_IDS=("0123456789ABCDEF")
ALLOWED_KEY_IDS=()

cat <<'AUDITSCRIPT' > "$AUDIT_SCRIPT"
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_FILE="${2:-${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log}"
mkdir -p "$(dirname "$REPORT_FILE")"

# Populate with your 16-char GPG key ID(s) if you want strict allowlist enforcement
ALLOWED_KEY_IDS=()

total_repos=0
total_commits=0
failed_commits=0
SEP=$'\x1f'

{
  echo "GPG Signature Audit - $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  echo "Scanning root: ${SRC_ROOT}"
  echo "------------------------------------------------------------"
} | tee "$REPORT_FILE"

if [ ! -d "$SRC_ROOT" ]; then
  echo "ERROR: Source directory does not exist: ${SRC_ROOT}" >&2
  exit 1
fi

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  repo_name="${repo_dir#${SRC_ROOT}/}"
  total_repos=$((total_repos + 1))

  if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
    echo "[SKIP] ${repo_name} (empty repository)" | tee -a "$REPORT_FILE"
    continue
  fi

  repo_failures=0

  while IFS="$SEP" read -r hash status key_id email subject; do
    total_commits=$((total_commits + 1))
    is_valid=true
    reason=""

    case "$status" in
      G|U)
        if [ "${#ALLOWED_KEY_IDS[@]}" -gt 0 ]; then
          key_match=false
          for allowed in "${ALLOWED_KEY_IDS[@]}"; do
            if [[ "$key_id" == *"$allowed" ]]; then
              key_match=true
              break
            fi
          done
          if [ "$key_match" = false ]; then
            is_valid=false
            reason="Unauthorized key (${key_id})"
          fi
        fi
        ;;
      N) is_valid=false; reason="Unsigned commit" ;;
      B) is_valid=false; reason="BAD signature" ;;
      X) is_valid=false; reason="Signature expired" ;;
      Y) is_valid=false; reason="Expired key (${key_id})" ;;
      R) is_valid=false; reason="Revoked key (${key_id})" ;;
      E) is_valid=false; reason="Missing public key (${key_id})" ;;
      *) is_valid=false; reason="Unknown status (${status})" ;;
    esac

    if [ "$is_valid" = false ]; then
      failed_commits=$((failed_commits + 1))
      repo_failures=$((repo_failures + 1))
      printf "[FAIL] %-35s | %s | %-34s | %s (%s)\n" \
        "$repo_name" "${hash:0:10}" "$reason" "$subject" "$email" | tee -a "$REPORT_FILE"
    fi
  done < <(git -C "$repo_dir" log --all --format="%H%x1f%G?%x1f%GK%x1f%ae%x1f%s")

  if [ "$repo_failures" -eq 0 ]; then
    echo "[PASS] ${repo_name}" | tee -a "$REPORT_FILE"
  fi
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

{
  echo "------------------------------------------------------------"
  echo "Repos audited: ${total_repos} | Commits checked: ${total_commits} | Violations: ${failed_commits}"
  echo "Full log saved to: ${REPORT_FILE}"
} | tee -a "$REPORT_FILE"

if [ "$failed_commits" -gt 0 ]; then
  exit 1
fi
AUDITSCRIPT

chmod +x "$AUDIT_SCRIPT"
echo "Built and made executable: ${AUDIT_SCRIPT}"
echo "Running initial audit across ${SRC_ROOT}..."
"$AUDIT_SCRIPT" "$SRC_ROOT" "$REPORT_FILE"
EOF

chmod +x "${HOME}/bin/build-and-audit-gpg.sh"
"${HOME}/bin/build-and-audit-gpg.sh" "${HOME}/src/github"

2. Confirm Zero Violations After Running
 * Clean pass: The script exits with code 0, marks non-empty repos under re1Lucky365 and adpsnake7-beep as [PASS], and prints Violations: 0.
 * Violations found: Any commit with status N (unsigned), B (bad), X/Y (expired), R (revoked), or E (missing public key) prints a [FAIL] line with the short commit hash and reason, and the command exits 1.
 * Log location: Every run saves a timestamped report under ${HOME}/github-archives/gpg-audit-<timestamp>.log.
I don't have a local shell or filesystem on your machine to run commands directly—I'm running inside a cloud environment without access to your local workstation or ${HOME}/src/github.
You can run the verification steps directly on your Linux machine using the commands below to confirm zero GPG signature violations across re1Lucky365 and adpsnake7-beep.
1. Run the GPG Audit Across All Cloned RepositoriesRun or explain how to verify zero signature violations across all cloned repositories after re-signing.Rewriting Git history across multiple repositories (re1Lucky365 and adpsnake7-beep) is high-impact because every re-signed commit gets a new SHA-1/SHA-256 hash. To keep this safe, the script below:
 * Creates a pre-rewrite bare mirror bundle/archive of each repository before touching anything.
 * Operates in a temporary working directory (mktemp -d) so a failed rewrite never leaves a half-broken working tree in ${HOME}/src/github.
 * Uses git filter-repo --force --commit-callback to re-sign commits that lack a valid signature (%G? not in G|U) while preserving author/committer metadata and existing valid signatures, then re-signs annotated tags when requested.
 * Verifies signatures locally with %G? before pushing, and pushes with --force-with-lease --force-if-includes (supporting --dry-run first).
Batch Re-Sign Script (~/bin/resign-github-repos.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub accounts (default: "re1Lucky365 adpsnake7-beep")
  --src-root PATH            Local working clones root (default: "${HOME}/src/github")
  --key-id ID                GPG signing key ID (default: git config user.signingkey)
  --repos "a b"              Optional space-separated repo names to restrict to
  --sign-tags                Also re-sign annotated tags with the signing key
  --dry-run                  Audit and plan re-signing without rewriting or pushing
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"
KEY_ID="$(git config --get user.signingkey || true)"
FILTER_REPOS=""
SIGN_TAGS=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --key-id)
      KEY_ID="${2:?Missing value for --key-id}"
      shift 2
      ;;
    --repos)
      FILTER_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --sign-tags)
      SIGN_TAGS=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v git >/dev/null 2>&1 || { echo "git is required" >&2; exit 1; }
command -v gpg >/dev/null 2>&1 || { echo "gpg is required" >&2; exit 1; }
command -v git-filter-repo >/dev/null 2>&1 || { echo "git-filter-repo is required (install via pip/apt/brew)" >&2; exit 1; }

if [ -z "$KEY_ID" ]; then
  echo "ERROR: No GPG signing key specified (--key-id or git config user.signingkey)." >&2
  exit 1
fi

if ! gpg --list-secret-keys --keyid-format=long "$KEY_ID" >/dev/null 2>&1; then
  echo "ERROR: Secret key not found in local GPG keyring: $KEY_ID" >&2
  exit 1
fi

export GPG_TTY="${GPG_TTY:-$(tty 2>/dev/null || true)}"

STAMP="$(date +%Y-%m-%d_%H%M%S)"
BACKUP_ROOT="${HOME}/github-archives/pre-resign-${STAMP}"
WORK_TMP="$(mktemp -d)"
trap 'rm -rf "$WORK_TMP"' EXIT

mkdir -p "$BACKUP_ROOT"

repo_matches_filter() {
  local name="$1"
  if [ -z "$FILTER_REPOS" ]; then
    return 0
  fi
  for r in $FILTER_REPOS; do
    if [ "$r" = "$name" ]; then
      return 0
    fi
  done
  return 1
}

total_repos=0
rewritten_repos=0
skipped_repos=0
failed_repos=0

echo "Pre-resign backup directory : ${BACKUP_ROOT}"
echo "Signing key ID              : ${KEY_ID}"
echo "Dry-run mode                : ${DRY_RUN}"
echo "------------------------------------------------------------"

for acct in "${ACCOUNTS[@]}"; do
  acct_src="${SRC_ROOT}/${acct}"
  acct_bak="${BACKUP_ROOT}/${acct}"

  if [ ! -d "$acct_src" ]; then
    echo "[SKIP] Account directory missing: ${acct_src}"
    continue
  fi

  mkdir -p "$acct_bak"

  for repo_dir in "${acct_src}"/*; do
    [ -d "${repo_dir}/.git" ] || continue
    repo_name="$(basename "$repo_dir")"
    repo_matches_filter "$repo_name" || continue

    total_repos=$((total_repos + 1))
    remote_url="$(git -C "$repo_dir" remote get-url origin 2>/dev/null || echo "git@github.com:${acct}/${repo_name}.git")"

    # Skip empty repos with no commits
    if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
      echo "[SKIP] ${acct}/${repo_name} (empty repository)"
      skipped_repos=$((skipped_repos + 1))
      continue
    fi

    # Count commits without good/trusted GPG status (G or U)
    unsigned_count="$(
      git -C "$repo_dir" log --all --format="%G?" | grep -cvE '^[GU]$' || true
    )"

    if [ "$unsigned_count" -eq 0 ]; then
      echo "[PASS] ${acct}/${repo_name} (all commits already have valid GPG signatures)"
      skipped_repos=$((skipped_repos + 1))
      continue
    fi

    echo "[TARGET] ${acct}/${repo_name}: ${unsigned_count} commit(s) need signing"

    if [ "$DRY_RUN" = true ]; then
      rewritten_repos=$((rewritten_repos + 1))
      continue
    fi

    # 1. Create pre-rewrite compressed bare mirror backup
    mirror_dir="${WORK_TMP}/mirror_${acct}_${repo_name}.git"
    git clone --mirror "$repo_dir" "$mirror_dir"
    tar -cf - -C "$(dirname "$mirror_dir")" "$(basename "$mirror_dir")" \
      | zstd -T0 -19 -o "${acct_bak}/${repo_name}.tar.zst"

    # 2. Clone into an isolated working dir for rewriting
    stage_dir="${WORK_TMP}/stage_${acct}_${repo_name}"
    git clone --no-local "$mirror_dir" "$stage_dir"
    git -C "$stage_dir" config user.signingkey "$KEY_ID"
    git -C "$stage_dir" config commit.gpgsign true

    # Build a lookup file of commit hashes that currently lack valid G/U signature
    unsigned_shas_file="${WORK_TMP}/unsigned_${acct}_${repo_name}.txt"
    git -C "$stage_dir" log --all --format="%H%x09%G?" \
      | awk '$2 != "G" && $2 != "U" { print $1 }' > "$unsigned_shas_file"

    # 3. Rewrite history with git-filter-repo, signing non-G/U commits
    if ! (
      cd "$stage_dir"
      UNSIGNED_SHAS_FILE="$unsigned_shas_file" git filter-repo --force --commit-callback '
import os, subprocess

sha_file = os.environ["UNSIGNED_SHAS_FILE"]
with open(sha_file, "r", encoding="utf-8") as fh:
    targets = {line.strip() for line in fh if line.strip()}

orig_id = commit.original_id.decode("utf-8", errors="ignore")
if orig_id in targets:
    # Sign commit payload via gpg using configured signing key
    key_id = subprocess.check_output(
        ["git", "config", "--get", "user.signingkey"], text=True
    ).strip()
    proc = subprocess.run(
        ["gpg", "--batch", "--yes", "--detach-sign", "--armor", "-u", key_id],
        input=commit.message,
        capture_output=True,
        check=False,
    )
    if proc.returncode == 0 and proc.stdout:
        # Let git-filter-repo / git commit-tree handle signing natively via --sign-commits
        pass
' --sign-commits
    ); then
      echo "[FAIL] ${acct}/${repo_name}: git filter-repo failed (backup preserved at ${acct_bak}/${repo_name}.tar.zst)" >&2
      failed_repos=$((failed_repos + 1))
      continue
    fi

    # Optionally re-sign annotated tags
    if [ "$SIGN_TAGS" = true ]; then
      while IFS= read -r tag_name; do
        [ -n "$tag_name" ] || continue
        tag_msg="$(git -C "$stage_dir" tag -l --format='%(contents)' "$tag_name")"
        tag_target="$(git -C "$stage_dir" rev-parse "${tag_name}^{}")"
        git -C "$stage_dir" tag -f -s -u "$KEY_ID" -m "$tag_msg" "$tag_name" "$tag_target"
      done < <(git -C "$stage_dir" tag)
    fi

    # 4. Verify rewritten commits locally before pushing
    remaining_bad="$(
      git -C "$stage_dir" log --all --format="%G?" | grep -cvE '^[GU]$' || true
    )"
    if [ "$remaining_bad" -gt 0 ]; then
      echo "[FAIL] ${acct}/${repo_name}: ${remaining_bad} commit(s) still lack valid signatures after rewrite" >&2
      failed_repos=$((failed_repos + 1))
      continue
    fi

    # 5. Safe force push (--force-with-lease + --force-if-includes) to remote
    git -C "$stage_dir" remote add origin "$remote_url"
    git -C "$stage_dir" fetch origin --prune

    if git -C "$stage_dir" push --force-with-lease --force-if-includes origin --all && \
       git -C "$stage_dir" push --force-with-lease --force-if-includes origin --tags; then
      # Refresh local working clone to match rewritten remote history
      git -C "$repo_dir" fetch origin --prune --tags
      for branch in $(git -C "$stage_dir" for-each-ref --format='%(refname:short)' refs/heads/); do
        git -C "$repo_dir" checkout -B "$branch" "origin/${branch}"
      done
      echo "[OK] Re-signed and pushed ${acct}/${repo_name}"
      rewritten_repos=$((rewritten_repos + 1))
    else
      echo "[FAIL] ${acct}/${repo_name}: --force-with-lease push rejected (remote may have newer commits)" >&2
      failed_repos=$((failed_repos + 1))
    fi
  done
done

echo "------------------------------------------------------------"
echo "Summary: Audited=${total_repos} | Rewritten=${rewritten_repos} | Skipped=${skipped_repos} | Failed=${failed_repos}"
echo "Pre-rewrite backups saved in: ${BACKUP_ROOT}"

if [ "$failed_repos" -gt 0 ]; then
  exit 1
fi

Install, Test & Run
 * Make it executable:
   chmod +x ~/bin/resign-github-repos.sh

 * Preview first with --dry-run:
   ~/bin/resign-github-repos.sh --dry-run

 * Run on a single test repository first before running across all repos:
   ~/bin/resign-github-repos.sh --repos "your-test-repo" --sign-tags

 * Run across all repositories under re1Lucky365 and adpsnake7-beep:
   ~/bin/resign-github-repos.sh --sign-tags

> Important safety notes:
>  * Because commit SHAs change when re-signed, any open pull requests, CI runs, or other machines working off old commit SHAs will need updated branches/re-clones.
>  * If a push fails with --force-with-lease, fetch the latest changes first so you don't clobber concurrent commits, and you always have the .tar.zst pre-rewrite bare mirror in ~/github-archives/pre-resign-<timestamp>/ for rollback.
> 
commit.gpgsignI cannot execute commands directly on your system, so I cannot run the check on your local machine—here is how to verify zero GPG commit signature violations across every cloned repository under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep) after re-signing.Re-signing historical commits rewrites commit hashes across the rewritten history, so you should take a backup mirror first, verify your signing setup, rewrite cleanly, and push with --force-with-lease (plus --force-if-includes on modern Git) rather than a blind --force.
1. Pre-Flight: Backup First & Confirm Signing Key
Before touching history, snapshot the repository into a separate backup clone/mirror so you can roll back instantly if anything goes wrong:
REPO_DIR="${PWD}"
BACKUP_MIRROR="${HOME}/github-archives/pre-resign-$(basename "$REPO_DIR")-$(date +%Y%m%d_%H%M%S).git"

git clone --mirror "$REPO_DIR" "$BACKUP_MIRROR"
echo "Safety mirror saved to: $BACKUP_MIRROR"

Confirm your GPG signing key is available and configured in Git:
gpg --list-secret-keys --keyid-format=long
git config user.signingkey
git config commit.gpgsign

If needed, set them locally (or globally):
git config user.signingkey "YOUR_SIGNING_KEY_ID"
git config commit.gpgsign true
export GPG_TTY=$(tty)

2. Re-Sign Historical Commits Across Branches & Tags
Option A: Re-sign All Commits Across All Local Branches and Tags (git filter-repo)
If you have git-filter-repo installed, it is the cleanest, fastest way to rewrite history and re-sign commits with your active signing key while updating branch and tag refs:
# Run inside a dedicated working clone of the target repository
git filter-repo --force --commit-callback '
  commit.message = commit.message
' --sign-commits

(Note: If you want to re-sign commits across all branches/tags in one pass, make sure your working clone has fetched all refs: git fetch --all --tags --prune.)
Option B: Re-sign Only Unsigned Commits from a Specific Base Commit (git rebase)
If you only want to re-sign commits from a specific point onward (or keep older signed commits untouched):
# Re-sign everything reachable from current HEAD back to a base commit/root
git rebase --committer-date-is-author-date -S --force-rebase --root
# Or from a specific commit/tag:
git rebase --committer-date-is-author-date -S --force-rebase <BASE_COMMIT_OR_TAG>

> Watch out for merge commits and tags: Plain git rebase can flatten or drop complex merge topologies unless --rebase-merges is used, and neither filter-repo nor rebase automatically re-signs annotated tags. To re-sign annotated tags explicitly:
> for tag in $(git tag); do
>   git tag -f -s -m "$(git tag -l --format='%(contents)' "$tag")" "$tag" "$tag^{}"
> done
> 
> 
3. Verify Signatures Locally Before Pushing
Check the rewritten commits (%G? = signature validity, %GK = signing key ID):
git log --all --graph --oneline --show-signature
git log --all --format="%h %G? %GK %s"

Every re-signed commit should show status G (good signature) or U (good signature with unknown validity if trust hasn't been set), rather than N (unsigned).
4. Force-Push Safely (--force-with-lease)
Use --force-with-lease (and --force-if-includes in Git 2.30+) so Git refuses to overwrite the remote branch if someone else pushed new commits you haven't fetched yet:
git fetch origin
git push --force-with-lease --force-if-includes origin --all
git push --force-with-lease --force-if-includes origin --tags

Post-Push Notes & Rollback
 * Collaborators & automation: Any collaborator or local clone tracking the rewritten branches will have diverged history and must reset/re-clone rather than merge.
 * Roll back if needed: Point your branch back to your pre-flight mirror (git fetch "$BACKUP_MIRROR" refs/heads/*:refs/heads/* && git push --force-with-lease origin --all).
 * Audit check: Run ~/bin/audit-gpg-signatures.sh ~/src/github afterward to confirm zero remaining signature violations.
Here is a GitHub Actions workflow (.github/workflows/verify-gpg-signatures.yml) that checks every incoming commit on pushes and pull requests, imports your trusted public GPG key(s), verifies cryptographic signatures using Git's %G? and %GK status codes, and fails the check (exit 1) if any commit is unsigned (N), bad (B), expired/revoked (X/Y/R), missing a public key (E), or signed by an unauthorized key ID.
1. Workflow File (.github/workflows/verify-gpg-signatures.yml)
name: Verify GPG Commit Signatures

on:
  push:
    branches-ignore:
      - "dependabot/**"
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  audit-gpg-signatures:
    name: Audit Commit GPG Signatures
    runs-on: ubuntu-latest

    steps:
      - name: Checkout full commit history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Import trusted GPG public key(s)
        env:
          GPG_PUBLIC_KEYS_ARMORED: ${{ secrets.GPG_PUBLIC_KEYS_ARMORED }}
        run: |
          set -euo pipefail

          if [ -z "${GPG_PUBLIC_KEYS_ARMORED:-}" ]; then
            echo "ERROR: GPG_PUBLIC_KEYS_ARMORED secret is empty or missing." >&2
            exit 1
          fi

          printf '%s\n' "$GPG_PUBLIC_KEYS_ARMORED" | gpg --batch --import
          gpg --list-keys --keyid-format=long

      - name: Verify commit signatures across push or PR range
        env:
          EVENT_NAME: ${{ github.event_name }}
          BEFORE_SHA: ${{ github.event.before }}
          AFTER_SHA: ${{ github.sha }}
          PR_BASE_SHA: ${{ github.event.pull_request.base.sha }}
          PR_HEAD_SHA: ${{ github.event.pull_request.head.sha }}
          # Optional: comma-separated list of allowed 16-char or full GPG key IDs
          # Example: "0123456789ABCDEF,FEDCBA9876543210"
          ALLOWED_KEY_IDS: ${{ vars.ALLOWED_GPG_KEY_IDS || '' }}
        run: |
          set -euo pipefail

          if [ "$EVENT_NAME" = "pull_request" ]; then
            BASE_REF="${PR_BASE_SHA}"
            HEAD_REF="${PR_HEAD_SHA}"
          else
            # Handle new branch pushes where before SHA is all zeros
            if [[ -z "${BEFORE_SHA:-}" || "$BEFORE_SHA" =~ ^0+$ ]]; then
              BASE_REF="$(git rev-list --max-parents=0 "$AFTER_SHA" | head -n 1)"
              RANGE_SPEC="${AFTER_SHA}"
            else
              BASE_REF="${BEFORE_SHA}"
              RANGE_SPEC="${BASE_REF}..${AFTER_SHA}"
            fi
          fi

          if [ "$EVENT_NAME" = "pull_request" ]; then
            RANGE_SPEC="${BASE_REF}..${HEAD_REF}"
          fi

          echo "Auditing commit range: ${RANGE_SPEC}"

          IFS=',' read -r -a allowed_arr <<< "${ALLOWED_KEY_IDS}"
          SEP=$'\x1f'
          checked=0
          violations=0

          while IFS="$SEP" read -r sha status key_id signer email subject; do
            checked=$((checked + 1))
            valid=true
            reason=""

            case "$status" in
              G|U)
                if [ "${#allowed_arr[@]}" -gt 0 ] && [ -n "${allowed_arr[0]}" ]; then
                  matched=false
                  for allowed in "${allowed_arr[@]}"; do
                    allowed_trimmed="$(printf '%s' "$allowed" | xargs)"
                    [ -z "$allowed_trimmed" ] && continue
                    if [[ "$key_id" == *"$allowed_trimmed" || "$signer" == *"$allowed_trimmed" ]]; then
                      matched=true
                      break
                    fi
                  done
                  if [ "$matched" = false ]; then
                    valid=false
                    reason="Unauthorized signing key (${key_id:-unknown})"
                  fi
                fi
                ;;
              N) valid=false; reason="Unsigned commit" ;;
              B) valid=false; reason="BAD signature" ;;
              X) valid=false; reason="Signature expired" ;;
              Y) valid=false; reason="Signed by expired key (${key_id})" ;;
              R) valid=false; reason="Signed by revoked key (${key_id})" ;;
              E) valid=false; reason="Missing public key (${key_id})" ;;
              *) valid=false; reason="Unknown signature status (${status})" ;;
            esac

            if [ "$valid" = true ]; then
              echo "[PASS] ${sha:0:10} | status=${status} | key=${key_id} | ${subject}"
            else
              violations=$((violations + 1))
              echo "::error title=GPG Signature Violation::Commit ${sha} (${subject}) by ${email}: ${reason}"
            fi
          done < <(git log --format="%H%x1f%G?%x1f%GK%x1f%GS%x1f%ae%x1f%s" "$RANGE_SPEC")

          echo "------------------------------------------------------------"
          echo "Commits audited: ${checked} | Violations: ${violations}"

          if [ "$checked" -eq 0 ]; then
            echo "WARNING: No commits found in range ${RANGE_SPEC}." >&2
            exit 1
          fi

          if [ "$violations" -gt 0 ]; then
            exit 1
          fi

2. Required Secret & Optional Variable Setup
 * Export your ASCII-armored public key(s) locally:
   gpg --armor --export YOUR_KEY_ID

   (If you use multiple signing keys across machines or accounts, concatenate or export both public keys into the secret value.)
 * Add the repository or organization secret:
   * Create a secret named GPG_PUBLIC_KEYS_ARMORED containing the full -----BEGIN PGP PUBLIC KEY BLOCK----- ... -----END PGP PUBLIC KEY BLOCK----- output.
 * Optionally restrict to specific key IDs:
   * Set the repository/organization variable ALLOWED_GPG_KEY_IDS to your allowed key IDs (comma-separated, e.g., YOUR_KEY_ID_1,YOUR_KEY_ID_2).
3. Enforcing Hard Rejection on Protected Branches
A workflow check alone blocks merges or pushes only if you also require Audit Commit GPG Signatures as a Required status check in your repository's branch protection rules or GitHub Rulesets (main / default branch). Without a required status check on the branch or ruleset, a failing workflow run is visible on the commit/PR, but users with write/bypass access could still push or merge unless rulesets block it.

# #!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0
#!/usr/bin/env bash
# pipelines/rclone_push.sh
set -euo pipefail

VAULT_DIR="${1:?Vault directory required}"
CRYPT_REMOTE="${2:-adpbetty-crypt:recovery-vault}"

# One-way encrypted push only; never pulls or exposes unencrypted files
rclone copy "${VAULT_DIR}" "${CRYPT_REMOTE}" \
  --include "*.tar.zst*" \
  --checksum \
  --transfers 2 \
  --log-level INFO
#!/usr/bin/env bash
# pipelines/seal_archive.sh
set -euo pipefail

SOURCE_DIR="${1:?Source directory required}"
VAULT_DIR="${2:?Vault directory required}"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
ARCHIVE="${VAULT_DIR}/adpbetty_vault_${STAMP}.tar.zst"

mkdir -p "${VAULT_DIR}"
tar -cf - -C "${SOURCE_DIR}" . | zstd -T0 -19 -o "${ARCHIVE}"

sha256sum "${ARCHIVE}" > "${ARCHIVE}.sha256"
sha512sum "${ARCHIVE}" > "${ARCHIVE}.sha512"
chmod 400 "${ARCHIVE}" "${ARCHIVE}.sha256" "${ARCHIVE}.sha512"
evansrobert517@gmail.com#!/usr/bin/env python3
import hashlib
import json
import sqlite3
from pathlib import Path

DB_PATH = Path("/var/lib/adpbetty/recovery_inventory.db")
SCHEMA_PATH = Path(__file__).with_name("schema.sql")

SEED_ACCOUNTS = [
    (1, "Google / Gmail", "evansrobert517@gmail.com", 2019, 2026, "evansrobert517@yahoo.com", "T-Mobile", "Passkey + TOTP", "lockdown_required"),
    (1, "Yahoo", "evansrobert517@yahoo.com", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP", "lockdown_required"),
    (1, "T-Mobile", "Robert Edward Evans (Carrier)", 2019, 2026, "evansrobert517@gmail.com", "Primary Line", "Account PIN", "lockdown_required"),
    (1, "Samsung", "Galaxy S23 FE (SM-S711U) / Moto E5 Play", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "2FA", "pending_audit"),
    (2, "GitHub", "re1Lucky365", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP + GPG", "lockdown_required"),
    (2, "GitHub", "adpsnake7-beep", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP + GPG", "lockdown_required"),
    (2, "Google Workspace", "WK Precision Enterprise LLC", 2026, 2026, "evansrobert517@gmail.com", "T-Mobile", "Passkey", "pending_audit"),
    (2, "Microsoft / Azure", "Personal & Azure Dev", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "Authenticator", "lockdown_required"),
    (2, "AWS", "Root / IAM Developer", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "MFA", "pending_audit"),
    (3, "Instagram", "@ROBERTEDWARD4553", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP", "lockdown_required"),
    (3, "Instagram", "@E.ROBERTEDWARD", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP", "lockdown_required"),
    (3, "Instagram", "@ROBERTEDWARDEVAN", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP", "lockdown_required"),
    (3, "Discord / Reddit / Steam", "Associated Handles", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP", "pending_audit"),
    (4, "Panels & Research", "Product Testing / Surveys / Market Research", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "Email", "pending_audit"),
]

def init_and_seed() -> None:
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)
    with sqlite3.connect(DB_PATH) as conn:
        conn.executescript(SCHEMA_PATH.read_text(encoding="utf-8"))
        conn.executemany(
            """
            INSERT INTO accounts (
                tier, platform, identifier, year_first_active, year_last_active,
                recovery_email, recovery_phone, mfa_method, status
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT(platform, identifier) DO UPDATE SET
                year_last_active = excluded.year_last_active,
                updated_at = datetime('now');
            """,
            SEED_ACCOUNTS,
        )
        rows = conn.execute("SELECT tier, platform, identifier, status FROM accounts ORDER BY tier, platform").fetchall()
        manifest = [{"tier": r[0], "platform": r[1], "identifier": r[2], "status": r[3]} for r in rows]
        payload = json.dumps(manifest, indent=2).encode("utf-8")
        digest = hashlib.sha256(payload).hexdigest()
        Path("docs/sanitized-manifest.json").write_bytes(payload)
        Path("docs/sanitized-manifest.json.sha256").write_text(f"{digest}  sanitized-manifest.json\n", encoding="utf-8")

if __name__ == "__main__":
    init_and_seed()
#!/usr/bin/env bash
set -euo pipefail

# Default-deny inbound and outbound (fail-closed)
ufw --force reset
ufw default deny incoming
ufw default deny outgoing

# Allow loopback (FastAPI, Ollama, ChromaDB, Caddy internal mesh)
ufw allow in on lo
ufw allow out on lo

# Allow DNS & HTTPS strictly for allowlisted sync/API targets
ufw allow out 53/udp
ufw allow out 443/tcp
ufw allow out 22/tcp

ufw --force enable
ufw status verbose
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tier INTEGER NOT NULL CHECK (tier IN (1, 2, 3, 4)),
    platform TEXT NOT NULL,
    identifier TEXT NOT NULL,
    year_first_active INTEGER CHECK (year_first_active BETWEEN 2019 AND 2026),
    year_last_active INTEGER CHECK (year_last_active BETWEEN 2019 AND 2026),
    recovery_email TEXT,
    recovery_phone TEXT,
    mfa_method TEXT,
    status TEXT NOT NULL DEFAULT 'pending_audit',
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(platform, identifier)
);

CREATE TABLE IF NOT EXISTS evidence_artifacts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    account_id INTEGER NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    artifact_type TEXT NOT NULL, -- 'ssh_key', 'gpg_key', 'takeout_zst', 'smart_switch', 'billing_pdf'
    local_vault_path TEXT NOT NULL,
    sha256_hash TEXT NOT NULL,
    sha512_hash TEXT NOT NULL,
    verified_at TEXT NOT NULL DEFAULT (datetime('now'))
);
adpBetty-core/
├── .github/
│   └── workflows/
│       ├── verify-manifest-integrity.yml   # CI check on signed commits & manifest hashes
│       └── scheduled-health-audit.yml      # Monthly schema & recovery readiness check
├── firewall/
│   └── ufw-failclosed.sh                   # One-way-in / no-way-out egress lockdown
├── pipelines/
│   ├── schema.sql                          # 2019–2026 master platform & evidence schema
│   ├── ingest_inventory.py                 # Local SQLite + ChromaDB metadata indexer
│   ├── audit_git_gpg.sh                    # Bare-mirror & GPG signature auditor
│   ├── seal_archive.sh                     # zstd -19 compression + SHA-256/512 hashing
│   └── rclone_push.sh                      # Encrypted-only outbound push to rclone crypt
├── systemd/
│   ├── adpbetty-recovery.service
│   └── adpbetty-recovery.timer
└── docker-compose.yml                      # Local FastAPI, ChromaDB, Ollama, Caddy stack
adpBetty-core/
├── docker-compose.yml              # FastAPI, ChromaDB, Ollama, Caddy (internal-only routing)
├── systemd/
│   ├── adpbetty-ingest.service     # Runs local collectors & device backup checks
│   └── adpbetty-ingest.timer       # Daily execution + hash verification
├── pipelines/
│   ├── ingest_inventory.py         # Normalizes 2019–2026 accounts into SQLite + ChromaDB
│   ├── audit_git_gpg.sh            # Verifies GPG signatures & mirrors GitHub repos
│   ├── seal_archive.sh             # Compresses exports with zstd + generates SHA-256/512
│   └── rclone_push.sh              # One-way encrypted push to rclone crypt remote
└── firewall/
    └── ufw-failclosed.sh           # Default-deny egress ruleset for ingest/vault containers
#!/usr/bin/env bash
set -euo pipefail

SOURCE_DIR="${1:?Usage: seal_archive.sh <source_dir> <vault_dir>}"
VAULT_DIR="${2:?Usage: seal_archive.sh <source_dir> <vault_dir>}"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
ARCHIVE="${VAULT_DIR}/recovery_bundle_${STAMP}.tar.zst"

mkdir -p "${VAULT_DIR}"
tar -cf - -C "${SOURCE_DIR}" . | zstd -T0 -19 -o "${ARCHIVE}"

sha256sum "${ARCHIVE}" > "${ARCHIVE}.sha256"
sha512sum "${ARCHIVE}" > "${ARCHIVE}.sha512"
chmod 400 "${ARCHIVE}" "${ARCHIVE}.sha256" "${ARCHIVE}.sha512"
docker-compose.ymlhttp://127.0.0.1:8080/healthschema.sqlsudo bash firewall/ufw-failclosed.sh
python3 pipelines/ingest_inventory.py
python3 pipelines/scan_files_inventory.py
bash pipelines/audit_git_gpg.sh
bash pipelines/seal_archive.sh /var/spool/adpbetty/quarantine /var/lib/adpbetty/vault
docker compose up -d
curl -s http://127.0.0.1:8080/health
:8080 {
    @internal_only {
        remote_ip 127.0.0.1 ::1 172.16.0.0/12 10.0.0.0/8 192.168.0.0/16
    }
    handle @internal_only {
        reverse_proxy adpbetty-api:8000
    }
    respond 403
}
#!/usr/bin/env python3
import hashlib
import os
import sqlite3
from pathlib import Path
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

DB_PATH = Path(os.getenv("ADP_DB_PATH", "/var/lib/adpbetty/recovery_inventory.db"))
ALLOWED_STATUSES = {"pending_audit", "lockdown_required", "secured", "recovery_in_progress", "recovered"}

app = FastAPI(title="adpBetty-core Recovery & Security API", version="1.0.0")


class StatusUpdate(BaseModel):
    status: str = Field(..., description="Updated recovery/security status")
    mfa_method: str | None = Field(default=None, description="Verified MFA or passkey mechanism")


def get_conn() -> sqlite3.Connection:
    if not DB_PATH.exists():
        raise HTTPException(status_code=503, detail="Inventory database not initialized yet")
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn


@app.get("/health")
def health() -> dict[str, object]:
    return {
        "status": "ok",
        "egress_policy": "fail_closed_one_way_in",
        "db_present": DB_PATH.exists(),
    }


@app.get("/accounts")
def list_accounts(tier: int | None = None) -> dict[str, object]:
    with get_conn() as conn:
        if tier is not None:
            rows = conn.execute(
                "SELECT id, tier, platform, identifier, year_first_active, year_last_active, mfa_method, status, updated_at FROM accounts WHERE tier = ? ORDER BY tier, platform, id",
                (tier,),
            ).fetchall()
        else:
            rows = conn.execute(
                "SELECT id, tier, platform, identifier, year_first_active, year_last_active, mfa_method, status, updated_at FROM accounts ORDER BY tier, platform, id"
            ).fetchall()
    return {"count": len(rows), "accounts": [dict(r) for r in rows]}


@app.patch("/accounts/{account_id}/status")
def update_account_status(account_id: int, payload: StatusUpdate) -> dict[str, object]:
    if payload.status not in ALLOWED_STATUSES:
        raise HTTPException(status_code=400, detail="Invalid status value")
    with get_conn() as conn:
        cur = conn.execute(
            """
            UPDATE accounts
            SET status = ?,
                mfa_method = COALESCE(?, mfa_method),
                updated_at = datetime('now')
            WHERE id = ?
            """,
            (payload.status, payload.mfa_method, account_id),
        )
        if cur.rowcount == 0:
            raise HTTPException(status_code=404, detail="Account not found")
        row = conn.execute(
            "SELECT id, tier, platform, identifier, mfa_method, status, updated_at FROM accounts WHERE id = ?",
            (account_id,),
        ).fetchone()
    return {"updated": dict(row)}


@app.get("/files/summary")
def files_summary() -> dict[str, object]:
    with get_conn() as conn:
        rows = conn.execute(
            """
            SELECT file_class, sensitivity_level, allow_cloud_sync,
                   COUNT(*) AS file_count, COALESCE(SUM(size_bytes), 0) AS total_bytes
            FROM file_records
            GROUP BY file_class, sensitivity_level, allow_cloud_sync
            ORDER BY file_class
            """
        ).fetchall()
    return {"records": [dict(r) for r in rows]}


@app.post("/vault/verify-checksum")
def verify_checksum(archive_path: str) -> dict[str, object]:
    target = Path(archive_path)
    sha256_file = Path(f"{archive_path}.sha256")
    if not target.is_file() or not sha256_file.is_file():
        raise HTTPException(status_code=404, detail="Archive or .sha256 sidecar missing")
    expected = sha256_file.read_text(encoding="utf-8").strip().split()[0].lower()
    h = hashlib.sha256()
    with target.open("rb") as f:
        while chunk := f.read(1024 * 1024):
            h.update(chunk)
    actual = h.hexdigest().lower()
    return {"archive": str(target), "sha256": actual, "matches_sidecar": actual == expected}
#!/usr/bin/env bash
set -euo pipefail

DB_PATH="${ADP_DB_PATH:-/var/lib/adpbetty/recovery_inventory.db}"
SEARCH_ROOTS=("${HOME}/projects" "${HOME}/src" "${HOME}/repos" "/opt")
REPORT_FILE="docs/git-gpg-audit-sanitized.txt"

mkdir -p "$(dirname "${REPORT_FILE}")"
: > "${REPORT_FILE}"

echo "=== SSH Public Key Fingerprints ===" >> "${REPORT_FILE}"
if [ -d "${HOME}/.ssh" ]; then
  for pub in "${HOME}/.ssh"/*.pub; do
    [ -e "${pub}" ] || continue
    sha256sum "${pub}" >> "${REPORT_FILE}"
    sha512sum "${pub}" >> "${REPORT_FILE}"
    ssh-keygen -lf "${pub}" >> "${REPORT_FILE}" 2>/dev/null || true
  done
fi

echo "" >> "${REPORT_FILE}"
echo "=== GPG Public Key Fingerprints ===" >> "${REPORT_FILE}"
if command -v gpg >/dev/null 2>&1; then
  gpg --list-keys --with-colons 2>/dev/null | awk -F: '/^fpr:/ {print $10}' >> "${REPORT_FILE}" || true
fi

echo "" >> "${REPORT_FILE}"
echo "=== Local Git Repositories & Remotes ===" >> "${REPORT_FILE}"
for root in "${SEARCH_ROOTS[@]}"; do
  [ -d "${root}" ] || continue
  while IFS= read -r -d '' gitdir; do
    repo_dir="$(dirname "${gitdir}")"
    remotes="$(git -C "${repo_dir}" remote -v 2>/dev/null | head -n 4 || true)"
    last_commit="$(git -C "${repo_dir}" log -1 --format='%H %cI %G?' 2>/dev/null || true)"
    printf "REPO: %s | HEAD: %s\n%s\n---\n" "${repo_dir}" "${last_commit}" "${remotes}" >> "${REPORT_FILE}"
  done < <(find "${root}" -maxdepth 4 -name ".git" -type d -print0 2>/dev/null)
done

if [ -f "${DB_PATH}" ] && command -v sqlite3 >/dev/null 2>&1; then
  h256="$(sha256sum "${REPORT_FILE}" | awk '{print $1}')"
  h512="$(sha512sum "${REPORT_FILE}" | awk '{print $1}')"
  acct_id="$(sqlite3 "${DB_PATH}" "SELECT id FROM accounts WHERE platform='GitHub' AND identifier IN ('re1Lucky365','adpsnake7-beep') LIMIT 1;")"
  if [ -n "${acct_id}" ]; then
    sqlite3 "${DB_PATH}" "INSERT INTO evidence_artifacts (account_id, artifact_type, local_vault_path, sha256_hash, sha512_hash) VALUES (${acct_id}, 'gpg_key', '${REPORT_FILE}', '${h256}', '${h512}');"
  fi
fi

echo "Audit complete. Report written to ${REPORT_FILE}"
services:
  caddy:
    image: caddy:2-alpine
    container_name: adpbetty-caddy
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - adpbetty_caddy_data:/data
      - adpbetty_caddy_config:/config
    networks:
      - adpbetty_internal

  fastapi:
    build:
      context: .
      dockerfile: Dockerfile.api
    container_name: adpbetty-api
    restart: unless-stopped
    environment:
      - ADP_DB_PATH=/var/lib/adpbetty/recovery_inventory.db
      - ADP_CHROMA_HOST=chromadb
      - ADP_CHROMA_PORT=8000
      - ADP_OLLAMA_URL=http://ollama:11434
    volumes:
      - /var/lib/adpbetty:/var/lib/adpbetty
      - ./pipelines:/app/pipelines:ro
      - ./docs:/app/docs
    depends_on:
      - chromadb
      - ollama
    networks:
      - adpbetty_internal

  chromadb:
    image: chromadb/chroma:latest
    container_name: adpbetty-chromadb
    restart: unless-stopped
    volumes:
      - adpbetty_chroma_data:/chroma/chroma
    networks:
      - adpbetty_internal

  ollama:
    image: ollama/ollama:latest
    container_name: adpbetty-ollama
    restart: unless-stopped
    volumes:
      - adpbetty_ollama_data:/root/.ollama
    networks:
      - adpbetty_internal

networks:
  adpbetty_internal:
    driver: bridge
    internal: true

volumes:
  adpbetty_caddy_data:
  adpbetty_caddy_config:
  adpbetty_chroma_data:
  adpbetty_ollama_data:
name: Verify File & Spreadsheet Inventory Integrity

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  verify-file-manifest:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Validate CSV Header & No Restricted PII Leak
        run: |
          test -f docs/files-inventory-sanitized.csv
          head -n 1 docs/files-inventory-sanitized.csv | grep -q "file_class,extension,size_bytes,sensitivity_level,allow_cloud_sync,sha256_hash,captured_at"
          if grep -rnE '([0-9]{3}-[0-9]{2}-[0-9]{4}|BEGIN (RSA|OPENSSH|PGP) PRIVATE KEY)' docs/ pipelines/; then
            echo "ERROR: Restricted identifier or key pattern detected in repository!"
            exit 1
          fi
#!/usr/bin/env python3
import csv
import hashlib
import sqlite3
from pathlib import Path

DB_PATH = Path("/var/lib/adpbetty/recovery_inventory.db")
SCHEMA_PATH = Path(__file__).with_name("files_schema.sql")
MANIFEST_OUT = Path("docs/files-inventory-sanitized.csv")

WATCH_ROOTS = [
    Path.home() / "Downloads",
    Path.home() / "Documents",
    Path("/var/spool/adpbetty/quarantine"),
]

CLASS_MAP = {
    ".xlsx": ("spreadsheet", "vaulted", 0),
    ".ods": ("spreadsheet", "vaulted", 0),
    ".csv": ("spreadsheet", "sanitized", 1),
    ".tsv": ("spreadsheet", "sanitized", 1),
    ".pdf": ("pdf_document", "restricted", 0),
    ".docx": ("legal_document", "restricted", 0),
    ".odt": ("legal_document", "restricted", 0),
    ".zip": ("download_export", "vaulted", 0),
    ".tar": ("download_export", "vaulted", 0),
    ".zst": ("device_archive", "vaulted", 1),
}

def file_hashes(path: Path) -> tuple[str, str, int]:
    h256 = hashlib.sha256()
    h512 = hashlib.sha512()
    total = 0
    with path.open("rb") as f:
        while chunk := f.read(1024 * 1024):
            total += len(chunk)
            h256.update(chunk)
            h512.update(chunk)
    return h256.hexdigest(), h512.hexdigest(), total

def classify(path: Path) -> tuple[str, str, int] | None:
    ext = path.suffix.lower()
    if ext not in CLASS_MAP:
        return None
    f_class, sensitivity, sync_ok = CLASS_MAP[ext]
    lower_name = path.name.lower()
    if any(token in lower_name for token in ("llc", "legal", "notice", "court", "affidavit", "contract", "title", "deed")):
        return ("legal_document", "restricted", 0)
    return (f_class, sensitivity, sync_ok)

def run_scan() -> None:
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)
    MANIFEST_OUT.parent.mkdir(parents=True, exist_ok=True)

    with sqlite3.connect(DB_PATH) as conn:
        conn.executescript(SCHEMA_PATH.read_text(encoding="utf-8"))
        for root in WATCH_ROOTS:
            if not root.exists():
                continue
            for item in root.rglob("*"):
                if not item.is_file():
                    continue
                meta = classify(item)
                if not meta:
                    continue
                f_class, sensitivity, sync_ok = meta
                s256, s512, size = file_hashes(item)
                conn.execute(
                    """
                    INSERT INTO file_records (
                        file_class, original_name, extension, size_bytes,
                        source_path, sha256_hash, sha512_hash,
                        sensitivity_level, allow_cloud_sync
                    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                    ON CONFLICT(sha256_hash, file_class) DO UPDATE SET
                        size_bytes = excluded.size_bytes,
                        allow_cloud_sync = excluded.allow_cloud_sync,
#!/usr/bin/env bash
set -euo pipefail

# Default-deny inbound and outbound (fail-closed)
ufw --force reset
ufw default deny incoming
ufw default deny outgoing

# Allow loopback (FastAPI, Ollama, ChromaDB, Caddy internal mesh)
ufw allow in on lo
ufw allow out on lo

# Allow DNS & HTTPS strictly for allowlisted sync/API targets
ufw allow out 53/udp
ufw allow out 443/tcp
ufw allow out 22/tcp

ufw --force enable
ufw status verbose
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tier INTEGER NOT NULL CHECK (tier IN (1, 2, 3, 4)),
    platform TEXT NOT NULL,
    identifier TEXT NOT NULL,
    year_first_active INTEGER CHECK (year_first_active BETWEEN 2019 AND 2026),
    year_last_active INTEGER CHECK (year_last_active BETWEEN 2019 AND 2026),
    recovery_email TEXT,
    recovery_phone TEXT,
    mfa_method TEXT,
    status TEXT NOT NULL DEFAULT 'pending_audit',
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(platform, identifier)
);

CREATE TABLE IF NOT EXISTS evidence_artifacts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    account_id INTEGER NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    artifact_type TEXT NOT NULL, -- 'ssh_key', 'gpg_key', 'takeout_zst', 'smart_switch', 'billing_pdf'
    local_vault_path TEXT NOT NULL,
    sha256_hash TEXT NOT NULL,
    sha512_hash TEXT NOT NULL,
    verified_at TEXT NOT NULL DEFAULT (datetime('now'))
);
adpBetty-core/
├── .github/
│   └── workflows/
│       ├── verify-manifest-integrity.yml   # CI check on signed commits & manifest hashes
│       └── scheduled-health-audit.yml      # Monthly schema & recovery readiness check
├── firewall/
│   └── ufw-failclosed.sh                   # One-way-in / no-way-out egress lockdown
├── pipelines/
│   ├── schema.sql                          # 2019–2026 master platform & evidence schema
│   ├── ingest_inventory.py                 # Local SQLite + ChromaDB metadata indexer
│   ├── audit_git_gpg.sh                    # Bare-mirror & GPG signature auditor
│   ├── seal_archive.sh                     # zstd -19 compression + SHA-256/512 hashing
│   └── rclone_push.sh                      # Encrypted-only outbound push to rclone crypt
├── systemd/
│   ├── adpbetty-recovery.service
│   └── adpbetty-recovery.timer
└── docker-compose.yml                      # Local FastAPI, ChromaDB, Ollama, Caddy stack
#!/usr/bin/env bash
set -euo pipefail

SOURCE_DIR="${1:?Usage: seal_archive.sh <source_dir> <vault_dir>}"
VAULT_DIR="${2:?Usage: seal_archive.sh <source_dir> <vault_dir>}"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
ARCHIVE="${VAULT_DIR}/recovery_bundle_${STAMP}.tar.zst"

mkdir -p "${VAULT_DIR}"
tar -cf - -C "${SOURCE_DIR}" . | zstd -T0 -19 -o "${ARCHIVE}"

sha256sum "${ARCHIVE}" > "${ARCHIVE}.sha256"
sha512sum "${ARCHIVE}" > "${ARCHIVE}.sha512"
chmod 400 "${ARCHIVE}" "${ARCHIVE}.sha256" "${ARCHIVE}.sha512"
adpBetty-core/
├── docker-compose.yml              # FastAPI, ChromaDB, Ollama, Caddy (internal-only routing)
├── systemd/
│   ├── adpbetty-ingest.service     # Runs local collectors & device backup checks
│   └── adpbetty-ingest.timer       # Daily execution + hash verification
├── pipelines/
│   ├── ingest_inventory.py         # Normalizes 2019–2026 accounts into SQLite + ChromaDB
│   ├── audit_git_gpg.sh            # Verifies GPG signatures & mirrors GitHub repos
│   ├── seal_archive.sh             # Compresses exports with zstd + generates SHA-256/512
│   └── rclone_push.sh              # One-way encrypted push to rclone crypt remote
└── firewall/
    └── ufw-failclosed.sh           # Default-deny egress ruleset for ingest/vault containers

                        captured_at = datetime('now');
                    """,
                    (
                        f_class,
                        item.name,
                        item.suffix.lower(),
                        size,
                        str(item),
                        s256,
                        s512,
                        sensitivity,
                        sync_ok,
                    ),
                )

        rows = conn.execute(
            """
            SELECT file_class, extension, size_bytes, sensitivity_level, allow_cloud_sync, sha256_hash, captured_at
            FROM file_records
            ORDER BY file_class, captured_at DESC
            """
        ).fetchall()

    with MANIFEST_OUT.open("w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow(["file_class", "extension", "size_bytes", "sensitivity_level", "allow_cloud_sync", "sha256_hash", "captured_at"])
        writer.writerows(rows)

if __name__ == "__main__":
    run_scan()
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS file_records (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_class TEXT NOT NULL CHECK (file_class IN (
        'legal_document',
        'spreadsheet',
        'pdf_document',
        'download_export',
        'device_archive'
    )),
    original_name TEXT NOT NULL,
    extension TEXT NOT NULL,
    size_bytes INTEGER NOT NULL CHECK (size_bytes >= 0),
    source_path TEXT NOT NULL,
    vault_bundle TEXT,
    sha256_hash TEXT NOT NULL,
    sha512_hash TEXT NOT NULL,
    sensitivity_level TEXT NOT NULL CHECK (sensitivity_level IN ('restricted', 'vaulted', 'sanitized')),
    allow_cloud_sync INTEGER NOT NULL DEFAULT 0 CHECK (allow_cloud_sync IN (0, 1)),
    captured_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(sha256_hash, file_class)
);

CREATE INDEX IF NOT EXISTS idx_file_records_class ON file_records(file_class);
CREATE INDEX IF NOT EXISTS idx_file_records_sync ON file_records(allow_cloud_sync);
[Unit]
Description=Daily adpBetty-core Local Recovery Inventory & Vault Seal

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
name: Verify Manifest Integrity & Secret Guard

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  integrity-and-leak-guard:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Verify Sanitized Manifest SHA-256
        run: |
          cd docs
          sha256sum -c sanitized-manifest.json.sha256

      - name: Block Accidental Secret or SSN Patterns
        run: |
          if grep -rnE '([0-9]{3}-[0-9]{2}-[0-9]{4}|BEGIN (RSA|OPENSSH|PGP) PRIVATE KEY)' docs/ pipelines/; then
            echo "ERROR: Sensitive credential or SSN pattern detected!"
            exit 1
          fi
#!/usr/bin/env bash
# pipelines/rclone_push.sh
set -euo pipefail

VAULT_DIR="${1:?Vault directory required}"
CRYPT_REMOTE="${2:-adpbetty-crypt:recovery-vault}"

# One-way encrypted push only; never pulls or exposes unencrypted files
rclone copy "${VAULT_DIR}" "${CRYPT_REMOTE}" \
  --include "*.tar.zst*" \
  --checksum \
  --transfers 2 \
  --log-level INFO
#!/usr/bin/env bash
# pipelines/seal_archive.sh
set -euo pipefail

SOURCE_DIR="${1:?Source directory required}"
VAULT_DIR="${2:?Vault directory required}"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
ARCHIVE="${VAULT_DIR}/adpbetty_vault_${STAMP}.tar.zst"

mkdir -p "${VAULT_DIR}"
tar -cf - -C "${SOURCE_DIR}" . | zstd -T0 -19 -o "${ARCHIVE}"

sha256sum "${ARCHIVE}" > "${ARCHIVE}.sha256"
sha512sum "${ARCHIVE}" > "${ARCHIVE}.sha512"
chmod 400 "${ARCHIVE}" "${ARCHIVE}.sha256" "${ARCHIVE}.sha512"
#!/usr/bin/env python3
import hashlib
import json
import sqlite3
from pathlib import Path

DB_PATH = Path("/var/lib/adpbetty/recovery_inventory.db")
SCHEMA_PATH = Path(__file__).with_name("schema.sql")

SEED_ACCOUNTS = [
    (1, "Google / Gmail", "evansrobert517@gmail.com", 2019, 2026, "evansrobert517@yahoo.com", "T-Mobile", "Passkey + TOTP", "lockdown_required"),
    (1, "Yahoo", "evansrobert517@yahoo.com", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "TOTP", "lockdown_required"),
    (1, "T-Mobile", "Robert Edward Evans (Carrier)", 2019, 2026, "evansrobert517@gmail.com", "Primary Line", "Account PIN", "lockdown_required"),
    (1, "Samsung", "Galaxy S23 FE (SM-S711U) / Moto E5 Play", 2019, 2026, "evansrobert517@gmail.com", "T-Mobile", "2FA", "pending_audit"),
    (2https://github.com/mary-georgiou/NancyIt looks like your sentence cut off right after "that can"—what capability do you want to build next?
Since you already have adpBetty-core (Python, FastAPI, SQLite, ChromaDB, Ollama, Docker, Caddy, zstd, rclone, and UFW) and your recovery/security pipelines in motion, here are a few directions we can take it:
 * Automated Security & Drift Watcher: Watch local repos, SSH/GPG keys, or configuration files and alert you if remotes, permissions, or checksums change unexpectedly.
 * Local RAG Recovery & Evidence Assistant: Index sanitized metadata, notes, and recovery logs into ChromaDB + Ollama so you can query your 2019–2026 history and checklists offline.
 * Encrypted Backup & Vault Orchestrator: Automate scheduled zstd sealing, SHA-256/512 verification, and one-way rclone pushes across your Linux/Windows machines and devices.
How did you want to finish that thought—what should this tool do?
Running throttled background Data Encryption Key (DEK) re-wrapping (or payload re-encryption) across multi-region storage without hurting production latency requires separating control-plane key rotation from data-plane bulk migration, pacing I/O and KMS/HSM requests, and keeping live traffic strictly isolated from batch workers.
In most systems using envelope encryption, rotating a Key Encryption Key (KEK) is already instant for new writes once KEK-v2 is active—background re-wrapping is only needed to retire an old KEK version, comply with retention/crypto rules, or consolidate wrapped DEKs.
1. Core Architecture to Protect Production Latency
 * Re-Wrap DEKs, Not Full Payloads, Whenever Possible: If your data is encrypted with per-record or per-object DEKs, unwrapping the small DEK under KEK-v1 and re-wrapping it under KEK-v2 (updating only the envelope header/metadata) avoids reading, decrypting, and re-encrypting multi-gigabyte payloads—saving massive storage bandwidth and IOPS.
 * Isolate Background Workers & Quotas: Run re-wrapping workers on dedicated batch pools/queues with separate KMS/HSM API rate-limit budgets, separate storage connection pools, and lower priority than customer-facing read/write paths.
 * Use Token-Bucket Rate Limiting + Backoff: Cap operations per second (ops/sec), bytes/sec, and KMS unwrap/wrap calls per region, and automatically slow down or pause when regional storage latency, CPU, or KMS throttling (429/rate-limit) rises.
 * Keep Operations Idempotent & Resumable: Store durable cursor checkpoints (e.g., key prefix, partition ID, or last processed record ID) and verify current_kek_id != target_kek_id before modifying a record so retries after crashes or region failovers never corrupt data or duplicate work.
2. Multi-Region Execution Strategy
 * Run Region-Local Workers First: Point each region's background worker to its local storage bucket/database and local/regional KMS/HSM endpoint rather than pulling objects across regions over WAN links.
 * Use Lazy Re-Wrap on Read/Write + Background Sweep:
   * Let normal reads/writes re-wrap stale DEKs lazily when touched if safe, while a throttled background scanner sweeps cold/untouched keys at a low steady rate.
 * Gate Retirement on Completion & Verification: Only disable or retire KEK-v1 once your scan reports zero remaining objects/DEKs referencing kek_id == "kek-v1" across all regions and checksums/headers are verified.
3. Reference Pattern: Throttled, Resumable Re-Wrapper (throttled_rewrap.py)
import time
from dataclasses import dataclass


@dataclass
class TokenBucket:
    rate_per_sec: float
    capacity: float
    tokens: float
    last_check: float

    @classmethod
    def create(cls, rate_per_sec: float, burst: float | None = None) -> "TokenBucket":
        cap = burst if burst is not None else max(1.0, rate_per_sec)
        return cls(rate_per_sec=rate_per_sec, capacity=cap, tokens=cap, last_check=time.monotonic())

    def acquire(self, cost: float = 1.0) -> None:
        while True:
            now = time.monotonic()
            elapsed = now - self.last_check
            self.last_check = now
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate_per_sec)
            if self.tokens >= cost:
                self.tokens -= cost
                return
            sleep_for = (cost - self.tokens) / self.rate_per_sec
            time.sleep(min(max(sleep_for, 0.005), 0.25))


def rewrap_batch_with_throttle(
    records: list[dict],
    target_kek_id: str,
    unwrap_dek_fn,
    wrap_dek_fn,
    persist_envelope_fn,
    p99_latency_ms_fn,
    max_p99_ms: float = 120.0,
    base_rate_per_sec: float = 25.0,
) -> dict[str, int]:
    bucket = TokenBucket.create(rate_per_sec=base_rate_per_sec)
    stats = {"processed": 0, "rewrapped": 0, "skipped": 0, "throttled_pauses": 0}

    for rec in records:
        # Dynamic backpressure if live region latency rises
        if p99_latency_ms_fn() > max_p99_ms:
            stats["throttled_pauses"] += 1
            time.sleep(0.5)

        bucket.acquire(1.0)
        stats["processed"] += 1

        if rec.get("kek_id") == target_kek_id:
            stats["skipped"] += 1
            continue

        # Idempotent re-wrap: unwrap old DEK -> wrap under new KEK -> persist atomically
        plaintext_dek = unwrap_dek_fn(rec["kek_id"], rec["wrapped_dek"])
        new_wrapped_dek = wrap_dek_fn(target_kek_id, plaintext_dek)
        persist_envelope_fn(rec["id"], target_kek_id, new_wrapped_dek)
        stats["rewrapped"] += 1

    return stats

4. Guardrails & Failure Modes to Watch
 * Watch KMS / HSM Throttling Quotas: Background jobs often hit KMS request limits long before object storage saturates—allocate a dedicated KMS quota or batch/cache policy checks cleanly.
 * Avoid Hot-Partition Storms: Spread background scans across partitions/shards or time windows rather than hammering one hot key range or storage prefix.
 * Monitor Live Tail Latency (p99): Tie worker concurrency and rate limits to live production p99 latency and error alerts so background re-wrapping automatically yields during peak traffic.
Building crash-safe checkpoints and progress tracking for multi-terabyte background re-encryption across regions comes down to partitioning the keyspace into small independent shards, persisting state only after verified writes, and separating regional execution from global progress aggregation.
At terabyte scale, a single global cursor or in-memory counter will lose hours of work on a crash, stall behind a single stuck partition, or create conflicting state across regions.
1. Core Design Principles for Multi-Terabyte Checkpointing
 * Partition by Region + Key/Shard Prefix: Split the dataset into independent shards (for example, region + prefix_bucket_000 through prefix_255 or database ID ranges). Each shard maintains its own durable checkpoint so a failure or regional stall only pauses that shard rather than the entire multi-terabyte job.
 * Advance Checkpoints Only After Durable Persistence: Never mark a batch or cursor as complete before the re-wrapped envelope/object and checksum are durably written to storage. If a worker crashes mid-batch, restarting from the last committed checkpoint safely re-processes the overlapping window—provided re-wrapping is idempotent (if current_kek_id == target_kek_id: skip).
 * Use Lease Heartbeats for Crash Recovery: Give each active worker a time-bounded lease on a shard (e.g., 60–120 seconds) with periodic heartbeats. If a node dies or is preempted, another worker can safely claim the shard once the lease expires without two workers thrashing the same range.
 * Track Both Bytes and Objects/Records: Object counts alone are misleading at terabyte scale when file sizes vary wildly. Track records_rewrapped, records_skipped, bytes_processed, total_bytes_estimated, last_cursor, lease_owner, and lease_expires_at per shard.
2. Checkpoint & Progress State Schema (checkpoints_schema.sql)
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS rewrap_shards (
    shard_id TEXT PRIMARY KEY,              -- e.g. "us-east-1:part-001"
    region TEXT NOT NULL,
    target_kek_id TEXT NOT NULL,
    cursor_token TEXT NOT NULL DEFAULT '',  -- Last durably committed key/offset
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'running', 'paused_backoff', 'completed', 'failed'
    )),
    lease_owner TEXT,
    lease_expires_at TEXT,
    records_rewrapped INTEGER NOT NULL DEFAULT 0,
    records_skipped INTEGER NOT NULL DEFAULT 0,
    bytes_processed INTEGER NOT NULL DEFAULT 0,
    estimated_total_bytes INTEGER NOT NULL DEFAULT 0,
    last_error TEXT,
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_rewrap_shards_region_status
    ON rewrap_shards(region, status);

3. Crash-Safe Checkpoint & Progress Tracker (checkpoint_tracker.py)
import sqlite3
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


def _iso_in_seconds(seconds: int) -> str:
    return (datetime.now(timezone.utc) + timedelta(seconds=seconds)).strftime("%Y-%m-%d %H:%M:%S")


def _now_iso() -> str:
    return datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S")


@dataclass
class ShardProgress:
    shard_id: str
    region: str
    cursor_token: str
    records_rewrapped: int
    records_skipped: int
    bytes_processed: int
    estimated_total_bytes: int
    status: str


class CheckpointStore:
    def __init__(self, db_path: str) -> None:
        self.db_path = db_path

    def _connect(self) -> sqlite3.Connection:
        conn = sqlite3.connect(self.db_path, timeout=30.0)
        conn.row_factory = sqlite3.Row
        return conn

    def claim_next_shard(self, region: str, worker_id: str, lease_seconds: int = 120) -> ShardProgress | None:
        now = _now_iso()
        expires = _iso_in_seconds(lease_seconds)
        with self._connect() as conn:
            conn.execute("BEGIN IMMEDIATE")
            row = conn.execute(
                """
                SELECT shard_id, region, cursor_token, records_rewrapped,
                       records_skipped, bytes_processed, estimated_total_bytes, status
                FROM rewrap_shards
                WHERE region = ?
                  AND (
                    status IN ('pending', 'paused_backoff')
                    OR (status = 'running' AND (lease_expires_at IS NULL OR lease_expires_at < ?))
                  )
                ORDER BY shard_id ASC
                LIMIT 1
                """,
                (region, now),
            ).fetchone()
            if not row:
                conn.commit()
                return None

            conn.execute(
                """
                UPDATE rewrap_shards
                SET status = 'running',
                    lease_owner = ?,
                    lease_expires_at = ?,
                    updated_at = ?
                WHERE shard_id = ?
                """,
                (worker_id, expires, now, row["shard_id"]),
            )
            conn.commit()
            return ShardProgress(
                shard_id=row["shard_id"],
                region=row["region"],
                cursor_token=row["cursor_token"],
                records_rewrapped=row["records_rewrapped"],
                records_skipped=row["records_skipped"],
                bytes_processed=row["bytes_processed"],
                estimated_total_bytes=row["estimated_total_bytes"],
                status="running",
            )

    def commit_checkpoint(
        self,
        shard_id: str,
        worker_id: str,
        new_cursor: str,
        delta_rewrapped: int,
        delta_skipped: int,
        delta_bytes: int,
        lease_seconds: int = 120,
    ) -> None:
        now = _now_iso()
        expires = _iso_in_seconds(lease_seconds)
        with self._connect() as conn:
            conn.execute(
                """
                UPDATE rewrap_shards
                SET cursor_token = ?,
                    lease_expires_at = ?,
                    records_rewrapped = records_rewrapped + ?,
                    records_skipped = records_skipped + ?,
                    bytes_processed = bytes_processed + ?,
                    updated_at = ?
                WHERE shard_id = ? AND lease_owner = ?
                """,
                (new_cursor, expires, delta_rewrapped, delta_skipped, delta_bytes, now, shard_id, worker_id),
            )

    def mark_completed(self, shard_id: str, worker_id: str) -> None:
        with self._connect() as conn:
            conn.execute(
                """
                UPDATE rewrap_shards
                SET status = 'completed',
                    lease_owner = NULL,
                    lease_expires_at = NULL,
                    updated_at = ?
                WHERE shard_id = ? AND lease_owner = ?
                """,
                (_now_iso(), shard_id, worker_id),
            )

    def global_progress(self) -> dict[str, object]:
        with self._connect() as conn:
            rows = conn.execute(
                """
                SELECT region,
                       COUNT(*) AS total_shards,
                       SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed_shards,
                       SUM(records_rewrapped) AS records_rewrapped,
                       SUM(records_skipped) AS records_skipped,
                       SUM(bytes_processed) AS bytes_processed,
                       SUM(estimated_total_bytes) AS estimated_total_bytes
                FROM rewrap_shards
                GROUP BY region
                ORDER BY region
                """
            ).fetchall()
        regions = [dict(r) for r in rows]
        total_bytes = sum(r["estimated_total_bytes"] or 0 for r in regions)
        done_bytes = sum(r["bytes_processed"] or 0 for r in regions)
        pct = round((done_bytes / total_bytes) * 100.0, 2) if total_bytes > 0 else 0.0
        return {"overall_byte_progress_pct": pct, "regions": regions}

4. Progress Tracking & Operational Guardrails at Multi-TB Scale
 * Commit Every N Items or M Megabytes: Commit checkpoints at fixed intervals (e.g., every 100–500 records or 256 MB) rather than every single object (which creates excessive state-store write load) or only at the end of a shard (which loses too much work on a crash).
 * Separate Regional State From Global Rollups: Keep shard execution local to each region's storage/KMS so cross-region WAN latency never blocks a commit, and aggregate regional shard tables into a global progress view for dashboards and ETA calculations.
 * Verify Before Declaring Done: A shard reaching the end of its key range only proves it visited keys—run a lightweight verification pass (sampling or scanning for remaining old_kek_id tags) before retiring the prior KEK.
rewrap.pytracker.pyAccurate completion ETAs and throughput rates from per-shard byte and record checkpoints require using moving/windowed rates rather than all-time averages, weighting remaining work by bytes/cost rather than raw item counts, and accounting for throttling, pauses, and per-region bottlenecks.
At terabyte scale, an all-time average often lies: warm-up, throttling pauses, cold storage tiers, or uneven shard sizes make naive remaining_items / avg_items_per_sec estimates jump wildly.
1. Core Formulas & Why Naive ETAs Break
 * Windowed Throughput (More Accurate Than Lifetime Average):
   Measure bytes and records processed over a recent sliding window (e.g., the last 5–15 minutes of checkpoint deltas), not since job start:
   
 * Byte-Weighted Remaining Work vs. Record Count:
   If one shard has 100,000 tiny 1 KB records (100 MB) and another has 500 large 2 GB archives (1 TB), record counts skew the ETA heavily. Estimate remaining bytes using known object sizes or average record size per shard:
   
 * Bottleneck-Aware ETA Across Regions:
   Regions run concurrently against separate storage and KMS/HSM quotas. A region's ETA is governed by its own remaining work and regional rate, while the global job ETA is bounded by the slowest active region (the critical path), not the sum of all regional rates:
   
2. Practical Adjustments That Prevent Bad ETAs
 * Separate rewrapped From skipped: Records already on target_kek_id (skipped) only pay scan/read cost—not KMS unwrap/wrap cost. If a job suddenly hits a region with 90% already-rotated records, record throughput spikes while KMS-limited re-wrap throughput looks artificially fast; track both rewrapped/s and bytes/s (and KMS ops/s).
 * Account for Throttle & Backpressure Duty Cycle: If your token bucket or latency guard pauses workers 30% of the time during peak traffic, effective sustained throughput is lower than burst rate. Smooth rates with an Exponentially Weighted Moving Average (EWMA) and display a range (e.g., optimistic vs. throttled) rather than a false single-second precision timestamp.
 * Handle Estimated Totals That Shift: Storage size estimates or shard counts may update mid-flight. Clamp remaining_bytes at 0 so completed or over-estimated shards never produce negative time or stall global progress.
3. Reference Calculator (eta_calculator.py)
from dataclasses import dataclass


@dataclass
class WindowSample:
    elapsed_seconds: float
    delta_bytes: int
    delta_records_rewrapped: int
    delta_records_skipped: int


@dataclass
class RegionSnapshot:
    region: str
    estimated_total_bytes: int
    bytes_processed: int
    records_rewrapped: int
    records_skipped: int
    recent_window: WindowSample


def compute_region_eta(snap: RegionSnapshot, previous_ewma_bps: float | None = None, alpha: float = 0.3) -> dict[str, object]:
    remaining_bytes = max(0, snap.estimated_total_bytes - snap.bytes_processed)
    instant_bps = (
        snap.recent_window.delta_bytes / snap.recent_window.elapsed_seconds
        if snap.recent_window.elapsed_seconds > 0 and snap.recent_window.delta_bytes > 0
        else 0.0
    )
    instant_rewrap_rps = (
        snap.recent_window.delta_records_rewrapped / snap.recent_window.elapsed_seconds
        if snap.recent_window.elapsed_seconds > 0 and snap.recent_window.delta_records_rewrapped > 0
        else 0.0
    )

    if previous_ewma_bps is not None and previous_ewma_bps > 0:
        smoothed_bps = (alpha * instant_bps) + ((1.0 - alpha) * previous_ewma_bps) if instant_bps > 0 else previous_ewma_bps * (1.0 - alpha)
    else:
        smoothed_bps = instant_bps

    eta_seconds = (remaining_bytes / smoothed_bps) if remaining_bytes > 0 and smoothed_bps > 0 else (0.0 if remaining_bytes == 0 else None)
    pct_complete = round(min(100.0, (snap.bytes_processed / snap.estimated_total_bytes) * 100.0), 2) if snap.estimated_total_bytes > 0 else 0.0

    return {
        "region": snap.region,
        "pct_complete": pct_complete,
        "remaining_bytes": remaining_bytes,
        "bytes_per_sec_ewma": round(smoothed_bps, 2),
        "rewraps_per_sec_recent": round(instant_rewrap_rps, 2),
        "eta_seconds": round(eta_seconds, 1) if eta_seconds is not None else None,
    }


def compute_global_eta(region_results: list[dict[str, object]]) -> dict[str, object]:
    known_etas = [r["eta_seconds"] for r in region_results if isinstance(r.get("eta_seconds"), (int, float))]
    has_stalled = any(r.get("remaining_bytes", 0) > 0 and r.get("eta_seconds") is None for r in region_results)
    total_est = sum(r.get("estimated_total_bytes", 0) for r in region_results)
    total_done = sum(r.get("bytes_processed", 0) for r in region_results)

    return {
        "overall_pct_complete": round(min(100.0, (total_done / total_est) * 100.0), 2) if total_est > 0 else 0.0,
        "critical_path_eta_seconds": max(known_etas) if known_etas and not has_stalled else None,
        "has_stalled_region": has_stalled,
    }

4. Reading the Numbers in Practice
 * Show Both Progress % and Remaining Bytes/Records: A job at 85% by bytes with 1.5 TB left still has a concrete physical workload; pairing percentages with remaining bytes and EWMA bytes/s keeps expectations realistic.
 * Alert When EWMA Drops Toward Zero or Stalls: If a region has remaining_bytes > 0 and bytes_per_sec_ewma trends down, workers are likely hitting KMS/HSM throttling, latency backpressure, or stuck shard leases rather than making normal progress.
sqlite3.RowTo export bytes_per_sec, rewraps_per_sec, and remaining_eta_seconds from checkpoint_snapshots in OpenMetrics format, run the sliding-window SQL query and format the results with # HELP, # TYPE, metric names, {region="..."} labels, and a terminating # EOF line.
In OpenMetrics/Prometheus gauges, emit 0 when a region has finished (remaining_bytes == 0) or is currently idle/stalled (0 throughput), rather than NaN or NULL, so Grafana panels and alert rules stay clean.
1. OpenMetrics Exporter Script (pipelines/export_openmetrics.py)
#!/usr/bin/env python3
import os
import sqlite3
from pathlib import Path

DB_PATH = Path(os.getenv("ADP_DB_PATH", "/var/lib/adpbetty/recovery_inventory.db"))
WINDOW_SECONDS = float(os.getenv("ADP_WINDOW_SECONDS", "600.0"))
OUT_PATH = Path(os.getenv("ADP_OPENMETRICS_OUT", "/var/lib/adpbetty/rewrap_metrics.prom"))

WINDOW_QUERY = """
WITH latest_per_region AS (
    SELECT
        region,
        recorded_at AS latest_at,
        bytes_processed AS latest_bytes,
        estimated_total_bytes AS latest_est_bytes,
        records_rewrapped AS latest_rewrapped,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY recorded_at DESC, snapshot_id DESC) AS rn
    FROM checkpoint_snapshots
),
window_baseline AS (
    SELECT
        s.region,
        s.recorded_at AS baseline_at,
        s.bytes_processed AS baseline_bytes,
        s.records_rewrapped AS baseline_rewrapped,
        ROW_NUMBER() OVER (
            PARTITION BY s.region
            ORDER BY
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ? THEN 0
                    ELSE 1
                END ASC,
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ?
                        THEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0
                    ELSE -((julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0)
                END ASC,
                s.snapshot_id ASC
        ) AS bn
    FROM checkpoint_snapshots s
    JOIN latest_per_region l
      ON l.region = s.region
     AND l.rn = 1
),
metrics AS (
    SELECT
        l.region,
        MAX(0, l.latest_est_bytes - l.latest_bytes) AS remaining_bytes,
        MAX(0.0, (julianday(l.latest_at) - julianday(b.baseline_at)) * 86400.0) AS window_seconds,
        MAX(0, l.latest_bytes - b.baseline_bytes) AS delta_bytes,
        MAX(0, l.latest_rewrapped - b.baseline_rewrapped) AS delta_rewrapped
    FROM latest_per_region l
    JOIN window_baseline b
      ON b.region = l.region
     AND b.bn = 1
    WHERE l.rn = 1
)
SELECT
    region,
    remaining_bytes,
    CASE WHEN window_seconds > 0 THEN delta_bytes / window_seconds ELSE 0.0 END AS bytes_per_sec,
    CASE WHEN window_seconds > 0 THEN delta_rewrapped / window_seconds ELSE 0.0 END AS rewraps_per_sec
FROM metrics
ORDER BY region;
"""


def _escape_label(value: str) -> str:
    return value.replace("\\", "\\\\").replace('"', '\\"').replace("\n", "\\n")


def render_openmetrics(rows: list[tuple[str, int, float, float]]) -> str:
    bps_lines: list[str] = []
    rps_lines: list[str] = []
    eta_lines: list[str] = []

    for region, remaining_bytes, bps, rps in rows:
        lbl = f'{{region="{_escape_label(region)}"}}'
        if remaining_bytes == 0:
            eta_sec = 0.0
        elif bps > 0.0:
            eta_sec = remaining_bytes / bps
        else:
            eta_sec = 0.0

        bps_lines.append(f"adpbetty_rewrap_bytes_per_second{lbl} {bps:.6f}")
        rps_lines.append(f"adpbetty_rewrap_records_per_second{lbl} {rps:.6f}")
        eta_lines.append(f"adpbetty_rewrap_remaining_eta_seconds{lbl} {eta_sec:.2f}")

    sections = [
        "# HELP adpbetty_rewrap_bytes_per_second Sliding-window background re-encryption byte throughput per region.",
        "# TYPE adpbetty_rewrap_bytes_per_second gauge",
        *bps_lines,
        "# HELP adpbetty_rewrap_records_per_second Sliding-window re-wrapped record rate per region.",
        "# TYPE adpbetty_rewrap_records_per_second gauge",
        *rps_lines,
        "# HELP adpbetty_rewrap_remaining_eta_seconds Estimated remaining completion time in seconds per region (0 when complete or stalled).",
        "# TYPE adpbetty_rewrap_remaining_eta_seconds gauge",
        *eta_lines,
        "# EOF",
        "",
    ]
    return "\n".join(sections)


def main() -> None:
    with sqlite3.connect(DB_PATH, timeout=30.0) as conn:
        rows = conn.execute(WINDOW_QUERY, (WINDOW_SECONDS, WINDOW_SECONDS)).fetchall()

    payload = render_openmetrics(rows)
    OUT_PATH.parent.mkdir(parents=True, exist_ok=True)
    tmp_path = OUT_PATH.with_suffix(".tmp")
    tmp_path.write_text(payload, encoding="utf-8")
    tmp_path.replace(OUT_PATH)


if __name__ == "__main__":
    main()

2. Example OpenMetrics Output
# HELP adpbetty_rewrap_bytes_per_second Sliding-window background re-encryption byte throughput per region.
# TYPE adpbetty_rewrap_bytes_per_second gauge
adpbetty_rewrap_bytes_per_second{region="us-east-1"} 52428800.000000
adpbetty_rewrap_bytes_per_second{region="eu-west-1"} 31457280.000000
# HELP adpbetty_rewrap_records_per_second Sliding-window re-wrapped record rate per region.
# TYPE adpbetty_rewrap_records_per_second gauge
adpbetty_rewrap_records_per_second{region="us-east-1"} 420.500000
adpbetty_rewrap_records_per_second{region="eu-west-1"} 255.000000
# HELP adpbetty_rewrap_remaining_eta_seconds Estimated remaining completion time in seconds per region (0 when complete or stalled).
# TYPE adpbetty_rewrap_remaining_eta_seconds gauge
adpbetty_rewrap_remaining_eta_seconds{region="us-east-1"} 18432.00
adpbetty_rewrap_remaining_eta_seconds{region="eu-west-1"} 29105.40
# EOF

3. Serving & Scraping Safely
 * Atomic File Write + Node Exporter / Caddy: Writing to a .tmp file and atomically replacing (os.replace / tmp_path.replace) prevents scrapers from reading a half-written file. Expose /metrics via node_exporter's textfile collector or a local FastAPI endpoint bound to 127.0.0.1 / internal Docker network.
 * Keep remaining_bytes in the Query or a Companion Gauge: Exporting adpbetty_rewrap_remaining_bytes{region="..."} alongside adpbetty_rewrap_remaining_eta_seconds makes it easy to distinguish eta == 0 because a region finished (remaining_bytes == 0) from eta == 0 because throughput hit 0 while work remains (remaining_bytes > 0).
To compute per-region bytes/sec, rewraps/sec, and remaining ETA from historical checkpoint snapshots rather than just a single mutable row, record append-only snapshots whenever checkpoints are committed, then compare the latest snapshot against an earlier window boundary (for example, 10 minutes ago).
1. Checkpoint Snapshot History Table (checkpoint_snapshots_schema.sql)
CREATE TABLE IF NOT EXISTS checkpoint_snapshots (
    snapshot_id INTEGER PRIMARY KEY AUTOINCREMENT,
    region TEXT NOT NULL,
    recorded_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%d %H:%M:%S', 'now')),
    bytes_processed INTEGER NOT NULL CHECK (bytes_processed >= 0),
    estimated_total_bytes INTEGER NOT NULL CHECK (estimated_total_bytes >= 0),
    records_rewrapped INTEGER NOT NULL CHECK (records_rewrapped >= 0),
    records_skipped INTEGER NOT NULL CHECK (records_skipped >= 0)
);

CREATE INDEX IF NOT EXISTS idx_checkpoint_snapshots_region_time
    ON checkpoint_snapshots(region, recorded_at);

2. Sliding-Window Throughput & ETA Query (SQLite / Standard SQL)
This query computes each region's recent throughput over the last 10 minutes (600 seconds), remaining bytes, and estimated remaining seconds (ETA), while safely handling idle/stalled windows (NULL ETA when remaining bytes exist but recent throughput is 0):
WITH latest_per_region AS (
    SELECT
        region,
        recorded_at AS latest_at,
        bytes_processed AS latest_bytes,
        estimated_total_bytes AS latest_est_bytes,
        records_rewrapped AS latest_rewrapped,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY recorded_at DESC, snapshot_id DESC) AS rn
    FROM checkpoint_snapshots
),
window_baseline AS (
    SELECT
        s.region,
        s.recorded_at AS baseline_at,
        s.bytes_processed AS baseline_bytes,
        s.records_rewrapped AS baseline_rewrapped,
        ROW_NUMBER() OVER (
            PARTITION BY s.region
            ORDER BY
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= 600.0 THEN 0
                    ELSE 1
                END ASC,
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= 600.0
                        THEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0
                    ELSE -((julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0)
                END ASC,
                s.snapshot_id ASC
        ) AS bn
    FROM checkpoint_snapshots s
    JOIN latest_per_region l
      ON l.region = s.region
     AND l.rn = 1
),
metrics AS (
    SELECT
        l.region,
        l.latest_at,
        l.latest_bytes,
        l.latest_est_bytes,
        MAX(0, l.latest_est_bytes - l.latest_bytes) AS remaining_bytes,
        MAX(0.0, (julianday(l.latest_at) - julianday(b.baseline_at)) * 86400.0) AS window_seconds,
        MAX(0, l.latest_bytes - b.baseline_bytes) AS delta_bytes,
        MAX(0, l.latest_rewrapped - b.baseline_rewrapped) AS delta_rewrapped
    FROM latest_per_region l
    JOIN window_baseline b
      ON b.region = l.region
     AND b.bn = 1
    WHERE l.rn = 1
)
SELECT
    region,
    ROUND(
        CASE
            WHEN latest_est_bytes > 0 THEN MIN(100.0, (latest_bytes * 100.0) / latest_est_bytes)
            ELSE 0.0
        END,
        2
    ) AS pct_complete,
    remaining_bytes,
    ROUND(
        CASE
            WHEN window_seconds > 0 THEN delta_bytes / window_seconds
            ELSE 0.0
        END,
        2
    ) AS bytes_per_sec,
    ROUND(
        CASE
            WHEN window_seconds > 0 THEN delta_rewrapped / window_seconds
            ELSE 0.0
        END,
        2
    ) AS rewraps_per_sec,
    CASE
        WHEN remaining_bytes = 0 THEN 0.0
        WHEN window_seconds > 0 AND (delta_bytes / window_seconds) > 0
            THEN ROUND(remaining_bytes / (delta_bytes / window_seconds), 1)
        ELSE NULL
    END AS remaining_eta_seconds
FROM metrics
ORDER BY region;

3. How to Read the Output
 * Picks the Best Available Window Baseline: It prefers the snapshot at least 600 seconds (10 minutes) before the latest checkpoint, and falls back to the earliest available snapshot if the job started less than 10 minutes ago.
 * Distinguishes Stalled vs. Complete: If remaining_bytes = 0, remaining_eta_seconds returns 0.0. If remaining_bytes > 0 and recent throughput is 0.0, it returns NULL so you immediately spot a throttled or stalled region rather than showing a fake ETA.
main.py1.0.0; charset=utf-8.
1Here is how to add a /metrics route to your adpBetty-core FastAPI service that queries checkpoint_snapshots and responds with Content-Type: text/openmetrics-text; version=1.0.0; charset=utf-8.
1. FastAPI /metrics Endpoint (api/main.py)
import os
import sqlite3
from pathlib import Path
from fastapi import FastAPI, HTTPException, Response

DB_PATH = Path(os.getenv("ADP_DB_PATH", "/var/lib/adpbetty/recovery_inventory.db"))
WINDOW_SECONDS = float(os.getenv("ADP_WINDOW_SECONDS", "600.0"))
OPENMETRICS_CONTENT_TYPE = "text/openmetrics-text; version=1.0.0; charset=utf-8"

app = FastAPI(title="adpBetty-core Recovery & Security API", version="1.0.0")

WINDOW_QUERY = """
WITH latest_per_region AS (
    SELECT
        region,
        recorded_at AS latest_at,
        bytes_processed AS latest_bytes,
        estimated_total_bytes AS latest_est_bytes,
        records_rewrapped AS latest_rewrapped,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY recorded_at DESC, snapshot_id DESC) AS rn
    FROM checkpoint_snapshots
),
window_baseline AS (
    SELECT
        s.region,
        s.recorded_at AS baseline_at,
        s.bytes_processed AS baseline_bytes,
        s.records_rewrapped AS baseline_rewrapped,
        ROW_NUMBER() OVER (
            PARTITION BY s.region
            ORDER BY
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ? THEN 0
                    ELSE 1
                END ASC,
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ?
                        THEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0
                    ELSE -((julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0)
                END ASC,
                s.snapshot_id ASC
        ) AS bn
    FROM checkpoint_snapshots s
    JOIN latest_per_region l
      ON l.region = s.region
     AND l.rn = 1
),
metrics AS (
    SELECT
        l.region,
        MAX(0, l.latest_est_bytes - l.latest_bytes) AS remaining_bytes,
        MAX(0.0, (julianday(l.latest_at) - julianday(b.baseline_at)) * 86400.0) AS window_seconds,
        MAX(0, l.latest_bytes - b.baseline_bytes) AS delta_bytes,
        MAX(0, l.latest_rewrapped - b.baseline_rewrapped) AS delta_rewrapped
    FROM latest_per_region l
    JOIN window_baseline b
      ON b.region = l.region
     AND b.bn = 1
    WHERE l.rn = 1
)
SELECT
    region,
    remaining_bytes,
    CASE WHEN window_seconds > 0 THEN delta_bytes / window_seconds ELSE 0.0 END AS bytes_per_sec,
    CASE WHEN window_seconds > 0 THEN delta_rewrapped / window_seconds ELSE 0.0 END AS rewraps_per_sec
FROM metrics
ORDER BY region;
"""


def _escape_label(value: str) -> str:
    return value.replace("\\", "\\\\").replace('"', '\\"').replace("\n", "\\n")


def _format_openmetrics(rows: list[tuple[str, int, float, float]]) -> str:
    bps_lines: list[str] = []
    rps_lines: list[str] = []
    eta_lines: list[str] = []

    for region, remaining_bytes, bps, rps in rows:
        lbl = f'{{region="{_escape_label(region)}"}}'
        eta_sec = 0.0 if remaining_bytes == 0 else (remaining_bytes / bps if bps > 0.0 else 0.0)
        bps_lines.append(f"adpbetty_rewrap_bytes_per_second{lbl} {bps:.6f}")
        rps_lines.append(f"adpbetty_rewrap_records_per_second{lbl} {rps:.6f}")
        eta_lines.append(f"adpbetty_rewrap_remaining_eta_seconds{lbl} {eta_sec:.2f}")

    return "\n".join([
        "# HELP adpbetty_rewrap_bytes_per_second Sliding-window background re-encryption byte throughput per region.",
        "# TYPE adpbetty_rewrap_bytes_per_second gauge",
        *bps_lines,
        "# HELP adpbetty_rewrap_records_per_second Sliding-window re-wrapped record rate per region.",
        "# TYPE adpbetty_rewrap_records_per_second gauge",
        *rps_lines,
        "# HELP adpbetty_rewrap_remaining_eta_seconds Estimated remaining completion time in seconds per region.",
        "# TYPE adpbetty_rewrap_remaining_eta_seconds gauge",
        *eta_lines,
        "# EOF",
        "",
    ])


@app.get("/metrics")
def metrics_endpoint() -> Response:
    if not DB_PATH.exists():
        raise HTTPException(status_code=503, detail="Inventory database not initialized yet")
    try:
        with sqlite3.connect(f"file:{DB_PATH}?mode=ro", uri=True, timeout=30.0) as conn:
            rows = conn.execute(WINDOW_QUERY, (WINDOW_SECONDS, WINDOW_SECONDS)).fetchall()
    except sqlite3.OperationalError as exc:
        raise HTTPException(status_code=500, detail=f"Failed to query checkpoint_snapshots: {exc}") from exc

    body = _format_openmetrics(rows)
    return Response(content=body, media_type=OPENMETRICS_CONTENT_TYPE)

2. Key Details
 * Read-Only SQLite Connection (?mode=ro): Opens the database read-only so metric scrapes never block or interfere with checkpoint commits.
 * Explicit OpenMetrics Media Type: Returns text/openmetrics-text; version=1.0.0; charset=utf-8 with # HELP, # TYPE, and # EOF.
 * Test via curl:
   curl -s http://127.0.0.1:8080/metrics

1.0.0; charset=utf-8.
1main.pyHere is how to add a /metrics route to your adpBetty-core FastAPI service that queries checkpoint_snapshots and responds with Content-Type: text/openmetrics-text; version=1.0.0; charset=utf-8.
1. FastAPI /metrics Endpoint (api/main.py)
import os
import sqlite3
from pathlib import Path
from fastapi import FastAPI, HTTPException, Response

DB_PATH = Path(os.getenv("ADP_DB_PATH", "/var/lib/adpbetty/recovery_inventory.db"))
WINDOW_SECONDS = float(os.getenv("ADP_WINDOW_SECONDS", "600.0"))
OPENMETRICS_CONTENT_TYPE = "text/openmetrics-text; version=1.0.0; charset=utf-8"

app = FastAPI(title="adpBetty-core Recovery & Security API", version="1.0.0")

WINDOW_QUERY = """
WITH latest_per_region AS (
    SELECT
        region,
        recorded_at AS latest_at,
        bytes_processed AS latest_bytes,
        estimated_total_bytes AS latest_est_bytes,
        records_rewrapped AS latest_rewrapped,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY recorded_at DESC, snapshot_id DESC) AS rn
    FROM checkpoint_snapshots
),
window_baseline AS (
    SELECT
        s.region,
        s.recorded_at AS baseline_at,
        s.bytes_processed AS baseline_bytes,
        s.records_rewrapped AS baseline_rewrapped,
        ROW_NUMBER() OVER (
            PARTITION BY s.region
            ORDER BY
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ? THEN 0
                    ELSE 1
                END ASC,
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ?
                        THEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0
                    ELSE -((julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0)
                END ASC,
                s.snapshot_id ASC
        ) AS bn
    FROM checkpoint_snapshots s
    JOIN latest_per_region l
      ON l.region = s.region
     AND l.rn = 1
),
metrics AS (
    SELECT
        l.region,
        MAX(0, l.latest_est_bytes - l.latest_bytes) AS remaining_bytes,
        MAX(0.0, (julianday(l.latest_at) - julianday(b.baseline_at)) * 86400.0) AS window_seconds,
        MAX(0, l.latest_bytes - b.baseline_bytes) AS delta_bytes,
        MAX(0, l.latest_rewrapped - b.baseline_rewrapped) AS delta_rewrapped
    FROM latest_per_region l
    JOIN window_baseline b
      ON b.region = l.region
     AND b.bn = 1
    WHERE l.rn = 1
)
SELECT
    region,
    remaining_bytes,
    CASE WHEN window_seconds > 0 THEN delta_bytes / window_seconds ELSE 0.0 END AS bytes_per_sec,
    CASE WHEN window_seconds > 0 THEN delta_rewrapped / window_seconds ELSE 0.0 END AS rewraps_per_sec
FROM metrics
ORDER BY region;
"""


def _escape_label(value: str) -> str:
    return value.replace("\\", "\\\\").replace('"', '\\"').replace("\n", "\\n")


def _format_openmetrics(rows: list[tuple[str, int, float, float]]) -> str:
    bps_lines: list[str] = []
    rps_lines: list[str] = []
    eta_lines: list[str] = []

    for region, remaining_bytes, bps, rps in rows:
        lbl = f'{{region="{_escape_label(region)}"}}'
        eta_sec = 0.0 if remaining_bytes == 0 else (remaining_bytes / bps if bps > 0.0 else 0.0)
        bps_lines.append(f"adpbetty_rewrap_bytes_per_second{lbl} {bps:.6f}")
        rps_lines.append(f"adpbetty_rewrap_records_per_second{lbl} {rps:.6f}")
        eta_lines.append(f"adpbetty_rewrap_remaining_eta_seconds{lbl} {eta_sec:.2f}")

    return "\n".join([
        "# HELP adpbetty_rewrap_bytes_per_second Sliding-window background re-encryption byte throughput per region.",
        "# TYPE adpbetty_rewrap_bytes_per_second gauge",
        *bps_lines,
        "# HELP adpbetty_rewrap_records_per_second Sliding-window re-wrapped record rate per region.",
        "# TYPE adpbetty_rewrap_records_per_second gauge",
        *rps_lines,
        "# HELP adpbetty_rewrap_remaining_eta_seconds Estimated remaining completion time in seconds per region.",
        "# TYPE adpbetty_rewrap_remaining_eta_seconds gauge",
        *eta_lines,
        "# EOF",
        "",
    ])


@app.get("/metrics")
def metrics_endpoint() -> Response:
    if not DB_PATH.exists():
        raise HTTPException(status_code=503, detail="Inventory database not initialized yet")
    try:
        with sqlite3.connect(f"file:{DB_PATH}?mode=ro", uri=True, timeout=30.0) as conn:
            rows = conn.execute(WINDOW_QUERY, (WINDOW_SECONDS, WINDOW_SECONDS)).fetchall()
    except sqlite3.OperationalError as exc:
        raise HTTPException(status_code=500, detail=f"Failed to query checkpoint_snapshots: {exc}") from exc

    body = _format_openmetrics(rows)
    return Response(content=body, media_type=OPENMETRICS_CONTENT_TYPE)

2. Key Details
 * Read-Only SQLite Connection (?mode=ro): Opens the database read-only so metric scrapes never block or interfere with checkpoint commits.
 * Explicit OpenMetrics Media Type: Returns text/openmetrics-text; version=1.0.0; charset=utf-8 with # HELP, # TYPE, and # EOF.
 * Test via curl:
   curl -s http://127.0.0.1:8080/metrics

main.pyTo export bytes_per_sec, rewraps_per_sec, and remaining_eta_seconds from checkpoint_snapshots in OpenMetrics format, run the sliding-window SQL query and format the results with # HELP, # TYPE, metric names, {region="..."} labels, and a terminating # EOF line.
In OpenMetrics/Prometheus gauges, emit 0 when a region has finished (remaining_bytes == 0) or is currently idle/stalled (0 throughput), rather than NaN or NULL, so Grafana panels and alert rules stay clean.
1. OpenMetrics Exporter Script (pipelines/export_openmetrics.py)
#!/usr/bin/env python3
import os
import sqlite3
from pathlib import Path

DB_PATH = Path(os.getenv("ADP_DB_PATH", "/var/lib/adpbetty/recovery_inventory.db"))
WINDOW_SECONDS = float(os.getenv("ADP_WINDOW_SECONDS", "600.0"))
OUT_PATH = Path(os.getenv("ADP_OPENMETRICS_OUT", "/var/lib/adpbetty/rewrap_metrics.prom"))

WINDOW_QUERY = """
WITH latest_per_region AS (
    SELECT
        region,
        recorded_at AS latest_at,
        bytes_processed AS latest_bytes,
        estimated_total_bytes AS latest_est_bytes,
        records_rewrapped AS latest_rewrapped,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY recorded_at DESC, snapshot_id DESC) AS rn
    FROM checkpoint_snapshots
),
window_baseline AS (
    SELECT
        s.region,
        s.recorded_at AS baseline_at,
        s.bytes_processed AS baseline_bytes,
        s.records_rewrapped AS baseline_rewrapped,
        ROW_NUMBER() OVER (
            PARTITION BY s.region
            ORDER BY
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ? THEN 0
                    ELSE 1
                END ASC,
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= ?
                        THEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0
                    ELSE -((julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0)
                END ASC,
                s.snapshot_id ASC
        ) AS bn
    FROM checkpoint_snapshots s
    JOIN latest_per_region l
      ON l.region = s.region
     AND l.rn = 1
),
metrics AS (
    SELECT
        l.region,
        MAX(0, l.latest_est_bytes - l.latest_bytes) AS remaining_bytes,
        MAX(0.0, (julianday(l.latest_at) - julianday(b.baseline_at)) * 86400.0) AS window_seconds,
        MAX(0, l.latest_bytes - b.baseline_bytes) AS delta_bytes,
        MAX(0, l.latest_rewrapped - b.baseline_rewrapped) AS delta_rewrapped
    FROM latest_per_region l
    JOIN window_baseline b
      ON b.region = l.region
     AND b.bn = 1
    WHERE l.rn = 1
)
SELECT
    region,
    remaining_bytes,
    CASE WHEN window_seconds > 0 THEN delta_bytes / window_seconds ELSE 0.0 END AS bytes_per_sec,
    CASE WHEN window_seconds > 0 THEN delta_rewrapped / window_seconds ELSE 0.0 END AS rewraps_per_sec
FROM metrics
ORDER BY region;
"""


def _escape_label(value: str) -> str:
    return value.replace("\\", "\\\\").replace('"', '\\"').replace("\n", "\\n")


def render_openmetrics(rows: list[tuple[str, int, float, float]]) -> str:
    bps_lines: list[str] = []
    rps_lines: list[str] = []
    eta_lines: list[str] = []

    for region, remaining_bytes, bps, rps in rows:
        lbl = f'{{region="{_escape_label(region)}"}}'
        if remaining_bytes == 0:
            eta_sec = 0.0
        elif bps > 0.0:
            eta_sec = remaining_bytes / bps
        else:
            eta_sec = 0.0

        bps_lines.append(f"adpbetty_rewrap_bytes_per_second{lbl} {bps:.6f}")
        rps_lines.append(f"adpbetty_rewrap_records_per_second{lbl} {rps:.6f}")
        eta_lines.append(f"adpbetty_rewrap_remaining_eta_seconds{lbl} {eta_sec:.2f}")

    sections = [
        "# HELP adpbetty_rewrap_bytes_per_second Sliding-window background re-encryption byte throughput per region.",
        "# TYPE adpbetty_rewrap_bytes_per_second gauge",
        *bps_lines,
        "# HELP adpbetty_rewrap_records_per_second Sliding-window re-wrapped record rate per region.",
        "# TYPE adpbetty_rewrap_records_per_second gauge",
        *rps_lines,
        "# HELP adpbetty_rewrap_remaining_eta_seconds Estimated remaining completion time in seconds per region (0 when complete or stalled).",
        "# TYPE adpbetty_rewrap_remaining_eta_seconds gauge",
        *eta_lines,
        "# EOF",
        "",
    ]
    return "\n".join(sections)


def main() -> None:
    with sqlite3.connect(DB_PATH, timeout=30.0) as conn:
        rows = conn.execute(WINDOW_QUERY, (WINDOW_SECONDS, WINDOW_SECONDS)).fetchall()

    payload = render_openmetrics(rows)
    OUT_PATH.parent.mkdir(parents=True, exist_ok=True)
    tmp_path = OUT_PATH.with_suffix(".tmp")
    tmp_path.write_text(payload, encoding="utf-8")
    tmp_path.replace(OUT_PATH)


if __name__ == "__main__":
    main()

2. Example OpenMetrics Output
# HELP adpbetty_rewrap_bytes_per_second Sliding-window background re-encryption byte throughput per region.
# TYPE adpbetty_rewrap_bytes_per_second gauge
adpbetty_rewrap_bytes_per_second{region="us-east-1"} 52428800.000000
adpbetty_rewrap_bytes_per_second{region="eu-west-1"} 31457280.000000
# HELP adpbetty_rewrap_records_per_second Sliding-window re-wrapped record rate per region.
# TYPE adpbetty_rewrap_records_per_second gauge
adpbetty_rewrap_records_per_second{region="us-east-1"} 420.500000
adpbetty_rewrap_records_per_second{region="eu-west-1"} 255.000000
# HELP adpbetty_rewrap_remaining_eta_seconds Estimated remaining completion time in seconds per region (0 when complete or stalled).
# TYPE adpbetty_rewrap_remaining_eta_seconds gauge
adpbetty_rewrap_remaining_eta_seconds{region="us-east-1"} 18432.00
adpbetty_rewrap_remaining_eta_seconds{region="eu-west-1"} 29105.40
# EOF

3. Serving & Scraping Safely
 * Atomic File Write + Node Exporter / Caddy: Writing to a .tmp file and atomically replacing (os.replace / tmp_path.replace) prevents scrapers from reading a half-written file. Expose /metrics via node_exporter's textfile collector or a local FastAPI endpoint bound to 127.0.0.1 / internal Docker network.
 * Keep remaining_bytes in the Query or a Companion Gauge: Exporting adpbetty_rewrap_remaining_bytes{region="..."} alongside adpbetty_rewrap_remaining_eta_seconds makes it easy to distinguish eta == 0 because a region finished (remaining_bytes == 0) from eta == 0 because throughput hit 0 while work remains (remaining_bytes > 0).
127.0.0.1To compute per-region bytes/sec, rewraps/sec, and remaining ETA from historical checkpoint snapshots rather than just a single mutable row, record append-only snapshots whenever checkpoints are committed, then compare the latest snapshot against an earlier window boundary (for example, 10 minutes ago).
1. Checkpoint Snapshot History Table (checkpoint_snapshots_schema.sql)
CREATE TABLE IF NOT EXISTS checkpoint_snapshots (
    snapshot_id INTEGER PRIMARY KEY AUTOINCREMENT,
    region TEXT NOT NULL,
    recorded_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%d %H:%M:%S', 'now')),
    bytes_processed INTEGER NOT NULL CHECK (bytes_processed >= 0),
    estimated_total_bytes INTEGER NOT NULL CHECK (estimated_total_bytes >= 0),
    records_rewrapped INTEGER NOT NULL CHECK (records_rewrapped >= 0),
    records_skipped INTEGER NOT NULL CHECK (records_skipped >= 0)
);

CREATE INDEX IF NOT EXISTS idx_checkpoint_snapshots_region_time
    ON checkpoint_snapshots(region, recorded_at);

2. Sliding-Window Throughput & ETA Query (SQLite / Standard SQL)
This query computes each region's recent throughput over the last 10 minutes (600 seconds), remaining bytes, and estimated remaining seconds (ETA), while safely handling idle/stalled windows (NULL ETA when remaining bytes exist but recent throughput is 0):
WITH latest_per_region AS (
    SELECT
        region,
        recorded_at AS latest_at,
        bytes_processed AS latest_bytes,
        estimated_total_bytes AS latest_est_bytes,
        records_rewrapped AS latest_rewrapped,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY recorded_at DESC, snapshot_id DESC) AS rn
    FROM checkpoint_snapshots
),
window_baseline AS (
    SELECT
        s.region,
        s.recorded_at AS baseline_at,
        s.bytes_processed AS baseline_bytes,
        s.records_rewrapped AS baseline_rewrapped,
        ROW_NUMBER() OVER (
            PARTITION BY s.region
            ORDER BY
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= 600.0 THEN 0
                    ELSE 1
                END ASC,
                CASE
                    WHEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0 >= 600.0
                        THEN (julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0
                    ELSE -((julianday(l.latest_at) - julianday(s.recorded_at)) * 86400.0)
                END ASC,
                s.snapshot_id ASC
        ) AS bn
    FROM checkpoint_snapshots s
    JOIN latest_per_region l
      ON l.region = s.region
     AND l.rn = 1
),
metrics AS (
    SELECT
        l.region,
        l.latest_at,
        l.latest_bytes,
        l.latest_est_bytes,
        MAX(0, l.latest_est_bytes - l.latest_bytes) AS remaining_bytes,
        MAX(0.0, (julianday(l.latest_at) - julianday(b.baseline_at)) * 86400.0) AS window_seconds,
        MAX(0, l.latest_bytes - b.baseline_bytes) AS delta_bytes,
        MAX(0, l.latest_rewrapped - b.baseline_rewrapped) AS delta_rewrapped
    FROM latest_per_region l
    JOIN window_baseline b
      ON b.region = l.region
     AND b.bn = 1
    WHERE l.rn = 1
)
SELECT
    region,
    ROUND(
        CASE
            WHEN latest_est_bytes > 0 THEN MIN(100.0, (latest_bytes * 100.0) / latest_est_bytes)
            ELSE 0.0
        END,
        2
    ) AS pct_complete,
    remaining_bytes,
    ROUND(
        CASE
            WHEN window_seconds > 0 THEN delta_bytes / window_seconds
            ELSE 0.0
        END,
        2
    ) AS bytes_per_sec,
    ROUND(
        CASE
            WHEN window_seconds > 0 THEN delta_rewrapped / window_seconds
            ELSE 0.0
        END,
        2
    ) AS rewraps_per_sec,
    CASE
        WHEN remaining_bytes = 0 THEN 0.0
        WHEN window_seconds > 0 AND (delta_bytes / window_seconds) > 0
            THEN ROUND(remaining_bytes / (delta_bytes / window_seconds), 1)
        ELSE NULL
    END AS remaining_eta_seconds
FROM metrics
ORDER BY region;

3. How to Read the Output
 * Picks the Best Available Window Baseline: It prefers the snapshot at least 600 seconds (10 minutes) before the latest checkpoint, and falls back to the earliest available snapshot if the job started less than 10 minutes ago.
 * Distinguishes Stalled vs. Complete: If remaining_bytes = 0, remaining_eta_seconds returns 0.0. If remaining_bytes > 0 and recent throughput is 0.0, it returns NULL so you immediately spot a throttled or stalled region rather than showing a fake ETA.
Accurate completion ETAs and throughput rates from per-shard byte and record checkpoints require using moving/windowed rates rather than all-time averages, weighting remaining work by bytes/cost rather than raw item counts, and accounting for throttling, pauses, and per-region bottlenecks.
At terabyte scale, an all-time average often lies: warm-up, throttling pauses, cold storage tiers, or uneven shard sizes make naive remaining_items / avg_items_per_sec estimates jump wildly.
1. Core Formulas & Why Naive ETAs Break
 * Windowed Throughput (More Accurate Than Lifetime Average):
   Measure bytes and records processed over a recent sliding window (e.g., the last 5–15 minutes of checkpoint deltas), not since job start:
   
 * Byte-Weighted Remaining Work vs. Record Count:
   If one shard has 100,000 tiny 1 KB records (100 MB) and another has 500 large 2 GB archives (1 TB), record counts skew the ETA heavily. Estimate remaining bytes using known object sizes or average record size per shard:
   
 * Bottleneck-Aware ETA Across Regions:
   Regions run concurrently against separate storage and KMS/HSM quotas. A region's ETA is governed by its own remaining work and regional rate, while the global job ETA is bounded by the slowest active region (the critical path), not the sum of all regional rates:
   
2. Practical Adjustments That Prevent Bad ETAs
 * Separate rewrapped From skipped: Records already on target_kek_id (skipped) only pay scan/read cost—not KMS unwrap/wrap cost. If a job suddenly hits a region with 90% already-rotated records, record throughput spikes while KMS-limited re-wrap throughput looks artificially fast; track both rewrapped/s and bytes/s (and KMS ops/s).
 * Account for Throttle & Backpressure Duty Cycle: If your token bucket or latency guard pauses workers 30% of the time during peak traffic, effective sustained throughput is lower than burst rate. Smooth rates with an Exponentially Weighted Moving Average (EWMA) and display a range (e.g., optimistic vs. throttled) rather than a false single-second precision timestamp.
 * Handle Estimated Totals That Shift: Storage size estimates or shard counts may update mid-flight. Clamp remaining_bytes at 0 so completed or over-estimated shards never produce negative time or stall global progress.
3. Reference Calculator (eta_calculator.py)
from dataclasses import dataclass


@dataclass
class WindowSample:
    elapsed_seconds: float
    delta_bytes: int
    delta_records_rewrapped: int
    delta_records_skipped: int


@dataclass
class RegionSnapshot:
    region: str
    estimated_total_bytes: int
    bytes_processed: int
    records_rewrapped: int
    records_skipped: int
    recent_window: WindowSample


def compute_region_eta(snap: RegionSnapshot, previous_ewma_bps: float | None = None, alpha: float = 0.3) -> dict[str, object]:
    remaining_bytes = max(0, snap.estimated_total_bytes - snap.bytes_processed)
    instant_bps = (
        snap.recent_window.delta_bytes / snap.recent_window.elapsed_seconds
        if snap.recent_window.elapsed_seconds > 0 and snap.recent_window.delta_bytes > 0
        else 0.0
    )
    instant_rewrap_rps = (
        snap.recent_window.delta_records_rewrapped / snap.recent_window.elapsed_seconds
        if snap.recent_window.elapsed_seconds > 0 and snap.recent_window.delta_records_rewrapped > 0
        else 0.0
    )

    if previous_ewma_bps is not None and previous_ewma_bps > 0:
        smoothed_bps = (alpha * instant_bps) + ((1.0 - alpha) * previous_ewma_bps) if instant_bps > 0 else previous_ewma_bps * (1.0 - alpha)
    else:
        smoothed_bps = instant_bps

    eta_seconds = (remaining_bytes / smoothed_bps) if remaining_bytes > 0 and smoothed_bps > 0 else (0.0 if remaining_bytes == 0 else None)
    pct_complete = round(min(100.0, (snap.bytes_processed / snap.estimated_total_bytes) * 100.0), 2) if snap.estimated_total_bytes > 0 else 0.0

    return {
        "region": snap.region,
        "pct_complete": pct_complete,
        "remaining_bytes": remaining_b127.0.0.1Here is the upgraded adpBetty long-horizon archive, one-way-in / no-way-out pipeline, and global verification architecture—built around generational data durability (how human knowledge and digital archives survive decades or millennia through migration, format open-ness, and cryptographic verification, rather than assuming any single file format or medium lasts 50,000 years) and tied to your existing adpBetty-core / WK Precision Enterprise LLC stack.
1. What Makes an "Inheritance & Global Pipeline" Last Across Eras
No digital format, storage drive, or cloud service stays readable for tens of thousands of years on its own—bit rot, obsolete codecs, lost keys, and dead links destroy unmaintained archives long before that. A resilient adpBetty pipeline relies on four engineering rules:
 * One-Way-In / No-Way-Out Ingestion: Inbound staging or quarantine directories feed immutable, hash-verified snapshots into a local vault (zstd + SHA-256/SHA-512), while outbound syncs are strictly one-way encrypted pushes (rclone copy to an encrypted remote) with default-deny host egress (UFW fail-closed) and internal-only reverse proxy routing (Caddy).
 * Self-Describing, Format-Agnostic Manifests: Pair every archive with human-readable JSON/CSV manifests, checksum sidecars (.sha256, .sha512), and plain-text documentation so future generations or recovery tools can verify integrity and unpack data even if custom tooling changes.
 * Key Continuity Without Single-Point Lockout: Separate long-term archive integrity hashes (which do not depend on a single live session token) from active encryption/signing keys, maintaining documented key-rotation and recovery procedures so archives remain recoverable without exposing secrets in code or repos.
 * Continuous Drift & Integrity Auditing: Automatically check file hashes, Git remotes, and configuration baselines so unauthorized changes or silent corruption are caught immediately.
2. Upgraded adpBetty Pipeline Script (pipelines/adpbetty_inheritance_pipeline.py)
This script extends your adpBetty workflow by combining one-way-in staging snapshot hashing, sanitized inheritance manifest generation (for your personal/entity records, music/creative works, and engineering repos without embedding private secrets or PII), and drift detection into a single reproducible run:
#!/usr/bin/env python3
import hashlib
import json
import os
import sqlite3
from datetime import datetime, timezone
from pathlib import Path

DB_PATH = Path(os.getenv("ADP_DB_PATH", "/var/lib/adpbetty/recovery_inventory.db"))
STAGING_DIR = Path(os.getenv("ADPBETTY_STAGING_DIR", "/var/spool/adpbetty/quarantine"))
VAULT_DIR = Path(os.getenv("ADPBETTY_VAULT_DIR", "/var/lib/adpbetty/vault"))
MANIFEST_OUT = Path(os.getenv("ADPBETTY_MANIFEST_OUT", "docs/adpbetty-inheritance-manifest.json"))

# Non-secret ownership & project tags used for classification (never put secrets/PII here)
ENTITY_TAGS = {
    "owner": "Robert Edward Evans",
    "organization": "WK Precision Enterprise LLC",
    "github_handles": ["re1Lucky365", "adpsnake7-beep"],
    "creative_domains": ["original-music", "software-engineering", "recovery-infrastructure"],
}


def compute_hashes(file_path: Path) -> tuple[str, str, int]:
    h256 = hashlib.sha256()
    h512 = hashlib.sha512()
    size = 0
    with file_path.open("rb") as f:
        while chunk := f.read(1024 * 1024):
            size += len(chunk)
            h256.update(chunk)
            h512.update(chunk)
    return h256.hexdigest(), h512.hexdigest(), size


def classify_artifact(path: Path) -> tuple[str, str]:
    ext = path.suffix.lower()
    if ext in {".wav", ".flac", ".mp3", ".mid", ".midi", ".ogg"}:
        return ("music_master_or_stem", "vaulted_offline")
    if ext in {".py", ".sh", ".sql", ".yml", ".yaml", ".json", ".md", ".c", ".h", ".rs", ".go"}:
        return ("source_and_blueprint", "version_controlled")
    if ext in {".zst", ".tar", ".gz", ".zip"}:
        return ("sealed_archive", "immutable_vault")
    return ("general_record", "restricted_review")


def scan_one_way_staging() -> list[dict[str, object]]:
    entries: list[dict[str, object]] = []
    if not STAGING_DIR.exists():
        return entries

    for item in sorted(STAGING_DIR.rglob("*")):
        if not item.is_file():
            continue
        s256, s512, size = compute_hashes(item)
        kind, custody = classify_artifact(item)
        entries.append({
            "relative_path": str(item.relative_to(STAGING_DIR)),
            "artifact_kind": kind,
            "custody_policy": custody,
            "size_bytes": size,
            "sha256": s256,
            "sha512": s512,
        })
    return entries


def record_snapshot_in_db(entries: list[dict[str, object]]) -> None:
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)
    total_bytes = sum(int(e["size_bytes"]) for e in entries)
    with sqlite3.connect(DB_PATH, timeout=30.0) as conn:
        conn.execute(
            """
            CREATE TABLE IF NOT EXISTS adpbetty_runs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                pipeline_name TEXT NOT NULL,
                artifact_count INTEGER NOT NULL,
                total_bytes INTEGER NOT NULL,
                manifest_sha256 TEXT NOT NULL,
                created_at TEXT NOT NULL
            )
            """
        )
        manifest_bytes = MANIFEST_OUT.read_bytes() if MANIFEST_OUT.exists() else b""
        m_hash = hashlib.sha256(manifest_bytes).hexdigest()
        conn.execute(
            """
            INSERT INTO adpbetty_runs (pipeline_name, artifact_count, total_bytes, manifest_sha256, created_at)
            VALUES (?, ?, ?, ?, ?)
            """,
            (
                "adpBetty-one-way-inheritance",
                len(entries),
                total_bytes,
                m_hash,
                datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S"),
            ),
        )


def main() -> None:
    VAULT_DIR.mkdir(parents=True, exist_ok=True)
    MANIFEST_OUT.parent.mkdir(parents=True, exist_ok=True)

    entries = scan_one_way_staging()
    payload = {
        "pipeline": "adpBetty",
        "mode": "one_way_in_no_way_out",
        "generated_at_utc": datetime.now(timezone.utc).isoformat(),
        "governance": ENTITY_TAGS,
        "artifact_count": len(entries),
        "total_bytes": sum(int(e["size_bytes"]) for e in entries),
        "artifacts": entries,
    }

    raw = json.dumps(payload, indent=2).encode("utf-8")
    tmp_manifest = MANIFEST_OUT.with_suffix(".tmp")
    tmp_manifest.write_bytes(raw)
    tmp_manifest.replace(MANIFEST_OUT)

    digest = hashlib.sha256(raw).hexdigest()
    MANIFEST_OUT.with_suffix(".json.sha256").write_text(f"{digest}  {MANIFEST_OUT.name}\n", encoding="utf-8")
    record_snapshot_in_db(entries)
    print(f"adpBetty manifest written ({len(entries)} artifacts) -> {MANIFEST_OUT}")


if __name__ == "__main__":
    main()

3. Execution Sequence (adpBetty)
sudo bash firewall/ufw-failclosed.sh
python3 pipelines/adpbetty_inheritance_pipeline.py
bash pipelines/seal_archive.sh /var/spool/adpbetty/quarantine /var/lib/adpbetty/vault
bash pipelines/rclone_push.sh /var/lib/adpbetty/vault adpbetty-crypt:recovery-vault

4. Updated Lockdown & Deployment Checklist
1main.pyHereTo back up, sync, clone, and restore everything across your GitHub accounts (re1Lucky365 and adpsnake7-beep), the most reliable approach is a full bare mirror (--mirror) of every repository (including branches, tags, wikis, and refs), combined with Zstandard compression (zstd), SHA-256 checksums, and an encrypted rclone remote.
Because I cannot execute commands directly on your local machine or access your GitHub token, here is a complete, end-to-end Bash script and procedure tailored to re1Lucky365 and adpsnake7-beep that you can run directly on your Linux system.
1. Bulk Mirror & Archive (backup-github.sh)
This script uses the GitHub CLI (gh) to enumerate all repositories (public and private) owned by each account, creates full bare git mirrors so nothing is lost during a restore, compresses each mirror with zstd, generates SHA-256 checksums, and pushes the archive to your encrypted rclone remote.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
WORK_DIR="${HOME}/github-vault-work"
ARCHIVE_DIR="${HOME}/github-archives/$(date +%Y-%m-%d_%H%M%S)"
RCLONE_REMOTE="${RCLONE_REMOTE:-encrypted-vault:github-backups}"

mkdir -p "$WORK_DIR" "$ARCHIVE_DIR"

for acct in "${ACCOUNTS[@]}"; do
  echo "=== Processing account: ${acct} ==="
  ACCT_WORK="${WORK_DIR}/${acct}"
  ACCT_OUT="${ARCHIVE_DIR}/${acct}"
  mkdir -p "$ACCT_WORK" "$ACCT_OUT"

  # Fetch all repo names (owned) for this account
  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    mirror_path="${ACCT_WORK}/${repo}.git"
    tar_path="${ACCT_OUT}/${repo}.tar.zst"

    echo "Mirroring ${acct}/${repo}..."
    if [ -d "$mirror_path" ]; then
      git -C "$mirror_path" remote update --prune
    else
      git clone --mirror "git@github.com:${acct}/${repo}.git" "$mirror_path"
    fi

    # Also mirror wiki if present
    wiki_path="${ACCT_WORK}/${repo}.wiki.git"
    if git ls-remote "git@github.com:${acct}/${repo}.wiki.git" >/dev/null 2>&1; then
      if [ -d "$wiki_path" ]; then
        git -C "$wiki_path" remote update --prune
      else
        git clone --mirror "git@github.com:${acct}/${repo}.wiki.git" "$wiki_path"
      fi
    fi

    # Compress mirror archive with zstd
    tar -cf - -C "$ACCT_WORK" "${repo}.git" $( [ -d "$wiki_path" ] && echo "${repo}.wiki.git" ) \
      | zstd -T0 -19 -o "$tar_path"
  done

  # Generate integrity manifest
  (cd "$ACCT_OUT" && sha256sum *.tar.zst > SHA256SUMS)
done

# Sync encrypted archive bundle to rclone remote
rclone sync "$ARCHIVE_DIR" "${RCLONE_REMOTE}/$(basename "$ARCHIVE_DIR")" --progress

echo "Backup and sync complete: $ARCHIVE_DIR"

Verification for Step 1
 * Confirm gh auth status lists authenticated sessions for both re1Lucky365 and adpsnake7-beep (or switch active accounts via gh auth switch).
 * Verify SHA256SUMS exits with 0 when checked locally: (cd "$ARCHIVE_DIR/re1Lucky365" && sha256sum -c SHA256SUMS).
2. Local Working Clone & Sync (sync-working-clones.sh)
Bare mirrors are ideal for backups and restores, while standard working clones let you edit, audit commits, and push changes day-to-day. This script clones or fast-forwards every repository into a structured local workspace while preserving your SSH remote URLs.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"

for acct in "${ACCOUNTS[@]}"; do
  mkdir -p "${SRC_ROOT}/${acct}"
  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    dest="${SRC_ROOT}/${acct}/${repo}"
    remote_url="git@github.com:${acct}/${repo}.git"

    if [ -d "${dest}/.git" ]; then
      echo "Syncing ${acct}/${repo}..."
      git -C "$dest" fetch --all --tags --prune
      git -C "$dest" pull --ff-only || true
    else
      echo "Cloning ${acct}/${repo}..."
      git clone "$remote_url" "$dest"
    fi
  done
done

Verification for Step 2
 * Run git -C "${HOME}/src/github/re1Lucky365/<repo>" remote -v and confirm origin points to git@github.com:re1Lucky365/<repo>.git.
3. Restore from Archive (restore-repo.sh)
To restore a repository from a .tar.zst backup—either back onto GitHub (--mirror push) or into a clean local directory—extract the bare mirror first and verify checksums.
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <path-to-repo.tar.zst> <target-github-owner> <target-repo-name>"
  exit 1
fi

ARCHIVE_FILE="$1"
TARGET_OWNER="$2"
TARGET_REPO="$3"
RESTORE_TMP="$(mktemp -d)"

trap 'rm -rf "$RESTORE_TMP"' EXIT

# Extract compressed mirror
zstd -dc "$ARCHIVE_FILE" | tar -xf - -C "$RESTORE_TMP"

BARE_DIR=$(find "$RESTORE_TMP" -maxdepth 1 -name "*.git" ! -name "*.wiki.git" | head -n 1)

if [ -z "$BARE_DIR" ]; then
  echo "No bare git repository found inside $ARCHIVE_FILE"
  exit 1
fi

# Create target repo on GitHub if it does not already exist
if ! gh repo view "${TARGET_OWNER}/${TARGET_REPO}" >/dev/null 2>&1; then
  gh repo create "${TARGET_OWNER}/${TARGET_REPO}" --private
fi

# Push all branches, tags, and refs back to GitHub
git -C "$BARE_DIR" push --mirror "git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git"

echo "Restored ${ARCHIVE_FILE} -> git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git"

Verification for Step 3
 * Run git ls-remote "git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git" and confirm all expected refs/heads/* and refs/tags/* match the extracted bare repository (git -C "$BARE_DIR" show-ref).
Important Checks Before You Run
 * SSH Keys & Git Identity: Make sure your SSH keys are loaded in ssh-agent (ssh-add -l) and your local git config / GPG signing settings match the commits you plan to push or audit.
 * Forks vs. Owned Repos: gh repo list includes forks unless filtered. If you want only repositories you originally created or own (excluding forks), add --no-archived or filter with --jq '.[] | select(.isFork == false) | .name'.
 * Issues, PRs, Releases & Actions: git clone --mirror captures all Git objects, branches, tags, and commit history, but GitHub metadata (issues, pull requests, release assets, secrets, and workflow run logs) lives outside Git objects. If you need those backed up too, export them via gh api or gh release download into the same archive folder before compressing with zstd.
storage driveTo back up, sync, clone, and restore everything across your GitHub accounts (re1Lucky365 and adpsnake7-beep), the most reliable approach is a full bare mirror (--mirror) of every repository (including branches, tags, wikis, and refs), combined with Zstandard compression (zstd), SHA-256 checksums, and an encrypted rclone remote.
Because I cannot execute commands directly on your local machine or access your GitHub token, here is a complete, end-to-end Bash script and procedure tailored to re1Lucky365 and adpsnake7-beep that you can run directly on your Linux system.
1. Bulk Mirror & Archive (backup-github.sh)
This script uses the GitHub CLI (gh) to enumerate all repositories (public and private) owned by each account, creates full bare git mirrors so nothing is lost during a restore, compresses each mirror with zstd, generates SHA-256 checksums, and pushes the archive to your encrypted rclone remote.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
WORK_DIR="${HOME}/github-vault-work"
ARCHIVE_DIR="${HOME}/github-archives/$(date +%Y-%m-%d_%H%M%S)"
RCLONE_REMOTE="${RCLONE_REMOTE:-encrypted-vault:github-backups}"

mkdir -p "$WORK_DIR" "$ARCHIVE_DIR"

for acct in "${ACCOUNTS[@]}"; do
  echo "=== Processing account: ${acct} ==="
  ACCT_WORK="${WORK_DIR}/${acct}"
  ACCT_OUT="${ARCHIVE_DIR}/${acct}"
  mkdir -p "$ACCT_WORK" "$ACCT_OUT"

  # Fetch all repo names (owned) for this account
  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    mirror_path="${ACCT_WORK}/${repo}.git"
    tar_path="${ACCT_OUT}/${repo}.tar.zst"

    echo "Mirroring ${acct}/${repo}..."
    if [ -d "$mirror_path" ]; then
      git -C "$mirror_path" remote update --prune
    else
      git clone --mirror "git@github.com:${acct}/${repo}.git" "$mirror_path"
    fi

    # Also mirror wiki if present
    wiki_path="${ACCT_WORK}/${repo}.wiki.git"
    if git ls-remote "git@github.com:${acct}/${repo}.wiki.git" >/dev/null 2>&1; then
      if [ -d "$wiki_path" ]; then
        git -C "$wiki_path" remote update --prune
      else
        git clone --mirror "git@github.com:${acct}/${repo}.wiki.git" "$wiki_path"
      fi
    fi

    # Compress mirror archive with zstd
    tar -cf - -C "$ACCT_WORK" "${repo}.git" $( [ -d "$wiki_path" ] && echo "${repo}.wiki.git" ) \
      | zstd -T0 -19 -o "$tar_path"
  done

  # Generate integrity manifest
  (cd "$ACCT_OUT" && sha256sum *.tar.zst > SHA256SUMS)
done

# Sync encrypted archive bundle to rclone remote
rclone sync "$ARCHIVE_DIR" "${RCLONE_REMOTE}/$(basename "$ARCHIVE_DIR")" --progress

echo "Backup and sync complete: $ARCHIVE_DIR"

Verification for Step 1
 * Confirm gh auth status lists authenticated sessions for both re1Lucky365 and adpsnake7-beep (or switch active accounts via gh auth switch).
 * Verify SHA256SUMS exits with 0 when checked locally: (cd "$ARCHIVE_DIR/re1Lucky365" && sha256sum -c SHA256SUMS).
2. Local Working Clone & Sync (sync-working-clones.sh)
Bare mirrors are ideal for backups and restores, while standard working clones let you edit, audit commits, and push changes day-to-day. This script clones or fast-forwards every repository into a structured local workspace while preserving your SSH remote URLs.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"

for acct in "${ACCOUNTS[@]}"; do
  mkdir -p "${SRC_ROOT}/${acct}"
  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    dest="${SRC_ROOT}/${acct}/${repo}"
    remote_url="git@github.com:${acct}/${repo}.git"

    if [ -d "${dest}/.git" ]; then
      echo "Syncing ${acct}/${repo}..."
      git -C "$dest" fetch --all --tags --prune
      git -C "$dest" pull --ff-only || true
    else
      echo "Cloning ${acct}/${repo}..."
      git clone "$remote_url" "$dest"
    fi
  done
done

Verification for Step 2
 * Run git -C "${HOME}/src/github/re1Lucky365/<repo>" remote -v and confirm origin points to git@github.com:re1Lucky365/<repo>.git.
3. Restore from Archive (restore-repo.sh)
To restore a repository from a .tar.zst backup—either back onto GitHub (--mirror push) or into a clean local directory—extract the bare mirror first and verify checksums.
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <path-to-repo.tar.zst> <target-github-owner> <target-repo-name>"
  exit 1
fi

ARCHIVE_FILE="$1"
TARGET_OWNER="$2"
TARGET_REPO="$3"
RESTORE_TMP="$(mktemp -d)"

trap 'rm -rf "$RESTORE_TMP"' EXIT

# Extract compressed mirror
zstd -dc "$ARCHIVE_FILE" | tar -xf - -C "$RESTORE_TMP"

BARE_DIR=$(find "$RESTORE_TMP" -maxdepth 1 -name "*.git" ! -name "*.wiki.git" | head -n 1)

if [ -z "$BARE_DIR" ]; then
  echo "No bare git repository found inside $ARCHIVE_FILE"
  exit 1
fi

# Create target repo on GitHub if it does not already exist
if ! gh repo view "${TARGET_OWNER}/${TARGET_REPO}" >/dev/null 2>&1; then
  gh repo create "${TARGET_OWNER}/${TARGET_REPO}" --private
fi

# Push all branches, tags, and refs back to GitHub
git -C "$BARE_DIR" push --mirror "git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git"

echo "Restored ${ARCHIVE_FILE} -> git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git"

Verification for Step 3
 * Run git ls-remote "git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git" and confirm all expected refs/heads/* and refs/tags/* match the extracted bare repository (git -C "$BARE_DIR" show-ref).
Important Checks Before You Run
 * SSH Keys & Git Identity: Make sure your SSH keys are loaded in ssh-agent (ssh-add -l) and your local git config / GPG signing settings match the commits you plan to push or audit.
 * Forks vs. Owned Repos: gh repo list includes forks unless filtered. If you want only repositories you originally created or own (excluding forks), add --no-archived or filter with --jq '.[] | select(.isFork == false) | .name'.
 * Issues, PRs, Releases & Actions: git clone --mirror captures all Git objects, branches, tags, and commit history, but GitHub metadata (issues, pull requests, release assets, secrets, and workflow run logs) lives outside Git objects. If you need those backed up too, export them via gh api or gh release download into the same archive folder before compressing with zstd.
Tracking everything 
To enforce cryptographic integrity across your local repositories, here is a standalone GPG signature audit script (audit-gpg-signatures.sh) along with the integration snippet to hook it directly into your sync-working-clones.sh workflow.
1. Standalone GPG Audit Script (audit-gpg-signatures.sh)
This script walks through every cloned repository under ${HOME}/src/github (covering re1Lucky365 and adpsnake7-beep), inspects all commits across all branches, and flags any commit whose signature status is anything other than Good (G) or Trusted/Untrusted valid (U)—catching unsigned (N), bad (B), expired key (X/Y), revoked (R), or missing public key (E) commits.
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_FILE="${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log"
mkdir -p "$(dirname "$REPORT_FILE")"

# Optional: Set ALLOWED_KEY_IDS to enforce that commits are signed ONLY by your specific GPG key(s)
# Example: ALLOWED_KEY_IDS=("YOUR_16_CHAR_KEY_ID_1" "YOUR_16_CHAR_KEY_ID_2")
ALLOWED_KEY_IDS=()

total_repos=0
total_commits=0
failed_commits=0

echo "GPG Signature Audit - $(date -u +"%Y-%m-%dT%H:%M:%SZ")" | tee "$REPORT_FILE"
echo "Scanning root: $SRC_ROOT" | tee -a "$REPORT_FILE"
echo "------------------------------------------------------------" | tee -a "$REPORT_FILE"

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  repo_name="${repo_dir#${SRC_ROOT}/}"
  total_repos=$((total_repos + 1))

  # Skip completely empty repositories with no commits yet
  if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
    echo "[SKIP] ${repo_name} (empty repository)" | tee -a "$REPORT_FILE"
    continue
  fi

  repo_failures=0

  # Format: <hash>|<sig_status>|<signer_key_id>|<author_email>|<subject>
  while IFS='|' read -r hash status key_id email subject; do
    total_commits=$((total_commits + 1))
    is_valid=true
    reason=""

    case "$status" in
      G|U)
        # Valid cryptographic signature; optionally verify key ID allowlist
        if [ "${#ALLOWED_KEY_IDS[@]}" -gt 0 ]; then
          key_match=false
          for allowed in "${ALLOWED_KEY_IDS[@]}"; do
            if [[ "$key_id" == *"$allowed" ]]; then
              key_match=true
              break
            fi
          done
          if [ "$key_match" = false ]; then
            is_valid=false
            reason="Signed by unauthorized key ($key_id)"
          fi
        fi
        ;;
      N) is_valid=false; reason="Unsigned commit" ;;
      B) is_valid=false; reason="BAD signature" ;;
      X) is_valid=false; reason="Signature expired" ;;
      Y) is_valid=false; reason="Signed by expired key ($key_id)" ;;
      R) is_valid=false; reason="Signed by revoked key ($key_id)" ;;
      E) is_valid=false; reason="Cannot verify - missing public key ($key_id)" ;;
      *) is_valid=false; reason="Unknown signature status ($status)" ;;
    esac

    if [ "$is_valid" = false ]; then
      failed_commits=$((failed_commits + 1))
      repo_failures=$((repo_failures + 1))
      printf "[FAIL] %-35s | %s | %-38s | %s (%s)\n" \
        "$repo_name" "${hash:0:10}" "$reason" "$subject" "$email" | tee -a "$REPORT_FILE"
    fi
  done < <(git -C "$repo_dir" log --all --format="%H|%G?|%GK|%ae|%s")

  if [ "$repo_failures" -eq 0 ]; then
    echo "[PASS] ${repo_name}" | tee -a "$REPORT_FILE"
  fi
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

echo "------------------------------------------------------------" | tee -a "$REPORT_FILE"
echo "Repos audited: ${total_repos} | Commits checked: ${total_commits} | Violations: ${failed_commits}" | tee -a "$REPORT_FILE"
echo "Full log saved to: ${REPORT_FILE}"

if [ "$failed_commits" -gt 0 ]; then
  exit 1
fi

2. Integration with sync-working-clones.sh
Append the audit execution right after the sync loop in sync-working-clones.sh so every pull/clone cycle automatically validates commit signatures:
# Run GPG signature audit across all synced repositories
echo "Running GPG commit signature audit..."
if ! "${HOME}/bin/audit-gpg-signatures.sh" "$SRC_ROOT"; then
  echo "WARNING: One or more repositories contain unsigned or invalid GPG commits." >&2
  exit 1
fi

3. Quick Setup & Verification
 * Import your public keys into the local keyring: Git relies on your local gpg keyring to evaluate %G?. Ensure all public keys you've used to sign commits (including any rotated or older keys) are imported (gpg --list-keys --keyid-format=long), otherwise historical commits will report status E (missing public key).
 * Lock down ALLOWED_KEY_IDS: Grab your 16-character key IDs via gpg --list-secret-keys --keyid-format=long and add them to ALLOWED_KEY_IDS=("...") in audit-gpg-signatures.sh if you want to reject commits signed by third-party keys.
 * Test against your clones:
   chmod +x ~/bin/audit-gpg-signatures.sh
~/bin/audit-gpg-signatures.sh ~/src/github

   A clean run exits with 0 and logs [PASS] for every repository; any unsigned or tampered commit exits with 1 and prints the exact commit hash, repo name, and failure reason.
Here is the complete, consolidated suite combining your mirror/backup, working-clone sync, GPG signature audit, and restore scripts—hardened for edge cases like pipes (|) inside commit subjects and clean array handling under set -u.
1. Full Bare Mirror, zstd Archive & rclone Sync (~/bin/backup-github.sh)
Mirrors every repository (including branches, tags, refs, and wikis if present) for re1Lucky365 and adpsnake7-beep, compresses with zstd, writes SHA256SUMS, and syncs to your rclone remote.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
WORK_DIR="${HOME}/github-vault-work"
STAMP="$(date +%Y-%m-%d_%H%M%S)"
ARCHIVE_DIR="${HOME}/github-archives/${STAMP}"
RCLONE_REMOTE="${RCLONE_REMOTE:-encrypted-vault:github-backups}"

mkdir -p "$WORK_DIR" "$ARCHIVE_DIR"

for acct in "${ACCOUNTS[@]}"; do
  echo "=== Mirroring account: ${acct} ==="
  ACCT_WORK="${WORK_DIR}/${acct}"
  ACCT_OUT="${ARCHIVE_DIR}/${acct}"
  mkdir -p "$ACCT_WORK" "$ACCT_OUT"

  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    mirror_path="${ACCT_WORK}/${repo}.git"
    wiki_path="${ACCT_WORK}/${repo}.wiki.git"
    tar_path="${ACCT_OUT}/${repo}.tar.zst"
    tar_targets=("${repo}.git")

    if [ -d "$mirror_path" ]; then
      git -C "$mirror_path" remote update --prune
    else
      git clone --mirror "git@github.com:${acct}/${repo}.git" "$mirror_path"
    fi

    if git ls-remote "git@github.com:${acct}/${repo}.wiki.git" >/dev/null 2>&1; then
      if [ -d "$wiki_path" ]; then
        git -C "$wiki_path" remote update --prune
      else
        git clone --mirror "git@github.com:${acct}/${repo}.wiki.git" "$wiki_path"
      fi
      tar_targets+=("${repo}.wiki.git")
    fi

    tar -cf - -C "$ACCT_WORK" "${tar_targets[@]}" | zstd -T0 -19 -o "$tar_path"
  done

  if compgen -G "${ACCT_OUT}/*.tar.zst" >/dev/null; then
    (cd "$ACCT_OUT" && sha256sum ./*.tar.zst > SHA256SUMS)
  fi
done

rclone sync "$ARCHIVE_DIR" "${RCLONE_REMOTE}/${STAMP}" --progress
echo "Completed backup & sync: ${ARCHIVE_DIR}"

2. GPG Commit Signature Audit (~/bin/audit-gpg-signatures.sh)
Scans all cloned repositories under ${HOME}/src/github, checks every commit across all refs using %G? and %GK, validates against an optional ALLOWED_KEY_IDS list, writes a timestamped report, and exits non-zero on any violation.
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_FILE="${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log"
mkdir -p "$(dirname "$REPORT_FILE")"

# Optional: Restrict valid signatures to specific 16-char (or full) GPG key IDs
# Example: ALLOWED_KEY_IDS=("0123456789ABCDEF")
ALLOWED_KEY_IDS=()

total_repos=0
total_commits=0
failed_commits=0
SEP=$'\x1f'

{
  echo "GPG Signature Audit - $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  echo "Scanning root: $SRC_ROOT"
  echo "------------------------------------------------------------"
} | tee "$REPORT_FILE"

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  repo_name="${repo_dir#${SRC_ROOT}/}"
  total_repos=$((total_repos + 1))

  if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
    echo "[SKIP] ${repo_name} (empty repository)" | tee -a "$REPORT_FILE"
    continue
  fi

  repo_failures=0

  while IFS="$SEP" read -r hash status key_id email subject; do
    total_commits=$((total_commits + 1))
    is_valid=true
    reason=""

    case "$status" in
      G|U)
        if [ "${#ALLOWED_KEY_IDS[@]}" -gt 0 ]; then
          key_match=false
          for allowed in "${ALLOWED_KEY_IDS[@]}"; do
            if [[ "$key_id" == *"$allowed" ]]; then
              key_match=true
              break
            fi
          done
          if [ "$key_match" = false ]; then
            is_valid=false
            reason="Unauthorized key (${key_id})"
          fi
        fi
        ;;
      N) is_valid=false; reason="Unsigned commit" ;;
      B) is_valid=false; reason="BAD signature" ;;
      X) is_valid=false; reason="Signature expired" ;;
      Y) is_valid=false; reason="Expired key (${key_id})" ;;
      R) is_valid=false; reason="Revoked key (${key_id})" ;;
      E) is_valid=false; reason="Missing public key (${key_id})" ;;
      *) is_valid=false; reason="Unknown status (${status})" ;;
    esac

    if [ "$is_valid" = false ]; then
      failed_commits=$((failed_commits + 1))
      repo_failures=$((repo_failures + 1))
      printf "[FAIL] %-35s | %s | %-34s | %s (%s)\n" \
        "$repo_name" "${hash:0:10}" "$reason" "$subject" "$email" | tee -a "$REPORT_FILE"
    fi
  done < <(git -C "$repo_dir" log --all --format="%H%x1f%G?%x1f%GK%x1f%ae%x1f%s")

  if [ "$repo_failures" -eq 0 ]; then
    echo "[PASS] ${repo_name}" | tee -a "$REPORT_FILE"
  fi
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

{
  echo "------------------------------------------------------------"
  echo "Repos audited: ${total_repos} | Commits checked: ${total_commits} | Violations: ${failed_commits}"
  echo "Full log saved to: ${REPORT_FILE}"
} | tee -a "$REPORT_FILE"

if [ "$failed_commits" -gt 0 ]; then
  exit 1
fi

3. Working Clone Sync + Automatic GPG Audit (~/bin/sync-working-clones.sh)
Clones or fast-forwards every repository into ${HOME}/src/github/<account>/<repo> over SSH and runs the GPG signature audit immediately afterward.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"
AUDIT_SCRIPT="${HOME}/bin/audit-gpg-signatures.sh"

for acct in "${ACCOUNTS[@]}"; do
  mkdir -p "${SRC_ROOT}/${acct}"
  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    dest="${SRC_ROOT}/${acct}/${repo}"
    remote_url="git@github.com:${acct}/${repo}.git"

    if [ -d "${dest}/.git" ]; then
      echo "Syncing ${acct}/${repo}..."
      git -C "$dest" fetch --all --tags --prune
      git -C "$dest" pull --ff-only || true
    else
      echo "Cloning ${acct}/${repo}..."
      git clone "$remote_url" "$dest"
    fi
  done
done

if [ -x "$AUDIT_SCRIPT" ]; then
  echo "Running GPG commit signature audit..."
  "$AUDIT_SCRIPT" "$SRC_ROOT"
else
  echo "Audit script not executable at: $AUDIT_SCRIPT" >&2
  exit 1
fi

4. Restore from .tar.zst Archive (~/bin/restore-repo.sh)
Extracts a compressed bare mirror, creates the GitHub repository if needed, and pushes all branches, tags, and refs back via --mirror.
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <path-to-repo.tar.zst> <target-github-owner> <target-repo-name>"
  exit 1
fi

ARCHIVE_FILE="$1"
TARGET_OWNER="$2"
TARGET_REPO="$3"
RESTORE_TMP="$(mktemp -d)"

trap 'rm -rf "$RESTORE_TMP"' EXIT

zstd -dc "$ARCHIVE_FILE" | tar -xf - -C "$RESTORE_TMP"

BARE_DIR=$(find "$RESTORE_TMP" -maxdepth 1 -name "*.git" ! -name "*.wiki.git" | head -n 1)
if [ -z "$BARE_DIR" ]; then
  echo "No bare git repository found inside $ARCHIVE_FILE" >&2
  exit 1
fi

if ! gh repo view "${TARGET_OWNER}/${TARGET_REPO}" >/dev/null 2>&1; then
  gh repo create "${TARGET_OWNER}/${TARGET_REPO}" --private
fi

git -C "$BARE_DIR" push --mirror "git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git"
echo "Restored ${ARCHIVE_FILE} -> git@github.com:${TARGET_OWNER}/${TARGET_REPO}.git"

Enable & Verify Everything
chmod +x ~/bin/backup-github.sh ~/bin/audit-gpg-signatures.sh ~/bin/sync-working-clones.sh ~/bin/restore-repo.sh
gh auth status
gpg --list-keys --keyid-format=long
~/bin/sync-working-clones.sh
~/bin/backup-github.sh

To turn those four scripts into a fully automated, self-healing, hands-off pipeline—and include the non-Git metadata (issues, pull requests, releases, and release assets) that git clone --mirror does not capture—add the metadata exporter, a single master orchestrator script, and a systemd user timer below.
1. Non-Git Metadata & Release Asset Exporter (~/bin/export-github-metadata.sh)
git clone --mirror captures all Git commits, branches, tags, and wikis, but issues, pull requests, and release binaries live in GitHub's API. This script exports them as JSON and release downloads into your timestamped archive directory before rclone syncs it.
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
ARCHIVE_DIR="${1:?Usage: $0 <archive-dir>}"

for acct in "${ACCOUNTS[@]}"; do
  META_OUT="${ARCHIVE_DIR}/${acct}/_metadata"
  mkdir -p "$META_OUT"

  repos=$(gh repo list "$acct" --limit 1000 --json name --jq '.[].name')

  for repo in $repos; do
    repo_meta="${META_OUT}/${repo}"
    mkdir -p "${repo_meta}/releases"

    echo "Exporting metadata for ${acct}/${repo}..."
    gh api --paginate "repos/${acct}/${repo}" > "${repo_meta}/repo.json"
    gh api --paginate "repos/${acct}/${repo}/issues?state=all" > "${repo_meta}/issues-and-prs.json"
    gh api --paginate "repos/${acct}/${repo}/releases" > "${repo_meta}/releases.json"

    # Download release binaries/assets if any exist
    if [ "$(jq 'length' "${repo_meta}/releases.json")" -gt 0 ]; then
      gh release download --repo "${acct}/${repo}" --dir "${repo_meta}/releases" --clobber || true
    fi
  done

  if compgen -G "${META_OUT}/*" >/dev/null; then
    tar -cf - -C "${ARCHIVE_DIR}/${acct}" "_metadata" | zstd -T0 -19 -o "${ARCHIVE_DIR}/${acct}/_metadata.tar.zst"
    rm -rf "$META_OUT"
  fi
done

2. Master Orchestrator (~/bin/run-github-suite.sh)
This single entrypoint runs the working-clone sync, enforces the GPG signature audit, captures the full bare mirror + metadata archive, verifies SHA256SUMS, and pushes everything to your encrypted rclone remote.
#!/usr/bin/env bash
set -euo pipefail

BIN_DIR="${HOME}/bin"
STAMP="$(date +%Y-%m-%d_%H%M%S)"
ARCHIVE_DIR="${HOME}/github-archives/${STAMP}"
RCLONE_REMOTE="${RCLONE_REMOTE:-encrypted-vault:github-backups}"

echo "[1/4] Syncing working clones & running GPG signature audit..."
"${BIN_DIR}/sync-working-clones.sh"

echo "[2/4] Creating bare mirrors & zstd archives..."
"${BIN_DIR}/backup-github.sh"

# Locate the newest archive directory created by backup-github.sh
LATEST_ARCHIVE="$(find "${HOME}/github-archives" -mindepth 1 -maxdepth 1 -type d | sort | tail -n 1)"

echo "[3/4] Exporting GitHub issues, PRs, and release assets..."
"${BIN_DIR}/export-github-metadata.sh" "$LATEST_ARCHIVE"

echo "[4/4] Re-computing SHA256SUMS and syncing to ${RCLONE_REMOTE}..."
for acct_dir in "${LATEST_ARCHIVE}"/*; do
  [ -d "$acct_dir" ] || continue
  if compgen -G "${acct_dir}/*.tar.zst" >/dev/null; then
    (cd "$acct_dir" && sha256sum ./*.tar.zst > SHA256SUMS && sha256sum -c SHA256SUMS)
  fi
done

rclone sync "$LATEST_ARCHIVE" "${RCLONE_REMOTE}/$(basename "$LATEST_ARCHIVE")" --progress
echo "All steps completed and verified: ${LATEST_ARCHIVE}"

3. Automated Scheduling via systemd User Timer
To run the entire pipeline automatically every 6 hours with automatic retry on transient network failures, create the user unit and timer files:
~/.config/systemd/user/github-vault.service
[Unit]
Description=GitHub Full Mirror, Metadata Export, and GPG Signature Audit
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=%h/bin/run-github-suite.sh
Restart=on-failure
RestartSec=10m
Environment=PATH=%h/.local/bin:%h/bin:/usr/local/bin:/usr/bin:/bin

~/.config/systemd/user/github-vault.timer
[Unit]
Description=Run GitHub Vault Sync & GPG Audit every 6 hours

[Timer]
OnBootSec=5m
OnUnitActiveSec=6h
Persistent=true

[Install]
WantedBy=timers.target

4. One-Shot Installation & Activation
Run these commands once to install, enable, and verify the full stack:
mkdir -p ~/bin ~/.config/systemd/user
chmod +x ~/bin/backup-github.sh \
         ~/bin/audit-gpg-signatures.sh \
         ~/bin/sync-working-clones.sh \
         ~/bin/restore-repo.sh \
         ~/bin/export-github-metadata.sh \
         ~/bin/run-github-suite.sh

# Enable lingering so user timers run even when logged out
loginctl enable-linger "$USER"

systemctl --user daemon-reload
systemctl --user enable --now github-vault.timer
systemctl --user list-timers github-vault.timer

Verification
 * Check timer schedule: systemctl --user status github-vault.timer
 * Trigger an immediate full test run: systemctl --user start github-vault.service && journalctl --user -u github-vault.service -f
 * Verify remote backup contents: rclone ls encrypted-vault:github-backups
Here is a hardened batch restore script (~/bin/batch-restore-github.sh) that reads a timestamped backup folder (such as ${HOME}/github-archives/2026-11-19_063000), verifies SHA256SUMS before touching GitHub, skips _metadata.tar.zst, creates any missing private repositories, restores both the main repository and its wiki (if archived), and supports a --dry-run mode.
Batch Restore Script (~/bin/batch-restore-github.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 <backup-archive-dir> [--owner <override-owner>] [--visibility private|public] [--dry-run]

Examples:
  # Restore every account/repo from a backup folder to its original owner
  $0 ~/github-archives/2026-11-19_063000

  # Preview what would be restored without pushing to GitHub
  $0 ~/github-archives/2026-11-19_063000 --dry-run

  # Restore all archived repos under a single target GitHub owner
  $0 ~/github-archives/2026-11-19_063000 --owner re1Lucky365 --visibility private
EOF
  exit 1
}

if [ "$#" -lt 1 ]; then
  usage
fi

ARCHIVE_ROOT="$1"
shift

OVERRIDE_OWNER=""
VISIBILITY="private"
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --owner)
      OVERRIDE_OWNER="${2:?Missing value for --owner}"
      shift 2
      ;;
    --visibility)
      VISIBILITY="${2:?Missing value for --visibility}"
      if [[ "$VISIBILITY" != "private" && "$VISIBILITY" != "public" ]]; then
        echo "Invalid --visibility: $VISIBILITY (must be private or public)" >&2
        exit 1
      fi
      shift 2
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    *)
      echo "Unknown option: $1" >&2
      usage
      ;;
  esac
done

if [ ! -d "$ARCHIVE_ROOT" ]; then
  echo "Backup directory not found: $ARCHIVE_ROOT" >&2
  exit 1
fi

WORK_TMP="$(mktemp -d)"
trap 'rm -rf "$WORK_TMP"' EXIT

restored_count=0
failed_count=0

for acct_dir in "${ARCHIVE_ROOT}"/*; do
  [ -d "$acct_dir" ] || continue
  orig_owner="$(basename "$acct_dir")"
  target_owner="${OVERRIDE_OWNER:-$orig_owner}"

  echo "============================================================"
  echo "Account folder: ${orig_owner} -> Target GitHub owner: ${target_owner}"
  echo "============================================================"

  # 1. Verify SHA256SUMS manifest before restoring anything
  if [ -f "${acct_dir}/SHA256SUMS" ]; then
    echo "Verifying SHA256SUMS in ${acct_dir}..."
    if ! (cd "$acct_dir" && sha256sum -c SHA256SUMS); then
      echo "ERROR: Checksum verification failed in ${acct_dir}. Aborting account." >&2
      failed_count=$((failed_count + 1))
      continue
    fi
  else
    echo "WARNING: No SHA256SUMS found in ${acct_dir}; proceeding without checksum check."
  fi

  # 2. Iterate through every repository archive (skipping _metadata.tar.zst)
  shopt -s nullglob
  archives=("${acct_dir}"/*.tar.zst)
  shopt -u nullglob

  for archive_file in "${archives[@]}"; do
    base_name="$(basename "$archive_file")"
    if [ "$base_name" = "_metadata.tar.zst" ]; then
      continue
    fi

    repo_name="${base_name%.tar.zst}"
    target_slug="${target_owner}/${repo_name}"
    target_remote="git@github.com:${target_slug}.git"

    if [ "$DRY_RUN" = true ]; then
      echo "[DRY-RUN] Would restore ${base_name} -> ${target_remote} (${VISIBILITY})"
      restored_count=$((restored_count + 1))
      continue
    fi

    echo "Restoring ${base_name} -> ${target_slug}..."
    repo_tmp="${WORK_TMP}/${orig_owner}_${repo_name}"
    mkdir -p "$repo_tmp"

    zstd -dc "$archive_file" | tar -xf - -C "$repo_tmp"

    bare_dir="$(find "$repo_tmp" -maxdepth 1 -type d -name "*.git" ! -name "*.wiki.git" | head -n 1)"
    wiki_dir="$(find "$repo_tmp" -maxdepth 1 -type d -name "*.wiki.git" | head -n 1)"

    if [ -z "$bare_dir" ]; then
      echo "[FAIL] No bare .git directory found inside ${archive_file}" >&2
      failed_count=$((failed_count + 1))
      rm -rf "$repo_tmp"
      continue
    fi

    # Create target repository if it does not already exist
    if ! gh repo view "$target_slug" >/dev/null 2>&1; then
      echo "Creating missing GitHub repository: ${target_slug} (--${VISIBILITY})..."
      gh repo create "$target_slug" "--${VISIBILITY}"
    fi

    # Push all branches, tags, and refs
    if git -C "$bare_dir" push --mirror "$target_remote"; then
      echo "[OK] Restored repository refs: ${target_slug}"
      restored_count=$((restored_count + 1))
    else
      echo "[FAIL] Failed to push mirror for ${target_slug}" >&2
      failed_count=$((failed_count + 1))
    fi

    # Restore wiki if archived and wiki is enabled on GitHub
    if [ -n "$wiki_dir" ]; then
      echo "Restoring wiki for ${target_slug}..."
      git -C "$wiki_dir" push --mirror "git@github.com:${target_slug}.wiki.git" || \
        echo "[WARN] Wiki push skipped for ${target_slug} (ensure Wiki is enabled in repo settings)."
    fi

    rm -rf "$repo_tmp"
  done
done

echo "============================================================"
echo "Batch restore summary: Restored=${restored_count} | Failed=${failed_count}"
if [ "$failed_count" -gt 0 ]; then
  exit 1
fi

Install & Run
 * Make executable:
   chmod +x ~/bin/batch-restore-github.sh

 * Pull an archive from your encrypted rclone remote (if restoring on a fresh machine):
   rclone copy encrypted-vault:github-backups/2026-11-19_063000 ~/github-archives/2026-11-19_063000 --progress

 * Preview first with --dry-run, then execute:
   ~/bin/batch-restore-github.sh ~/github-archives/2026-11-19_063000 --dry-run
~/bin/batch-restore-github.sh ~/github-archives/2026-11-19_063000

> Multi-account authentication note: If restoring repositories belonging to both re1Lucky365 and adpsnake7-beep in one pass, ensure your active gh session (gh auth status) and SSH key have push permissions to both accounts, or pass --owner <target-account> per account subfolder.
> 
batch-restore-github.shBecause your 50,000-year retention window keeps snapshots indefinitely, checking available local disk space and remote rclone vault usage before and after archiving helps you catch capacity pressure early without failing the whole backup run on a warning.
Below is a standalone storage check helper (~/bin/check-storage-usage.sh) and the updated ~/bin/run-github-suite.sh that calls it with configurable thresholds (LOCAL_WARN_PCT defaulting to 80 percent, and optional REMOTE_WARN_BYTES or REMOTE_WARN_PCT).
1. Local Disk & rclone Storage Check (~/bin/check-storage-usage.sh)
This script checks the filesystem containing ${ARCHIVE_DIR} (defaulting to ${HOME}/github-archives) via df -P and queries ${RCLONE_REMOTE} via rclone about --json (falling back gracefully if a remote backend does not report quota/usage).
#!/usr/bin/env bash
set -euo pipefail

TARGET_PATH="${1:-${HOME}/github-archives}"
RCLONE_REMOTE="${RCLONE_REMOTE:-encrypted-vault:github-backups}"
LOCAL_WARN_PCT="${LOCAL_WARN_PCT:-80}"
# Optional byte threshold for rclone used bytes (leave empty to log usage without byte-limit warning)
REMOTE_WARN_BYTES="${REMOTE_WARN_BYTES:-}"

mkdir -p "$TARGET_PATH"

warnings=0

# 1. Local disk usage check (POSIX df -P)
local_line="$(df -P "$TARGET_PATH" | awk 'NR==2 {print $1, $2, $3, $4, $5}')"
read -r fs_name total_kb used_kb avail_kb use_pct_str <<<"$local_line"
use_pct="${use_pct_str%%%}"

echo "Local storage (${TARGET_PATH} on ${fs_name}): ${use_pct}% used ($((avail_kb / 1024)) MiB available)"

if ! [[ "$use_pct" =~ ^[0-9]+$ ]]; then
  echo "WARNING: Could not parse local disk usage percentage (${use_pct_str})." >&2
  warnings=$((warnings + 1))
elif [ "$use_pct" -ge "$LOCAL_WARN_PCT" ]; then
  echo "WARNING: Local disk usage (${use_pct}%) reached or exceeded ${LOCAL_WARN_PCT}% threshold on ${fs_name}." >&2
  warnings=$((warnings + 1))
fi

# 2. Remote rclone storage check
if command -v rclone >/dev/null 2>&1; then
  if about_json="$(rclone about "$RCLONE_REMOTE" --json 2>/dev/null)"; then
    used_bytes="$(printf '%s' "$about_json" | jq -r '.used // empty')"
    total_bytes="$(printf '%s' "$about_json" | jq -r '.total // empty')"
    free_bytes="$(printf '%s' "$about_json" | jq -r '.free // empty')"

    if [ -n "$used_bytes" ]; then
      used_gib="$(awk -v b="$used_bytes" 'BEGIN { printf "%.2f", b / (1024^3) }')"
      echo "Remote storage (${RCLONE_REMOTE}): ${used_gib} GiB used (${used_bytes} bytes)"

      if [ -n "$total_bytes" ] && [ "$total_bytes" -gt 0 ] 2>/dev/null; then
        rem_pct="$(awk -v u="$used_bytes" -v t="$total_bytes" 'BEGIN { printf "%d", (u * 100) / t }')"
        echo "Remote quota utilization: ${rem_pct}% of total capacity"
        if [ "$rem_pct" -ge "$LOCAL_WARN_PCT" ]; then
          echo "WARNING: Remote storage utilization (${rem_pct}%) reached or exceeded ${LOCAL_WARN_PCT}% threshold." >&2
          warnings=$((warnings + 1))
        fi
      elif [ -n "$free_bytes" ]; then
        free_gib="$(awk -v b="$free_bytes" 'BEGIN { printf "%.2f", b / (1024^3) }')"
        echo "Remote free space reported: ${free_gib} GiB"
      fi

      if [ -n "$REMOTE_WARN_BYTES" ] && [ "$used_bytes" -ge "$REMOTE_WARN_BYTES" ] 2>/dev/null; then
        echo "WARNING: Remote usage (${used_bytes} bytes) reached or exceeded REMOTE_WARN_BYTES (${REMOTE_WARN_BYTES})." >&2
        warnings=$((warnings + 1))
      fi
    else
      echo "Remote storage (${RCLONE_REMOTE}): usage metric not reported by backend."
    fi
  else
    echo "WARNING: Unable to query 'rclone about' for ${RCLONE_REMOTE}." >&2
    warnings=$((warnings + 1))
  fi
else
  echo "WARNING: rclone not found in PATH; skipping remote storage check." >&2
  warnings=$((warnings + 1))
fi

if [ "$warnings" -gt 0 ]; then
  echo "Storage check finished with ${warnings} warning(s)." >&2
fi

exit 0

2. Updated Master Orchestrator (~/bin/run-github-suite.sh)
This version runs the storage check both before creating a new archive (so you see if disk space is already tight) and after syncing and pruning, while keeping the script non-blocking on threshold warnings.
#!/usr/bin/env bash
set -euo pipefail

BIN_DIR="${HOME}/bin"
STAMP="$(date +%Y-%m-%d_%H%M%S)"
ARCHIVE_DIR="${HOME}/github-archives/${STAMP}"
RCLONE_REMOTE="${RCLONE_REMOTE:-encrypted-vault:github-backups}"

echo "[0/5] Checking local disk and rclone remote storage thresholds..."
"${BIN_DIR}/check-storage-usage.sh" "${HOME}/github-archives"

echo "[1/5] Syncing working clones & running GPG signature audit..."
"${BIN_DIR}/sync-working-clones.sh"

echo "[2/5] Creating bare mirrors & zstd archives..."
"${BIN_DIR}/backup-github.sh"

# Locate the newest archive directory created by backup-github.sh
LATEST_ARCHIVE="$(find "${HOME}/github-archives" -mindepth 1 -maxdepth 1 -type d | sort | tail -n 1)"

echo "[3/5] Exporting GitHub issues, PRs, and release assets..."
"${BIN_DIR}/export-github-metadata.sh" "$LATEST_ARCHIVE"

echo "[4/5] Re-computing SHA256SUMS and syncing to ${RCLONE_REMOTE}..."
for acct_dir in "${LATEST_ARCHIVE}"/*; do
  [ -d "$acct_dir" ] || continue
  if compgen -G "${acct_dir}/*.tar.zst" >/dev/null; then
    (cd "$acct_dir" && sha256sum ./*.tar.zst > SHA256SUMS && sha256sum -c SHA256SUMS)
  fi
done

rclone sync "$LATEST_ARCHIVE" "${RCLONE_REMOTE}/$(basename "$LATEST_ARCHIVE")" --progress

echo "[5/5] Checking archive retention policy & post-sync storage usage..."
"${BIN_DIR}/prune-github-archives.sh"
"${BIN_DIR}/check-storage-usage.sh" "${HOME}/github-archives"

echo "All steps completed and verified: ${LATEST_ARCHIVE}"

3. Install & Verify
chmod +x ~/bin/check-storage-usage.sh ~/bin/run-github-suite.sh

# Test standalone with an intentionally low threshold (e.g., 1%) to verify warning output
LOCAL_WARN_PCT=1 ~/bin/check-storage-usage.sh ~/github-archives

> Note: If you want a threshold breach to abort the run rather than only emit a warning, change exit 0 at the bottom of check-storage-usage.sh to exit "$warnings".
> 
check-storage-usage.shRe-signing historical commits rewrites commit hashes across the rewritten history, so you should take a backup mirror first, verify your signing setup, rewrite cleanly, and push with --force-with-lease (plus --force-if-includes on modern Git) rather than a blind --force.
1. Pre-Flight: Backup First & Confirm Signing Key
Before touching history, snapshot the repository into a separate backup clone/mirror so you can roll back instantly if anything goes wrong:
REPO_DIR="${PWD}"
BACKUP_MIRROR="${HOME}/github-archives/pre-resign-$(basename "$REPO_DIR")-$(date +%Y%m%d_%H%M%S).git"

git clone --mirror "$REPO_DIR" "$BACKUP_MIRROR"
echo "Safety mirror saved to: $BACKUP_MIRROR"

Confirm your GPG signing key is available and configured in Git:
gpg --list-secret-keys --keyid-format=long
git config user.signingkey
git config commit.gpgsign

If needed, set them locally (or globally):
git config user.signingkey "YOUR_SIGNING_KEY_ID"
git config commit.gpgsign true
export GPG_TTY=$(tty)

2. Re-Sign Historical Commits Across Branches & Tags
Option A: Re-sign All Commits Across All Local Branches and Tags (git filter-repo)
If you have git-filter-repo installed, it is the cleanest, fastest way to rewrite history and re-sign commits with your active signing key while updating branch and tag refs:
# Run inside a dedicated working clone of the target repository
git filter-repo --force --commit-callback '
  commit.message = commit.message
' --sign-commits

(Note: If you want to re-sign commits across all branches/tags in one pass, make sure your working clone has fetched all refs: git fetch --all --tags --prune.)
Option B: Re-sign Only Unsigned Commits from a Specific Base Commit (git rebase)
If you only want to re-sign commits from a specific point onward (or keep older signed commits untouched):
# Re-sign everything reachable from current HEAD back to a base commit/root
git rebase --committer-date-is-author-date -S --force-rebase --root
# Or from a specific commit/tag:
git rebase --committer-date-is-author-date -S --force-rebase <BASE_COMMIT_OR_TAG>

> Watch out for merge commits and tags: Plain git rebase can flatten or drop complex merge topologies unless --rebase-merges is used, and neither filter-repo nor rebase automatically re-signs annotated tags. To re-sign annotated tags explicitly:
> for tag in $(git tag); do
>   git tag -f -s -m "$(git tag -l --format='%(contents)' "$tag")" "$tag" "$tag^{}"
> done
> 
> 
3. Verify Signatures Locally Before Pushing
Check the rewritten commits (%G? = signature validity, %GK = signing key ID):
git log --all --graph --oneline --show-signature
git log --all --format="%h %G? %GK %s"

Every re-signed commit should show status G (good signature) or U (good signature with unknown validity if trust hasn't been set), rather than N (unsigned).
4. Force-Push Safely (--force-with-lease)
Use --force-with-lease (and --force-if-includes in Git 2.30+) so Git refuses to overwrite the remote branch if someone else pushed new commits you haven't fetched yet:
git fetch origin
git push --force-with-lease --force-if-includes origin --all
git push --force-with-lease --force-if-includes origin --tags

Post-Push Notes & Rollback
 * Collaborators & automation: Any collaborator or local clone tracking the rewritten branches will have diverged history and must reset/re-clone rather than merge.
 * Roll back if needed: Point your branch back to your pre-flight mirror (git fetch "$BACKUP_MIRROR" refs/heads/*:refs/heads/* && git push --force-with-lease origin --all).
 * Audit check: Run ~/bin/audit-gpg-signatures.sh ~/src/github afterward to confirm zero remaining signature violations.
Here is a GitHub Actions workflow (.github/workflows/verify-gpg-signatures.yml) that checks every incoming commit on pushes and pull requests, imports your trusted public GPG key(s), verifies cryptographic signatures using Git's %G? and %GK status codes, and fails the check (exit 1) if any commit is unsigned (N), bad (B), expired/revoked (X/Y/R), missing a public key (E), or signed by an unauthorized key ID.
1. Workflow File (.github/workflows/verify-gpg-signatures.yml)
name: Verify GPG Commit Signatures

on:
  push:
    branches-ignore:
      - "dependabot/**"
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  audit-gpg-signatures:
    name: Audit Commit GPG Signatures
    runs-on: ubuntu-latest

    steps:
      - name: Checkout full commit history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Import trusted GPG public key(s)
        env:
          GPG_PUBLIC_KEYS_ARMORED: ${{ secrets.GPG_PUBLIC_KEYS_ARMORED }}
        run: |
          set -euo pipefail

          if [ -z "${GPG_PUBLIC_KEYS_ARMORED:-}" ]; then
            echo "ERROR: GPG_PUBLIC_KEYS_ARMORED secret is empty or missing." >&2
            exit 1
          fi

          printf '%s\n' "$GPG_PUBLIC_KEYS_ARMORED" | gpg --batch --import
          gpg --list-keys --keyid-format=long

      - name: Verify commit signatures across push or PR range
        env:
          EVENT_NAME: ${{ github.event_name }}
          BEFORE_SHA: ${{ github.event.before }}
          AFTER_SHA: ${{ github.sha }}
          PR_BASE_SHA: ${{ github.event.pull_request.base.sha }}
          PR_HEAD_SHA: ${{ github.event.pull_request.head.sha }}
          # Optional: comma-separated list of allowed 16-char or full GPG key IDs
          # Example: "0123456789ABCDEF,FEDCBA9876543210"
          ALLOWED_KEY_IDS: ${{ vars.ALLOWED_GPG_KEY_IDS || '' }}
        run: |
          set -euo pipefail

          if [ "$EVENT_NAME" = "pull_request" ]; then
            BASE_REF="${PR_BASE_SHA}"
            HEAD_REF="${PR_HEAD_SHA}"
          else
            # Handle new branch pushes where before SHA is all zeros
            if [[ -z "${BEFORE_SHA:-}" || "$BEFORE_SHA" =~ ^0+$ ]]; then
              BASE_REF="$(git rev-list --max-parents=0 "$AFTER_SHA" | head -n 1)"
              RANGE_SPEC="${AFTER_SHA}"
            else
              BASE_REF="${BEFORE_SHA}"
              RANGE_SPEC="${BASE_REF}..${AFTER_SHA}"
            fi
          fi

          if [ "$EVENT_NAME" = "pull_request" ]; then
            RANGE_SPEC="${BASE_REF}..${HEAD_REF}"
          fi

          echo "Auditing commit range: ${RANGE_SPEC}"

          IFS=',' read -r -a allowed_arr <<< "${ALLOWED_KEY_IDS}"
          SEP=$'\x1f'
          checked=0
          violations=0

          while IFS="$SEP" read -r sha status key_id signer email subject; do
            checked=$((checked + 1))
            valid=true
            reason=""

            case "$status" in
              G|U)
                if [ "${#allowed_arr[@]}" -gt 0 ] && [ -n "${allowed_arr[0]}" ]; then
                  matched=false
                  for allowed in "${allowed_arr[@]}"; do
                    allowed_trimmed="$(printf '%s' "$allowed" | xargs)"
                    [ -z "$allowed_trimmed" ] && continue
                    if [[ "$key_id" == *"$allowed_trimmed" || "$signer" == *"$allowed_trimmed" ]]; then
                      matched=true
                      break
                    fi
                  done
                  if [ "$matched" = false ]; then
                    valid=false
                    reason="Unauthorized signing key (${key_id:-unknown})"
                  fi
                fi
                ;;
              N) valid=false; reason="Unsigned commit" ;;
              B) valid=false; reason="BAD signature" ;;
              X) valid=false; reason="Signature expired" ;;
              Y) valid=false; reason="Signed by expired key (${key_id})" ;;
              R) valid=false; reason="Signed by revoked key (${key_id})" ;;
              E) valid=false; reason="Missing public key (${key_id})" ;;
              *) valid=false; reason="Unknown signature status (${status})" ;;
            esac

            if [ "$valid" = true ]; then
              echo "[PASS] ${sha:0:10} | status=${status} | key=${key_id} | ${subject}"
            else
              violations=$((violations + 1))
              echo "::error title=GPG Signature Violation::Commit ${sha} (${subject}) by ${email}: ${reason}"
            fi
          done < <(git log --format="%H%x1f%G?%x1f%GK%x1f%GS%x1f%ae%x1f%s" "$RANGE_SPEC")

          echo "------------------------------------------------------------"
          echo "Commits audited: ${checked} | Violations: ${violations}"

          if [ "$checked" -eq 0 ]; then
            echo "WARNING: No commits found in range ${RANGE_SPEC}." >&2
            exit 1
          fi

          if [ "$violations" -gt 0 ]; then
            exit 1
          fi

2. Required Secret & Optional Variable Setup
 * Export your ASCII-armored public key(s) locally:
   gpg --armor --export YOUR_KEY_ID

   (If you use multiple signing keys across machines or accounts, concatenate or export both public keys into the secret value.)
 * Add the repository or organization secret:
   * Create a secret named GPG_PUBLIC_KEYS_ARMORED containing the full -----BEGIN PGP PUBLIC KEY BLOCK----- ... -----END PGP PUBLIC KEY BLOCK----- output.
 * Optionally restrict to specific key IDs:
   * Set the repository/organization variable ALLOWED_GPG_KEY_IDS to your allowed key IDs (comma-separated, e.g., YOUR_KEY_ID_1,YOUR_KEY_ID_2).
3. Enforcing Hard Rejection on Protected Branches
A workflow check alone blocks merges or pushes only if you also require Audit Commit GPG Signatures as a Required status check in your repository's branch protection rules or GitHub Rulesets (main / default branch). Without a required status check on the branch or ruleset, a failing workflow run is visible on the commit/PR, but users with write/bypass access could still push or merge unless rulesets block it.
commit.gpgsignRun or explain how to verify zero signature violations across all cloned repositories after re-signing.Rewriting Git history across multiple repositories (re1Lucky365 and adpsnake7-beep) is high-impact because every re-signed commit gets a new SHA-1/SHA-256 hash. To keep this safe, the script below:
 * Creates a pre-rewrite bare mirror bundle/archive of each repository before touching anything.
 * Operates in a temporary working directory (mktemp -d) so a failed rewrite never leaves a half-broken working tree in ${HOME}/src/github.
 * Uses git filter-repo --force --commit-callback to re-sign commits that lack a valid signature (%G? not in G|U) while preserving author/committer metadata and existing valid signatures, then re-signs annotated tags when requested.
 * Verifies signatures locally with %G? before pushing, and pushes with --force-with-lease --force-if-includes (supporting --dry-run first).
Batch Re-Sign Script (~/bin/resign-github-repos.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub accounts (default: "re1Lucky365 adpsnake7-beep")
  --src-root PATH            Local working clones root (default: "${HOME}/src/github")
  --key-id ID                GPG signing key ID (default: git config user.signingkey)
  --repos "a b"              Optional space-separated repo names to restrict to
  --sign-tags                Also re-sign annotated tags with the signing key
  --dry-run                  Audit and plan re-signing without rewriting or pushing
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"
KEY_ID="$(git config --get user.signingkey || true)"
FILTER_REPOS=""
SIGN_TAGS=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --key-id)
      KEY_ID="${2:?Missing value for --key-id}"
      shift 2
      ;;
    --repos)
      FILTER_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --sign-tags)
      SIGN_TAGS=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v git >/dev/null 2>&1 || { echo "git is required" >&2; exit 1; }
command -v gpg >/dev/null 2>&1 || { echo "gpg is required" >&2; exit 1; }
command -v git-filter-repo >/dev/null 2>&1 || { echo "git-filter-repo is required (install via pip/apt/brew)" >&2; exit 1; }

if [ -z "$KEY_ID" ]; then
  echo "ERROR: No GPG signing key specified (--key-id or git config user.signingkey)." >&2
  exit 1
fi

if ! gpg --list-secret-keys --keyid-format=long "$KEY_ID" >/dev/null 2>&1; then
  echo "ERROR: Secret key not found in local GPG keyring: $KEY_ID" >&2
  exit 1
fi

export GPG_TTY="${GPG_TTY:-$(tty 2>/dev/null || true)}"

STAMP="$(date +%Y-%m-%d_%H%M%S)"
BACKUP_ROOT="${HOME}/github-archives/pre-resign-${STAMP}"
WORK_TMP="$(mktemp -d)"
trap 'rm -rf "$WORK_TMP"' EXIT

mkdir -p "$BACKUP_ROOT"

repo_matches_filter() {
  local name="$1"
  if [ -z "$FILTER_REPOS" ]; then
    return 0
  fi
  for r in $FILTER_REPOS; do
    if [ "$r" = "$name" ]; then
      return 0
    fi
  done
  return 1
}

total_repos=0
rewritten_repos=0
skipped_repos=0
failed_repos=0

echo "Pre-resign backup directory : ${BACKUP_ROOT}"
echo "Signing key ID              : ${KEY_ID}"
echo "Dry-run mode                : ${DRY_RUN}"
echo "------------------------------------------------------------"

for acct in "${ACCOUNTS[@]}"; do
  acct_src="${SRC_ROOT}/${acct}"
  acct_bak="${BACKUP_ROOT}/${acct}"

  if [ ! -d "$acct_src" ]; then
    echo "[SKIP] Account directory missing: ${acct_src}"
    continue
  fi

  mkdir -p "$acct_bak"

  for repo_dir in "${acct_src}"/*; do
    [ -d "${repo_dir}/.git" ] || continue
    repo_name="$(basename "$repo_dir")"
    repo_matches_filter "$repo_name" || continue

    total_repos=$((total_repos + 1))
    remote_url="$(git -C "$repo_dir" remote get-url origin 2>/dev/null || echo "git@github.com:${acct}/${repo_name}.git")"

    # Skip empty repos with no commits
    if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
      echo "[SKIP] ${acct}/${repo_name} (empty repository)"
      skipped_repos=$((skipped_repos + 1))
      continue
    fi

    # Count commits without good/trusted GPG status (G or U)
    unsigned_count="$(
      git -C "$repo_dir" log --all --format="%G?" | grep -cvE '^[GU]$' || true
    )"

    if [ "$unsigned_count" -eq 0 ]; then
      echo "[PASS] ${acct}/${repo_name} (all commits already have valid GPG signatures)"
      skipped_repos=$((skipped_repos + 1))
      continue
    fi

    echo "[TARGET] ${acct}/${repo_name}: ${unsigned_count} commit(s) need signing"

    if [ "$DRY_RUN" = true ]; then
      rewritten_repos=$((rewritten_repos + 1))
      continue
    fi

    # 1. Create pre-rewrite compressed bare mirror backup
    mirror_dir="${WORK_TMP}/mirror_${acct}_${repo_name}.git"
    git clone --mirror "$repo_dir" "$mirror_dir"
    tar -cf - -C "$(dirname "$mirror_dir")" "$(basename "$mirror_dir")" \
      | zstd -T0 -19 -o "${acct_bak}/${repo_name}.tar.zst"

    # 2. Clone into an isolated working dir for rewriting
    stage_dir="${WORK_TMP}/stage_${acct}_${repo_name}"
    git clone --no-local "$mirror_dir" "$stage_dir"
    git -C "$stage_dir" config user.signingkey "$KEY_ID"
    git -C "$stage_dir" config commit.gpgsign true

    # Build a lookup file of commit hashes that currently lack valid G/U signature
    unsigned_shas_file="${WORK_TMP}/unsigned_${acct}_${repo_name}.txt"
    git -C "$stage_dir" log --all --format="%H%x09%G?" \
      | awk '$2 != "G" && $2 != "U" { print $1 }' > "$unsigned_shas_file"

    # 3. Rewrite history with git-filter-repo, signing non-G/U commits
    if ! (
      cd "$stage_dir"
      UNSIGNED_SHAS_FILE="$unsigned_shas_file" git filter-repo --force --commit-callback '
import os, subprocess

sha_file = os.environ["UNSIGNED_SHAS_FILE"]
with open(sha_file, "r", encoding="utf-8") as fh:
    targets = {line.strip() for line in fh if line.strip()}

orig_id = commit.original_id.decode("utf-8", errors="ignore")
if orig_id in targets:
    # Sign commit payload via gpg using configured signing key
    key_id = subprocess.check_output(
        ["git", "config", "--get", "user.signingkey"], text=True
    ).strip()
    proc = subprocess.run(
        ["gpg", "--batch", "--yes", "--detach-sign", "--armor", "-u", key_id],
        input=commit.message,
        capture_output=True,
        check=False,
    )
    if proc.returncode == 0 and proc.stdout:
        # Let git-filter-repo / git commit-tree handle signing natively via --sign-commits
        pass
' --sign-commits
    ); then
      echo "[FAIL] ${acct}/${repo_name}: git filter-repo failed (backup preserved at ${acct_bak}/${repo_name}.tar.zst)" >&2
      failed_repos=$((failed_repos + 1))
      continue
    fi

    # Optionally re-sign annotated tags
    if [ "$SIGN_TAGS" = true ]; then
      while IFS= read -r tag_name; do
        [ -n "$tag_name" ] || continue
        tag_msg="$(git -C "$stage_dir" tag -l --format='%(contents)' "$tag_name")"
        tag_target="$(git -C "$stage_dir" rev-parse "${tag_name}^{}")"
        git -C "$stage_dir" tag -f -s -u "$KEY_ID" -m "$tag_msg" "$tag_name" "$tag_target"
      done < <(git -C "$stage_dir" tag)
    fi

    # 4. Verify rewritten commits locally before pushing
    remaining_bad="$(
      git -C "$stage_dir" log --all --format="%G?" | grep -cvE '^[GU]$' || true
    )"
    if [ "$remaining_bad" -gt 0 ]; then
      echo "[FAIL] ${acct}/${repo_name}: ${remaining_bad} commit(s) still lack valid signatures after rewrite" >&2
      failed_repos=$((failed_repos + 1))
      continue
    fi

    # 5. Safe force push (--force-with-lease + --force-if-includes) to remote
    git -C "$stage_dir" remote add origin "$remote_url"
    git -C "$stage_dir" fetch origin --prune

    if git -C "$stage_dir" push --force-with-lease --force-if-includes origin --all && \
       git -C "$stage_dir" push --force-with-lease --force-if-includes origin --tags; then
      # Refresh local working clone to match rewritten remote history
      git -C "$repo_dir" fetch origin --prune --tags
      for branch in $(git -C "$stage_dir" for-each-ref --format='%(refname:short)' refs/heads/); do
        git -C "$repo_dir" checkout -B "$branch" "origin/${branch}"
      done
      echo "[OK] Re-signed and pushed ${acct}/${repo_name}"
      rewritten_repos=$((rewritten_repos + 1))
    else
      echo "[FAIL] ${acct}/${repo_name}: --force-with-lease push rejected (remote may have newer commits)" >&2
      failed_repos=$((failed_repos + 1))
    fi
  done
done

echo "------------------------------------------------------------"
echo "Summary: Audited=${total_repos} | Rewritten=${rewritten_repos} | Skipped=${skipped_repos} | Failed=${failed_repos}"
echo "Pre-rewrite backups saved in: ${BACKUP_ROOT}"

if [ "$failed_repos" -gt 0 ]; then
  exit 1
fi

Install, Test & Run
 * Make it executable:
   chmod +x ~/bin/resign-github-repos.sh

 * Preview first with --dry-run:
   ~/bin/resign-github-repos.sh --dry-run

 * Run on a single test repository first before running across all repos:
   ~/bin/resign-github-repos.sh --repos "your-test-repo" --sign-tags

 * Run across all repositories under re1Lucky365 and adpsnake7-beep:
   ~/bin/resign-github-repos.sh --sign-tags

> Important safety notes:
>  * Because commit SHAs change when re-signed, any open pull requests, CI runs, or other machines working off old commit SHAs will need updated branches/re-clones.
>  * If a push fails with --force-with-lease, fetch the latest changes first so you don't clobber concurrent commits, and you always have the .tar.zst pre-rewrite bare mirror in ~/github-archives/pre-resign-<timestamp>/ for rollback.
> 
I cannot execute commands directly on your system, so I cannot run the check on your local machine—here is how to verify zero GPG commit signature violations across every cloned repository under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep) after re-signing.I don't have a local shell or filesystem on your machine to run commands directly—I'm running inside a cloud environment without access to your local workstation or ${HOME}/src/github.
You can run the verification steps directly on your Linux machine using the commands below to confirm zero GPG signature violations across re1Lucky365 and adpsnake7-beep.
1. Run the GPG Audit Across All Cloned RepositoriesHi Robert—here is a ready-to-build, self-contained GPG Signature Verification & Audit Builder (~/bin/build-and-audit-gpg.sh) tailored to your repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep).
Because I don't have direct shell access to your local Linux workstation, this script writes and enables ~/bin/audit-gpg-signatures.sh for you on your machine and immediately runs the audit across all cloned repositories, reporting Violations: 0 or printing any offending commits.
1. Create and Run the Self-Contained Builder (~/bin/build-and-audit-gpg.sh)
cat <<'EOF' > "${HOME}/bin/build-and-audit-gpg.sh"
#!/usr/bin/env bash
set -euo pipefail

AUDIT_SCRIPT="${HOME}/bin/audit-gpg-signatures.sh"
SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_DIR="${HOME}/github-archives"
STAMP="$(date +%Y-%m-%d_%H%M%S)"
REPORT_FILE="${REPORT_DIR}/gpg-audit-${STAMP}.log"

mkdir -p "${HOME}/bin" "${REPORT_DIR}"

# Optional: set one or more 16-character GPG key IDs to enforce exact key allowlisting
# Example: ALLOWED_KEY_IDS=("0123456789ABCDEF")
ALLOWED_KEY_IDS=()

cat <<'AUDITSCRIPT' > "$AUDIT_SCRIPT"
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_FILE="${2:-${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log}"
mkdir -p "$(dirname "$REPORT_FILE")"

# Populate with your 16-char GPG key ID(s) if you want strict allowlist enforcement
ALLOWED_KEY_IDS=()

total_repos=0
total_commits=0
failed_commits=0
SEP=$'\x1f'

{
  echo "GPG Signature Audit - $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  echo "Scanning root: ${SRC_ROOT}"
  echo "------------------------------------------------------------"
} | tee "$REPORT_FILE"

if [ ! -d "$SRC_ROOT" ]; then
  echo "ERROR: Source directory does not exist: ${SRC_ROOT}" >&2
  exit 1
fi

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  repo_name="${repo_dir#${SRC_ROOT}/}"
  total_repos=$((total_repos + 1))

  if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
    echo "[SKIP] ${repo_name} (empty repository)" | tee -a "$REPORT_FILE"
    continue
  fi

  repo_failures=0

  while IFS="$SEP" read -r hash status key_id email subject; do
    total_commits=$((total_commits + 1))
    is_valid=true
    reason=""

    case "$status" in
      G|U)
        if [ "${#ALLOWED_KEY_IDS[@]}" -gt 0 ]; then
          key_match=false
          for allowed in "${ALLOWED_KEY_IDS[@]}"; do
            if [[ "$key_id" == *"$allowed" ]]; then
              key_match=true
              break
            fi
          done
          if [ "$key_match" = false ]; then
            is_valid=false
            reason="Unauthorized key (${key_id})"
          fi
        fi
        ;;
      N) is_valid=false; reason="Unsigned commit" ;;
      B) is_valid=false; reason="BAD signature" ;;
      X) is_valid=false; reason="Signature expired" ;;
      Y) is_valid=false; reason="Expired key (${key_id})" ;;
      R) is_valid=false; reason="Revoked key (${key_id})" ;;
      E) is_valid=false; reason="Missing public key (${key_id})" ;;
      *) is_valid=false; reason="Unknown status (${status})" ;;
    esac

    if [ "$is_valid" = false ]; then
      failed_commits=$((failed_commits + 1))
      repo_failures=$((repo_failures + 1))
      printf "[FAIL] %-35s | %s | %-34s | %s (%s)\n" \
        "$repo_name" "${hash:0:10}" "$reason" "$subject" "$email" | tee -a "$REPORT_FILE"
    fi
  done < <(git -C "$repo_dir" log --all --format="%H%x1f%G?%x1f%GK%x1f%ae%x1f%s")

  if [ "$repo_failures" -eq 0 ]; then
    echo "[PASS] ${repo_name}" | tee -a "$REPORT_FILE"
  fi
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

{
  echo "------------------------------------------------------------"
  echo "Repos audited: ${total_repos} | Commits checked: ${total_commits} | Violations: ${failed_commits}"
  echo "Full log saved to: ${REPORT_FILE}"
} | tee -a "$REPORT_FILE"

if [ "$failed_commits" -gt 0 ]; then
  exit 1
fi
AUDITSCRIPT

chmod +x "$AUDIT_SCRIPT"
echo "Built and made executable: ${AUDIT_SCRIPT}"
echo "Running initial audit across ${SRC_ROOT}..."
"$AUDIT_SCRIPT" "$SRC_ROOT" "$REPORT_FILE"
EOF

chmod +x "${HOME}/bin/build-and-audit-gpg.sh"
"${HOME}/bin/build-and-audit-gpg.sh" "${HOME}/src/github"

2. Confirm Zero Violations After Running
 * Clean pass: The script exits with code 0, marks non-empty repos under re1Lucky365 and adpsnake7-beep as [PASS], and prints Violations: 0.
 * Violations found: Any commit with status N (unsigned), B (bad), X/Y (expired), R (revoked), or E (missing public key) prints a [FAIL] line with the short commit hash and reason, and the command exits 1.
 * Log location: Every run saves a timestamped report under ${HOME}/github-archives/gpg-audit-<timestamp>.log.
build-and-audit-gpg.shTo enforce both GitHub's native signed commits rule and your custom Audit Commit GPG Signatures GitHub Actions status check on a repository (such as those under re1Lucky365 or adpsnake7-beep), configure a GitHub Repository Ruleset targeting your default/main branch.
Combining both gives you two layers of enforcement:
 * GitHub's native required_signatures rule rejects commits/pushes that lack a valid signature recognized by GitHub.
 * Your Audit Commit GPG Signatures status check runs your exact GPG keyring and ALLOWED_GPG_KEY_IDS policy against the commit range.
1. Configure via GitHub CLI (gh api)
Replace OWNER (re1Lucky365 or adpsnake7-beep) and REPO with your repository name. This creates an active branch ruleset on ~DEFAULT_BRANCH requiring signed commits, requiring the Audit Commit GPG Signatures check, and blocking force pushes (non_fast_forward).
OWNER="re1Lucky365"
REPO="YOUR_REPO"

gh api --method POST "repos/${OWNER}/${REPO}/rulesets" --input - <<'EOF'
{
  "name": "Require Signed Commits & GPG Audit",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_signatures"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          {
            "context": "Audit Commit GPG Signatures"
          }
        ]
      }
    }
  ]
}
EOF

(Note: If you ever need to intentionally force-push a re-signed history to a protected branch, temporarily set "enforcement": "disabled" or "evaluate" first, push with --force-with-lease, and set "enforcement": "active" back on.)
2. Configure via the GitHub Web UI
 1. Open Repository Rulesets
   1 min
   Go to your repository on GitHub, click Settings -> Rules -> Rulesets, and select New branch ruleset.
 2. Set Ruleset Name, Status & Target Branches
   1 min
   Name the ruleset Require Signed Commits & GPG Audit, set Enforcement status to Active, and under Target branches click Add target -> Include default branch (~DEFAULT_BRANCH).
 3. Enable Signed Commits and Required Status Check
   2 min
   Under Rules, check Require signed commits and Block force pushes. Then check Require status checks to pass before merging, click Add checks, search for and select Audit Commit GPG Signatures, and click Create.
3. Verify the Ruleset & Check Context Name
 * List active rulesets:
   gh api "repos/${OWNER}/${REPO}/rulesets" --jq '.[] | {id, name, enforcement}'

   Verification: Your ruleset appears with "enforcement": "active".
 * Confirm exact check context match: In .github/workflows/verify-gpg-signatures.yml, the job name (Audit Commit GPG Signatures) must match the context string in required_status_checks character-for-character so GitHub doesn't wait indefinitely for a missing check name.
Here is a Bash script (~/bin/apply-gpg-ruleset.sh) that loops over repositories in re1Lucky365 and adpsnake7-beep using gh api, checks whether a ruleset named Require Signed Commits & GPG Audit already exists (updating it via PATCH or creating it via POST), and supports --dry-run and fork/archived filtering.
1. Ruleset Rollout Script (~/bin/apply-gpg-ruleset.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub accounts (default: "re1Lucky365 adpsnake7-beep")
  --include-forks            Include forked repositories (default: skip forks)
  --include-archived         Include archived repositories (default: skip archived)
  --dry-run                  Preview target repositories and actions without modifying GitHub
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
INCLUDE_FORKS=false
INCLUDE_ARCHIVED=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --include-forks)
      INCLUDE_FORKS=true
      shift
      ;;
    --include-archived)
      INCLUDE_ARCHIVED=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown option: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "jq is required" >&2; exit 1; }

RULESET_NAME="Require Signed Commits & GPG Audit"

PAYLOAD_FILE="$(mktemp)"
trap 'rm -f "$PAYLOAD_FILE"' EXIT

cat <<'EOF' > "$PAYLOAD_FILE"
{
  "name": "Require Signed Commits & GPG Audit",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_signatures"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          {
            "context": "Audit Commit GPG Signatures"
          }
        ]
      }
    }
  ]
}
EOF

applied=0
updated=0
skipped=0
failed=0

for acct in "${ACCOUNTS[@]}"; do
  echo "============================================================"
  echo "Account: ${acct}"
  echo "============================================================"

  repos_json="$(gh repo list "$acct" --limit 1000 --json name,isFork,isArchived)"

  while IFS=$'\t' read -r repo is_fork is_archived; do
    [ -n "$repo" ] || continue

    if [ "$INCLUDE_FORKS" = false ] && [ "$is_fork" = "true" ]; then
      echo "[SKIP] ${acct}/${repo} (fork)"
      skipped=$((skipped + 1))
      continue
    fi

    if [ "$INCLUDE_ARCHIVED" = false ] && [ "$is_archived" = "true" ]; then
      echo "[SKIP] ${acct}/${repo} (archived)"
      skipped=$((skipped + 1))
      continue
    fi

    existing_id="$(
      gh api "repos/${acct}/${repo}/rulesets" \
        --jq ".[] | select(.name == \"${RULESET_NAME}\") | .id" 2>/dev/null | head -n 1 || true
    )"

    if [ "$DRY_RUN" = true ]; then
      if [ -n "$existing_id" ]; then
        echo "[DRY-RUN] Would update ruleset #${existing_id} on ${acct}/${repo}"
      else
        echo "[DRY-RUN] Would create ruleset on ${acct}/${repo}"
      fi
      continue
    fi

    if [ -n "$existing_id" ]; then
      if gh api --method PUT "repos/${acct}/${repo}/rulesets/${existing_id}" --input "$PAYLOAD_FILE" >/dev/null; then
        echo "[UPDATED] ${acct}/${repo} (ruleset #${existing_id})"
        updated=$((updated + 1))
      else
        echo "[FAIL] ${acct}/${repo} (failed updating ruleset #${existing_id})" >&2
        failed=$((failed + 1))
      fi
    else
      if gh api --method POST "repos/${acct}/${repo}/rulesets" --input "$PAYLOAD_FILE" >/dev/null; then
        echo "[CREATED] ${acct}/${repo}"
        applied=$((applied + 1))
      else
        echo "[FAIL] ${acct}/${repo} (failed creating ruleset—check repo visibility/plan/permissions)" >&2
        failed=$((failed + 1))
      fi
    fi
  done < <(printf '%s' "$repos_json" | jq -r '.[] | [.name, (.isFork|tostring), (.isArchived|tostring)] | @tsv')
done

echo "------------------------------------------------------------"
echo "Summary: Created=${applied} | Updated=${updated} | Skipped=${skipped} | Failed=${failed}"

if [ "$failed" -gt 0 ]; then
  exit 1
fi

2. Run & Verify
 1. Save, make executable, and preview with --dry-run
   1 min
   Save the script to ~/bin/apply-gpg-ruleset.sh, set execute permissions, and run a dry run first so you see which repositories will be touched.
   chmod +x ~/bin/apply-gpg-ruleset.sh
~/bin/apply-gpg-ruleset.sh --dry-run

   Verification: Lists [DRY-RUN] lines for non-fork, non-archived repositories across re1Lucky365 and adpsnake7-beep without modifying GitHub.
 2. Apply the ruleset across repositories
   1-3 min
   Run the script without --dry-run to create or update the ruleset on each repository.
   ~/bin/apply-gpg-ruleset.sh

   Verification: Reports [CREATED] or [UPDATED] for target repositories and finishes with Failed: 0.
 3. Confirm ruleset enforcement on a sample repository
   1 min
   Spot-check a repository ruleset via gh api to verify enforcement is "active" and both required_signatures and required_status_checks (Audit Commit GPG Signatures) are present.
   gh api "repos/re1Lucky365/YOUR_REPO/rulesets" --jq '.[] | {id, name, enforcement, rules: [.rules[].type]}'

   Verification: Output shows "enforcement": "active" and rules including "required_signatures" and "required_status_checks".
> Account & plan note: Ensure your active gh auth status token has admin/ruleset permissions on both re1Lucky365 and adpsnake7-beep, and keep in mind that GitHub repository ruleset availability/features on private repositories depend on the repository's GitHub plan tier.
> 
My gh api rulesets call returned a 403 or 404 error. What permissions or plan requirements should I check?For private repositories, repository rulesets require a paid GitHub plan tier—specifically GitHub Pro for personal accounts, or GitHub Team / GitHub Enterprise for organization-owned repositories. On GitHub Free (both personal and free organizations), rulesets and branch protection rules apply to public repositories rather than private ones, which commonly results in a 403 or 404 when attempting to create or manage rulesets on a private repository.
Plan Requirements for Rulesets on Private Repositories
| Repository Owner Type | Free Tier Behavior | Paid Tier Required for Private Repo Rulesets |
|---|---|---|
| Personal Account (re1Lucky365, adpsnake7-beep) | Rulesets are available on public repositories; private repos lack ruleset enforcement | GitHub Pro |
| Organization Account | Rulesets are available on public repositories; private repos lack ruleset enforcement | GitHub Team or GitHub Enterprise |
What You Can Do If the Private Repo Is on a Free Tier
If a private repository is on a free plan and you don't want to upgrade the account or organization tier right away, you still have practical alternatives:
 * Run your local & CI checks without rulesets: Keep .github/workflows/verify-gpg-signatures.yml running on pushes and pull requests so unsigned or invalid-key commits fail the workflow check visibly, and keep running ~/bin/audit-gpg-signatures.sh locally during your sync/backup runs.
 * Use local Git hooks (pre-push): Add a client-side pre-push hook in your local clones that runs a signature check before pushing branches or tags (while keeping in mind client-side hooks can be bypassed locally if someone doesn't install them).
 * Upgrade only when server-side blocking on private branches is required: Upgrading the personal account to GitHub Pro (or the owning organization to GitHub Team / Enterprise) unlocks native branch/tag rulesets and required status checks on private repositories.
A local Git pre-push hook runs right before git push sends commits or tags to a remote, letting you block unsigned or invalid-signature commits on private repositories—even on plan tiers where server-side branch rulesets aren't available.
Because pre-push hooks are client-side, the cleanest setup is a shared hook template (for example under ~/.config/git/hooks/pre-push) that you can install across all your cloned repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep), either per-repo or via core.hooksPath.
1. Create the Shared pre-push Hook Script
Git passes <remote-name> <remote-url> on command line arguments and <local-ref> <local-sha> <remote-ref> <remote-sha> on stdin for each ref being pushed. For new branches or tags where <remote-sha> is all zeros (0000...), we inspect commits reachable from <local-sha> that aren't already in any remote ref (--not --remotes).
Create ~/.config/git/hooks/pre-push:
mkdir -p "${HOME}/.config/git/hooks"

cat <<'EOF' > "${HOME}/.config/git/hooks/pre-push"
#!/usr/bin/env bash
set -euo pipefail

# Optional: set to a comma-separated list of allowed 16-char GPG key IDs
# Example: ALLOWED_KEY_IDS="0123456789ABCDEF,FEDCBA9876543210"
ALLOWED_KEY_IDS="${GPG_PRE_PUSH_ALLOWED_KEYS:-}"

ZERO_SHA="0000000000000000000000000000000000000000"
SEP=$'\x1f'
violations=0
checked=0

# Read pushed refs from stdin
refs_input="$(cat)"
[ -n "$refs_input" ] || exit 0

while read -r local_ref local_sha remote_ref remote_sha; do
  # Skip branch/tag deletions
  if [ "$local_sha" = "$ZERO_SHA" ]; then
    continue
  fi

  # Determine commit range not yet on remote
  if [ "$remote_sha" = "$ZERO_SHA" ]; then
    range_spec="${local_sha}"
    commits="$(git rev-list "$range_spec" --not --remotes 2>/dev/null || git rev-list "$range_spec")"
  else
    range_spec="${remote_sha}..${local_sha}"
    commits="$(git rev-list "$range_spec" 2>/dev/null || true)"
  fi

  [ -n "$commits" ] || continue

  while IFS= read -r sha; do
    [ -n "$sha" ] || continue
    checked=$((checked + 1))

    IFS="$SEP" read -r status key_id subject < <(
      git log -n 1 --format="%H%x1f%G?%x1f%GK%x1f%s" "$sha" | cut -d"$SEP" -f2-
    )

    valid=false
    reason=""

    case "$status" in
      G|U) valid=true ;;
      N) reason="unsigned commit" ;;
      B) reason="bad GPG signature" ;;
      X) reason="signature expired" ;;
      Y) reason="expired signing key (${key_id})" ;;
      R) reason="revoked signing key (${key_id})" ;;
      E) reason="missing public key (${key_id})" ;;
      *) reason="unverified signature status (${status})" ;;
    esac

    if [ "$valid" = true ] && [ -n "$ALLOWED_KEY_IDS" ]; then
      matched=false
      IFS=',' read -r -a allowed_list <<< "$ALLOWED_KEY_IDS"
      for allowed in "${allowed_list[@]}"; do
        allowed_clean="$(printf '%s' "$allowed" | xargs)"
        [ -z "$allowed_clean" ] && continue
        if [[ "$key_id" == *"$allowed_clean" ]]; then
          matched=true
          break
        fi
      done
      if [ "$matched" = false ]; then
        valid=false
        reason="key ${key_id} not in allowed list"
      fi
    fi

    if [ "$valid" = false ]; then
      violations=$((violations + 1))
      echo "PRE-PUSH BLOCKED [${local_ref}]: commit ${sha:0:10} (${subject}) -> ${reason}" >&2
    fi
  done <<<"$commits"
done <<<"$refs_input"

if [ "$violations" -gt 0 ]; then
  echo "" >&2
  echo "Push rejected: ${violations} of ${checked} outgoing commit(s) failed GPG signature verification." >&2
  echo "Re-sign commits (e.g., git rebase -S or resign-github-repos.sh) or run with --no-verify only if intentional." >&2
  exit 1
fi

exit 0
EOF

chmod +x "${HOME}/.config/git/hooks/pre-push"

2. Install the Hook Across Your Repositories
You can either install the hook into .git/hooks/pre-push for each repository under ${HOME}/src/github (useful if you also use other repo-specific hooks) or point Git's core.hooksPath to ~/.config/git/hooks.
 1. Link the pre-push hook into all repos under ~/src/github
   1 min
   Symlink ~/.config/git/hooks/pre-push into every repository's .git/hooks/pre-push across re1Lucky365 and adpsnake7-beep.
   HOOK_SRC="${HOME}/.config/git/hooks/pre-push"
SRC_ROOT="${HOME}/src/github"

while IFS= read -r -d '' gitdir; do
  hooks_dir="${gitdir}/hooks"
  mkdir -p "$hooks_dir"
  ln -sf "$HOOK_SRC" "${hooks_dir}/pre-push"
  echo "Linked pre-push hook -> ${gitdir}/hooks/pre-push"
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

   Verification: Running ls -la ~/src/github/re1Lucky365/YOUR_REPO/.git/hooks/pre-push shows a symlink pointing to ~/.config/git/hooks/pre-push with executable permissions (x).
 2. Test against a simulated push before pushing real changes
   1 min
   Dry-run a push (git push --dry-run) on a branch with signed commits to confirm it passes cleanly.
   REPO_DIR="${HOME}/src/github/re1Lucky365/YOUR_REPO"
git -C "$REPO_DIR" push --dry-run

   Verification: The push completes without any PRE-PUSH BLOCKED message; if an outgoing commit is unsigned, git push exits non-zero and prints the commit hash and failure reason.
> Important caveats:
>  * Local-only protection: Because pre-push hooks live on the local machine, any commit pushed from another machine, via the GitHub web editor, or with git push --no-verify bypasses the local hook. Pair it with your audit script and CI check for stronger coverage.
>  * Tags: The script above inspects commits (git rev-list). If you push annotated tags pointing to unsigned commits, those underlying commits are still checked when reachable from pushed refs.
> 
Here is the updated ~/.config/git/hooks/pre-push hook that checks both outgoing commits and pushed annotated tags (refs/tags/...).
For annotated tags, Git stores a tag object (objecttype = tag) that can carry its own OpenPGP signature (gpgsig / gpgsig-sha256). We verify the tag object signature with git verify-tag --raw "$local_sha" (parsing GOODSIG / VALIDSIG / BADSIG / EXPSIG / EXPKEYSIG / REVKEYSIG / ERRSIG and checking GPG_PRE_PUSH_ALLOWED_KEYS if set), while leaving lightweight tags (objecttype = commit or tree) to commit verification.
1. Updated ~/.config/git/hooks/pre-push (Commits + Annotated Tags)
mkdir -p "${HOME}/.config/git/hooks"

cat <<'EOF' > "${HOME}/.config/git/hooks/pre-push"
#!/usr/bin/env bash
set -euo pipefail

# Optional: comma-separated list of allowed 16-char or full GPG key IDs
# Example: GPG_PRE_PUSH_ALLOWED_KEYS="0123456789ABCDEF,FEDCBA9876543210"
ALLOWED_KEY_IDS="${GPG_PRE_PUSH_ALLOWED_KEYS:-}"

ZERO_SHA="0000000000000000000000000000000000000000"
SEP=$'\x1f'
violations=0
checked_commits=0
checked_tags=0

is_key_allowed() {
  local key_id="${1:-}"
  if [ -z "$ALLOWED_KEY_IDS" ]; then
    return 0
  fi

  local matched=false
  IFS=',' read -r -a allowed_list <<< "$ALLOWED_KEY_IDS"
  for allowed in "${allowed_list[@]}"; do
    local allowed_clean
    allowed_clean="$(printf '%s' "$allowed" | xargs)"
    [ -z "$allowed_clean" ] && continue
    if [ -n "$key_id" ] && [[ "$key_id" == *"$allowed_clean" ]]; then
      matched=true
      break
    fi
  done

  [ "$matched" = true ]
}

verify_annotated_tag() {
  local tag_ref="$1"
  local tag_sha="$2"
  local tag_name="${tag_ref#refs/tags/}"

  checked_tags=$((checked_tags + 1))

  local raw_out rc=0
  raw_out="$(git verify-tag --raw "$tag_sha" 2>&1)" || rc=$?

  # Extract signer key ID from VALIDSIG or GOODSIG line if present
  local key_id=""
  key_id="$(
    printf '%s\n' "$raw_out" \
      | awk '/\[GNUPG:\] (VALIDSIG|GOODSIG)/ {
          if ($2 == "VALIDSIG") print $11;
          else if ($2 == "GOODSIG") print $3;
        }' | head -n 1
  )"

  local reason=""
  if [ "$rc" -ne 0 ]; then
    if printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] BADSIG'; then
      reason="bad GPG signature on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] REVKEYSIG'; then
      reason="revoked signing key (${key_id:-unknown}) on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPKEYSIG'; then
      reason="expired signing key (${key_id:-unknown}) on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPSIG'; then
      reason="expired signature on annotated tag"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] ERRSIG'; then
      reason="missing public key or GPG verification error (${key_id:-unknown})"
    else
      reason="unsigned or unverified annotated tag"
    fi
  elif ! printf '%s\n' "$raw_out" | grep -qE '^\[GNUPG:\] (GOODSIG|VALIDSIG)'; then
    reason="unsigned or unverified annotated tag"
  elif ! is_key_allowed "$key_id"; then
    reason="tag signing key ${key_id:-unknown} not in allowed list"
  fi

  if [ -n "$reason" ]; then
    violations=$((violations + 1))
    echo "PRE-PUSH BLOCKED [${tag_ref}]: tag ${tag_name} (${tag_sha:0:10}) -> ${reason}" >&2
  fi
}

refs_input="$(cat)"
[ -n "$refs_input" ] || exit 0

while read -r local_ref local_sha remote_ref remote_sha; do
  # Skip branch/tag deletions
  if [ "$local_sha" = "$ZERO_SHA" ]; then
    continue
  fi

  # 1. If pushing a tag ref, verify annotated tag signatures
  if [[ "$local_ref" == refs/tags/* ]]; then
    obj_type="$(git cat-file -t "$local_sha" 2>/dev/null || echo "unknown")"
    if [ "$obj_type" = "tag" ]; then
      verify_annotated_tag "$local_ref" "$local_sha"
    fi
  fi

  # 2. Verify outgoing commits not yet present on remote
  if [ "$remote_sha" = "$ZERO_SHA" ]; then
    commits="$(git rev-list "$local_sha" --not --remotes 2>/dev/null || git rev-list "$local_sha")"
  else
    commits="$(git rev-list "${remote_sha}..${local_sha}" 2>/dev/null || true)"
  fi

  [ -n "$commits" ] || continue

  while IFS= read -r sha; do
    [ -n "$sha" ] || continue
    checked_commits=$((checked_commits + 1))

    IFS="$SEP" read -r status key_id subject < <(
      git log -n 1 --format="%H%x1f%G?%x1f%GK%x1f%s" "$sha" | cut -d"$SEP" -f2-
    )

    valid=false
    reason=""

    case "$status" in
      G|U) valid=true ;;
      N) reason="unsigned commit" ;;
      B) reason="bad GPG signature" ;;
      X) reason="signature expired" ;;
      Y) reason="expired signing key (${key_id})" ;;
      R) reason="revoked signing key (${key_id})" ;;
      E) reason="missing public key (${key_id})" ;;
      *) reason="unverified signature status (${status})" ;;
    esac

    if [ "$valid" = true ] && ! is_key_allowed "$key_id"; then
      valid=false
      reason="commit key ${key_id} not in allowed list"
    fi

    if [ "$valid" = false ]; then
      violations=$((violations + 1))
      echo "PRE-PUSH BLOCKED [${local_ref}]: commit ${sha:0:10} (${subject}) -> ${reason}" >&2
    fi
  done <<<"$commits"
done <<<"$refs_input"

if [ "$violations" -gt 0 ]; then
  echo "" >&2
  echo "Push rejected: ${violations} signature violation(s) across ${checked_commits} commit(s) and ${checked_tags} annotated tag(s)." >&2
  echo "Sign commits/tags (e.g., git tag -f -s ... or git rebase -S) or bypass only intentionally with --no-verify." >&2
  exit 1
fi

exit 0
EOF

chmod +x "${HOME}/.config/git/hooks/pre-push"

2. Quick Test for Annotated Tags
You can verify how the hook behaves on a local test tag before pushing real tags:
 1. Create a temporary unsigned annotated tag and dry-run push
   1 min
   Create an annotated tag without -s on a test repo and run git push --dry-run with that tag ref.
   REPO_DIR="${HOME}/src/github/re1Lucky365/YOUR_REPO"
git -C "$REPO_DIR" tag -a -m "test unsigned tag" test-unsigned-tag HEAD
git -C "$REPO_DIR" push --dry-run origin refs/tags/test-unsigned-tag || true
git -C "$REPO_DIR" tag -d test-unsigned-tag

   Verification: The dry-run push exits non-zero and prints PRE-PUSH BLOCKED [refs/tags/test-unsigned-tag]: ... unsigned or unverified annotated tag, then the temporary local tag is deleted cleanly.
 2. Test with a signed annotated tag
   1 min
   Create a signed annotated tag (git tag -s) and run git push --dry-run to confirm it passes.
   git -C "$REPO_DIR" tag -s -m "test signed tag" test-signed-tag HEAD
git -C "$REPO_DIR" push --dry-run origin refs/tags/test-signed-tag
git -C "$REPO_DIR" tag -d test-signed-tag

   Verification: git push --dry-run exits 0 with no PRE-PUSH BLOCKED output.
> Note: Since your hook is symlinked into .git/hooks/pre-push across your repositories under ${HOME}/src/github, updating ~/.config/git/hooks/pre-push automatically applies the annotated tag check to all linked repos without re-linking.
> 
Here is the updated ~/bin/audit-gpg-signatures.sh that audits both commits and annotated tags across your repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep).
Lightweight tags (objecttype = commit or tree) do not carry tag-object signatures of their own, so the script inspects refs/tags/* objects of type tag using git verify-tag --raw, checks for GOODSIG / VALIDSIG status and optional ALLOWED_KEY_IDS allowlisting, and counts any unsigned, bad, expired, revoked, or missing-key tags toward Violations.
1. Updated ~/bin/audit-gpg-signatures.sh (Commits + Annotated Tags)
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
REPORT_FILE="${2:-${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log}"
mkdir -p "$(dirname "$REPORT_FILE")"

# Optional: populate with one or more 16-char or full GPG key IDs to enforce allowlisting
# Example: ALLOWED_KEY_IDS=("0123456789ABCDEF")
ALLOWED_KEY_IDS=()

total_repos=0
total_commits=0
total_tags=0
failed_items=0
SEP=$'\x1f'

is_key_allowed() {
  local key_id="${1:-}"
  if [ "${#ALLOWED_KEY_IDS[@]}" -eq 0 ]; then
    return 0
  fi

  local matched=false
  for allowed in "${ALLOWED_KEY_IDS[@]}"; do
    [ -n "$allowed" ] || continue
    if [ -n "$key_id" ] && [[ "$key_id" == *"$allowed" ]]; then
      matched=true
      break
    fi
  done

  [ "$matched" = true ]
}

verify_annotated_tag_item() {
  local repo_dir="$1"
  local repo_name="$2"
  local tag_name="$3"
  local tag_sha="$4"

  total_tags=$((total_tags + 1))

  local raw_out rc=0
  raw_out="$(git -C "$repo_dir" verify-tag --raw "$tag_sha" 2>&1)" || rc=$?

  local key_id=""
  key_id="$(
    printf '%s\n' "$raw_out" \
      | awk '/\[GNUPG:\] (VALIDSIG|GOODSIG)/ {
          if ($2 == "VALIDSIG") print $11;
          else if ($2 == "GOODSIG") print $3;
        }' | head -n 1
  )"

  local reason=""
  if [ "$rc" -ne 0 ]; then
    if printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] BADSIG'; then
      reason="BAD tag signature (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] REVKEYSIG'; then
      reason="Revoked key on tag (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPKEYSIG'; then
      reason="Expired key on tag (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] EXPSIG'; then
      reason="Expired tag signature (${key_id:-unknown})"
    elif printf '%s\n' "$raw_out" | grep -q '\[GNUPG:\] ERRSIG'; then
      reason="Missing public key on tag (${key_id:-unknown})"
    else
      reason="Unsigned annotated tag"
    fi
  elif ! printf '%s\n' "$raw_out" | grep -qE '^\[GNUPG:\] (GOODSIG|VALIDSIG)'; then
    reason="Unsigned annotated tag"
  elif ! is_key_allowed "$key_id"; then
    reason="Unauthorized tag key (${key_id:-unknown})"
  fi

  if [ -n "$reason" ]; then
    failed_items=$((failed_items + 1))
    repo_failures=$((repo_failures + 1))
    printf "[FAIL] %-35s | tag:%-18s | %-34s | %s\n" \
      "$repo_name" "$tag_name" "$reason" "${tag_sha:0:10}" | tee -a "$REPORT_FILE"
  fi
}

{
  echo "GPG Signature Audit (Commits & Annotated Tags) - $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  echo "Scanning root: ${SRC_ROOT}"
  echo "------------------------------------------------------------"
} | tee "$REPORT_FILE"

if [ ! -d "$SRC_ROOT" ]; then
  echo "ERROR: Source directory does not exist: ${SRC_ROOT}" >&2
  exit 1
fi

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  repo_name="${repo_dir#${SRC_ROOT}/}"
  total_repos=$((total_repos + 1))

  if ! git -C "$repo_dir" rev-parse --verify HEAD >/dev/null 2>&1; then
    echo "[SKIP] ${repo_name} (empty repository)" | tee -a "$REPORT_FILE"
    continue
  fi

  repo_failures=0

  # 1. Audit commits across all refs
  while IFS="$SEP" read -r hash status key_id email subject; do
    total_commits=$((total_commits + 1))
    is_valid=true
    reason=""

    case "$status" in
      G|U)
        if ! is_key_allowed "$key_id"; then
          is_valid=false
          reason="Unauthorized key (${key_id})"
        fi
        ;;
      N) is_valid=false; reason="Unsigned commit" ;;
      B) is_valid=false; reason="BAD signature" ;;
      X) is_valid=false; reason="Signature expired" ;;
      Y) is_valid=false; reason="Expired key (${key_id})" ;;
      R) is_valid=false; reason="Revoked key (${key_id})" ;;
      E) is_valid=false; reason="Missing public key (${key_id})" ;;
      *) is_valid=false; reason="Unknown status (${status})" ;;
    esac

    if [ "$is_valid" = false ]; then
      failed_items=$((failed_items + 1))
      repo_failures=$((repo_failures + 1))
      printf "[FAIL] %-35s | %-22s | %-34s | %s (%s)\n" \
        "$repo_name" "${hash:0:10}" "$reason" "$subject" "$email" | tee -a "$REPORT_FILE"
    fi
  done < <(git -C "$repo_dir" log --all --format="%H%x1f%G?%x1f%GK%x1f%ae%x1f%s")

  # 2. Audit annotated tags (objecttype == tag)
  while IFS=$'\t' read -r tag_name tag_sha obj_type; do
    [ -n "$tag_name" ] || continue
    if [ "$obj_type" = "tag" ]; then
      verify_annotated_tag_item "$repo_dir" "$repo_name" "$tag_name" "$tag_sha"
    fi
  done < <(git -C "$repo_dir" for-each-ref --format="%(refname:short)%09%(objectname)%09%(objecttype)" refs/tags)

  if [ "$repo_failures" -eq 0 ]; then
    echo "[PASS] ${repo_name}" | tee -a "$REPORT_FILE"
  fi
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

{
  echo "------------------------------------------------------------"
  echo "Repos audited: ${total_repos} | Commits checked: ${total_commits} | Annotated tags checked: ${total_tags} | Violations: ${failed_items}"
  echo "Full log saved to: ${REPORT_FILE}"
} | tee -a "$REPORT_FILE"

if [ "$failed_items" -gt 0 ]; then
  exit 1
fi

2. Install & Verify
 1. Save and run the updated audit script
   1 min
   Write the script to ~/bin/audit-gpg-signatures.sh, mark it executable, and run it against ${HOME}/src/github.
   chmod +x ~/bin/audit-gpg-signatures.sh
~/bin/audit-gpg-signatures.sh "${HOME}/src/github"

   Verification: The summary line displays Commits checked, Annotated tags checked, and Violations: 0, exiting with status 0.
 2. Check the log output for any tag or commit failures
   1 min
   Inspect the generated log file in ${HOME}/github-archives/ to verify that both commits and annotated tags are covered.
   LATEST_LOG="$(ls -t "${HOME}/github-archives"/gpg-audit-*.log | head -n 1)"
tail -n 10 "$LATEST_LOG"

   Verification: The output shows Annotated tags checked and Violations: 0 with no [FAIL] entries.
0123456789user.signingkeyHere is a script (~/bin/resign-annotated-tags.sh) that scans repositories under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep), identifies annotated tags (objecttype == tag) that lack a valid signature (git verify-tag --raw), backs up a compressed bare mirror first, re-signs each unsigned/invalid annotated tag onto its original target object while preserving its tagger message and date, and optionally force-pushes updated tag refs with --force-with-lease --force-if-includes.
Lightweight tags (objecttype = commit or tree) are left untouched because they are plain pointers rather than tag objects with OpenPGP signatures.
1. Batch Annotated Tag Re-Signer (~/bin/resign-annotated-tags.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub accounts (default: "re1Lucky365 adpsnake7-beep")
  --src-root PATH            Local working clones root (default: "${HOME}/src/github")
  --key-id ID                GPG signing key ID (default: git config user.signingkey)
  --repos "a b"              Optional space-separated repo names to restrict to
  --push                     Force-with-lease push updated tags to origin
  --dry-run                  Audit and list unsigned/invalid annotated tags without modifying anything
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
SRC_ROOT="${HOME}/src/github"
KEY_ID="$(git config --get user.signingkey || true)"
FILTER_REPOS=""
DO_PUSH=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --key-id)
      KEY_ID="${2:?Missing value for --key-id}"
      shift 2
      ;;
    --repos)
      FILTER_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --push)
      DO_PUSH=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v git >/dev/null 2>&1 || { echo "git is required" >&2; exit 1; }
command -v gpg >/dev/null 2>&1 || { echo "gpg is required" >&2; exit 1; }

if [ -z "$KEY_ID" ]; then
  echo "ERROR: No GPG signing key specified (--key-id or git config user.signingkey)." >&2
  exit 1
fi

if ! gpg --list-secret-keys --keyid-format=long "$KEY_ID" >/dev/null 2>&1; then
  echo "ERROR: Secret key not found in local GPG keyring: $KEY_ID" >&2
  exit 1
fi

export GPG_TTY="${GPG_TTY:-$(tty 2>/dev/null || true)}"

STAMP="$(date +%Y-%m-%d_%H%M%S)"
BACKUP_ROOT="${HOME}/github-archives/pre-tag-resign-${STAMP}"
WORK_TMP="$(mktemp -d)"
trap 'rm -rf "$WORK_TMP"' EXIT

repo_matches_filter() {
  local name="$1"
  [ -z "$FILTER_REPOS" ] && return 0
  for r in $FILTER_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

tag_needs_signing() {
  local repo_dir="$1"
  local tag_sha="$2"
  local raw_out rc=0

  raw_out="$(git -C "$repo_dir" verify-tag --raw "$tag_sha" 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    return 0
  fi
  if ! printf '%s\n' "$raw_out" | grep -qE '^\[GNUPG:\] (GOODSIG|VALIDSIG)'; then
    return 0
  fi
  return 1
}

repos_scanned=0
repos_updated=0
tags_resigned=0
failed_repos=0

echo "Signing key ID  : ${KEY_ID}"
echo "Push to origin  : ${DO_PUSH}"
echo "Dry-run mode    : ${DRY_RUN}"
echo "------------------------------------------------------------"

for acct in "${ACCOUNTS[@]}"; do
  acct_src="${SRC_ROOT}/${acct}"
  acct_bak="${BACKUP_ROOT}/${acct}"

  if [ ! -d "$acct_src" ]; then
    echo "[SKIP] Account directory missing: ${acct_src}"
    continue
  fi

  for repo_dir in "${acct_src}"/*; do
    [ -d "${repo_dir}/.git" ] || continue
    repo_name="$(basename "$repo_dir")"
    repo_matches_filter "$repo_name" || continue

    repos_scanned=$((repos_scanned + 1))

    target_tags=()
    while IFS=$'\t' read -r tag_name tag_sha obj_type; do
      [ -n "$tag_name" ] || continue
      if [ "$obj_type" = "tag" ] && tag_needs_signing "$repo_dir" "$tag_sha"; then
        target_tags+=("${tag_name}:${tag_sha}")
      fi
    done < <(git -C "$repo_dir" for-each-ref --format="%(refname:short)%09%(objectname)%09%(objecttype)" refs/tags)

    if [ "${#target_tags[@]}" -eq 0 ]; then
      echo "[PASS] ${acct}/${repo_name} (no unsigned/invalid annotated tags)"
      continue
    fi

    echo "[TARGET] ${acct}/${repo_name}: ${#target_tags[@]} annotated tag(s) to re-sign"
    for entry in "${target_tags[@]}"; do
      echo "         - ${entry%%:*} (${entry##*:})"
    done

    if [ "$DRY_RUN" = true ]; then
      tags_resigned=$((tags_resigned + ${#target_tags[@]}))
      continue
    fi

    # 1. Back up repository mirror before rewriting tag objects
    mkdir -p "$acct_bak"
    mirror_dir="${WORK_TMP}/mirror_${acct}_${repo_name}.git"
    git clone --mirror "$repo_dir" "$mirror_dir"
    tar -cf - -C "$(dirname "$mirror_dir")" "$(basename "$mirror_dir")" \
      | zstd -T0 -19 -o "${acct_bak}/${repo_name}.tar.zst"

    # 2. Re-sign each unsigned/invalid annotated tag while preserving message & tagger date
    repo_fail=false
    resigned_names=()

    for entry in "${target_tags[@]}"; do
      tag_name="${entry%%:*}"
      tag_target="$(git -C "$repo_dir" rev-parse "${tag_name}^{}")"
      tag_msg="$(git -C "$repo_dir" for-each-ref --format="%(contents)" "refs/tags/${tag_name}")"
      tag_date="$(git -C "$repo_dir" for-each-ref --format="%(taggerdate:iso8601)" "refs/tags/${tag_name}")"

      # Strip trailing single newline added by %(contents) if original message didn't end with one
      if ! GIT_COMMITTER_DATE="$tag_date" git -C "$repo_dir" \
        tag -f -s -u "$KEY_ID" -m "$tag_msg" "$tag_name" "$tag_target"; then
        echo "[FAIL] ${acct}/${repo_name}: failed re-signing tag ${tag_name}" >&2
        repo_fail=true
        break
      fi

      # Verify tag signature immediately after signing
      if tag_needs_signing "$repo_dir" "refs/tags/${tag_name}"; then
        echo "[FAIL] ${acct}/${repo_name}: tag ${tag_name} still failed verification after signing" >&2
        repo_fail=true
        break
      fi

      resigned_names+=("$tag_name")
      tags_resigned=$((tags_resigned + 1))
    done

    if [ "$repo_fail" = true ]; then
      failed_repos=$((failed_repos + 1))
      continue
    fi

    # 3. Optionally push updated tag refs with --force-with-lease --force-if-includes
    if [ "$DO_PUSH" = true ]; then
      push_args=(push --force-with-lease --force-if-includes origin)
      for tname in "${resigned_names[@]}"; do
        push_args+=("refs/tags/${tname}:refs/tags/${tname}")
      done
      if ! git -C "$repo_dir" "${push_args[@]}"; then
        echo "[FAIL] ${acct}/${repo_name}: failed pushing re-signed tags to origin" >&2
        failed_repos=$((failed_repos + 1))
        continue
      fi
    fi

    echo "[OK] ${acct}/${repo_name}: re-signed ${#resigned_names[@]} tag(s)"
    repos_updated=$((repos_updated + 1))
  done
done

echo "------------------------------------------------------------"
echo "Summary: ReposScanned=${repos_scanned} | ReposUpdated=${repos_updated} | TagsResigned=${tags_resigned} | FailedRepos=${failed_repos}"
if [ -d "$BACKUP_ROOT" ]; then
  echo "Pre-tag-resign backups saved in: ${BACKUP_ROOT}"
fi

if [ "$failed_repos" -gt 0 ]; then
  exit 1
fi

2. Install, Preview & Run
 1. Save, make executable, and run a dry-run preview
   1 min
   Write the script to ~/bin/resign-annotated-tags.sh, make it executable, and run --dry-run first to list every annotated tag that needs re-signing across re1Lucky365 and adpsnake7-beep.
   chmod +x ~/bin/resign-annotated-tags.sh
~/bin/resign-annotated-tags.sh --dry-run

   Verification: Shows [TARGET] lines for repositories with unsigned/invalid annotated tags and [PASS] for clean ones without modifying any Git objects or remotes.
 2. Re-sign locally and push with --force-with-lease
   1-3 min
   Run the script with --push (or run without --push first if you want to inspect git verify-tag locally before pushing).
   ~/bin/resign-annotated-tags.sh --push

   Verification: Outputs [OK] for updated repositories, saves .tar.zst pre-rewrite mirrors under ~/github-archives/pre-tag-resign-<timestamp>/, and exits with status 0.
 3. Confirm zero tag or commit violations in your audit script
   1 min
   Run your updated ~/bin/audit-gpg-signatures.sh to verify that both commits and annotated tags pass with zero violations.
   ~/bin/audit-gpg-signatures.sh "${HOME}/src/github"

   Verification: Reports Violations: 0 and exits with status 0.
> Important notes:
>  * Existing releases & downstream consumers: Re-signing an annotated tag replaces the tag object's SHA on the remote when pushed with --force-with-lease. Anyone who previously fetched or pinned that tag SHA will see an updated tag object.
>  * Rollback: If you ever need to restore a repository's pre-run tags and refs, extract its .tar.zst mirror from ~/github-archives/pre-tag-resign-<timestamp>/ and fetch/push refs back as needed.
> 
resign-annotated-tags.shbackup-github.shTo add automated SHA-256 checksum verification and rclone check validation directly into ~/bin/run-github-suite.sh, wire a dedicated verify_snapshot_integrity() function right after your backup/sync steps so a corrupted archive or remote mismatch fails the suite run loudly instead of going unnoticed.
1. Drop-In Verification Function & run-github-suite.sh Integration
Set RCLONE_REMOTE_BASE (for example crypt-github:github-archives) and SNAP_DIR to the snapshot directory created during that run. If your SHA256SUMS file uses relative paths inside the snapshot folder, run sha256sum -c inside SNAP_DIR; if it contains paths relative to ARCHIVE_ROOT, run sha256sum -c from ARCHIVE_ROOT.
verify_snapshot_integrity() {
  local snap_dir="$1"
  local archive_root="$2"
  local rclone_remote_base="${3:-}"

  if [ ! -d "$snap_dir" ]; then
    echo "[ERROR] Snapshot directory not found: ${snap_dir}" >&2
    return 1
  fi

  local sums_file="${snap_dir}/SHA256SUMS"
  if [ ! -f "$sums_file" ]; then
    echo "[ERROR] SHA256SUMS manifest missing in: ${snap_dir}" >&2
    return 1
  fi

  echo "[VERIFY] Checking local SHA-256 manifest (${sums_file})..."
  if (cd "$snap_dir" && sha256sum --status -c "$sums_file"); then
    echo "[PASS] Local SHA-256 checksums verified for $(basename "$snap_dir")"
  elif (cd "$archive_root" && sha256sum --status -c "$sums_file"); then
    echo "[PASS] Local SHA-256 checksums verified from ${archive_root}"
  else
    echo "[FAIL] SHA-256 checksum mismatch in ${snap_dir}" >&2
    return 1
  fi

  if [ -n "$rclone_remote_base" ]; then
    command -v rclone >/dev/null 2>&1 || {
      echo "[ERROR] rclone is required when RCLONE_REMOTE_BASE is set" >&2
      return 1
    }

    local snap_name
    snap_name="$(basename "$snap_dir")"
    local remote_target="${rclone_remote_base%/}/${snap_name}"

    echo "[VERIFY] Running rclone check against ${remote_target}..."
    if rclone check "$snap_dir" "$remote_target" --one-way --checkers=8; then
      echo "[PASS] Remote vault matches local snapshot (${snap_name})"
    else
      echo "[FAIL] rclone check detected differences between ${snap_dir} and ${remote_target}" >&2
      return 1
    fi
  else
    echo "[SKIP] RCLONE_REMOTE_BASE not configured; skipped remote rclone check"
  fi
}

2. Example ~/bin/run-github-suite.sh Flow
Here is how the verification step fits cleanly after backup, metadata export, GPG audit, and rclone sync:
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${SRC_ROOT:-${HOME}/src/github}"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
RCLONE_REMOTE_BASE="${RCLONE_REMOTE_BASE:-}" # e.g., "crypt-github:github-archives"

"${HOME}/bin/backup-github.sh"
"${HOME}/bin/export-github-metadata.sh"
"${HOME}/bin/audit-gpg-signatures.sh" "$SRC_ROOT"

LATEST_SNAP="$(
  find "$ARCHIVE_ROOT" -mindepth 1 -maxdepth 1 -type d \
    ! -name 'pre-resign-*' ! -name 'pre-tag-resign-*' \
    | sort | tail -n 1
)"

if [ -n "$RCLONE_REMOTE_BASE" ] && [ -n "$LATEST_SNAP" ]; then
  SNAP_NAME="$(basename "$LATEST_SNAP")"
  rclone copy "$LATEST_SNAP" "${RCLONE_REMOTE_BASE%/}/${SNAP_NAME}" \
    --transfers=4 --checkers=8 --fast-list
fi

verify_snapshot_integrity "$LATEST_SNAP" "$ARCHIVE_ROOT" "$RCLONE_REMOTE_BASE"

3. Rollout & Verification Steps
 1. Add verify_snapshot_integrity to run-github-suite.sh
   1 min
   Paste verify_snapshot_integrity into ~/bin/run-github-suite.sh, set RCLONE_REMOTE_BASE in your environment or script config, and ensure ~/bin/run-github-suite.sh is executable (chmod +x ~/bin/run-github-suite.sh).
   Verification: Running bash -n ~/bin/run-github-suite.sh exits 0 with no syntax errors.
 2. Run the suite with RCLONE_REMOTE_BASE set
   1-3 min
   Execute ~/bin/run-github-suite.sh with your remote target configured so both local checksums and remote vault objects are verified in one pass.
   RCLONE_REMOTE_BASE="YOUR_RCLONE_REMOTE:github-archives" ~/bin/run-github-suite.sh

   Verification: Output prints [PASS] Local SHA-256 checksums verified... and [PASS] Remote vault matches local snapshot... and exits with status 0.
 3. Confirm failure behavior on a mismatched test file
   1 min
   Temporarily test on a non-production copy or inspect exit status 1 behavior when a file fails sha256sum -c or rclone check so systemd alerts/logs catch integrity failures immediately.
   Verification: Any checksum or remote mismatch exits 1 before the run is treated as healthy.
> Crypt remotes & rclone check note: On crypt remotes, rclone check uses size and modification time/hash checks supported by the backend (--one-way verifies every file in snap_dir exists and matches on the remote without complaining about extra remote files).
> 
To trigger an alert whenever run-github-suite.service exits non-zero (or fails), add OnFailure=run-github-suite-failure@%i.service (or a non-template OnFailure=run-github-suite-alert.service) to the [Unit] section of your systemd user service and define a companion oneshot unit under ~/.config/systemd/user/.
Because run-github-suite.service runs in your user systemd manager (systemctl --user), the OnFailure= target unit must also live in ~/.config/systemd/user/ so the user manager can resolve and start it.
1. Add OnFailure= to run-github-suite.service
Edit ~/.config/systemd/user/run-github-suite.service and include OnFailure= under [Unit] (using a drop-in override file is even cleaner so you don't overwrite your base unit):
mkdir -p "${HOME}/.config/systemd/user/run-github-suite.service.d"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite.service.d/on-failure.conf"
[Unit]
OnFailure=run-github-suite-alert.service
EOF

(If you run a template service like run-github-suite@.service, use OnFailure=run-github-suite-alert@%i.service in that unit instead so %i passes the instance name.)
2. Create the Alert Handler Script & OnFailure User Unit
Create a local alert script (~/bin/notify-github-suite-failure.sh) that captures recent journalctl --user lines for run-github-suite.service and dispatches whatever alert mechanism you prefer (for example notify-send for desktop notifications, a webhook/email/script, or appending to a dedicated failure log).
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
ALERT_LOG="${HOME}/github-archives/systemd-failures.log"
mkdir -p "$(dirname "$ALERT_LOG")"

RECENT_LOGS="$(
  journalctl --user -u "$UNIT_NAME" -n 25 --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

SUMMARY="FAILURE in ${UNIT_NAME} on ${HOST_NAME} at $(date -u +"%Y-%m-%dT%H:%M:%SZ")"

{
  echo "============================================================"
  echo "$SUMMARY"
  echo "------------------------------------------------------------"
  printf '%s\n' "$RECENT_LOGS"
  echo ""
} >> "$ALERT_LOG"

# Desktop notification if running inside a graphical session
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" "${UNIT_NAME} exited non-zero on ${HOST_NAME}. See ${ALERT_LOG}" || true
fi

# Optional webhook example (uncomment and set WEBHOOK_URL if desired):
# if [ -n "${WEBHOOK_URL:-}" ] && command -v curl >/dev/null 2>&1; then
#   curl -fsS -X POST -H "Content-Type: application/json" \
#     -d "$(jq -n --arg text "$SUMMARY" --arg logs "$RECENT_LOGS" '{text: ($text + "\n```\n" + $logs + "\n```")}')" \
#     "$WEBHOOK_URL" || true
# fi
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

Now create ~/.config/systemd/user/run-github-suite-alert.service:
cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.service"
[Unit]
Description=Alert when run-github-suite.service exits non-zero
After=run-github-suite.service

[Service]
Type=oneshot
ExecStart=%h/bin/notify-github-suite-failure.sh run-github-suite.service
EOF

3. Reload Systemd & Test Non-Zero Exit Handling
 1. Reload the systemd user daemon
   1 min
   Reload user unit definitions so run-github-suite.service picks up the OnFailure= drop-in and run-github-suite-alert.service is registered.
   systemctl --user daemon-reload
systemctl --user cat run-github-suite.service | grep -n "OnFailure"

   Verification: grep prints OnFailure=run-github-suite-alert.service.
 2. Verify failure trigger behavior
   1 min
   Check systemctl --user status run-github-suite-alert.service after a failed run (or test your alert script directly with ~/bin/notify-github-suite-failure.sh run-github-suite.service).
   ~/bin/notify-github-suite-failure.sh run-github-suite.service
tail -n 20 "${HOME}/github-archives/systemd-failures.log"
systemctl --user status run-github-suite-alert.service --no-pager || true

   Verification: ${HOME}/github-archives/systemd-failures.log records the timestamped failure block and recent journal lines for run-github-suite.service.
 3. Ensure user services run even when logged out (if needed)
   1 min
   If run-github-suite.timer runs unattended while you are logged out of an interactive session, verify lingering is enabled for your user account (loginctl show-user "$USER" -p Linger).
   loginctl show-user "$USER" -p Linger

   Verification: Shows Linger=yes if background user units are enabled across logouts (enable with sudo loginctl enable-linger "$USER" if needed).
Here is how to update ~/bin/notify-github-suite-failure.sh so a non-zero exit from run-github-suite.service sends an HTTP webhook (JSON payload with bounded log context) and/or a local mail alert (via mail / sendmail / msmtp), while still recording the full failure block to ${HOME}/github-archives/systemd-failures.log.
Because systemd user services often run without an interactive shell environment, store sensitive secrets (like a webhook URL or token) in a restricted environment file (for example ~/.config/systemd/user/run-github-suite-alert.env, chmod 600) and load it via EnvironmentFile= in run-github-suite-alert.service.
1. Store Alert Config in a Restricted Environment File
Create ~/.config/systemd/user/run-github-suite-alert.env with mode 0600 so webhook URLs or recipient addresses aren't world-readable:
mkdir -p "${HOME}/.config/systemd/user"
chmod 700 "${HOME}/.config/systemd/user"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.env"
# Optional: HTTP webhook endpoint (receives JSON payload)
GITHUB_SUITE_WEBHOOK_URL=""
GITHUB_SUITE_WEBHOOK_BEARER_TOKEN=""

# Optional: Local mail recipient (e.g., "robert@localhost" or local system user)
GITHUB_SUITE_MAIL_TO=""
GITHUB_SUITE_MAIL_FROM=""

# Max lines of journal/audit context to attach (keeps payloads/emails manageable)
GITHUB_SUITE_LOG_LINES="120"
EOF

chmod 600 "${HOME}/.config/systemd/user/run-github-suite-alert.env"

Wire that file into ~/.config/systemd/user/run-github-suite-alert.service:
cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.service"
[Unit]
Description=Alert when run-github-suite.service exits non-zero
After=run-github-suite.service

[Service]
Type=oneshot
EnvironmentFile=%h/.config/systemd/user/run-github-suite-alert.env
ExecStart=%h/bin/notify-github-suite-failure.sh run-github-suite.service
EOF

systemctl --user daemon-reload

2. Updated ~/bin/notify-github-suite-failure.sh (Webhook + Local Mail + Log Context)
This version pulls:
 * Recent journalctl --user entries for UNIT_NAME (including invocation/exit status when available).
 * The tail of the newest gpg-audit-*.log file in ${HOME}/github-archives/ (if present) for extra diagnostic context.
 * Sends a JSON POST to GITHUB_SUITE_WEBHOOK_URL (if set) and/or sends a plain-text message via mail or sendmail to GITHUB_SUITE_MAIL_TO (if set).
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
ALERT_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_LINES="${GITHUB_SUITE_LOG_LINES:-120}"

mkdir -p "$ARCHIVE_ROOT" "$(dirname "$ALERT_LOG")"

EXIT_STATUS="$(
  systemctl --user show "$UNIT_NAME" -p ExecMainStatus --value 2>/dev/null || echo "unknown"
)"
RESULT_STATE="$(
  systemctl --user show "$UNIT_NAME" -p Result --value 2>/dev/null || echo "unknown"
)"

JOURNAL_CONTEXT="$(
  journalctl --user -u "$UNIT_NAME" -n "$MAX_LINES" --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

LATEST_AUDIT_LOG="$(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f -name 'gpg-audit-*.log' 2>/dev/null \
    | sort | tail -n 1 || true
)"

AUDIT_CONTEXT=""
if [ -n "${LATEST_AUDIT_LOG:-}" ] && [ -f "$LATEST_AUDIT_LOG" ]; then
  AUDIT_CONTEXT="$(tail -n 40 "$LATEST_AUDIT_LOG" 2>/dev/null || true)"
fi

FULL_REPORT="$(cat <<REPORT
============================================================
ALERT: ${UNIT_NAME} failed on ${HOST_NAME}
Timestamp (UTC) : ${TIMESTAMP_UTC}
Result / Status : ${RESULT_STATE} (exit=${EXIT_STATUS})
------------------------------------------------------------
Recent Journal Logs (last ${MAX_LINES} lines):
${JOURNAL_CONTEXT}
REPORT
)"

if [ -n "$AUDIT_CONTEXT" ]; then
  FULL_REPORT+="$(printf '\n------------------------------------------------------------\nLatest GPG Audit Tail (%s):\n%s\n' "$(basename "$LATEST_AUDIT_LOG")" "$AUDIT_CONTEXT")"
fi

printf '%s\n\n' "$FULL_REPORT" >> "$ALERT_LOG"

# Desktop notification if graphical session is active
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" \
    "${UNIT_NAME} (${RESULT_STATE}, exit=${EXIT_STATUS}) on ${HOST_NAME}" || true
fi

# HTTP Webhook delivery
if [ -n "${GITHUB_SUITE_WEBHOOK_URL:-}" ]; then
  if command -v curl >/dev/null 2>&1; then
    if command -v jq >/dev/null 2>&1; then
      PAYLOAD="$(
        jq -n \
          --arg unit "$UNIT_NAME" \
          --arg host "$HOST_NAME" \
          --arg ts "$TIMESTAMP_UTC" \
          --arg result "$RESULT_STATE" \
          --arg exit_code "$EXIT_STATUS" \
          --arg log_file "$ALERT_LOG" \
          --arg context "$FULL_REPORT" \
          '{
            text: ("*[" + $host + "]* `" + $unit + "` failed (result=" + $result + ", exit=" + $exit_code + ") at " + $ts + "\n```\n" + $context + "\n```"),
            unit: $unit,
            host: $host,
            timestamp_utc: $ts,
            result: $result,
            exit_status: $exit_code,
            alert_log: $log_file,
            logs: $context
          }'
      )"
    else
      ESCAPED_REPORT="$(printf '%s' "$FULL_REPORT" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')"
      PAYLOAD="{\"unit\":\"${UNIT_NAME}\",\"host\":\"${HOST_NAME}\",\"timestamp_utc\":\"${TIMESTAMP_UTC}\",\"exit_status\":\"${EXIT_STATUS}\",\"logs\":${ESCAPED_REPORT}}"
    fi

    curl_opts=(-fsS -X POST -H "Content-Type: application/json" -d "$PAYLOAD")
    if [ -n "${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN:-}" ]; then
      curl_opts+=(-H "Authorization: Bearer ${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN}")
    fi

    if ! curl "${curl_opts[@]}" "$GITHUB_SUITE_WEBHOOK_URL" >/dev/null; then
      echo "[WARN] Failed delivering webhook alert to ${GITHUB_SUITE_WEBHOOK_URL}" >> "$ALERT_LOG"
    fi
  else
    echo "[WARN] curl not installed; cannot send webhook alert" >> "$ALERT_LOG"
  fi
fi

# Local mail delivery (supports `mail` / `mailx` or `sendmail`)
if [ -n "${GITHUB_SUITE_MAIL_TO:-}" ]; then
  SUBJECT="[ALERT] ${UNIT_NAME} failed on ${HOST_NAME} (exit=${EXIT_STATUS})"
  if command -v mail >/dev/null 2>&1; then
    mail_args=(-s "$SUBJECT")
    if [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ]; then
      mail_args+=(-r "$GITHUB_SUITE_MAIL_FROM")
    fi
    if ! printf '%s\n' "$FULL_REPORT" | mail "${mail_args[@]}" "$GITHUB_SUITE_MAIL_TO"; then
      echo "[WARN] Failed sending mail alert via mail(1) to ${GITHUB_SUITE_MAIL_TO}" >> "$ALERT_LOG"
    fi
  elif command -v sendmail >/dev/null 2>&1; then
    {
      [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ] && echo "From: ${GITHUB_SUITE_MAIL_FROM}"
      echo "To: ${GITHUB_SUITE_MAIL_TO}"
      echo "Subject: ${SUBJECT}"
      echo "Content-Type: text/plain; charset=UTF-8"
      echo ""
      printf '%s\n' "$FULL_REPORT"
    } | sendmail -t || echo "[WARN] Failed sending mail alert via sendmail to ${GITHUB_SUITE_MAIL_TO}" >> "$ALERT_LOG"
  else
    echo "[WARN] Neither mail nor sendmail is installed; cannot send mail alert" >> "$ALERT_LOG"
  fi
fi

exit 0
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

3. Verify Webhook & Local Mail Delivery
 1. Test the alert script directly with your configured env vars
   1 min
   Load ~/.config/systemd/user/run-github-suite-alert.env and invoke the script once to confirm webhook and/or mail delivery without waiting for a real failure.
   set -a
source "${HOME}/.config/systemd/user/run-github-suite-alert.env"
set +a
"${HOME}/bin/notify-github-suite-failure.sh" run-github-suite.service

   Verification: Your webhook endpoint receives the JSON payload, your local mail spool/inbox gets the alert message, and ${HOME}/github-archives/systemd-failures.log has no [WARN] lines.
 2. Check local mail queue if using local MTA
   1 min
   If you configured a local mail address (GITHUB_SUITE_MAIL_TO), check local mail delivery (mail or postqueue -p / journal logs for your MTA).
   tail -n 15 "${HOME}/github-archives/systemd-failures.log"

   Verification: Confirms the alert block was written cleanly and no transport warnings were logged.
> Payload & security note: Because FULL_REPORT includes recent journal lines and GPG audit output, avoid sending full raw logs over unencrypted HTTP or to shared webhook channels unless your environment and destination are trusted.
> 
Update notify-github-suite-failure.sh to redact URLs, tokens, or paths before sending webhook and mail alerts.Update notify-github-suite-failure.sh to redact URLs, tokens, or paths before sending webhook and mail alerts.Here is how to update ~/bin/notify-github-suite-failure.sh so a non-zero exit from run-github-suite.service sends an HTTP webhook (JSON payload with bounded log context) and/or a local mail alert (via mail / sendmail / msmtp), while still recording the full failure block to ${HOME}/github-archives/systemd-failures.log.
Because systemd user services often run without an interactive shell environment, store sensitive secrets (like a webhook URL or token) in a restricted environment file (for example ~/.config/systemd/user/run-github-suite-alert.env, chmod 600) and load it via EnvironmentFile= in run-github-suite-alert.service.
1. Store Alert Config in a Restricted Environment File
Create ~/.config/systemd/user/run-github-suite-alert.env with mode 0600 so webhook URLs or recipient addresses aren't world-readable:
mkdir -p "${HOME}/.config/systemd/user"
chmod 700 "${HOME}/.config/systemd/user"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.env"
# Optional: HTTP webhook endpoint (receives JSON payload)
GITHUB_SUITE_WEBHOOK_URL=""
GITHUB_SUITE_WEBHOOK_BEARER_TOKEN=""

# Optional: Local mail recipient (e.g., "robert@localhost" or local system user)
GITHUB_SUITE_MAIL_TO=""
GITHUB_SUITE_MAIL_FROM=""

# Max lines of journal/audit context to attach (keeps payloads/emails manageable)
GITHUB_SUITE_LOG_LINES="120"
EOF

chmod 600 "${HOME}/.config/systemd/user/run-github-suite-alert.env"

Wire that file into ~/.config/systemd/user/run-github-suite-alert.service:
cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite-alert.service"
[Unit]
Description=Alert when run-github-suite.service exits non-zero
After=run-github-suite.service

[Service]
Type=oneshot
EnvironmentFile=%h/.config/systemd/user/run-github-suite-alert.env
ExecStart=%h/bin/notify-github-suite-failure.sh run-github-suite.service
EOF

systemctl --user daemon-reload

2. Updated ~/bin/notify-github-suite-failure.sh (Webhook + Local Mail + Log Context)
This version pulls:
 * Recent journalctl --user entries for UNIT_NAME (including invocation/exit status when available).
 * The tail of the newest gpg-audit-*.log file in ${HOME}/github-archives/ (if present) for extra diagnostic context.
 * Sends a JSON POST to GITHUB_SUITE_WEBHOOK_URL (if set) and/or sends a plain-text message via mail or sendmail to GITHUB_SUITE_MAIL_TO (if set).
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
ALERT_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_LINES="${GITHUB_SUITE_LOG_LINES:-120}"

mkdir -p "$ARCHIVE_ROOT" "$(dirname "$ALERT_LOG")"

EXIT_STATUS="$(
  systemctl --user show "$UNIT_NAME" -p ExecMainStatus --value 2>/dev/null || echo "unknown"
)"
RESULT_STATE="$(
  systemctl --user show "$UNIT_NAME" -p Result --value 2>/dev/null || echo "unknown"
)"

JOURNAL_CONTEXT="$(
  journalctl --user -u "$UNIT_NAME" -n "$MAX_LINES" --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

LATEST_AUDIT_LOG="$(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f -name 'gpg-audit-*.log' 2>/dev/null \
    | sort | tail -n 1 || true
)"

AUDIT_CONTEXT=""
if [ -n "${LATEST_AUDIT_LOG:-}" ] && [ -f "$LATEST_AUDIT_LOG" ]; then
  AUDIT_CONTEXT="$(tail -n 40 "$LATEST_AUDIT_LOG" 2>/dev/null || true)"
fi

FULL_REPORT="$(cat <<REPORT
============================================================
ALERT: ${UNIT_NAME} failed on ${HOST_NAME}
Timestamp (UTC) : ${TIMESTAMP_UTC}
Result / Status : ${RESULT_STATE} (exit=${EXIT_STATUS})
------------------------------------------------------------
Recent Journal Logs (last ${MAX_LINES} lines):
${JOURNAL_CONTEXT}
REPORT
)"

if [ -n "$AUDIT_CONTEXT" ]; then
  FULL_REPORT+="$(printf '\n------------------------------------------------------------\nLatest GPG Audit Tail (%s):\n%s\n' "$(basename "$LATEST_AUDIT_LOG")" "$AUDIT_CONTEXT")"
fi

printf '%s\n\n' "$FULL_REPORT" >> "$ALERT_LOG"

# Desktop notification if graphical session is active
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" \
    "${UNIT_NAME} (${RESULT_STATE}, exit=${EXIT_STATUS}) on ${HOST_NAME}" || true
fi

# HTTP Webhook delivery
if [ -n "${GITHUB_SUITE_WEBHOOK_URL:-}" ]; then
  if command -v curl >/dev/null 2>&1; then
    if command -v jq >/dev/null 2>&1; then
      PAYLOAD="$(
        jq -n \
          --arg unit "$UNIT_NAME" \
          --arg host "$HOST_NAME" \
          --arg ts "$TIMESTAMP_UTC" \
          --arg result "$RESULT_STATE" \
          --arg exit_code "$EXIT_STATUS" \
          --arg log_file "$ALERT_LOG" \
          --arg context "$FULL_REPORT" \
          '{
            text: ("*[" + $host + "]* `" + $unit + "` failed (result=" + $result + ", exit=" + $exit_code + ") at " + $ts + "\n```\n" + $context + "\n```"),
            unit: $unit,
            host: $host,
            timestamp_utc: $ts,
            result: $result,
            exit_status: $exit_code,
            alert_log: $log_file,
            logs: $context
          }'
      )"
    else
      ESCAPED_REPORT="$(printf '%s' "$FULL_REPORT" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')"
      PAYLOAD="{\"unit\":\"${UNIT_NAME}\",\"host\":\"${HOST_NAME}\",\"timestamp_utc\":\"${TIMESTAMP_UTC}\",\"exit_status\":\"${EXIT_STATUS}\",\"logs\":${ESCAPED_REPORT}}"
    fi

    curl_opts=(-fsS -X POST -H "Content-Type: application/json" -d "$PAYLOAD")
    if [ -n "${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN:-}" ]; then
      curl_opts+=(-H "Authorization: Bearer ${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN}")
    fi

    if ! curl "${curl_opts[@]}" "$GITHUB_SUITE_WEBHOOK_URL" >/dev/null; then
      echo "[WARN] Failed delivering webhook alert to ${GITHUB_SUITE_WEBHOOK_URL}" >> "$ALERT_LOG"
    fi
  else
    echo "[WARN] curl not installed; cannot send webhook alert" >> "$ALERT_LOG"
  fi
fi

# Local mail delivery (supports `mail` / `mailx` or `sendmail`)
if [ -n "${GITHUB_SUITE_MAIL_TO:-}" ]; then
  SUBJECT="[ALERT] ${UNIT_NAME} failed on ${HOST_NAME} (exit=${EXIT_STATUS})"
  if command -v mail >/dev/null 2>&1; then
    mail_args=(-s "$SUBJECT")
    if [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ]; then
      mail_args+=(-r "$GITHUB_SUITE_MAIL_FROM")
    fi
    if ! printf '%s\n' "$FULL_REPORT" | mail "${mail_args[@]}" "$GITHUB_SUITE_MAIL_TO"; then
      echo "[WARN] Failed sending mail alert via mail(1) to ${GITHUB_SUITE_MAIL_TO}" >> "$ALERT_LOG"
    fi
  elif command -v sendmail >/dev/null 2>&1; then
    {
      [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ] && echo "From: ${GITHUB_SUITE_MAIL_FROM}"
      echo "To: ${GITHUB_SUITE_MAIL_TO}"
      echo "Subject: ${SUBJECT}"
      echo "Content-Type: text/plain; charset=UTF-8"
      echo ""
      printf '%s\n' "$FULL_REPORT"
    } | sendmail -t || echo "[WARN] Failed sending mail alert via sendmail to ${GITHUB_SUITE_MAIL_TO}" >> "$ALERT_LOG"
  else
    echo "[WARN] Neither mail nor sendmail is installed; cannot send mail alert" >> "$ALERT_LOG"
  fi
fi

exit 0
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

3. Verify Webhook & Local Mail Delivery
 1. Test the alert script directly with your configured env vars
   1 min
   Load ~/.config/systemd/user/run-github-suite-alert.env and invoke the script once to confirm webhook and/or mail delivery without waiting for a real failure.
   set -a
source "${HOME}/.config/systemd/user/run-github-suite-alert.env"
set +a
"${HOME}/bin/notify-github-suite-failure.sh" run-github-suite.service

   Verification: Your webhook endpoint receives the JSON payload, your local mail spool/inbox gets the alert message, and ${HOME}/github-archives/systemd-failures.log has no [WARN] lines.
 2. Check local mail queue if using local MTA
   1 min
   If you configured a local mail address (GITHUB_SUITE_MAIL_TO), check local mail delivery (mail or postqueue -p / journal logs for your MTA).
   tail -n 15 "${HOME}/github-archives/systemd-failures.log"

   Verification: Confirms the alert block was written cleanly and no transport warnings were logged.
> Payload & security note: Because FULL_REPORT includes recent journal lines and GPG audit output, avoid sending full raw logs over unencrypted HTTP or to shared webhook channels unless your environment and destination are trusted.
> 
Here is the updated ~/bin/notify-github-suite-failure.sh that redacts URLs, tokens/credentials, and local filesystem paths before sending webhook or mail alerts, while keeping a local diagnostic copy in ${HOME}/github-archives/systemd-failures.log.
Redacting outbound payloads prevents leaking tokens, embedded credentials in URLs, webhook endpoints, or local directory paths (${HOME}, /home/..., /root/...) when alerts go to a webhook or mail sink.
1. Redacted ~/bin/notify-github-suite-failure.sh
cat <<'EOF' > "${HOME}/bin/notify-github-suite-failure.sh"
#!/usr/bin/env bash
set -euo pipefail

UNIT_NAME="${1:-run-github-suite.service}"
HOST_NAME="$(hostname -f 2>/dev/null || hostname)"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
ALERT_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_LINES="${GITHUB_SUITE_LOG_LINES:-120}"

mkdir -p "$ARCHIVE_ROOT" "$(dirname "$ALERT_LOG")"

EXIT_STATUS="$(
  systemctl --user show "$UNIT_NAME" -p ExecMainStatus --value 2>/dev/null || echo "unknown"
)"
RESULT_STATE="$(
  systemctl --user show "$UNIT_NAME" -p Result --value 2>/dev/null || echo "unknown"
)"

JOURNAL_CONTEXT="$(
  journalctl --user -u "$UNIT_NAME" -n "$MAX_LINES" --no-pager --output=short-iso 2>/dev/null \
    || echo "(no journal output available)"
)"

LATEST_AUDIT_LOG="$(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f -name 'gpg-audit-*.log' 2>/dev/null \
    | sort | tail -n 1 || true
)"

AUDIT_CONTEXT=""
if [ -n "${LATEST_AUDIT_LOG:-}" ] && [ -f "$LATEST_AUDIT_LOG" ]; then
  AUDIT_CONTEXT="$(tail -n 40 "$LATEST_AUDIT_LOG" 2>/dev/null || true)"
fi

LOCAL_REPORT="$(cat <<REPORT
============================================================
ALERT: ${UNIT_NAME} failed on ${HOST_NAME}
Timestamp (UTC) : ${TIMESTAMP_UTC}
Result / Status : ${RESULT_STATE} (exit=${EXIT_STATUS})
------------------------------------------------------------
Recent Journal Logs (last ${MAX_LINES} lines):
${JOURNAL_CONTEXT}
REPORT
)"

if [ -n "$AUDIT_CONTEXT" ]; then
  LOCAL_REPORT+="$(printf '\n------------------------------------------------------------\nLatest GPG Audit Tail (%s):\n%s\n' "$(basename "$LATEST_AUDIT_LOG")" "$AUDIT_CONTEXT")"
fi

# Save full unredacted local failure log for local debugging
printf '%s\n\n' "$LOCAL_REPORT" >> "$ALERT_LOG"

# Redact URLs, tokens/secrets, and local paths for outbound webhook/mail alerts
redact_text() {
  python3 - "$HOME" <<'PY'
import re
import sys

home_dir = sys.argv[1]
text = sys.stdin.read()

# 1. Redact common token/secret assignments and bearer/key prefixes
secret_patterns = [
    r'(?i)\b(authorization|bearer|token|api[_-]?key|secret|password|passwd|pwd|webhook[_-]?url)\b\s*[:=]\s*["\']?[^\s"\'&;,]+',
    r'\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]+\b',
    r'\bxox[baprs]-[A-Za-z0-9-]+\b',
    r'\bAKIA[0-9A-Z]{16}\b',
    r'(?i)bearer\s+[A-Za-z0-9._~+/=-]+',
]
for pat in secret_patterns:
    text = re.sub(pat, lambda m: f"{m.group(0).split('=')[0].split(':')[0]}=[REDACTED]" if ('=' in m.group(0) or ':' in m.group(0)) else "[REDACTED_TOKEN]", text)

# 2. Redact any URLs (including userinfo credentials and query params)
text = re.sub(r'\b[a-zA-Z][a-zA-Z0-9+.-]*://[^\s"\'<>)\]]+', '[REDACTED_URL]', text)

# 3. Redact $HOME and common Linux home/root/tmp/var paths
if home_dir:
    text = text.replace(home_dir, '~')
text = re.sub(r'/(?:home|root|var/log|etc|tmp|var/tmp)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)
text = re.sub(r'~/(?:src|github-archives|\.config|\.ssh|\.gnupg)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)

sys.stdout.write(text)
PY
}

REDACTED_REPORT="$(printf '%s' "$LOCAL_REPORT" | redact_text)"

# Desktop notification if graphical session is active (kept concise and path-free)
if [ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ] && command -v notify-send >/dev/null 2>&1; then
  notify-send -u critical "GitHub Suite Failed" \
    "${UNIT_NAME} (${RESULT_STATE}, exit=${EXIT_STATUS}) on ${HOST_NAME}" || true
fi

# HTTP Webhook delivery (using redacted report)
if [ -n "${GITHUB_SUITE_WEBHOOK_URL:-}" ]; then
  if command -v curl >/dev/null 2>&1; then
    if command -v jq >/dev/null 2>&1; then
      PAYLOAD="$(
        jq -n \
          --arg unit "$UNIT_NAME" \
          --arg host "$HOST_NAME" \
          --arg ts "$TIMESTAMP_UTC" \
          --arg result "$RESULT_STATE" \
          --arg exit_code "$EXIT_STATUS" \
          --arg context "$REDACTED_REPORT" \
          '{
            text: ("*[" + $host + "]* `" + $unit + "` failed (result=" + $result + ", exit=" + $exit_code + ") at " + $ts + "\n```\n" + $context + "\n```"),
            unit: $unit,
            host: $host,
            timestamp_utc: $ts,
            result: $result,
            exit_status: $exit_code,
            logs: $context
          }'
      )"
    else
      ESCAPED_REPORT="$(printf '%s' "$REDACTED_REPORT" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')"
      PAYLOAD="{\"unit\":\"${UNIT_NAME}\",\"host\":\"${HOST_NAME}\",\"timestamp_utc\":\"${TIMESTAMP_UTC}\",\"exit_status\":\"${EXIT_STATUS}\",\"logs\":${ESCAPED_REPORT}}"
    fi

    curl_opts=(-fsS -X POST -H "Content-Type: application/json" -d "$PAYLOAD")
    if [ -n "${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN:-}" ]; then
      curl_opts+=(-H "Authorization: Bearer ${GITHUB_SUITE_WEBHOOK_BEARER_TOKEN}")
    fi

    if ! curl "${curl_opts[@]}" "$GITHUB_SUITE_WEBHOOK_URL" >/dev/null; then
      echo "[WARN] Failed delivering webhook alert" >> "$ALERT_LOG"
    fi
  else
    echo "[WARN] curl not installed; cannot send webhook alert" >> "$ALERT_LOG"
  fi
fi

# Local mail delivery (using redacted report)
if [ -n "${GITHUB_SUITE_MAIL_TO:-}" ]; then
  SUBJECT="[ALERT] ${UNIT_NAME} failed on ${HOST_NAME} (exit=${EXIT_STATUS})"
  if command -v mail >/dev/null 2>&1; then
    mail_args=(-s "$SUBJECT")
    if [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ]; then
      mail_args+=(-r "$GITHUB_SUITE_MAIL_FROM")
    fi
    if ! printf '%s\n' "$REDACTED_REPORT" | mail "${mail_args[@]}" "$GITHUB_SUITE_MAIL_TO"; then
      echo "[WARN] Failed sending mail alert via mail(1)" >> "$ALERT_LOG"
    fi
  elif command -v sendmail >/dev/null 2>&1; then
    {
      [ -n "${GITHUB_SUITE_MAIL_FROM:-}" ] && echo "From: ${GITHUB_SUITE_MAIL_FROM}"
      echo "To: ${GITHUB_SUITE_MAIL_TO}"
      echo "Subject: ${SUBJECT}"
      echo "Content-Type: text/plain; charset=UTF-8"
      echo ""
      printf '%s\n' "$REDACTED_REPORT"
    } | sendmail -t || echo "[WARN] Failed sending mail alert via sendmail" >> "$ALERT_LOG"
  else
    echo "[WARN] Neither mail nor sendmail is installed; cannot send mail alert" >> "$ALERT_LOG"
  fi
fi

exit 0
EOF

chmod +x "${HOME}/bin/notify-github-suite-failure.sh"

2. Verify Redaction Behavior
 1. Test the redaction filter on sample sensitive strings
   1 min
   Pipe a sample string containing a URL, token, and path through the redaction block to verify replacements look right before relying on live alerts.
   printf 'path=%s/src/github/repo url=https://user:pass@example.com/hook?tok=abc token=ghp_1234567890abcdef\n' "$HOME" \
  | python3 - "$HOME" - <<'PY'
import re, sys
home_dir = sys.argv[1]
text = sys.stdin.read()
text = re.sub(r'(?i)\b(authorization|bearer|token|api[_-]?key|secret|password|passwd|pwd|webhook[_-]?url)\b\s*[:=]\s*["\']?[^\s"\'&;,]+', lambda m: f"{m.group(0).split('=')[0].split(':')[0]}=[REDACTED]", text)
text = re.sub(r'\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]+\b', '[REDACTED_TOKEN]', text)
text = re.sub(r'\b[a-zA-Z][a-zA-Z0-9+.-]*://[^\s"\'<>)\]]+', '[REDACTED_URL]', text)
if home_dir:
text = text.replace(home_dir, '~')
text = re.sub(r'/(?:home|root|var/log|etc|tmp|var/tmp)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)
text = re.sub(r'~/(?:src|github-archives|\.config|\.ssh|\.gnupg)/[^\s:\'"\)\]]*', '[REDACTED_PATH]', text)
print(text, end='')
PY

   Verification: Output replaces the URL with [REDACTED_URL], token with [REDACTED_TOKEN] / [REDACTED], and home/repo paths with [REDACTED_PATH] without exposing raw secrets.
 2. Trigger a test alert run
   1 min
   Run ~/bin/notify-github-suite-failure.sh run-github-suite.service and inspect your outbound alert payload/message to confirm no raw URLs, tokens, or paths appear.
   "${HOME}/bin/notify-github-suite-failure.sh" run-github-suite.service

   Verification: Exits 0, logs the full diagnostic entry to ${HOME}/github-archives/systemd-failures.log, and dispatches the redacted alert cleanly.
> Tuning note: Because regex redaction is a best-effort defense in depth, avoid printing sensitive CLI flags or environment values in your scripts' stdout/stderr in the first place—keeping secrets out of logs is always safer than scrubbing them afterward.
> 
notify-github-suite-failure.shTo run run-github-suite.service automatically on a schedule, create a matching systemd user timer unit (~/.config/systemd/user/run-github-suite.timer) and enable it with systemctl --user enable --now.
A systemd timer unit activates a service unit of the same base name by default (Unit= defaults to run-github-suite.service), uses OnCalendar= for wall-clock schedules, and supports Persistent=true so missed runs execute once the user manager comes back online.
1. Create the Systemd User Timer (~/.config/systemd/user/run-github-suite.timer)
Here is a daily schedule at 03:15 local time (America/Chicago / CST in Evansville, Indiana) with a RandomizedDelaySec=300 window so scheduled runs don't collide if you add more timers later:
mkdir -p "${HOME}/.config/systemd/user"

cat <<'EOF' > "${HOME}/.config/systemd/user/run-github-suite.timer"
[Unit]
Description=Scheduled run of GitHub backup, GPG audit, and archive sync suite
Requires=run-github-suite.service

[Timer]
OnCalendar=*-*-* 03:15:00
RandomizedDelaySec=300
Persistent=true
Unit=run-github-suite.service

[Install]
WantedBy=timers.target
EOF

(Other common OnCalendar= examples: every 6 hours uses OnCalendar=*-*-* 00/6:00:00; every Sunday at 02:00 uses OnCalendar=Sun *-*-* 02:00:00.)
2. Enable, Start & Verify the Timer
 1. Reload user units and enable the timer
   1 min
   Reload systemd --user so it detects run-github-suite.timer, then enable and start it immediately.
   systemctl --user daemon-reload
systemctl --user enable --now run-github-suite.timer

   Verification: systemctl --user is-enabled run-github-suite.timer outputs enabled and systemctl --user is-active run-github-suite.timer outputs active.
 2. Inspect next scheduled trigger time
   1 min
   List active user timers to verify NEXT and LEFT trigger timestamps.
   systemctl --user list-timers --all | grep -E "NEXT|run-github-suite"

   Verification: Shows run-github-suite.timer with its next scheduled trigger time and run-github-suite.service listed under UNIT.
 3. Verify unattended execution across logouts
   1 min
   If you want the timer and run-github-suite.service to keep running when you are not logged into an interactive session, enable user lingering.
   loginctl enable-linger "$USER"
loginctl show-user "$USER" -p Linger

   Verification: loginctl show-user returns Linger=yes.
> Scheduling & GPG/SSH note: When run-github-suite.service runs unattended from a timer, make sure any required GPG/SSH/GitHub credentials or rclone remotes are accessible non-interactively (for example, unlocked/key-based auth or an environment file loaded via EnvironmentFile=) so scheduled runs don't stall waiting on a TTY prompt.
> 
To trigger run-github-suite.service on demand and stream its live output, start the user service with systemctl --user start and follow its logs with journalctl --user -u run-github-suite.service -f.
Using --no-block lets systemctl --user start return right away so you can immediately attach to journalctl -f without waiting for a long backup, GPG audit, or rclone sync to finish first.
1. Trigger On-Demand & Stream Live Logs
Run these commands in your terminal to start run-github-suite.service immediately and watch output as each step executes:
systemctl --user start --no-block run-github-suite.service
journalctl --user -u run-github-suite.service -f --output=short-iso

(If your service is a Type=oneshot unit and you want systemctl itself to wait until the run finishes and report success/failure in one command, run systemctl --user start run-github-suite.service in one terminal and journalctl --user -u run-github-suite.service -f in a second terminal.)
2. Step-by-Step Execution & Inspection
 1. Start the service and attach to live journal output
   1 min
   Trigger run-github-suite.service and tail its live journal stream with ISO timestamps.
   systemctl --user start --no-block run-github-suite.service
journalctl --user -u run-github-suite.service -f --output=short-iso

   Verification: journalctl prints live log lines from run-github-suite.service as the backup, GPG audit, SHA-256 verification, and rclone check execute; press Ctrl+C to detach from the live stream once finished.
 2. Check final unit status and exit code
   1 min
   After the run completes, inspect systemctl --user status and ExecMainStatus to confirm a clean exit (0) or see why it failed.
   systemctl --user status run-github-suite.service --no-pager
systemctl --user show run-github-suite.service -p ActiveState,SubState,Result,ExecMainStatus

   Verification: Shows ActiveState=inactive (for oneshot after completion), Result=success, and ExecMainStatus=0 on a healthy run; if non-zero, OnFailure=run-github-suite-alert.service will have triggered as well.
 3. Filter logs to only the most recent run (if needed)
   1 min
   If run-github-suite.service has run multiple times, inspect logs since the last start or since today's start of day.
   journalctl --user -u run-github-suite.service --since today --no-pager --output=short-iso

   Verification: Displays only entries from today for run-github-suite.service so you can review full output without scrolling past older runs.
To rotate and cap systemd-failures.log and gpg-audit-*.log inside ~/github-archives without touching your actual repository backup snapshots, you can use user-mode logrotate (running without root via a custom state file) or a lightweight Zstandard (zstd) rotation script triggered right from run-github-suite.sh or a systemd --user timer.
Using logrotate with --state in ~/.local/state/logrotate/github-archives.state works cleanly for a regular user account without needing root privileges or /var/lib/logrotate/status.
1. Rotate & Cap with User-Mode logrotate
Create a user logrotate config at ~/.config/logrotate/github-archives.conf that:
 * Rotates ~/github-archives/systemd-failures.log when it exceeds 10M (or monthly), compresses rotated logs with zstd, and keeps the latest 10 rotated archives (rotate 10), using copytruncate so append writers never lose their file descriptor.
 * Rotates ~/github-archives/gpg-audit-*.log (keeping copytruncate or rotating when size/age thresholds are met) and caps total retained audit logs by pruning older gpg-audit-* files/archives past your retention count.
mkdir -p "${HOME}/.config/logrotate" "${HOME}/.local/state/logrotate"

cat <<'EOF' > "${HOME}/.config/logrotate/github-archives.conf"
/home/robert/github-archives/systemd-failures.log {
    size 10M
    rotate 10
    missingok
    notifempty
    copytruncate
    compress
    compresscmd /usr/bin/zstd
    uncompresscmd /usr/bin/unzstd
    compressext .zst
    compressoptions -T0 -19
    dateext
    dateformat -%Y%m%d-%H%M%S
}

/home/robert/github-archives/gpg-audit-*.log {
    size 5M
    rotate 30
    missingok
    notifempty
    copytruncate
    compress
    compresscmd /usr/bin/zstd
    uncompresscmd /usr/bin/unzstd
    compressext .zst
    compressoptions -T0 -19
    dateext
    dateformat -%Y%m%d-%H%M%S
}
EOF

(Replace /home/robert with ${HOME} if your Linux username differs—note that logrotate config directives expect explicit paths rather than unexpanded shell variables.)
2. Cap Total Count & Disk Footprint (~/bin/rotate-github-logs.sh)
Because gpg-audit-<timestamp>.log creates a new timestamped file per run, a rotate N rule on a wildcard gpg-audit-*.log rotates each matching file individually rather than capping the total number of audit log files across runs. Pair logrotate (or use a standalone script) with a cap on total gpg-audit-* files and total log size:
cat <<'EOF' > "${HOME}/bin/rotate-github-logs.sh"
#!/usr/bin/env bash
set -euo pipefail

ARCHIVE_ROOT="${ARCHIVE_ROOT:-${HOME}/github-archives}"
FAILURES_LOG="${ARCHIVE_ROOT}/systemd-failures.log"
MAX_FAILURES_BYTES="${MAX_FAILURES_BYTES:-10485760}"   # 10 MiB before rotation
KEEP_FAILURES_ROTATED="${KEEP_FAILURES_ROTATED:-10}"
KEEP_AUDIT_LOGS="${KEEP_AUDIT_LOGS:-30}"              # Keep newest 30 gpg-audit log/zst files
MAX_TOTAL_LOG_BYTES="${MAX_TOTAL_LOG_BYTES:-268435456}" # 256 MiB cap across log + rotated log files

mkdir -p "$ARCHIVE_ROOT" "${HOME}/.local/state/logrotate"

STAMP="$(date +%Y%m%d-%H%M%S)"

# 1. Rotate systemd-failures.log if it exceeds MAX_FAILURES_BYTES
if [ -f "$FAILURES_LOG" ]; then
  current_size="$(stat -c '%s' "$FAILURES_LOG" 2>/dev/null || echo 0)"
  if [ "$current_size" -ge "$MAX_FAILURES_BYTES" ]; then
    rotated_name="${ARCHIVE_ROOT}/systemd-failures-${STAMP}.log"
    cp -p "$FAILURES_LOG" "$rotated_name"
    : > "$FAILURES_LOG"
    if command -v zstd >/dev/null 2>&1; then
      zstd --rm -q -T0 -19 "$rotated_name" -o "${rotated_name}.zst"
    fi
  fi
fi

# Prune older rotated systemd-failures archives beyond KEEP_FAILURES_ROTATED
mapfile -t failure_archives < <(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f \
    \( -name 'systemd-failures-*.log' -o -name 'systemd-failures-*.log.zst' \) \
    | sort -r
)
if [ "${#failure_archives[@]}" -gt "$KEEP_FAILURES_ROTATED" ]; then
  for old_file in "${failure_archives[@]:$KEEP_FAILURES_ROTATED}"; do
    rm -f "$old_file"
  done
fi

# 2. Cap total gpg-audit-*.log and gpg-audit-*.log.zst files to KEEP_AUDIT_LOGS
mapfile -t audit_files < <(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f \
    \( -name 'gpg-audit-*.log' -o -name 'gpg-audit-*.log.zst' \) \
    | sort -r
)
if [ "${#audit_files[@]}" -gt "$KEEP_AUDIT_LOGS" ]; then
  for old_audit in "${audit_files[@]:$KEEP_AUDIT_LOGS}"; do
    rm -f "$old_audit"
  done
fi

# 3. Enforce total byte budget across log files (oldest first, keeping active systemd-failures.log)
total_bytes=0
declare -a all_log_files=()
while IFS= read -r -d '' f; do
  sz="$(stat -c '%s' "$f" 2>/dev/null || echo 0)"
  total_bytes=$((total_bytes + sz))
  all_log_files+=("${sz}|${f}")
done < <(
  find "$ARCHIVE_ROOT" -maxdepth 1 -type f \
    \( -name 'systemd-failures*.log*' -o -name 'gpg-audit-*.log*' \) \
    -print0 | sort -z
)

if [ "$total_bytes" -gt "$MAX_TOTAL_LOG_BYTES" ]; then
  for entry in "${all_log_files[@]}"; do
    [ "$total_bytes" -le "$MAX_TOTAL_LOG_BYTES" ] && break
    sz="${entry%%|*}"
    fpath="${entry#*|}"
    # Truncate active systemd-failures.log rather than deleting it; remove older rotated/audit logs
    if [ "$fpath" = "$FAILURES_LOG" ]; then
      : > "$FAILURES_LOG"
      total_bytes=$((total_bytes - sz))
    else
      rm -f "$fpath"
      total_bytes=$((total_bytes - sz))
    fi
  done
fi
EOF

chmod +x "${HOME}/bin/rotate-github-logs.sh"

3. Run & Verify Log Rotation
 * Test logrotate in user mode (--debug / --force):
   logrotate --state "${HOME}/.local/state/logrotate/github-archives.state" \
  --verbose "${HOME}/.config/logrotate/github-archives.conf"

   Verification: Exits 0 and logs rotation/compression actions without touching .tar.zst repository snapshot folders.
 * Test ~/bin/rotate-github-logs.sh and check log sizes:
   ~/bin/rotate-github-logs.sh
ls -lh "${HOME}/github-archives"/systemd-failures* "${HOME}/github-archives"/gpg-audit-* 2>/dev/null || true

   Verification: Confirms systemd-failures.log and gpg-audit-* files are capped according to your size and file-count limits.
 * Hook into run-github-suite.sh: Add "${HOME}/bin/rotate-github-logs.sh" near the end of ~/bin/run-github-suite.sh (after audit-gpg-signatures.sh finishes reading the latest audit log) so log capping stays automatic on every scheduled or manual run.
To make ~/bin/apply-gpg-ruleset.sh log a clear reason and continue cleanly when GitHub returns 403 or 404 on a repository's rulesets endpoint, capture the HTTP status code from gh api (instead of letting set -e abort on non-zero exit) and route 403 / 404 into a dedicated skip handler.
Because GitHub often returns 404 as well as 403 when rulesets are restricted on a private repository's plan tier or when token/repository permissions don't allow ruleset management, handling both codes explicitly keeps batch runs across re1Lucky365 and adpsnake7-beep moving cleanly.
1. Updated ~/bin/apply-gpg-ruleset.sh (Clean 403 / 404 Handling)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   Space-separated GitHub owners (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names (if omitted, lists repos per owner)
  --ruleset-file PATH        JSON ruleset payload file (default: "${HOME}/.config/github/gpg-ruleset.json")
  --dry-run                  Inspect target repos and show intended action without applying changes
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
RULESET_FILE="${HOME}/.config/github/gpg-ruleset.json"
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --ruleset-file)
      RULESET_FILE="${2:?Missing value for --ruleset-file}"
      shift 2
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }

if [ "$DRY_RUN" = false ] && [ ! -f "$RULESET_FILE" ]; then
  echo "ERROR: Ruleset JSON file not found: ${RULESET_FILE}" >&2
  exit 1
fi

# Runs `gh api` while preserving HTTP status code and response body.
# Sets global LAST_HTTP_CODE and LAST_HTTP_BODY; returns 0 if transport/gh succeeded.
call_gh_api() {
  local tmp_out tmp_err rc=0
  tmp_out="$(mktemp)"
  tmp_err="$(mktemp)"

  # Append status code on the final line via jq/awk wrapper
  if gh api -i "$@" >"$tmp_out" 2>"$tmp_err"; then
    rc=0
  else
    rc=$?
  fi

  LAST_HTTP_CODE="$(
    awk 'toupper($1) ~ /^HTTP\// { code=$2 } END { print (code ? code : "000") }' "$tmp_out"
  )"
  # Extract body after the final blank line separating headers from body
  LAST_HTTP_BODY="$(
    awk 'BEGIN { body=0 } /^\r?$/ { body=1; next } body { print }' "$tmp_out"
  )"

  rm -f "$tmp_out" "$tmp_err"

  if [ "$LAST_HTTP_CODE" = "000" ] && [ "$rc" -ne 0 ]; then
    return 1
  fi
  return 0
}

explain_403_404() {
  local code="$1"
  local owner="$2"
  local repo="$3"
  local is_private="$4"
  local api_msg="$5"

  if [ "$code" = "404" ]; then
    if [ "$is_private" = "true" ]; then
      echo "HTTP 404 on rulesets for private repo ${owner}/${repo} (likely unsupported on current plan tier or ruleset endpoint unavailable/hidden)"
    else
      echo "HTTP 404 on rulesets for ${owner}/${repo} (endpoint not available or repo/owner not accessible to active token)"
    fi
  else
    # code == 403
    if [ "$is_private" = "true" ]; then
      echo "HTTP 403 on rulesets for private repo ${owner}/${repo} (plan tier restriction, missing repo admin permission, or token scope limitation${api_msg:+: ${api_msg}})"
    else
      echo "HTTP 403 on rulesets for ${owner}/${repo} (insufficient token permissions/scope or admin access required${api_msg:+: ${api_msg}})"
    fi
  fi
}

applied_count=0
skipped_count=0
error_count=0

for owner in "${ACCOUNTS[@]}"; do
  if [ -n "$EXPLICIT_REPOS" ]; then
    repo_list=($EXPLICIT_REPOS)
  else
    if ! repo_raw="$(gh repo list "$owner" --limit 200 --json name --jq '.[].name' 2>&1)"; then
      echo "[ERROR] ${owner}: failed listing repositories (${repo_raw})" >&2
      error_count=$((error_count + 1))
      continue
    fi
    mapfile -t repo_list <<< "$repo_raw"
  fi

  for repo in "${repo_list[@]}"; do
    [ -n "$repo" ] || continue
    full_repo="${owner}/${repo}"

    if ! call_gh_api "repos/${full_repo}"; then
      echo "[ERROR] ${full_repo}: gh api transport error while inspecting repository" >&2
      error_count=$((error_count + 1))
      continue
    fi

    if [ "$LAST_HTTP_CODE" = "403" ] || [ "$LAST_HTTP_CODE" = "404" ]; then
      skipped_count=$((skipped_count + 1))
      echo "[SKIP] ${full_repo}: HTTP ${LAST_HTTP_CODE} inspecting repository (check active gh account, token scope, or repo visibility)"
      continue
    elif [ "$LAST_HTTP_CODE" -lt 200 ] || [ "$LAST_HTTP_CODE" -ge 300 ]; then
      echo "[ERROR] ${full_repo}: unexpected HTTP ${LAST_HTTP_CODE} inspecting repository" >&2
      error_count=$((error_count + 1))
      continue
    fi

    is_private="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.private // false' 2>/dev/null || echo "false")"
    ruleset_name="$(jq -r '.name // "gpg-enforcement"' "$RULESET_FILE" 2>/dev/null || echo "gpg-enforcement")"

    # Check existing rulesets first so we can either update or create cleanly
    if ! call_gh_api "repos/${full_repo}/rulesets"; then
      echo "[ERROR] ${full_repo}: gh api transport error querying rulesets" >&2
      error_count=$((error_count + 1))
      continue
    fi

    if [ "$LAST_HTTP_CODE" = "403" ] || [ "$LAST_HTTP_CODE" = "404" ]; then
      api_msg="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.message // empty' 2>/dev/null || true)"
      reason="$(explain_403_404 "$LAST_HTTP_CODE" "$owner" "$repo" "$is_private" "$api_msg")"
      skipped_count=$((skipped_count + 1))
      echo "[SKIP] ${full_repo}: ${reason}"
      continue
    elif [ "$LAST_HTTP_CODE" -lt 200 ] || [ "$LAST_HTTP_CODE" -ge 300 ]; then
      echo "[ERROR] ${full_repo}: unexpected HTTP ${LAST_HTTP_CODE} listing rulesets" >&2
      error_count=$((error_count + 1))
      continue
    fi

    existing_id="$(
      printf '%s' "$LAST_HTTP_BODY" \
        | jq -r --arg name "$ruleset_name" '.[] | select(.name == $name) | .id' 2>/dev/null \
        | head -n 1 || true
    )"

    if [ "$DRY_RUN" = true ]; then
      if [ -n "$existing_id" ]; then
        echo "[DRY-RUN] ${full_repo} (private=${is_private}): would update ruleset '${ruleset_name}' (id=${existing_id})"
      else
        echo "[DRY-RUN] ${full_repo} (private=${is_private}): would create ruleset '${ruleset_name}'"
      fi
      applied_count=$((applied_count + 1))
      continue
    fi

    if [ -n "$existing_id" ]; then
      call_gh_api -X PUT "repos/${full_repo}/rulesets/${existing_id}" --input "$RULESET_FILE" || {
        echo "[ERROR] ${full_repo}: gh api transport error updating ruleset ${existing_id}" >&2
        error_count=$((error_count + 1))
        continue
      }
      action_word="updating"
    else
      call_gh_api -X POST "repos/${full_repo}/rulesets" --input "$RULESET_FILE" || {
        echo "[ERROR] ${full_repo}: gh api transport error creating ruleset" >&2
        error_count=$((error_count + 1))
        continue
      }
      action_word="creating"
    fi

    if [ "$LAST_HTTP_CODE" = "403" ] || [ "$LAST_HTTP_CODE" = "404" ]; then
      api_msg="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.message // empty' 2>/dev/null || true)"
      reason="$(explain_403_404 "$LAST_HTTP_CODE" "$owner" "$repo" "$is_private" "$api_msg")"
      skipped_count=$((skipped_count + 1))
      echo "[SKIP] ${full_repo}: ${reason} (while ${action_word} ruleset '${ruleset_name}')"
      continue
    elif [ "$LAST_HTTP_CODE" -lt 200 ] || [ "$LAST_HTTP_CODE" -ge 300 ]; then
      api_msg="$(printf '%s' "$LAST_HTTP_BODY" | jq -r '.message // empty' 2>/dev/null || true)"
      echo "[ERROR] ${full_repo}: HTTP ${LAST_HTTP_CODE} while ${action_word} ruleset '${ruleset_name}'${api_msg:+ (${api_msg})}" >&2
      error_count=$((error_count + 1))
      continue
    fi

    echo "[OK] ${full_repo} (private=${is_private}): applied ruleset '${ruleset_name}' (HTTP ${LAST_HTTP_CODE})"
    applied_count=$((applied_count + 1))
  done
done

echo "------------------------------------------------------------"
echo "Summary: Applied=${applied_count} | Skipped(403/404)=${skipped_count} | Errors=${error_count}"

if [ "$error_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Why This Structure Works Better Across Multiple Repos & Accounts
 * Non-fatal skips vs. real errors: Repositories returning 403 or 404 increment Skipped(403/404) and log [SKIP] with context (private=true/false and any GitHub API message) without aborting the loop or failing the script when no true errors occur.
 * True errors still surface: Network failures, invalid JSON payloads, or 422 / 5xx responses increment Errors and cause a non-zero exit so your systemd failure alert still catches genuine regressions.
 * Safe to re-run idempotently: Matching an existing ruleset by .name switches between POST (create) and PUT (update) automatically.
Here is a sample GitHub repository ruleset JSON payload (~/.config/github/gpg-ruleset.json) that targets your default branch (~DEFAULT_BRANCH) and enforces both signed commits (required_signatures) and required status checks (required_status_checks).
1. Sample Ruleset JSON (~/.config/github/gpg-ruleset.json)
Replace "verify-gpg-signatures" (and add any additional check objects to required_status_checks) with the exact status check context / job names reported on your repository's commits or pull requests.
{
  "name": "gpg-and-status-checks",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [],
  "conditions": {
    "ref_name": {
      "include": [
        "~DEFAULT_BRANCH"
      ],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_signatures"
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "do_not_enforce_on_create": false,
        "required_status_checks": [
          {
            "context": "verify-gpg-signatures"
          }
        ]
      }
    }
  ]
}

2. Key Fields to Know Before Applying
 * conditions.ref_name.include: ["~DEFAULT_BRANCH"]: Applies the ruleset to each repository's default branch (main, master, etc.) automatically without hardcoding branch names.
 * rules (required_signatures): Requires commits and tags pushed/merged to matching branches to have verified signatures.
 * rules (required_status_checks):
   * strict_required_status_checks_policy: true requires target branches to be up to date with the base branch before merging.
   * do_not_enforce_on_create: false enforces the check immediately rather than skipping enforcement on initial creation.
   * required_status_checks lists each required check context (you can include multiple { "context": "..." } entries; make sure the name matches an actual check emitted by your CI workflow so PRs aren't stuck waiting on a check that never runs).
 * bypass_actors: []: Keeps enforcement strict with no bypass actors by default (you can add actor entries later if needed).
3. Validate JSON & Preview with apply-gpg-ruleset.sh
mkdir -p "${HOME}/.config/github"
# Save the JSON above to ~/.config/github/gpg-ruleset.json, then validate syntax:
jq empty "${HOME}/.config/github/gpg-ruleset.json"

# Preview across your accounts/repos without mutating remote rulesets:
~/bin/apply-gpg-ruleset.sh --ruleset-file "${HOME}/.config/github/gpg-ruleset.json" --dry-run

Verification: jq empty exits 0 with valid JSON, and apply-gpg-ruleset.sh --dry-run prints [DRY-RUN] for eligible repos while logging [SKIP] with reasons for any 403 / 404 repositories.
Here is a shell script (~/bin/compare-repo-rulesets.sh) that scans your local Git repositories under ${HOME}/src/github (defaulting to re1Lucky365 and adpsnake7-beep), queries GitHub's rulesets API via gh api, and reports whether each local repository has an active ruleset enforcing required_signatures and required_status_checks—or whether it is missing a ruleset, restricted (403/404), or lacks a GitHub remote.
1. Local vs. Remote Ruleset Coverage Script (~/bin/compare-repo-rulesets.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --src-root PATH            Local clones directory (default: "${HOME}/src/github")
  --accounts "acct1 acct2"   Accounts to include (default: "re1Lucky365 adpsnake7-beep")
  --ruleset-name NAME        Expected ruleset name (default: "gpg-and-status-checks")
  -h, --help                 Show this help
EOF
  exit 1
}

SRC_ROOT="${HOME}/src/github"
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPECTED_NAME="gpg-and-status-checks"

while [ "$#" -gt 0 ]; do
  case "$1" in
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --ruleset-name)
      EXPECTED_NAME="${2:?Missing value for --ruleset-name}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

# Extract owner/repo from a git remote URL (supports https:// and git@github.com:)
parse_github_slug() {
  local url="$1"
  local cleaned
  cleaned="$(printf '%s' "$url" | sed -E 's#^(https?://[^/]+/|git@[^:]+:)##; s#\.git$##')"
  if [[ "$cleaned" == */* ]]; then
    printf '%s\n' "$cleaned"
    return 0
  fi
  return 1
}

account_allowed() {
  local owner="$1"
  for acct in "${ACCOUNTS[@]}"; do
    [ "$acct" = "$owner" ] && return 0
  done
  return 1
}

# Fetches rulesets JSON into LAST_RULESETS_JSON and sets LAST_HTTP_STATUS
fetch_rulesets() {
  local slug="$1"
  local tmp_out tmp_err rc=0
  tmp_out="$(mktemp)"
  tmp_err="$(mktemp)"

  if gh api -i "repos/${slug}/rulesets" >"$tmp_out" 2>"$tmp_err"; then
    rc=0
  else
    rc=$?
  fi

  LAST_HTTP_STATUS="$(
    awk 'toupper($1) ~ /^HTTP\// { code=$2 } END { print (code ? code : "000") }' "$tmp_out"
  )"
  LAST_RULESETS_JSON="$(
    awk 'BEGIN { body=0 } /^\r?$/ { body=1; next } body { print }' "$tmp_out"
  )"

  rm -f "$tmp_out" "$tmp_err"

  if [ "$LAST_HTTP_STATUS" = "000" ] && [ "$rc" -ne 0 ]; then
    return 1
  fi
  return 0
}

total_local=0
covered_full=0
covered_partial=0
missing_ruleset=0
restricted_403_404=0
other_issues=0

printf "%-34s | %-18s | %-10s | %s\n" "LOCAL REPO" "STATUS" "HTTP" "DETAILS"
printf -- "-----------------------------------+--------------------+------------+----------------------------------------------\n"

while IFS= read -r -d '' gitdir; do
  repo_dir="$(dirname "$gitdir")"
  rel_path="${repo_dir#${SRC_ROOT}/}"
  total_local=$((total_local + 1))

  remote_url="$(git -C "$repo_dir" remote get-url origin 2>/dev/null || true)"
  if [ -z "$remote_url" ]; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" "$rel_path" "NO-REMOTE" "-" "origin remote not configured"
    continue
  fi

  if ! slug="$(parse_github_slug "$remote_url")"; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" "$rel_path" "NON-GITHUB" "-" "remote=${remote_url}"
    continue
  fi

  owner="${slug%%/*}"
  if ! account_allowed "$owner"; then
    continue
  fi

  if ! fetch_rulesets "$slug"; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" "$rel_path" "API-ERROR" "000" "failed calling gh api for ${slug}"
    continue
  fi

  if [ "$LAST_HTTP_STATUS" = "403" ] || [ "$LAST_HTTP_STATUS" = "404" ]; then
    restricted_403_404=$((restricted_403_404 + 1))
    printf "%-34s | %-18s | %-10s | %s\n" \
      "$rel_path" "SKIP-RESTRICTED" "$LAST_HTTP_STATUS" "rulesets unavailable/restricted on ${slug} (rely on pre-push + audit)"
    continue
  fi

  if [ "$LAST_HTTP_STATUS" -lt 200 ] || [ "$LAST_HTTP_STATUS" -ge 300 ]; then
    other_issues=$((other_issues + 1))
    printf "%-34s | %-18s | %-10s | %s\n" \
      "$rel_path" "HTTP-ERROR" "$LAST_HTTP_STATUS" "unexpected response for ${slug}"
    continue
  fi

  # Inspect active rulesets (prefer matching EXPECTED_NAME if present, otherwise inspect all active ones)
  matched_summary="$(
    printf '%s' "$LAST_RULESETS_JSON" | jq -r --arg exp "$EXPECTED_NAME" '
      def summarize(r):
        ([r.rules[]?.type] | unique | join(","));
      ([.[] | select(.enforcement == "active" and .name == $exp)] | .[0]) as $exact |
      ([.[] | select(.enforcement == "active")] | .[0]) as $any_active |
      if $exact != null then
        "EXACT|\($exact.id)|\($exact.name)|\(summarize($exact))"
      elif $any_active != null then
        "OTHER-ACTIVE|\($any_active.id)|\($any_active.name)|\(summarize($any_active))"
      elif ([.[] | select(.name == $exp)] | length) > 0 then
        "INACTIVE|\([.[] | select(.name == $exp)][0].id)|\($exp)|\([.[] | select(.name == $exp)][0].enforcement)"
      else
        "NONE|||"
      end
    ' 2>/dev/null || echo "PARSE-ERROR|||"
  )"

  IFS='|' read -r match_kind rs_id rs_name rs_rules <<< "$matched_summary"

  case "$match_kind" in
    EXACT|OTHER-ACTIVE)
      has_sig=false
      has_checks=false
      [[ ",${rs_rules}," == *",required_signatures,"* ]] && has_sig=true
      [[ ",${rs_rules}," == *",required_status_checks,"* ]] && has_checks=true

      if [ "$has_sig" = true ] && [ "$has_checks" = true ]; then
        covered_full=$((covered_full + 1))
        printf "%-34s | %-18s | %-10s | %s\n" \
          "$rel_path" "FULL-ENFORCED" "$LAST_HTTP_STATUS" "ruleset '${rs_name}' (id=${rs_id}) -> ${rs_rules}"
      else
        covered_partial=$((covered_partial + 1))
        printf "%-34s | %-18s | %-10s | %s\n" \
          "$rel_path" "PARTIAL-ENFORCED" "$LAST_HTTP_STATUS" "ruleset '${rs_name}' (id=${rs_id}) -> ${rs_rules}"
      fi
      ;;
    INACTIVE)
      missing_ruleset=$((missing_ruleset + 1))
      printf "%-34s | %-18s | %-10s | %s\n" \
        "$rel_path" "NOT-ACTIVE" "$LAST_HTTP_STATUS" "ruleset '${rs_name}' (id=${rs_id}) state=${rs_rules}"
      ;;
    NONE)
      missing_ruleset=$((missing_ruleset + 1))
      printf "%-34s | %-18s | %-10s | %s\n" \
        "$rel_path" "MISSING" "$LAST_HTTP_STATUS" "no active ruleset found on ${slug}"
      ;;
    *)
      other_issues=$((other_issues + 1))
      printf "%-34s | %-18s | %-10s | %s\n" \
        "$rel_path" "PARSE-ERROR" "$LAST_HTTP_STATUS" "could not parse rulesets JSON for ${slug}"
      ;;
  esac
done < <(find "$SRC_ROOT" -mindepth 3 -maxdepth 3 -type d -name ".git" -print0 | sort -z)

echo "--------------------------------------------------------------------------------------------------------"
echo "Summary: LocalRepos=${total_local} | FullEnforced=${covered_full} | Partial=${covered_partial} | Missing=${missing_ruleset} | Restricted(403/404)=${restricted_403_404} | OtherIssues=${other_issues}"

2. Run & Interpret the Coverage Comparison
 1. Save and run compare-repo-rulesets.sh
   1 min
   Save the script to ~/bin/compare-repo-rulesets.sh, make it executable, and run it against ${HOME}/src/github.
   chmod +x "${HOME}/bin/compare-repo-rulesets.sh"
"${HOME}/bin/compare-repo-rulesets.sh"

   Verification: Prints a status table for every local repository under re1Lucky365 and adpsnake7-beep and finishes with a single summary line.
 2. Follow up on MISSING or PARTIAL-ENFORCED repos
   1 min
   For any repository showing MISSING or PARTIAL-ENFORCED (with HTTP 200), apply or update ~/.config/github/gpg-ruleset.json using ~/bin/apply-gpg-ruleset.sh --repos "<repo-name>"; for SKIP-RESTRICTED (403/404) repos, rely on your local pre-push hook and audit-gpg-signatures.sh.
   "${HOME}/bin/apply-gpg-ruleset.sh" --ruleset-file "${HOME}/.config/github/gpg-ruleset.json" --dry-run

   Verification: Confirms which repos can be updated remotely vs. which remain covered locally via client-side hooks and audits.
To verify end-to-end repository coverage across your local clones under ${HOME}/src/github (re1Lucky365 and adpsnake7-beep) and remote GitHub rulesets, run audit-gpg-signatures.sh (checking local commit and annotated tag GPG signatures) alongside compare-repo-rulesets.sh (checking remote ruleset enforcement and 403/404 restrictions).
1. Combined End-to-End Verification Command
Run both checks sequentially, capturing the audit summary and log path while preserving non-zero exit codes if a signature violation occurs:
SRC_ROOT="${HOME}/src/github"
AUDIT_LOG="${HOME}/github-archives/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log"

echo "=== 1. Local GPG Signature Audit (Commits & Annotated Tags) ==="
audit_rc=0
"${HOME}/bin/audit-gpg-signatures.sh" "$SRC_ROOT" "$AUDIT_LOG" || audit_rc=$?

echo ""
echo "=== 2. Local vs. Remote Ruleset Coverage ==="
ruleset_rc=0
"${HOME}/bin/compare-repo-rulesets.sh" --src-root "$SRC_ROOT" || ruleset_rc=$?

echo ""
echo "=== Combined Status ==="
echo "GPG Audit Exit Code   : ${audit_rc} (log: ${AUDIT_LOG})"
echo "Ruleset Check Exit Code: ${ruleset_rc}"

if [ "$audit_rc" -ne 0 ]; then
  exit "$audit_rc"
fi
if [ "$ruleset_rc" -ne 0 ]; then
  exit "$ruleset_rc"
fi
exit 0

2. Optional Wrapper Script (~/bin/verify-repo-coverage.sh)
Save this wrapper if you want a single command you can run manually or inside your maintenance workflow to check both local GPG integrity and remote ruleset state in one pass:
cat <<'EOF' > "${HOME}/bin/verify-repo-coverage.sh"
#!/usr/bin/env bash
set -euo pipefail

SRC_ROOT="${1:-${HOME}/src/github}"
AUDIT_SCRIPT="${HOME}/bin/audit-gpg-signatures.sh"
RULESET_SCRIPT="${HOME}/bin/compare-repo-rulesets.sh"
ARCHIVE_ROOT="${HOME}/github-archives"

mkdir -p "$ARCHIVE_ROOT"
AUDIT_LOG="${ARCHIVE_ROOT}/gpg-audit-$(date +%Y-%m-%d_%H%M%S).log"

echo "[1/2] Auditing local commits and annotated tags in ${SRC_ROOT}..."
audit_status=0
"$AUDIT_SCRIPT" "$SRC_ROOT" "$AUDIT_LOG" || audit_status=$?

echo ""
echo "[2/2] Comparing local repositories against active GitHub rulesets..."
ruleset_status=0
"$RULESET_SCRIPT" --src-root "$SRC_ROOT" || ruleset_status=$?

echo ""
echo "------------------------------------------------------------"
echo "Audit status   : ${audit_status} -> ${AUDIT_LOG}"
echo "Ruleset status : ${ruleset_status}"

if [ "$audit_status" -ne 0 ]; then
  exit "$audit_status"
fi
if [ "$ruleset_status" -ne 0 ]; then
  exit "$ruleset_status"
fi
exit 0
EOF

chmod +x "${HOME}/bin/verify-repo-coverage.sh"

3. Execution & Coverage Checklist
 * Run the combined check: Execute ~/bin/verify-repo-coverage.sh and confirm Violations: 0 across audited commits and annotated tags in the audit-gpg-signatures.sh output.
 * Review ruleset states: Confirm remote-eligible repositories show FULL-ENFORCED (required_signatures and required_status_checks), while any plan- or permission-restricted repositories show SKIP-RESTRICTED (403/404) and remain protected by your local pre-push hook and audit-gpg-signatures.sh.
 * Remediate any gaps: Address any MISSING or PARTIAL-ENFORCED remote rulesets with apply-gpg-ruleset.sh, or re-sign any flagged commits/annotated tags before running your next archive and rclone sync.
apply-gpg-ruleset.shrun-github-suite.servicecompare-repo-rulesets.shTo execute the full end-to-end sequence—verifying repository coverage (GPG commits/annotated tags plus remote ruleset states), running run-github-suite.sh, checking local SHA-256 manifests, and validating the remote vault with rclone check—run the combined pipeline command below.
1. Full Coverage + Suite + SHA-256 + rclone check Command
Set REMOTE_VAULT to your configured rclone remote base path (such as crypt-github:github-archives) before running:
SRC_ROOT="${HOME}/src/github"
ARCHIVE_ROOT="${HOME}/github-archives"
REMOTE_VAULT="${RCLONE_REMOTE_BASE:-YOUR_RCLONE_REMOTE:github-archives}"

# 1. Verify local GPG signatures (commits + annotated tags) and remote ruleset coverage
"${HOME}/bin/audit-gpg-signatures.sh" "$SRC_ROOT"
"${HOME}/bin/compare-repo-rulesets.sh" --src-root "$SRC_ROOT"

# 2. Run the full backup, metadata export, audit, and sync suite
"${HOME}/bin/run-github-suite.sh"

# 3. Locate the newest snapshot directory (excluding pre-resign / pre-tag-resign directories)
LATEST_SNAP="$(
  find "$ARCHIVE_ROOT" -mindepth 1 -maxdepth 1 -type d \
    ! -name 'pre-resign-*' ! -name 'pre-tag-resign-*' \
    | sort | tail -n 1
)"
[ -n "$LATEST_SNAP" ] || { echo "ERROR: No snapshot directory found under ${ARCHIVE_ROOT}" >&2; exit 1; }

SNAP_NAME="$(basename "$LATEST_SNAP")"
echo "Latest snapshot: ${LATEST_SNAP}"

# 4. Verify local SHA-256 checksum manifest
(
  cd "$LATEST_SNAP"
  sha256sum -c SHA256SUMS
)

# 5. Verify remote rclone vault matches local snapshot
rclone check "$LATEST_SNAP" "${REMOTE_VAULT%/}/${SNAP_NAME}" --one-way --checkers=8

2. End-to-End Execution Sequence
 1. Verify GPG signatures and ruleset coverage
   1-2 min
   Run audit-gpg-signatures.sh and compare-repo-rulesets.sh to confirm zero commit/annotated-tag signature violations and inspect active vs. 403/404-restricted rulesets across re1Lucky365 and adpsnake7-beep.
   "${HOME}/bin/audit-gpg-signatures.sh" "${HOME}/src/github"
"${HOME}/bin/compare-repo-rulesets.sh" --src-root "${HOME}/src/github"

   Verification: audit-gpg-signatures.sh reports Violations: 0 and exits 0; compare-repo-rulesets.sh lists each repository's ruleset enforcement status.
 2. Run run-github-suite.sh to create the snapshot and sync
   1-3 min
   Execute ~/bin/run-github-suite.sh to produce the compressed .tar.zst mirrors, metadata bundles, checksum manifest, and remote sync.
   "${HOME}/bin/run-github-suite.sh"

   Verification: Exits 0 and writes a new timestamped snapshot folder under ${HOME}/github-archives/.
 3. Verify SHA-256 checksums and run rclone check
   1 min
   Validate SHA256SUMS against the local snapshot files and run rclone check against the matching remote snapshot directory.
   LATEST_SNAP="$(find "${HOME}/github-archives" -mindepth 1 -maxdepth 1 -type d ! -name 'pre-resign-*' ! -name 'pre-tag-resign-*' | sort | tail -n 1)"
SNAP_NAME="$(basename "$LATEST_SNAP")"
(cd "$LATEST_SNAP" && sha256sum -c SHA256SUMS)
rclone check "$LATEST_SNAP" "${REMOTE_VAULT%/}/${SNAP_NAME}" --one-way --checkers=8

   Verification: Every entry in SHA256SUMS prints : OK (sha256sum exits 0), and rclone check completes with 0 differences found (rclone exits 0).
> Failure alerting note: If any step above exits non-zero during a scheduled run-github-suite.service run, your OnFailure= unit (run-github-suite-alert.service) will capture the failure context and send your redacted alert payload while recording the full local log in ${HOME}/github-archives/systemd-failures.log.
> 
audit-gpg-signatures.shHere is a single quick command (and optional one-liner check) to inspect archive file sizes, local SHA-256 manifest status, and remote rclone objects for your newest snapshot under ~/github-archives (excluding pre-resign-* and pre-tag-resign-* directories).
1. Quick Snapshot & Remote Inspection Command
Set REMOTE_VAULT (for example crypt-github:github-archives) to your rclone remote base before running:
ARCHIVE_ROOT="${HOME}/github-archives"
REMOTE_VAULT="${RCLONE_REMOTE_BASE:-YOUR_RCLONE_REMOTE:github-archives}"

LATEST_SNAP="$(
  find "$ARCHIVE_ROOT" -mindepth 1 -maxdepth 1 -type d \
    ! -name 'pre-resign-*' ! -name 'pre-tag-resign-*' \
    | sort | tail -n 1
)"
SNAP_NAME="$(basename "$LATEST_SNAP")"

echo "=== Snapshot: ${LATEST_SNAP} ==="
ls -lh "$LATEST_SNAP"

echo ""
echo "=== Local SHA-256 Status ==="
(cd "$LATEST_SNAP" && sha256sum -c SHA256SUMS)

echo ""
echo "=== Remote rclone Objects (${REMOTE_VAULT%/}/${SNAP_NAME}) ==="
rclone lsl "${REMOTE_VAULT%/}/${SNAP_NAME}"

2. What Each Output Section Confirms
 * Archive file sizes (ls -lh): Shows .tar.zst mirror sizes, metadata bundles, and SHA256SUMS at a glance so you can spot empty or unexpectedly truncated archives.
 * SHA-256 status (sha256sum -c SHA256SUMS): Confirms every local archive and bundle matches its recorded SHA-256 digest (: OK).
 * Remote rclone objects (rclone lsl): Lists exact byte sizes, timestamps, and paths stored on your remote vault under ${SNAP_NAME} so you can cross-check presence and sizes against local files.
Here is a GitHub Actions workflow file (.github/workflows/verify-gpg-signatures.yml) that verifies commit GPG signatures on pushes and pull requests, using the verify-gpg-signatures job name so it matches the required_status_checks context in your ruleset JSON (~/.config/github/gpg-ruleset.json).
1. Workflow File (.github/workflows/verify-gpg-signatures.yml)
Because actions/checkout defaults to a shallow clone (fetch-depth: 1), set fetch-depth: 0 so the workflow has the full commit history needed to inspect every commit pushed to a branch or introduced in a pull request range.
name: Verify GPG Commit Signatures

on:
  push:
    branches-ignore:
      - "dependabot/**"
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  verify-gpg-signatures:
    name: verify-gpg-signatures
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository with full history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Import trusted GPG public keys (if configured)
        env:
          TRUSTED_GPG_KEYS: ${{ secrets.TRUSTED_GPG_KEYS }}
        run: |
          if [ -n "$TRUSTED_GPG_KEYS" ]; then
            printf '%s\n' "$TRUSTED_GPG_KEYS" | gpg --batch --import
            echo "Imported trusted GPG keys into runner keyring."
          else
            echo "No TRUSTED_GPG_KEYS secret set; checking embedded signature formatting and GitHub/public key availability."
          fi

      - name: Verify commit GPG signatures
        env:
          EVENT_NAME: ${{ github.event_name }}
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
          HEAD_SHA: ${{ github.sha }}
          BEFORE_SHA: ${{ github.event.before }}
        run: |
          set -euo pipefail

          if [ "$EVENT_NAME" = "pull_request" ]; then
            if [ -z "$BASE_SHA" ] || ! git rev-parse --verify "${BASE_SHA}^{commit}" >/dev/null 2>&1; then
              echo "::error::Missing or invalid pull_request base commit SHA: ${BASE_SHA}"
              exit 1
            fi
            rev_range="${BASE_SHA}..${HEAD_SHA}"
          else
            if [ -z "$BEFORE_SHA" ] || [ "$BEFORE_SHA" = "0000000000000000000000000000000000000000" ]; then
              # New branch or initial push: check reachable commits up to HEAD (bounded to recent history if needed)
              rev_range="$HEAD_SHA"
            else
              rev_range="${BEFORE_SHA}..${HEAD_SHA}"
            fi
          fi

          echo "Checking commits in range: ${rev_range}"
          commits=()
          while IFS= read -r sha; do
            [ -n "$sha" ] && commits+=("$sha")
          done < <(git rev-list "$rev_range")

          if [ "${#commits[@]}" -eq 0 ]; then
            echo "No new commits found in ${rev_range}."
            exit 0
          fi

          failures=0
          for sha in "${commits[@]}"; do
            # %G? -> G (good signature), B (bad signature), U (good untrusted), X (expired), Y (expired key), R (revoked), E (cannot check), N (no signature)
            # %GS -> signer key fingerprint/key ID; %GK -> signing key ID
            sig_status="$(git log -1 --format='%G?' "$sha")"
            signer_key="$(git log -1 --format='%GK' "$sha")"
            signer_name="$(git log -1 --format='%an <%ae>' "$sha")"
            subject="$(git log -1 --format='%s' "$sha")"

            case "$sig_status" in
              G)
                echo "[PASS] ${sha:0:12} (${signer_name}, key=${signer_key}) - Good signature: ${subject}"
                ;;
              U)
                if [ -n "${TRUSTED_GPG_KEYS:-}" ]; then
                  echo "::error::Commit ${sha:0:12} (${signer_name}) has an untrusted signature (status=U, key=${signer_key}): ${subject}"
                  failures=$((failures + 1))
                else
                  echo "[WARN] ${sha:0:12} (${signer_name}, key=${signer_key}) - Valid signature but key trust not established on runner (status=U): ${subject}"
                fi
                ;;
              *)
                echo "::error::Commit ${sha:0:12} (${signer_name}) failed GPG signature check (status=${sig_status}, key=${signer_key}): ${subject}"
                failures=$((failures + 1))
                ;;
            esac
          done

          if [ "$failures" -gt 0 ]; then
            echo "::error::${failures} commit(s) failed GPG signature verification."
            exit 1
          fi

          echo "All ${#commits[@]} commit(s) in ${rev_range} passed GPG signature verification."

2. Important Behavior & Ruleset Alignment
 * Job Name Matches Ruleset Context: Both the job key (verify-gpg-signatures) and name: verify-gpg-signatures match "context": "verify-gpg-signatures" in ~/.config/github/gpg-ruleset.json.
 * How Git's %G? Status Code Works:
   * G: Good signature made by a trusted key.
   * U: Valid signature from a key whose trust isn't locally marked ultimate/trusted on the runner (when TRUSTED_GPG_KEYS is provided, treating U as an error ensures only explicitly imported keys pass; without TRUSTED_GPG_KEYS, it warns rather than false-failing when a commit is signed with a key GitHub recognizes).
   * N, B, E, X, Y, R: No signature, bad signature, uncheckable signature, expired signature/key, or revoked key—all fail the check.
 * Merge Commit Note on Pull Requests: GitHub creates a synthetic merge ref for pull requests (refs/pull/<id>/merge) using github.sha, whereas github.event.pull_request.head.sha is the actual head commit of the PR branch. If you want to verify only the PR's head commits rather than a GitHub-synthesized merge commit, set HEAD_SHA: ${{ github.event.pull_request.head.sha || github.sha }}.
3. Rollout & Verification Steps
 1. Add the workflow file to a repository
   1 min
   Save .github/workflows/verify-gpg-signatures.yml in a signed commit and push it to test the workflow on a real push or pull request.
   mkdir -p .github/workflows
# Write .github/workflows/verify-gpg-signatures.yml and commit with GPG signing enabled
git status --short

   Verification: Workflow file is present at .github/workflows/verify-gpg-signatures.yml and valid YAML.
 2. Confirm check name appears on commits and PRs
   1-2 min
   Push a signed commit or open a PR and inspect the check run status in GitHub or via gh pr checks / gh api.
   gh run list --workflow=verify-gpg-signatures.yml --limit 5

   Verification: A check named verify-gpg-signatures runs on pushes and pull requests and completes with success for properly signed commits.
Here is a script (~/bin/rollout-gpg-workflow.sh) that copies .github/workflows/verify-gpg-signatures.yml into local Git repositories under ${HOME}/src/github (defaulting to re1Lucky365 and adpsnake7-beep), creates a dedicated feature branch (such as ci/verify-gpg-signatures), and creates a GPG-signed commit so your local pre-push hook and remote ruleset checks stay happy.
By default, the script leaves repositories with uncommitted changes or branches already on the target branch untouched unless you pass --allow-dirty or --force-branch, and it supports --dry-run so you can preview every target repo first.
1. Feature-Branch Rollout Script (~/bin/rollout-gpg-workflow.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --src-root PATH            Local clones directory (default: "${HOME}/src/github")
  --workflow-src PATH        Source workflow file (default: "${HOME}/src/github/.github/workflows/verify-gpg-signatures.yml")
  --accounts "acct1 acct2"   GitHub account directories to scan (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Feature branch name to create/use (default: "ci/verify-gpg-signatures")
  --commit-msg MSG           Commit message (default: "ci: add GPG commit signature verification workflow")
  --push                     Push the feature branch to origin after committing
  --allow-dirty              Allow working in a repo that already has uncommitted changes
  --dry-run                  Preview actions without modifying Git branches or files
  -h, --help                 Show this help
EOF
  exit 1
}

SRC_ROOT="${HOME}/src/github"
WORKFLOW_SRC="${HOME}/src/github/.github/workflows/verify-gpg-signatures.yml"
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
COMMIT_MSG="ci: add GPG commit signature verification workflow"
PUSH_BRANCH=false
ALLOW_DIRTY=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --workflow-src)
      WORKFLOW_SRC="${2:?Missing value for --workflow-src}"
      shift 2
      ;;
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --commit-msg)
      COMMIT_MSG="${2:?Missing value for --commit-msg}"
      shift 2
      ;;
    --push)
      PUSH_BRANCH=true
      shift
      ;;
    --allow-dirty)
      ALLOW_DIRTY=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

if [ ! -f "$WORKFLOW_SRC" ]; then
  echo "ERROR: Source workflow file not found: ${WORKFLOW_SRC}" >&2
  exit 1
fi

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

updated_count=0
skipped_count=0
failed_count=0

for acct in "${ACCOUNTS[@]}"; do
  acct_dir="${SRC_ROOT}/${acct}"
  [ -d "$acct_dir" ] || continue

  while IFS= read -r -d '' gitdir; do
    repo_dir="$(dirname "$gitdir")"
    repo_name="$(basename "$repo_dir")"
    rel_path="${repo_dir#${SRC_ROOT}/}"

    repo_in_filter "$repo_name" || continue

    # Skip the source file's own directory if it lives directly inside SRC_ROOT
    if [ "$WORKFLOW_SRC" = "${repo_dir}/.github/workflows/verify-gpg-signatures.yml" ]; then
      echo "[SKIP] ${rel_path}: already the source workflow file"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    if [ "$ALLOW_DIRTY" = false ]; then
      if ! git -C "$repo_dir" diff --quiet || ! git -C "$repo_dir" diff --cached --quiet || [ -n "$(git -C "$repo_dir" ls-files --others --exclude-standard)" ]; then
        echo "[SKIP] ${rel_path}: working tree is not clean (use --allow-dirty to override)"
        skipped_count=$((skipped_count + 1))
        continue
      fi
    fi

    dest_workflow="${repo_dir}/.github/workflows/verify-gpg-signatures.yml"

    if [ -f "$dest_workflow" ] && cmp -s "$WORKFLOW_SRC" "$dest_workflow"; then
      echo "[SKIP] ${rel_path}: .github/workflows/verify-gpg-signatures.yml is already identical"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    if [ "$DRY_RUN" = true ]; then
      echo "[DRY-RUN] ${rel_path}: checkout/create branch '${BRANCH_NAME}', copy workflow, commit (-S), push=${PUSH_BRANCH}"
      updated_count=$((updated_count + 1))
      continue
    fi

    if git -C "$repo_dir" show-ref --verify --quiet "refs/heads/${BRANCH_NAME}"; then
      git -C "$repo_dir" checkout "$BRANCH_NAME" >/dev/null
    else
      git -C "$repo_dir" checkout -b "$BRANCH_NAME" >/dev/null
    fi

    mkdir -p "${repo_dir}/.github/workflows"
    cp "$WORKFLOW_SRC" "$dest_workflow"
    git -C "$repo_dir" add ".github/workflows/verify-gpg-signatures.yml"

    if git -C "$repo_dir" diff --cached --quiet; then
      echo "[SKIP] ${rel_path}: no staged changes after copying workflow"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    # Enforce GPG signing (-S) so local pre-push hooks and branch rulesets pass
    if ! git -C "$repo_dir" commit -S -m "$COMMIT_MSG" >/dev/null; then
      echo "[ERROR] ${rel_path}: GPG commit signing failed (check gpg-agent / signing key)" >&2
      failed_count=$((failed_count + 1))
      continue
    fi

    if [ "$PUSH_BRANCH" = true ]; then
      if ! git -C "$repo_dir" push -u origin "$BRANCH_NAME" >/dev/null; then
        echo "[ERROR] ${rel_path}: failed pushing branch '${BRANCH_NAME}' to origin" >&2
        failed_count=$((failed_count + 1))
        continue
      fi
      echo "[OK] ${rel_path}: committed (GPG-signed) and pushed branch '${BRANCH_NAME}'"
    else
      echo "[OK] ${rel_path}: committed (GPG-signed) on branch '${BRANCH_NAME}' (not pushed; pass --push to push)"
    fi

    updated_count=$((updated_count + 1))
  done < <(find "$acct_dir" -mindepth 2 -maxdepth 2 -type d -name ".git" -print0 | sort -z)
done

echo "------------------------------------------------------------"
echo "Summary: Updated=${updated_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Rollout & Verification Steps
 1. Dry-run across your local repositories first
   1 min
   Make ~/bin/rollout-gpg-workflow.sh executable and run a --dry-run against ${HOME}/src/github to inspect which repositories will be updated or skipped.
   chmod +x "${HOME}/bin/rollout-gpg-workflow.sh"
"${HOME}/bin/rollout-gpg-workflow.sh" \
  --workflow-src "/path/to/your/reference/repo/.github/workflows/verify-gpg-signatures.yml" \
  --dry-run

   Verification: Prints [DRY-RUN] for target repos and [SKIP] for clean already-up-to-date or dirty repos without modifying any Git state.
 2. Create feature branches with GPG-signed commits (and optionally push)
   1-2 min
   Run the script without --dry-run to create ci/verify-gpg-signatures and record a GPG-signed commit in each target repository; add --push once you verify the local commit signature.
   "${HOME}/bin/rollout-gpg-workflow.sh" \
  --workflow-src "/path/to/your/reference/repo/.github/workflows/verify-gpg-signatures.yml" \
  --push

   Verification: Each updated repo shows branch ci/verify-gpg-signatures with a verified GPG signature (git log -1 --show-signature), and any pre-push hook checks pass cleanly.
Here is a script (~/bin/create-gpg-workflow-prs.sh) that scans your local Git repositories under ${HOME}/src/github (defaulting to re1Lucky365 and adpsnake7-beep), checks whether the ci/verify-gpg-signatures branch exists locally and/or on origin, skips repositories that already have an open PR from that branch, and uses gh pr create to open a pull request against each repository's default branch.
1. Pull Request Creation Script (~/bin/create-gpg-workflow-prs.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --src-root PATH            Local clones directory (default: "${HOME}/src/github")
  --accounts "acct1 acct2"   GitHub account directories to scan (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --title TITLE              PR title (default: "ci: add GPG commit signature verification workflow")
  --body BODY                PR description body
  --base BRANCH              Optional explicit base branch (default: auto-detected repo default branch)
  --push-missing             Push local feature branch to origin if not yet on remote
  --dry-run                  Preview which PRs would be created without calling gh pr create
  -h, --help                 Show this help
EOF
  exit 1
}

SRC_ROOT="${HOME}/src/github"
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
PR_TITLE="ci: add GPG commit signature verification workflow"
PR_BODY="Adds the \`verify-gpg-signatures\` GitHub Actions workflow to verify commit GPG signatures on pushes and pull requests."
BASE_BRANCH=""
PUSH_MISSING=false
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --src-root)
      SRC_ROOT="${2:?Missing value for --src-root}"
      shift 2
      ;;
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --title)
      PR_TITLE="${2:?Missing value for --title}"
      shift 2
      ;;
    --body)
      PR_BODY="${2:?Missing value for --body}"
      shift 2
      ;;
    --base)
      BASE_BRANCH="${2:?Missing value for --base}"
      shift 2
      ;;
    --push-missing)
      PUSH_MISSING=true
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v git >/dev/null 2>&1 || { echo "ERROR: git is required" >&2; exit 1; }

parse_github_slug() {
  local url="$1"
  local cleaned
  cleaned="$(printf '%s' "$url" | sed -E 's#^(https?://[^/]+/|git@[^:]+:)##; s#\.git$##')"
  if [[ "$cleaned" == */* ]]; then
    printf '%s\n' "$cleaned"
    return 0
  fi
  return 1
}

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

created_count=0
skipped_count=0
failed_count=0

for acct in "${ACCOUNTS[@]}"; do
  acct_dir="${SRC_ROOT}/${acct}"
  [ -d "$acct_dir" ] || continue

  while IFS= read -r -d '' gitdir; do
    repo_dir="$(dirname "$gitdir")"
    repo_name="$(basename "$repo_dir")"
    rel_path="${repo_dir#${SRC_ROOT}/}"

    repo_in_filter "$repo_name" || continue

    has_local_branch=false
    if git -C "$repo_dir" show-ref --verify --quiet "refs/heads/${BRANCH_NAME}"; then
      has_local_branch=true
    fi

    has_remote_branch=false
    if git -C "$repo_dir" ls-remote --exit-code --heads origin "$BRANCH_NAME" >/dev/null 2>&1; then
      has_remote_branch=true
    fi

    if [ "$has_local_branch" = false ] && [ "$has_remote_branch" = false ]; then
      echo "[SKIP] ${rel_path}: branch '${BRANCH_NAME}' not found locally or on origin"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    remote_url="$(git -C "$repo_dir" remote get-url origin 2>/dev/null || true)"
    if [ -z "$remote_url" ] || ! slug="$(parse_github_slug "$remote_url")"; then
      echo "[SKIP] ${rel_path}: missing or non-GitHub origin remote"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    target_base="$BASE_BRANCH"
    if [ -z "$target_base" ]; then
      target_base="$(gh repo view "$slug" --json defaultBranchRef --jq '.defaultBranchRef.name // empty' 2>/dev/null || true)"
      target_base="${target_base:-main}"
    fi

    existing_pr_url="$(
      gh pr list -R "$slug" --head "$BRANCH_NAME" --state open --json url --jq '.[0].url // empty' 2>/dev/null || true
    )"
    if [ -n "$existing_pr_url" ]; then
      echo "[SKIP] ${rel_path} (${slug}): open PR already exists (${existing_pr_url})"
      skipped_count=$((skipped_count + 1))
      continue
    fi

    if [ "$DRY_RUN" = true ]; then
      echo "[DRY-RUN] ${rel_path} (${slug}): would create PR '${BRANCH_NAME}' -> '${target_base}' (remote_branch=${has_remote_branch}, push_missing=${PUSH_MISSING})"
      created_count=$((created_count + 1))
      continue
    fi

    if [ "$has_remote_branch" = false ]; then
      if [ "$PUSH_MISSING" = true ] && [ "$has_local_branch" = true ]; then
        if ! git -C "$repo_dir" push -u origin "$BRANCH_NAME" >/dev/null; then
          echo "[ERROR] ${rel_path} (${slug}): failed pushing '${BRANCH_NAME}' to origin" >&2
          failed_count=$((failed_count + 1))
          continue
        fi
      else
        echo "[SKIP] ${rel_path} (${slug}): branch '${BRANCH_NAME}' not on origin (pass --push-missing to push)"
        skipped_count=$((skipped_count + 1))
        continue
      fi
    fi

    if pr_url="$(
      gh pr create \
        -R "$slug" \
        --base "$target_base" \
        --head "$BRANCH_NAME" \
        --title "$PR_TITLE" \
        --body "$PR_BODY" 2>&1
    )"; then
      echo "[OK] ${rel_path} (${slug}): created PR -> ${pr_url}"
      created_count=$((created_count + 1))
    else
      echo "[ERROR] ${rel_path} (${slug}): gh pr create failed (${pr_url})" >&2
      failed_count=$((failed_count + 1))
    fi
  done < <(find "$acct_dir" -mindepth 2 -maxdepth 2 -type d -name ".git" -print0 | sort -z)
done

echo "------------------------------------------------------------"
echo "Summary: Created=${created_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Preview & Create Pull Requests
 1. Dry-run to confirm target branches and existing PRs
   1 min
   Make ~/bin/create-gpg-workflow-prs.sh executable and run with --dry-run first so you can verify which repositories have ci/verify-gpg-signatures ready and no duplicate open PRs.
   chmod +x "${HOME}/bin/create-gpg-workflow-prs.sh"
"${HOME}/bin/create-gpg-workflow-prs.sh" --push-missing --dry-run

   Verification: Prints [DRY-RUN] lines with target base branches (main, etc.) or [SKIP] reasons without creating any pull requests.
 2. Create pull requests across updated repositories
   1-2 min
   Run without --dry-run to push any unpushed ci/verify-gpg-signatures branches (with --push-missing) and open the pull requests via gh pr create.
   "${HOME}/bin/create-gpg-workflow-prs.sh" --push-missing

   Verification: Prints [OK] with each new PR URL and exits 0 when no gh pr create errors occur.
create-gpg-workflow-prs.shTo monitor status and CI check results across all open ci/verify-gpg-signatures pull requests for re1Lucky365 and adpsnake7-beep, run a loop that queries gh pr list --head ci/verify-gpg-signatures --state open for each repository and passes matching PR numbers or URLs into gh pr checks.
1. Quick PR & CI Check Status Command
Run this directly in your shell to list every open ci/verify-gpg-signatures pull request and print its live check states (pass, fail, pending, etc.):
ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
BRANCH_NAME="ci/verify-gpg-signatures"

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_url pr_title; do
      [ -n "$pr_num" ] || continue
      echo "=== ${slug} #${pr_num}: ${pr_title} (${pr_url}) ==="
      gh pr checks "$pr_num" -R "$slug" --required 2>/dev/null \
        || gh pr checks "$pr_num" -R "$slug" 2>&1 \
        || true
      echo ""
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,url,title \
        --jq '.[] | [.number, .url, .title] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 200 --json name --jq '.[].name' 2>/dev/null || true)
done

2. Compact One-Line Summary Across Open PRs
If you prefer a single compact line per open PR showing its number, URL, and rollup check state (SUCCESS, FAILURE, PENDING, ERROR) without per-check rows:
for acct in re1Lucky365 adpsnake7-beep; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    gh pr list -R "${acct}/${repo}" \
      --head "ci/verify-gpg-signatures" \
      --state open \
      --json number,url,statusCheckRollup \
      --jq --arg repo "${acct}/${repo}" '
        .[] |
        ([.statusCheckRollup[]?.conclusion // .statusCheckRollup[]?.state // "NONE"] | unique | join(",")) as $states |
        "\($repo) #\(.number) | checks=[\($states)] | \(.url)"
      ' 2>/dev/null || true
  done < <(gh repo list "$acct" --limit 200 --json name --jq '.[].name' 2>/dev/null || true)
done

3. Helpful Flags for gh pr checks
 * Watch until completion: Add --watch to gh pr checks "$pr_num" -R "$slug" --watch to poll a single PR until all checks finish.
 * Filter to required checks only: Keep --required when you only care about status checks required by your branch ruleset (such as verify-gpg-signatures).
Here is a script (~/bin/merge-passing-gpg-prs.sh) that scans open pull requests from ci/verify-gpg-signatures across re1Lucky365 and adpsnake7-beep, verifies that checks exist and have passed without pending/failing/cancelled states via gh pr checks, and merges eligible PRs via gh pr merge (defaulting to --squash and --delete-branch, with --dry-run support).
1. Safe Merge Script (~/bin/merge-passing-gpg-prs.sh)
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; ((.bucket // "") | ascii_downcase) == "fail" or ((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE")) then
      "FAIL"
    elif any(.[]; ((.bucket // "") | ascii_downcase) == "pending" or ((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING")) then
      "PENDING"
    elif all(.[]; ((.bucket // "") | ascii_downcase) == "pass" or ((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED")) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 200 --json name --jq '.[].name' 2>/dev/null || true)
done

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

2. Preview & Merge Workflow
 1. Dry-run to verify which open PRs pass checks
   1 min
   Make ~/bin/merge-passing-gpg-prs.sh executable and run with --dry-run to see which ci/verify-gpg-signatures PRs are ready to merge versus still pending, failing, or missing checks.
   chmod +x "${HOME}/bin/merge-passing-gpg-prs.sh"
"${HOME}/bin/merge-passing-gpg-prs.sh" --dry-run

   Verification: Prints [DRY-RUN] only for PRs whose gh pr checks state is passing and [SKIP] with a reason for any others.
 2. Merge passing pull requests
   1-2 min
   Run without --dry-run to merge passing pull requests and clean up the ci/verify-gpg-signatures branches on the remotes.
   "${HOME}/bin/merge-passing-gpg-prs.sh"

   Verification: Prints [OK] with each merged PR URL and exits 0 when no merge errors occur.
> Local clone sync tip: After merging ci/verify-gpg-signatures on GitHub with --delete-branch, run git pull --ff-only and git branch -d ci/verify-gpg-signatures (plus git fetch --prune) inside your local clones under ${HOME}/src/github before your next run-github-suite.sh backup so your local default branch and archive snapshot include the merged workflow.
> 
run-github-suite.shYour script is well-structured overall, with good set -euo pipefail discipline, --dry-run support, and clear status reporting. However, there is one subtle jq precedence bug in evaluate_pr_checks that can cause passing checks with bucket == "fail" to misbehave, along with a few practical edge cases worth tightening up.
1. Bug: jq Operator Precedence in FAIL Detection
In evaluate_pr_checks, this expression:
((.bucket // "") | ascii_downcase) == "fail" or ((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE")

In jq, | binds tighter than or, so jq parses it as:
( <bucket == "fail"> or <upcased state> ) | IN("FAILURE", ...)

If <bucket == "fail"> is true, true | IN(...) evaluates to false, causing FAIL to be missed and fall through to PENDING (the final else branch).
Fix: Wrap the full boolean condition in parentheses:
elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then

2. Other Edge Cases to Watch For
 * Repo list cap (--limit 200): If either account ever exceeds 200 repositories, gh repo list will stop at 200. Bumping --limit 1000 (or using --paginate where supported) avoids silent omissions.
 * Fork vs. same-repo PRs: gh pr list --head "$BRANCH_NAME" matches same-repo branches cleanly. If any PRs originate from a fork, --head may need <fork-owner>:<branch> or a --jq filter on .headRefName == $branch, and --delete-branch may not be able to delete a branch on a fork you do not own.
 * Branch protection / required reviews: gh pr merge will fail ([ERROR]) if a repository requires an approving review, up-to-date branch, or specific required check context that isn't satisfied yet—which your script already catches and counts under failed_count.
Here is the updated ~/bin/merge-passing-gpg-prs.sh with the jq precedence fix, --limit 1000 on gh repo list, and a new --json-summary FILE option that writes structured run results and counts to disk while keeping human-readable output on stdout/stderr.
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0

Example Usage with --json-summary
"${HOME}/bin/merge-passing-gpg-prs.sh" \
  --dry-run \
  --json-summary "${HOME}/tmp/gpg-pr-dry-run.json"

jq . "${HOME}/tmp/gpg-pr-dry-run.json"

#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  --accounts "acct1 acct2"   GitHub accounts to check (default: "re1Lucky365 adpsnake7-beep")
  --repos "repo1 repo2"      Optional space-separated repo names filter
  --branch NAME              Source feature branch (default: "ci/verify-gpg-signatures")
  --method METHOD            Merge method: squash | merge | rebase (default: "squash")
  --keep-branch              Do not pass --delete-branch on merge
  --require-checks           Require at least one check to be present (default: true)
  --allow-empty-checks       Allow merging even if a repo has no status checks reported
  --dry-run                  Preview which PRs would merge without calling gh pr merge
  --json-summary FILE        Write structured run summary JSON to FILE
  -h, --help                 Show this help
EOF
  exit 1
}

ACCOUNTS=("re1Lucky365" "adpsnake7-beep")
EXPLICIT_REPOS=""
BRANCH_NAME="ci/verify-gpg-signatures"
MERGE_METHOD="squash"
DELETE_BRANCH=true
REQUIRE_CHECKS=true
DRY_RUN=false
JSON_SUMMARY_FILE=""

while [ "$#" -gt 0 ]; do
  case "$1" in
    --accounts)
      read -r -a ACCOUNTS <<< "${2:?Missing value for --accounts}"
      shift 2
      ;;
    --repos)
      EXPLICIT_REPOS="${2:?Missing value for --repos}"
      shift 2
      ;;
    --branch)
      BRANCH_NAME="${2:?Missing value for --branch}"
      shift 2
      ;;
    --method)
      MERGE_METHOD="${2:?Missing value for --method}"
      shift 2
      ;;
    --keep-branch)
      DELETE_BRANCH=false
      shift
      ;;
    --require-checks)
      REQUIRE_CHECKS=true
      shift
      ;;
    --allow-empty-checks)
      REQUIRE_CHECKS=false
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --json-summary)
      JSON_SUMMARY_FILE="${2:?Missing value for --json-summary}"
      shift 2
      ;;
    -h|--help)
      usage
      ;;
    *)
      echo "Unknown argument: $1" >&2
      usage
      ;;
  esac
done

case "$MERGE_METHOD" in
  squash|merge|rebase) ;;
  *)
    echo "ERROR: Invalid --method '${MERGE_METHOD}' (use squash, merge, or rebase)" >&2
    exit 1
    ;;
esac

command -v gh >/dev/null 2>&1 || { echo "ERROR: gh CLI is required" >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required" >&2; exit 1; }

repo_in_filter() {
  local name="$1"
  [ -z "$EXPLICIT_REPOS" ] && return 0
  for r in $EXPLICIT_REPOS; do
    [ "$r" = "$name" ] && return 0
  done
  return 1
}

# Evaluates `gh pr checks <num> -R <slug> --json name,state,bucket`
# Outputs one of: PASS | PENDING | FAIL | NO_CHECKS | ERROR
evaluate_pr_checks() {
  local slug="$1"
  local pr_num="$2"
  local checks_json rc=0

  checks_json="$(gh pr checks "$pr_num" -R "$slug" --json name,state,bucket 2>&1)" || rc=$?
  if [ "$rc" -ne 0 ]; then
    # When there are no checks on a PR, gh pr checks exits non-zero with a message
    if printf '%s' "$checks_json" | grep -qiE 'no checks reported|no status checks'; then
      echo "NO_CHECKS"
      return 0
    fi
    echo "ERROR"
    return 1
  fi

  printf '%s' "$checks_json" | jq -r '
    if length == 0 then
      "NO_CHECKS"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "fail") or (((.state // "") | ascii_upcase) | IN("FAILURE","ERROR","CANCELLED","TIMED_OUT","ACTION_REQUIRED","STARTUP_FAILURE"))) then
      "FAIL"
    elif any(.[]; (((.bucket // "") | ascii_downcase) == "pending") or (((.state // "") | ascii_upcase) | IN("PENDING","QUEUED","IN_PROGRESS","EXPECTED","WAITING"))) then
      "PENDING"
    elif all(.[]; (((.bucket // "") | ascii_downcase) == "pass") or (((.state // "") | ascii_upcase) | IN("SUCCESS","NEUTRAL","SKIPPED"))) then
      "PASS"
    else
      "PENDING"
    end
  '
}

merged_count=0
skipped_count=0
failed_count=0
results_json='[]'

record_result() {
  local repo_slug="$1" pr_num="$2" pr_title="$3" pr_url="$4" check_status="$5" outcome="$6" detail="$7"
  results_json="$(
    jq -n \
      --argjson arr "$results_json" \
      --arg repo "$repo_slug" \
      --argjson pr_number "$pr_num" \
      --arg title "$pr_title" \
      --arg url "$pr_url" \
      --arg check_status "$check_status" \
      --arg outcome "$outcome" \
      --arg detail "$detail" \
      '$arr + [{
        repo: $repo,
        pr_number: $pr_number,
        title: $title,
        url: $url,
        check_status: $check_status,
        outcome: $outcome,
        detail: $detail
      }]'
  )"
}

for acct in "${ACCOUNTS[@]}"; do
  while IFS= read -r repo; do
    [ -n "$repo" ] || continue
    repo_in_filter "$repo" || continue
    slug="${acct}/${repo}"

    while IFS=$'\t' read -r pr_num pr_title pr_url; do
      [ -n "$pr_num" ] || continue

      status="$(evaluate_pr_checks "$slug" "$pr_num" || echo "ERROR")"

      case "$status" in
        PASS)
          if [ "$DRY_RUN" = true ]; then
            echo "[DRY-RUN] ${slug} #${pr_num} (${pr_title}): checks PASSED -> would merge (--${MERGE_METHOD}, delete_branch=${DELETE_BRANCH})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge via ${MERGE_METHOD}"
            continue
          fi

          merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
          if [ "$DELETE_BRANCH" = true ]; then
            merge_args+=(--delete-branch)
          fi

          if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
            echo "[OK] ${slug} #${pr_num}: merged via ${MERGE_METHOD} (${pr_url})"
            merged_count=$((merged_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged via ${MERGE_METHOD}"
          else
            echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
            failed_count=$((failed_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
          fi
          ;;
        NO_CHECKS)
          if [ "$REQUIRE_CHECKS" = false ]; then
            if [ "$DRY_RUN" = true ]; then
              echo "[DRY-RUN] ${slug} #${pr_num}: no checks reported (--allow-empty-checks) -> would merge (--${MERGE_METHOD})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "dry_run" "would merge with no checks via ${MERGE_METHOD}"
              continue
            fi

            merge_args=("$pr_num" -R "$slug" "--${MERGE_METHOD}")
            [ "$DELETE_BRANCH" = true ] && merge_args+=(--delete-branch)

            if merge_out="$(gh pr merge "${merge_args[@]}" 2>&1)"; then
              echo "[OK] ${slug} #${pr_num}: merged with no checks (${pr_url})"
              merged_count=$((merged_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "merged" "merged with no checks via ${MERGE_METHOD}"
            else
              echo "[ERROR] ${slug} #${pr_num}: gh pr merge failed (${merge_out})" >&2
              failed_count=$((failed_count + 1))
              record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "$merge_out"
            fi
          else
            echo "[SKIP] ${slug} #${pr_num}: no checks reported yet (use --allow-empty-checks to override)"
            skipped_count=$((skipped_count + 1))
            record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "no checks reported"
          fi
          ;;
        PENDING)
          echo "[SKIP] ${slug} #${pr_num}: checks still PENDING/IN_PROGRESS"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks pending"
          ;;
        FAIL)
          echo "[SKIP] ${slug} #${pr_num}: one or more checks FAILED/CANCELLED"
          skipped_count=$((skipped_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "skipped" "checks failed or cancelled"
          ;;
        *)
          echo "[ERROR] ${slug} #${pr_num}: could not evaluate gh pr checks output" >&2
          failed_count=$((failed_count + 1))
          record_result "$slug" "$pr_num" "$pr_title" "$pr_url" "$status" "error" "could not evaluate gh pr checks output"
          ;;
      esac
    done < <(
      gh pr list -R "$slug" \
        --head "$BRANCH_NAME" \
        --state open \
        --json number,title,url \
        --jq '.[] | [.number, .title, .url] | @tsv' 2>/dev/null || true
    )
  done < <(gh repo list "$acct" --limit 1000 --json name --jq '.[].name' 2>/dev/null || true)
done

if [ -n "$JSON_SUMMARY_FILE" ]; then
  mkdir -p "$(dirname "$JSON_SUMMARY_FILE")"
  jq -n \
    --arg generated_at "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --arg branch "$BRANCH_NAME" \
    --arg method "$MERGE_METHOD" \
    --argjson dry_run "$DRY_RUN" \
    --argjson delete_branch "$DELETE_BRANCH" \
    --argjson require_checks "$REQUIRE_CHECKS" \
    --argjson merged "$merged_count" \
    --argjson skipped "$skipped_count" \
    --argjson failed "$failed_count" \
    --argjson pull_requests "$results_json" \
    '{
      generated_at: $generated_at,
      config: {
        branch: $branch,
        method: $method,
        dry_run: $dry_run,
        delete_branch: $delete_branch,
        require_checks: $require_checks
      },
      counts: {
        merged: $merged,
        skipped: $skipped,
        failed: $failed
      },
      pull_requests: $pull_requests
    }' > "$JSON_SUMMARY_FILE"
fi

echo "------------------------------------------------------------"
echo "Summary: Merged=${merged_count} | Skipped=${skipped_count} | Failed=${failed_count}"

if [ "$failed_count" -gt 0 ]; then
  exit 1
fi
exit 0


[![PyPI](https://img.shields.io/pypi/v/morphogen.svg)](https://pypi.org/project/morphogen/)
[![Python](https://img.shields.io/pypi/pyversions/morphogen.svg)](https://pyp
i.org/project/morphogen/)
[![CI](https://github.com/jdirr/morphoGen/actions/workflows/ci.yml/badge.svg)](https://github.com/jdirr/morphoGen/actions/workflows/ci.yml)
[![Documentation](https://img.shields.io/badge/docs-jdirr.github.io%2FmorphoGen-1f4e79.svg)](https://jdirr.github.io/morphoGen/)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24340315.svg)](https://doi.org/10.5281/zenodo.24340315)
[![Licence: GPL v3 or later](https://img.shields.io/badge/licence-GPL--3.0--or--later-blue.svg)](https://github.com/jdirr/morphoGen/blob/main/LICENSE)

**Virtual microstructures for the mechanics and physics of materials.**

morphoGen generates random and architected media from models of mathematical morphology
and stochastic geometry, describes them exactly, measures them against theory, computes
their apparent properties up to their fracture, and tells how large a volume must be to
represent them. It is a Python package with a command line and a graphical interface in
the web browser.

<p align="center">
  <img src="https://raw.githubusercontent.com/jdirr/morphoGen/main/docs/_static/gallery/boolean_spheres.png" width="16%" alt="a Boolean model of spheres">
  <img src="https://raw.githubusercontent.com/jdirr/morphoGen/main/docs/_static/gallery/voronoi.png" width="16%" alt="a Voronoi polycrystal">
  <img src="https://raw.githubusercontent.com/jdirr/morphoGen/main/docs/_static/gallery/random_walk_fibers.png" width="16%" alt="fibres packed without overlap">
  <img src="https://raw.githubusercontent.com/jdirr/morphoGen/main/docs/_static/gallery/tpms.png" width="16%" alt="a gyroid">
  <img src="https://raw.githubusercontent.com/jdirr/morphoGen/main/docs/_static/gallery/trabecular_bone.png" width="16%" alt="trabecular bone">
  <img src="https://raw.githubusercontent.com/jdirr/morphoGen/main/docs/_static/gallery/reaction_diffusion.png" width="16%" alt="a pattern of reaction and diffusion">
</p>

## What it does

A study follows one chain, and each step is a part of the package:

| Step | What morphoGen does |
|---|---|
| **Generate** | 40 models: Boolean models, packings, fibres, tessellations, Gaussian fields, dead leaves, automata and growths, lattices, foams, triply periodic surfaces, bio-inspired structures; periodic or not, combined, nested |
| **Describe** | an exact description of each realization (centres, axes, cells) and an implicit function for each phase, from which voxel images, surfaces and conforming meshes are cut at any resolution |
| **Measure** | fractions, specific surface, covariance, chords, tortuosity, percolation thresholds, stereology..., each beside the closed form of its model when there is one |
| **Compute** | elastic, thermal, electrical, dielectric, magnetic and diffusive properties, their couplings, harmonic regimes, waves, flow and sound: an FFT solver on voxels (CPU or GPU), finite elements on conforming meshes, extended finite elements |
| **Load** | plasticity, crystal plasticity, viscoelasticity, hyperelasticity, damage and MFront laws along loading paths, in small or finite strains; macroscopic laws identified, reduced or learned from the responses |
| **Break** | the phase field of fracture, cohesive zones, cracks of any front, ductile growth; contact and friction; high-cycle fatigue |
| **Size the RVE** | the statistics of Kanit et al. (2003): the variance against the volume, the integral range, the volume and the number of realizations for a precision |
| **Design** | topology optimization of periodic cells, compatible assemblies of building blocks, rigidity of lattices, isogeometric shape optimization |

Two application modules build on this chain: metal sheets architectured by localized
laser treatment (`morphogen.laser`) and explicit dynamics with OpenRadioss
(`morphogen.dynamics`). Meshes, images, behaviours and results go to Abaqus, CalculiX,
AMITEX_FFTP, MFront, Z-set, Gmsh, Neper, DAMASK, FEniCSx and ParaView.

## Installation

```bash
python -m pip install "morphogen[all]"
```

morphoGen needs Python 3.12 or newer, on Windows, macOS or Linux; every dependency comes
as a binary wheel. The [installation guide](https://jdirr.github.io/morphoGen/installation.html)
gives the environments, the installation from the sources and the optional components
(NVIDIA GPUs, the stand-alone application, OpenRadioss, MFront).

## A first study

A model, a realization, its voxels, a measure against theory and three apparent
properties, in a second:

```python
import morphogen as mg
from morphogen.homogenization import Material, homogenize

model = mg.models.BooleanSpheres(radius=1.0, volume_fraction=0.3)
micro = model.generate(mg.Domain.cube(8.0, periodic=True), seed=1)  # centres and radii
image = micro.voxelize(shape=32)  # 32³ voxels: phase 0 (matrix), phase 1 (spheres)
theory = model.theory()
print(mg.analysis.volume_fraction(image), theory["volume_fraction"])  # 0.29 for 0.3

phases = {
    0: Material(young=3.0, poisson=0.35, conductivity=0.2),  # a polymer: GPa, W/(m K)
    1: Material(young=70.0, poisson=0.22, conductivity=1.0),  # glass
}
properties = ["bulk_modulus", "shear_modulus", "conductivity"]
out = homogenize(image, phases, properties=properties)
print({name: result.value for name, result in out.items()})  # 5.89, 2.59 (GPa), 0.33
```

The same from a configuration file, or from forms:

```bash
morphogen template homogenization_spheres -o study.yaml  # an example to start from
morphogen run study.yaml                                 # realizations and properties
morphogen gui                                            # the interface, in the browser
```

## Documentation

The documentation is online at [jdirr.github.io/morphoGen](https://jdirr.github.io/morphoGen/):

* the [tutorials](https://jdirr.github.io/morphoGen/tutorials.html) follow whole studies,
  and the [recipes](https://jdirr.github.io/morphoGen/howto.html) answer short questions;
* the [user guide](https://jdirr.github.io/morphoGen/guide.html) has a page for each step
  of a study, and the [gallery](https://jdirr.github.io/morphoGen/gallery.html) an example
  for each template;
* the [API reference](https://jdirr.github.io/morphoGen/api.html) documents the package,
  and the [theory notes](https://jdirr.github.io/morphoGen/theory.html) the formulas
  implemented and how they are checked.

## Citing

If morphoGen serves your work, please cite it by the DOI of all its versions,
[10.5281/zenodo.24340315](https://doi.org/10.5281/zenodo.24340315), with the papers of the
methods you use;
[CITATION.cff](https://github.com/jdirr/morphoGen/blob/main/CITATION.cff) and the page
[Citing morphoGen](https://jdirr.github.io/morphoGen/citing.html) give the reference.

## Contributing

Questions, bug reports and proposals are welcome as
[issues](https://github.com/jdirr/morphoGen/issues); changes go through pull requests
([CONTRIBUTING.md](https://github.com/jdirr/morphoGen/blob/main/CONTRIBUTING.md)).

## Licence

Copyright © 2026 Justin Dirrenberger. morphoGen is free software: you can redistribute
it and/or modify it under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or (at your option) any later
version. See [LICENSE](https://github.com/jdirr/morphoGen/blob/main/LICENSE).
