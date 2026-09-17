# Bombastic SBOM Generator

This skill helps you create comprehensive Software Bill of Materials documents for compliance, supply chain security, license tracking and dependency management.

## When to Use

Generate an SBOM when you need to:
- **Compliance**: Document all components and dependencies for regulatory audits
- **Supply chain security**: Identify and track vulnerable components
- **Dependency management**: Understand what's in your codebase at a point in time
- **Vulnerability scanning**: Cross-reference dependencies against known vulnerabilities
- **Audit trails**: Create timestamped records of software composition

## Workflow Overview

The skill works in three main phases:

1. **Extract dependencies** from your source code (Git repo or manifests)
2. **Generate SBOMs** in your chosen format(s) with metadata
3. **Validate and optionally check** for known vulnerabilities

## Phase 1: Extracting Dependencies

### From a Git Repository

Use syft to automatically scan the repository:

```bash
syft <repo-path> -o json > sbom-raw.json
```

This discovers dependencies from all detected package managers and lock files in the repo.

### From Package Manifests

If you prefer manual extraction or have specific manifests:

**Node.js (package.json):**
- Use `npm list --json` or `yarn list --json` to get dependency tree
- Include lock file hash (from package-lock.json or yarn.lock)

**Python (requirements.txt / Pipenv / Poetry):**
- Extract from requirements.txt or poetry.lock
- For each dependency, note version, Python version compatibility

**Java (pom.xml, build.gradle):**
- Use Maven's dependency tree plugin: `mvn dependency:tree -DoutputFile=deps.txt`
- Or gradle: `./gradlew dependencies`

**Ruby (Gemfile):**
- Use `bundle list --path` to get versions
- Check Gemfile.lock for exact pinned versions

**Go (go.mod, go.sum):**
- Parse go.mod for direct dependencies
- Cross-reference with go.sum for hashes

### Enriching with Metadata

For each component, collect:
- **Name** (package/component identifier)
- **Version** (exact semver or commit hash)
- **License** ⭐ (SPDX identifier; check LICENSE files or package metadata) — **This is critical for compliance and legal review**
- **Hash** (SHA256 or other hash of the component; from lock files or package registries)
- **Source URL** (repository URL, package registry URL)
- **Vulnerability data** (optional; from CVE databases if checking)

**License Tracking is Essential:** Every component MUST include its license. This is required for:
- GPL/LGPL compliance (copy-left license requirements)
- Patent indemnification clauses
- License conflict detection (e.g., GPL + proprietary)
- Regulatory compliance (GDPR, HIPAA, SOX require license audit trails)

## Phase 2: Generating SBOMs

### Detail Levels

**Minimal** — only essential fields:
- Component name, version, license
- Excludes: hashes, URLs, vulnerability details

**Standard** — complete audit trail (default):
- All minimal fields plus: hashes, source URLs, timestamps, metadata
- Includes vulnerability data if available

### Output Formats

#### Markdown

Readable, human-friendly format. Best for documentation and quick reviews.

**Structure:**
```markdown
# Software Bill of Materials

- **Generated**: [ISO timestamp]
- **Author**: [user/tool name]
- **Tool**: SBOM Generator
- **Detail Level**: Standard

## Components

| Name | Version | License | Hash (SHA256) | Source |
| --- | --- | --- | --- | --- |
| component-name | 1.2.3 | MIT | abc123... | https://... |

## Vulnerabilities (if applicable)

| Component | CVE | Severity | Description |
| --- | --- | --- | --- |
| ... | ... | ... | ... |
```

#### SPDX JSON (Latest: 2.3)

Standard format for software bill of materials, widely used in compliance workflows.

**Key fields to include:**
- `spdxVersion`, `dataLicense`, `NOASSERTION`
- `creationInfo` (creator, created timestamp, licenseListVersion)
- `packages` array with:
  - `name`, `version`, `downloadLocation`
  - `filesAnalyzed` (true/false)
  - `licenseDeclared`, `licenseConcluded`
  - `externalRefs` for hashes and source URLs
- `relationships` array linking packages

**Example minimal package entry:**
```json
{
  "name": "lodash",
  "version": "4.17.21",
  "downloadLocation": "https://registry.npmjs.org/lodash/4.17.21",
  "filesAnalyzed": false,
  "licenseDeclared": "MIT",
  "externalRefs": [
    {
      "referenceCategory": "PACKAGE-MANAGER",
      "referenceType": "purl",
      "referenceLocator": "pkg:npm/lodash@4.17.21"
    }
  ]
}
```

#### CycloneDX JSON (Latest: 1.5)

Supply-chain security focused format, emphasizes vulnerability tracking.

**Key structure:**
```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "serialNumber": "urn:uuid:...",
  "version": 1,
  "metadata": {
    "timestamp": "2026-09-17T...",
    "tools": [{
      "vendor": "Anthropic",
      "name": "SBOM Generator",
      "version": "1.0"
    }],
    "authors": [{"name": "..."}]
  },
  "components": [
    {
      "type": "library",
      "name": "lodash",
      "version": "4.17.21",
      "licenses": [{"license": {"name": "MIT"}}],
      "purl": "pkg:npm/lodash@4.17.21",
      "hashes": [
        {"alg": "SHA-256", "content": "abc123..."}
      ],
      "vulnerabilities": [
        {
          "ref": "CVE-2021-23337",
          "id": "CVE-2021-23337",
          "severity": "high"
        }
      ]
    }
  ]
}
