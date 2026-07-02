# Endor Patches

Security backports for widely-used Java libraries, produced by [Endor Labs](https://www.endorlabs.com). This repository contains two fully attested example patches that demonstrate the complete Endor Patches workflow: from CVE fix through hermetic build, upstream test verification, and transparent deployment.

Learn more: [Endor Patches](https://www.endorlabs.com/lp/patches) | [Documentation](https://docs.endorlabs.com/risk-remediation/endor-patches)

---

## What is Endor Patches?

Endor Patches is a curated collection of software packages with backported vulnerability fixes. Endor Labs identifies vulnerable functions and the upstream commits that fixed each vulnerability, then applies those fixes to older versions that **cannot be upgraded** to the next major release without breaking API or runtime compatibility.

The result is a minimum viable security patch: the smallest possible change to neutralize a known CVE, without forcing a disruptive version bump across your dependency tree.

Three principles govern every patch:

| Principle | What it means |
|-----------|--------------|
| **Transparent** | Every code change, build command, and test result is published and independently auditable |
| **Hermetic** | Builds run inside pinned Docker containers with no network access at compile time; the output is consistent and reproducible |
| **Verified** | The full upstream test suite runs against the patched source; a patch is only published when all tests pass |

### Version naming convention

Every Endor patch is published under one of three version identifiers:

| Identifier | Example | When to use |
|------------|---------|-------------|
| `-endor-YYYY-MM-DD` | `6.3.1-endor-2024-09-03` | Pin to a specific patch date for reproducible builds |
| `-endor-latest` | `6.3.1-endor-latest` | Always resolve to the most current patch for that version |
| Original version | `6.3.1` | Auto-patching mode: Endor transparently serves the patched artifact |

---

## Patches in this repository

This repository contains two example patches that illustrate the full Endor Patches workflow end-to-end.

| Library | Patched Version | CVE | Severity | CVSS | EPSS |
|---------|----------------|-----|----------|------|------|
| `com.fasterxml.woodstox:woodstox-core` | `6.3.1-endor-2024-09-03` | [CVE-2022-40152](https://nvd.nist.gov/vuln/detail/CVE-2022-40152) | High | 7.5 | 81st pct |
| `org.eclipse.jgit:org.eclipse.jgit` | `6.6.0.202305301015-r-endor-2024-11-25` | [CVE-2023-4759](https://nvd.nist.gov/vuln/detail/CVE-2023-4759) | High | 8.8 | 77th pct |

---
## Patch Transparency

Every patch in this repository is fully auditable: you can read every line of code that was changed, see exactly how the artifact was built, and confirm that the upstream tests passed.

### Attestation files included with each patch

Each patch directory ships four evidence files:

| File | What it contains |
|------|-----------------|
| `security_attestation.json` | The exact code diffs applied to the source; maps each change to a CVE or build requirement; identifies whether each commit is upstream or Endor-authored |
| `build_logs.txt` | Full Maven/Gradle output from the hermetic Docker build; shows every compiler step, dependency resolution, and output artifact |
| `test_logs.txt` | Full upstream test suite output; confirms all tests ran and passed |
| `deploy_logs.txt` | Full publish output; shows the exact artifact path and registry URL where the JAR was deployed |

### What `security_attestation.json` shows

The `patches` array is the core of the transparency story. Each entry represents one logical set of changes:

| Field | What it means |
|-------|---------------|
| `fix_type` | `FIX_TYPE_VULN` (security fix) or `FIX_TYPE_BUILD` (build system change required to compile the patched version) |
| `fixed_vulns[].aliases` | CVE identifiers addressed by this change |
| `included_commits[].commit_type` | `COMMIT_TYPE_UPSTREAM` (backported from the original maintainer's commit) or `COMMIT_TYPE_MANUAL` (written by Endor Labs) |
| `included_commits[].author` | Who authored the commit |
| `included_commits[].sha` | The upstream commit SHA you can independently verify on GitHub |
| `patch_files[]` | Unified diffs: the exact line-by-line changes applied to the source |

A patch entry that contains only `COMMIT_TYPE_UPSTREAM` commits means Endor Labs applied an existing community fix without writing any new code. If `COMMIT_TYPE_MANUAL` commits are present, read `patch_files` to understand precisely what Endor Labs changed and why.

---

## Using the patched JARs from this repository

The patched JARs are in `patches/<library>/artifacts/`. Follow these steps to install and use them.

### Step 1: Verify the JAR

Always verify the SHA256 before installing. Expected hashes are in `SHA256SUM.txt` in each patch directory.

```bash
# woodstox-core
sha256sum patches/woodstox-core-6.3.1/artifacts/woodstox-core-6.3.1-endor-2024-09-03.jar
# expected: 821c773baa1b8121d1302404a143eb9ed4f84c9a3409fbc81d63fecf48471fbe

# org.eclipse.jgit
sha256sum patches/org.eclipse.jgit-6.6.0.202305301015-r/artifacts/org.eclipse.jgit-6.6.0.202305301015-r-endor-2024-11-25.jar
# expected: e3dce31a9c0b63591bbc8988a7a4e3d4ecb47898ca90d5e1ae8087d757d86f36
```

### Step 2: Install to your local Maven repository

```bash
# woodstox-core
mvn install:install-file \
  -Dfile=patches/woodstox-core-6.3.1/artifacts/woodstox-core-6.3.1-endor-2024-09-03.jar \
  -DpomFile=patches/woodstox-core-6.3.1/artifacts/woodstox-core-6.3.1-endor-2024-09-03.pom \
  -DgroupId=com.fasterxml.woodstox \
  -DartifactId=woodstox-core \
  -Dversion=6.3.1-endor-2024-09-03 \
  -Dpackaging=jar

# org.eclipse.jgit
mvn install:install-file \
  -Dfile=patches/org.eclipse.jgit-6.6.0.202305301015-r/artifacts/org.eclipse.jgit-6.6.0.202305301015-r-endor-2024-11-25.jar \
  -DpomFile=patches/org.eclipse.jgit-6.6.0.202305301015-r/artifacts/org.eclipse.jgit-6.6.0.202305301015-r-endor-2024-11-25.pom \
  -DgroupId=org.eclipse.jgit \
  -DartifactId=org.eclipse.jgit \
  -Dversion=6.6.0.202305301015-r-endor-2024-11-25 \
  -Dpackaging=jar
```

### Step 3: Declare the dependency

**Maven - direct dependency:**

```xml
<!-- woodstox-core -->
<dependency>
    <groupId>com.fasterxml.woodstox</groupId>
    <artifactId>woodstox-core</artifactId>
    <version>6.3.1-endor-2024-09-03</version>
</dependency>

<!-- org.eclipse.jgit -->
<dependency>
    <groupId>org.eclipse.jgit</groupId>
    <artifactId>org.eclipse.jgit</artifactId>
    <version>6.6.0.202305301015-r-endor-2024-11-25</version>
</dependency>
```

**Maven - override a transitive dependency (e.g. pulled in via a Spring Boot BOM):**

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.woodstox</groupId>
            <artifactId>woodstox-core</artifactId>
            <version>6.3.1-endor-2024-09-03</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**Gradle - force a patched version across all transitive dependencies:**

```groovy
configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
        if (details.requested.group == 'com.fasterxml.woodstox'
                && details.requested.name == 'woodstox-core') {
            details.useVersion '6.3.1-endor-2024-09-03'
        }
    }
}
```

---

## Auditing a patch independently

1. Open `security_attestation.json` in the patch directory you are evaluating.
2. Locate entries with `"fix_type": "FIX_TYPE_VULN"`. Read `patch_files`; these contain the security-relevant diffs.
3. For `COMMIT_TYPE_UPSTREAM` commits: look up the `sha` on the upstream GitHub repository and confirm the diff matches exactly.
4. For `COMMIT_TYPE_MANUAL` commits (Endor Labs-authored): read `patch_files` carefully. For woodstox-core there are none; the fix is a pure upstream backport. For org.eclipse.jgit the 4 manual commits are build system changes only (skipping license checks, removing submodules not mounted in the Docker container); they do not touch security-sensitive code.
5. Read `build_logs.txt` to confirm the pinned Docker image tag and exact build commands used.
6. Read `test_logs.txt` to verify the upstream test suite ran and all tests passed.

---

## Repository layout

```
patches/
├── woodstox-core-6.3.1/
│   ├── security_attestation.json   # code diffs, CVE mapping, commit provenance
│   ├── build_logs.txt              # full hermetic build output
│   ├── test_logs.txt               # full upstream test suite output
│   ├── deploy_logs.txt             # full deploy output
│   ├── SHA256SUM.txt               # SHA256 of the published JAR
│   └── artifacts/
│       ├── woodstox-core-6.3.1-endor-2024-09-03.jar
│       └── woodstox-core-6.3.1-endor-2024-09-03.pom
└── org.eclipse.jgit-6.6.0.202305301015-r/
    ├── security_attestation.json
    ├── build_logs.txt
    ├── test_logs.txt
    ├── deploy_logs.txt
    ├── SHA256SUM.txt
    └── artifacts/
        ├── org.eclipse.jgit-6.6.0.202305301015-r-endor-2024-11-25.jar
        └── org.eclipse.jgit-6.6.0.202305301015-r-endor-2024-11-25.pom
```

---

## License

Patch documentation and scripts in this repository are released under [Apache 2.0](LICENSE). The patched JARs inherit the license of their upstream projects: Apache 2.0 (woodstox-core) and Eclipse Distribution License 1.0 (org.eclipse.jgit).

---

## Contact

Patches are produced by [Endor Labs](https://www.endorlabs.com). To learn more about Endor Patches, visit [endorlabs.com/lp/patches](https://www.endorlabs.com/lp/patches) or the [Endor Patches documentation](https://docs.endorlabs.com/risk-remediation/endor-patches). To report an issue with a patched artifact, open an issue in this repository.
