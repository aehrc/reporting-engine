# Running Reports Locally

This guide covers running individual report scripts on a developer workstation, without the full reporting-service / ActiveMQ / MySQL stack.

---

## 1 Prerequisites

| Requirement | Notes |
|---|---|
| **JDK 17+** | JDK 25 also works. The parent BOM targets 17. |
| **Maven 3.8+** | No wrapper (`mvnw`) is included in the repo. |
| **Maven `settings.xml`** | Required to resolve IHTSDO SNAPSHOT dependencies (see below). |
| **Release archive(s)** | RF2 ZIP files — see [Section 5](#5-release-archives). |

### 1.1 Maven Settings

Create `~/.m2/settings.xml` with the IHTSDO Nexus repositories. Without this, Maven cannot resolve the `snomed-parent-bom` or other SNAPSHOT dependencies:

```xml
<settings>
  <profiles>
    <profile>
      <id>ihtsdo</id>
      <repositories>
        <repository>
          <id>ihtsdo-releases</id>
          <url>https://nexus3.ihtsdotools.org/repository/maven-releases/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>ihtsdo-snapshots</id>
          <url>https://nexus3.ihtsdotools.org/repository/maven-snapshots/</url>
          <releases><enabled>false</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>ihtsdo-releases</id>
          <url>https://nexus3.ihtsdotools.org/repository/maven-releases/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>ihtsdo</activeProfile>
  </activeProfiles>
</settings>
```

---

## 2 Build

```bash
mvn clean install -DskipTests -Ddependency-check.skip=true
```

- `-DskipTests` — skips unit tests for faster builds.
- `-Ddependency-check.skip=true` — skips the OWASP NVD vulnerability database download, which is very slow without an API key.
- Always run from the repo root to build all modules. Partial builds (`-pl reporting-engine-worker`) can fail if SNAPSHOT dependencies have drifted on Nexus.

---

## 3 Extract the Worker JAR

The `reporting-engine-worker` fat JAR uses Spring Boot's `BOOT-INF` layout and cannot be used directly with `java -cp`. Extract it to a working directory:

```bash
mkdir -p ~/re-run && cd ~/re-run
jar xf /path/to/reporting-engine/reporting-engine-worker/target/reporting-engine-worker-*.jar BOOT-INF/
```

### 3.1 Create `application-local.properties`

Create `~/re-run/BOOT-INF/classes/application-local.properties` with local storage config:

```properties
# Storage — all local, no S3
archives.readonly=true
archives.local.path=releases
archives.useCloud=false
archives.cloud.bucketName=none
archives.cloud.path=none

builds.readonly=true
builds.local.path=builds
builds.useCloud=false
builds.cloud.bucketName=none
builds.cloud.path=none

resources.readonly=true
resources.local.path=resources
resources.useCloud=false
resources.cloud.bucketName=none
resources.cloud.path=none

# Report output storage (prefix: reports.s3)
reports.s3.readonly=false
reports.s3.local.path=results
reports.s3.useCloud=false
reports.s3.cloud.bucketName=none
reports.s3.cloud.path=none

aws.key=
aws.secretKey=
cloud.aws.region.static=us-east-1
```

### 3.2 Create directories

```bash
cd ~/re-run
mkdir -p releases results reports
```

---

## 4 Report Output

By default, reports write to **Google Sheets** (requires a service account — see `application.properties` for the `google.client.*` settings). For local development, **LOCAL_FILE** output is simpler.

To enable local file output, add these two parameters to the report's `main()` method:

```java
params.put("ReportOutputTypes", "LOCAL_FILE");
params.put("ReportFormatType", "CSV");
```

> **Both parameters are required.** If only one is set, `ReportConfiguration.isValid()` returns false and output silently falls back to Google Sheets.

Output files are written to `reports/<env>/results_<ReportName>_<timestamp>.csv`.

---

## 5 Release Archives

Place RF2 release ZIP files in the `~/re-run/releases/` directory.

### Option A: Daily Build (simplest)

Pass the ZIP filename as the project with `-p`:

```bash
java ... -p SnomedCT_ManagedServiceAU_DAILYBUILD_BETA_AU1000036_20260430T120000Z.zip
```

This loads the archive directly as a single coherent edition. No Snowstorm connection needed. Works well for reports that only need the current snapshot state.

### Option B: Snowstorm + Release Archives

For reports that set `ensureSnapshotPlusDeltaLoad=true` (e.g. `NewDescriptions`, `InactiveConceptInRefset`), the engine loads:

1. **International release** — e.g. `SnomedCT_InternationalRF2_PRODUCTION_20260301T120000Z.zip`
2. **Extension-only release** — e.g. `EXT_ONLY_SnomedCT_ManagedServiceAU_PRODUCTION_AU1000036_20260228T120000Z.zip`
3. **Delta from Snowstorm** — exported automatically at runtime

The expected filenames are derived from Snowstorm branch metadata. The engine will log what it expects:

```
ArchiveDataLoader set to local source. Will expect releases\EXT_ONLY_SnomedCT_ManagedServiceAU_PRODUCTION_AU1000036_20260228T120000Z.zip to be available.
```

> **Important:** The extension release must be **extension-only** (just extension module content). Full editions that bundle International content will cause integrity errors from double-loading. See [Known Issues](#8-known-issues).

---

## 6 Running a Report

### 6.1 Interactive mode

```bash
cd ~/re-run
java -Xms1g -Xmx10g \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  -cp "BOOT-INF/classes;BOOT-INF/lib/*" \
  org.ihtsdo.termserver.scripting.reports.qi.FullyDefinedParentsInSubHierarchy \
  -p SnomedCT_ManagedServiceAU_DAILYBUILD_BETA_AU1000036_20260430T120000Z.zip
```

When connecting to Snowstorm (Option B), the script prompts for:
1. **Environment** — numbered list (e.g. `2` for UAT authoring)
2. **Auth cookie** — pass via `-c "uat-ims-ihtsdo=<JWT>"` to skip the prompt
3. **Project** — pass via `-p MAIN/SNOMEDCT-AU` to pre-fill

### 6.2 Using a run script

Create a `.env` file to avoid pasting the cookie each time:

```bash
# .env
IHTSDO_COOKIE="uat-ims-ihtsdo=<your-jwt-token>"
ENV_CHOICE=2
PROJECT="MAIN/SNOMEDCT-AU"
```

And a `run-report.sh`:

```bash
#!/bin/bash
set -e
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$SCRIPT_DIR"
source .env

CLASS="${1:?Usage: $0 <fully.qualified.ClassName>}"
echo "Running report: $CLASS"

java -Xms1g -Xmx10g \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  -cp "BOOT-INF/classes;BOOT-INF/lib/*" \
  "$CLASS" \
  -p "$PROJECT" \
  -c "$IHTSDO_COOKIE" <<EOF
$ENV_CHOICE

EOF
```

Usage:
```bash
./run-report.sh org.ihtsdo.termserver.scripting.reports.qi.FullyDefinedParentsInSubHierarchy
```

### 6.3 Compiling local changes

If you modify a report's source code (e.g. to add `LOCAL_FILE` output), you can recompile just that file against the extracted classpath without a full Maven rebuild:

```bash
cd ~/re-run
javac -cp "BOOT-INF/classes;BOOT-INF/lib/*" \
  -d BOOT-INF/classes \
  /path/to/reporting-engine/reporting-engine-worker/src/main/java/org/ihtsdo/.../YourReport.java
```

---

## 7 Memory

Reports load the full SNOMED ontology into memory. Recommended heap sizes:

| Scenario | `-Xmx` |
|---|---|
| International only | 4g |
| International + Extension (edition) | 8-10g |
| International + Extension (double-loaded from full edition) | 10g+ |

---

## 8 Known Issues

### Full Edition Double-Load

When using Option B (Snowstorm + release archives), the engine expects an **extension-only** package for the extension release. If a full edition (containing both International and extension content) is provided, International content is loaded twice at potentially different effective times. This can cause:

- `IllegalStateException: Attempt to check active status on non-populated Concept component`
- `Integrity concern: concept X does not appear in concept file`

**Workaround:** Use Option A (daily build) for reports that don't require snapshot+delta comparison.

### Corrupt Snapshot Cache

If a report fails mid-load, a partial snapshot may be cached in `~/re-run/snapshots/`. Subsequent runs will reuse this corrupt cache and fail with:

```
Insufficient number of concepts loaded N - Snapshot archive damaged?
```

**Fix:** Delete the cache directory:
```bash
rm -rf ~/re-run/snapshots/<project-name>
```

The safest default is to clear `snapshots/` on every run (trades ~5 min of extra load time for avoiding stale-cache issues). A one-liner prelude in your run script:

```powershell
# PowerShell
if (Test-Path ".\snapshots") { Remove-Item -Recurse -Force ".\snapshots" }
```

```bash
# Bash
rm -rf ./snapshots
```

### Expected Archive Filename Drift

Snowstorm's branch metadata advertises the specific release archive to load. As new releases are promoted, the filename the loader looks for moves forward. If you hit:

```
FileNotFoundException: releases\SnomedCT_ManagedServiceAU_PRODUCTION_AU1000036_<newer-date>.zip
```

…but only have the previous month's archive, options are:

1. **Download the new archive** (authoritative — required for production validation).
2. **Copy/rename a daily build to the expected production name** (quick unblock, not production-faithful):

   ```powershell
   Copy-Item "releases\SnomedCT_ManagedServiceAU_DAILYBUILD_BETA_AU1000036_<date>.zip" `
             "releases\SnomedCT_ManagedServiceAU_PRODUCTION_AU1000036_<date>.zip"
   ```

   This triggers the full-edition double-load (see above) — expect stubs and spurious MRCM integrity warnings. Useful for iterating on report *logic* but not for validating *data*.

Also note: the loader filename may flip between `EXT_ONLY_SnomedCT_*` and `SnomedCT_*` (no prefix) depending on how Snowstorm's metadata is configured. Match the filename it logs, character-for-character.
