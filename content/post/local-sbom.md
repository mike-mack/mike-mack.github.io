---
title: "Local SBOM Toolchains for Dev Environments"
date: 2025-12-10T09:50:22-05:00
draft: false
tags: ["security", "dependencies", "supply-chain"]
---

Modern applications pull in hundreds of dependencies. Security teams demand visibility, but developers often discover vulnerabilities too late, during CI/CD or even in production. The fix: catch issues locally by building a small, open-source SBOM toolchain.

If your dev environment is blind to dependencies, you’ll fail security checks later. That’s wasted time, risk, and frustration. A local SBOM setup gives you visibility and actionable feedback before anything hits CI/CD or production.

Minimal Viable SBOM Stack
Use tools that are lightweight, free, and production-proven:
- Syft – generates SBOMs from directories, images, or binaries
- Grype – scans SBOMs for vulnerabilities
- Optional: CycloneDX support for standardized SBOMs

Flow: local project → Syft generates SBOM → Grype scans → issues surfaced to dev

###### Install Syft and Grype:

macOS
```
brew install anchore/syft/syft
brew install anchore/grype
```

Linux
```
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh
```

###### Generate an SBOM for your project:
```
syft dir:. -o json > sbom.json
```

Scan the SBOM for vulnerabilities:
```
grype sbom:sbom.json
```


Review output. Integrate this into a pre-commit hook or local CI check to ensure dependencies stay clean.

Alternatively you can use this script to automate the process and create a vulnerability report either on demand or time-stamped.








```bash
#!/usr/bin/env bash
set -euo pipefail

TS=$(date -u +%Y%m%d-%H%M%S)
BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SBOM_DIR="$BASE_DIR/../docs/sboms"
VULN_DIR="$BASE_DIR/../docs/vulns"

##
# Usage:
#   ./scripts/sbom.sh [--timestamp|-t]
#
# Behavior:
#   - Without flag: generates docs/sboms/sbom.json and docs/vulns/vulnerability-report.{json,txt}
#   - With flag: appends UTC timestamp to all outputs (SBOM and vulnerability reports)
##

# Default SBOM filename
SBOM_FILE="$SBOM_DIR/sbom.json"

# If "--timestamp" or "-t" is passed, append timestamp to the SBOM filename
if [[ "${1:-}" == "--timestamp" || "${1:-}" == "-t" ]]; then
    SBOM_FILE="$SBOM_DIR/sbom-${TS}.json"
    echo "Generating timestamped SBOM: $(basename "$SBOM_FILE")"
else
    echo "Generating default SBOM: $(basename "$SBOM_FILE")"
fi

# Ensure output directories exist
mkdir -p "$SBOM_DIR" "$VULN_DIR"

# Determine vulnerability report filenames (timestamped only when flag is provided)
if [[ "${1:-}" == "--timestamp" || "${1:-}" == "-t" ]]; then
    VULN_JSON_FILE="$VULN_DIR/vulnerability-report-${TS}.json"
    VULN_TABLE_FILE="$VULN_DIR/vulnerability-report-${TS}.txt"
    echo "Generating timestamped vulnerability reports: $(basename "$VULN_JSON_FILE"), $(basename "$VULN_TABLE_FILE")"
else
    VULN_JSON_FILE="$VULN_DIR/vulnerability-report.json"
    VULN_TABLE_FILE="$VULN_DIR/vulnerability-report.txt"
    echo "Generating default vulnerability reports: $(basename "$VULN_JSON_FILE"), $(basename "$VULN_TABLE_FILE")"
fi

# Generate SBOM
syft "$BASE_DIR/.." -o json | jq '.' > "$SBOM_FILE"

# Generate vulnerability reports
grype "sbom:$SBOM_FILE" -o json | jq '.' > "$VULN_JSON_FILE"
grype "sbom:$SBOM_FILE" -o table > "$VULN_TABLE_FILE"

echo "SBOM and vulnerability reports generated successfully."
```

