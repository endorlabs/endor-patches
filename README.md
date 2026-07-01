# endor-patches-open-source

Security backports for widely-used Java libraries, produced by [Endor Labs](https://www.endorlabs.com). This repository contains two fully attested example patches that demonstrate the complete Endor Patches workflow — from CVE fix through hermetic build, upstream test verification, and transparent deployment.

---

## What is Endor Patches?

Endor Patches is a curated collection of software packages with backported vulnerability fixes. Endor Labs identifies vulnerable functions and the upstream commits that fixed each vulnerability, then applies those fixes to older versions that **cannot be upgraded** to the next major release without breaking API or runtime compatibility.

The result is a minimum viable security patch — the smallest possible change to neutralize a known CVE — without forcing a disruptive version bump across your dependency tree.

Three principles govern every patch:

| Principle | What it means |
|-----------|--------------|
| **Transparent** | Every code change, build command, and test result is published and independently auditable |
| **Hermetic** | Builds run inside pinned Docker containers with no network access at compile time — the output is consistent and reproducible |
| **Verified** | The full upstream test suite runs against the patched source; a patch is only published when all tests pass |

### Version naming convention

Every Endor patch is published under one of three version identifiers:

| Identifier | Example | When to use |
|------------|---------|-------------|
| `-endor-YYYY-MM-DD` | `6.3.1-endor-2024-09-03` | Pin to a specific patch date for reproducible builds |
| `-endor-latest` | `6.3.1-endor-latest` | Always resolve to the most current patch for that version |
| Original version | `6.3.1` | Auto-patching mode — Endor transparently serves the patched artifact |

---

## Patches in this repository

This repository contains two example patches that illustrate the full Endor Patches workflow end-to-end.

| Library | Patched Version | CVE | Severity | CVSS | EPSS |
|---------|----------------|-----|----------|------|------|
| `com.fasterxml.woodstox:woodstox-core` | `6.3.1-endor-2024-09-03` | [CVE-2022-40152](https://nvd.nist.gov/vuln/detail/CVE-2022-40152) | High | 7.5 | 81st pct |
| `org.eclipse.jgit:org.eclipse.jgit` | `6.6.0.202305301015-r-endor-2024-11-25` | [CVE-2023-4759](https://nvd.nist.gov/vuln/detail/CVE-2023-4759) | High | 8.8 | 77th pct |

The full Endor Patches catalog — including patches for okio, jackson-databind, sqlite-jdbc, spring-cloud-function-context, and many others — is available through the [Endor Patch Factory](https://docs.endorlabs.com/risk-remediation/endor-patches/connecting-to-the-factory/).

---

## What each patch fixes

### woodstox-core 6.3.1 — CVE-2022-40152

`FullDTDReader.readContentSpec()` recurses without bound when parsing nested `( )` groups in XML DTDs. A crafted document causes a `StackOverflowError` — an `Error`, not an `Exception` — so it bypasses all standard `catch` blocks and crashes the JVM thread without any opportunity for recovery or logging.

The fix adds a configurable recursion depth limit (default 500) and throws a catchable `XMLStreamException` when exceeded. The patch is a pure upstream backport of commit `7e93907` by PJ Fanning — no new security logic was written by Endor Labs.

woodstox-core is a transitive dependency of Jackson Dataformat XML and Spring OXM, making this a common exposure in Spring-based applications.

### org.eclipse.jgit 6.6.0 — CVE-2023-4759

JGit did not reject repository paths containing `.git` as a component during checkout, stash apply, merge, and patch operations. An attacker with write access to a shared repository could craft a commit that silently overwrites files inside the `.git` directory of any machine that clones or fetches from it — enabling arbitrary code execution via git hook injection.

The security fix rejects any path whose component equals `.git` (case-insensitive). Endor Labs also authored 4 `FIX_TYPE_BUILD` commits to make the module compile in an isolated Docker container (skipping license checks, removing unmounted submodules). These build changes do not touch any security-sensitive code paths.

---

## Patch Transparency

In security, trust must be earned. Every patch in this repository is fully auditable: you can read every line of code that was changed, see exactly how the artifact was built, and confirm that the upstream tests passed.

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
| `fix_type` | `FIX_TYPE_VULN` — a security fix; `FIX_TYPE_BUILD` — a build system change required to compile the patched version |
| `fixed_vulns[].aliases` | CVE identifiers addressed by this change |
| `included_commits[].commit_type` | `COMMIT_TYPE_UPSTREAM` — backported directly from the original maintainer's commit; `COMMIT_TYPE_MANUAL` — written by Endor Labs |
| `included_commits[].author` | Who authored the commit |
| `included_commits[].sha` | The upstream commit SHA you can independently verify on GitHub |
| `patch_files[]` | Unified diffs — the exact line-by-line changes applied to the source |

A patch entry that contains only `COMMIT_TYPE_UPSTREAM` commits means Endor Labs applied an existing community fix without writing any new code. If `COMMIT_TYPE_MANUAL` commits are present, read `patch_files` to understand precisely what Endor Labs changed and why.

### Querying live attestation data via endorctl

You can fetch structured attestation data directly from the Endor Labs API. The patches in this repository are available in the public `oss` namespace:

**woodstox-core 6.3.1:**

```bash
# Security attestation — code diffs, CVE mapping, commit provenance
endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://com.fasterxml.woodstox:woodstox-core@6.3.1-endor-2024-09-03" \
  --field-mask="spec.security_attestation"

# Build attestation — Docker build command and signed URL to full build log
endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://com.fasterxml.woodstox:woodstox-core@6.3.1-endor-2024-09-03" \
  --field-mask="spec.build_attestation"

# Test attestation — Docker test command and signed URL to full test log
endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://com.fasterxml.woodstox:woodstox-core@6.3.1-endor-2024-09-03" \
  --field-mask="spec.test_attestation"

# Deploy attestation — publish command and signed URL to deploy log
endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://com.fasterxml.woodstox:woodstox-core@6.3.1-endor-2024-09-03" \
  --field-mask="spec.deploy_attestation"

# Signed URL to download the reproducible source tarball
endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://com.fasterxml.woodstox:woodstox-core@6.3.1-endor-2024-09-03" \
  --field-mask="spec.reproducible_build_source_code_url"
```

**org.eclipse.jgit 6.6.0:**

```bash
endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://org.eclipse.jgit:org.eclipse.jgit@6.6.0.202305301015-r-endor-2024-11-25" \
  --field-mask="spec.security_attestation"

endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://org.eclipse.jgit:org.eclipse.jgit@6.6.0.202305301015-r-endor-2024-11-25" \
  --field-mask="spec.build_attestation"

endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://org.eclipse.jgit:org.eclipse.jgit@6.6.0.202305301015-r-endor-2024-11-25" \
  --field-mask="spec.test_attestation"

endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://org.eclipse.jgit:org.eclipse.jgit@6.6.0.202305301015-r-endor-2024-11-25" \
  --field-mask="spec.deploy_attestation"

endorctl api get -r AssuredPackageVersion -n oss \
  --name="mvn://org.eclipse.jgit:org.eclipse.jgit@6.6.0.202305301015-r-endor-2024-11-25" \
  --field-mask="spec.reproducible_build_source_code_url"
```

The `spec.build_attestation` and `spec.test_attestation` responses include a `logs_url` — a signed download link to the full log content (identical to what is stored in `build_logs.txt` and `test_logs.txt` in this repository). The `spec.reproducible_build_source_code_url` returns a signed URL to a `.tar.gz` of the exact source tree used to produce the JAR.

> **Note:** Reproducing the build locally requires Bazel and Docker.

---

## Using the patched JARs from this repository

The patched JARs are in `patches/<library>/artifacts/`. Follow these steps to install and use them.

### Step 1 — Verify the JAR

Always verify the SHA256 before installing. Expected hashes are in `SHA256SUM.txt` in each patch directory.

```bash
# woodstox-core
sha256sum patches/woodstox-core-6.3.1/artifacts/woodstox-core-6.3.1-endor-2024-09-03.jar
# expected: 821c773baa1b8121d1302404a143eb9ed4f84c9a3409fbc81d63fecf48471fbe

# org.eclipse.jgit
sha256sum patches/org.eclipse.jgit-6.6.0.202305301015-r/artifacts/org.eclipse.jgit-6.6.0.202305301015-r-endor-2024-11-25.jar
# expected: e3dce31a9c0b63591bbc8988a7a4e3d4ecb47898ca90d5e1ae8087d757d86f36
```

### Step 2 — Install to your local Maven repository

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

### Step 3 — Declare the dependency

**Maven — direct dependency:**

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

**Maven — override a transitive dependency (e.g. pulled in via a Spring Boot BOM):**

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

**Gradle — force a patched version across all transitive dependencies:**

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

## Connecting to the Endor Patch Factory

For access to the full Endor Patches catalog and automatic patching of both direct and transitive dependencies, connect your build tool to the Endor Patch Factory — a secure hosted Maven repository.

### Step 1 — Create an API key

1. In the Endor Labs console, go to **User menu → Settings → Access Control → API Keys**.
2. Select **Generate API Key**, give it a descriptive name (e.g. `endor-patch-factory`), grant at least **Read Only** permission, and set an expiration (30, 60, or 90 days).
3. Save the key ID as `ENDOR_API_CREDENTIALS_KEY` and the secret as `ENDOR_API_CREDENTIALS_SECRET`.

### Step 2 — Configure your build tool

**Maven — add to `pom.xml`:**

```xml
<repositories>
  <repository>
    <id>endorlabs</id>
    <url>https://factory.endorlabs.com/v1/namespaces/<your-namespace>/maven2</url>
    <releases><enabled>true</enabled></releases>
    <snapshots><enabled>false</enabled></snapshots>
  </repository>
</repositories>
```

Add credentials to `~/.m2/settings.xml`:

```xml
<servers>
  <server>
    <id>endorlabs</id>
    <username>${env.ENDOR_API_CREDENTIALS_KEY}</username>
    <password>${env.ENDOR_API_CREDENTIALS_SECRET}</password>
  </server>
</servers>
```

**Gradle — add to `build.gradle`:**

```groovy
repositories {
    mavenCentral()
    maven {
        url "https://factory.endorlabs.com/v1/namespaces/<your-namespace>/maven2"
        credentials {
            username "$ENDOR_API_CREDENTIALS_KEY"
            password "$ENDOR_API_CREDENTIALS_SECRET"
        }
    }
}
```

### Step 3 — Reference the patched version

```xml
<!-- Maven: pin to a specific patch date -->
<dependency>
    <groupId>com.fasterxml.woodstox</groupId>
    <artifactId>woodstox-core</artifactId>
    <version>6.3.1-endor-2024-09-03</version>
</dependency>

<!-- Maven: always use the latest available patch -->
<dependency>
    <groupId>com.fasterxml.woodstox</groupId>
    <artifactId>woodstox-core</artifactId>
    <version>6.3.1-endor-latest</version>
</dependency>
```

For enterprise environments using JFrog Artifactory or Sonatype Nexus, see the dedicated proxy configuration guides in the [Endor Patches documentation](https://docs.endorlabs.com/risk-remediation/endor-patches/).

---

## Auditing a patch independently

1. Open `security_attestation.json` in the patch directory you are evaluating.
2. Locate entries with `"fix_type": "FIX_TYPE_VULN"`. Read `patch_files` — these contain the security-relevant diffs.
3. For `COMMIT_TYPE_UPSTREAM` commits: look up the `sha` on the upstream GitHub repository and confirm the diff matches exactly.
4. For `COMMIT_TYPE_MANUAL` commits (Endor Labs-authored): read `patch_files` carefully. For woodstox-core there are none — the fix is a pure upstream backport. For org.eclipse.jgit the 4 manual commits are build system changes only (skipping license checks, removing submodules not mounted in the Docker container); they do not touch security-sensitive code.
5. Read `build_logs.txt` to confirm the pinned Docker image tag and exact build commands used.
6. Read `test_logs.txt` to verify the upstream test suite ran and all tests passed.
7. Use `spec.reproducible_build_source_code_url` (from endorctl) to download the source tarball, run the build command from `spec.build_attestation`, and verify the output JAR's SHA256 matches `SHA256SUM.txt`.

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

Patches produced by [Endor Labs](https://www.endorlabs.com). To report an issue with a patched artifact, open an issue in this repository.
