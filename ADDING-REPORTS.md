# Adding a New Report

This guide walks you through writing a new report for the reporting-engine. It assumes you've already set up your local dev environment per [LOCAL-SETUP.md](LOCAL-SETUP.md).

> **Note for AI coding agents**: Do NOT invoke `run-report.sh` / `run-report.ps1` directly during development sessions. A single run can take 3-15+ minutes and produces tens of thousands of log lines, wasting significant context. Ask the human to run it and paste back the tail of the output, or targeted `grep` results.

---

## 1 Anatomy of a Report

Every report is a Java class in `reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/<category>/` with the following shape:

```java
public class YourReport extends TermServerReport implements ReportClass {

    public static void main(String[] args) throws TermServerScriptException {
        Map<String, String> params = new HashMap<>();
        TermServerScript.run(YourReport.class, args, params);
    }

    @Override
    public void init(JobRun run) throws TermServerScriptException {
        // Configure archive loading, read parameters
        super.init(run);
    }

    @Override
    public void postInit() throws TermServerScriptException {
        // Set up tabs + column headings AFTER the graph is loaded
        String[] columnHeadings = { "Col A, Col B, Col C" };
        String[] tabNames       = { "Results" };
        super.postInit(tabNames, columnHeadings);
    }

    @Override
    public Job getJob() {
        // Metadata for the Reporting UI: name, description, parameters
        JobParameters params = new JobParameters()
            .add(ECL).withType(JobParameter.Type.ECL)
            .build();
        return new Job()
            .withCategory(new JobCategory(JobType.REPORT, JobCategory.RELEASE_STATS))
            .withName("Your Report")
            .withDescription("What this report does.")
            .withProductionStatus(ProductionStatus.PROD_READY)
            .withParameters(params)
            .withTag(INT).withTag(MS)
            .build();
    }

    @Override
    public void runJob() throws TermServerScriptException {
        // The actual report logic.
        for (Concept c : gl.getAllConcepts()) {
            if (someCondition(c)) {
                report(PRIMARY_REPORT, c, "extra", "columns");
            }
        }
    }
}
```

### Lifecycle

```
main()
  └─ TermServerScript.run()
      └─ instantiate(jobRun, null)
          ├─ preInit()                  ← override to set flags BEFORE env prompts
          ├─ checkSettingsWithUser()    ← reads -p, -c, env index
          ├─ init(jobRun)               ← set archive flags, read params
          ├─ loadProjectSnapshot()      ← loads the graph (slow!)
          ├─ postInit()                 ← build tab/column structure
          ├─ runJob()                   ← YOUR LOGIC
          ├─ flushFilesWithWait()       ← write report output
          └─ finish()
```

> **Gotcha**: `postInit()` only runs after the graph loads. Don't do heavy initialisation in it — but DO put column-heading setup there, because the Report Manager isn't available until after loading.

---

## 2 Finding a Template

Always start by copying an existing report that resembles yours. Reading sibling code is faster than reading framework docs.

| If your report needs... | Start with |
|---|---|
| Inactivated concepts appearing in refsets | [`release/InactiveConceptInRefset.java`](reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/release/InactiveConceptInRefset.java) |
| Current members of "other" refsets (simple, map, etc.) in the graph | [`ListMapEntries.java`](reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/ListMapEntries.java), [`release/RefsetMaintenanceReport.java`](reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/release/RefsetMaintenanceReport.java) |
| Descriptions / annotations in current authoring cycle | [`release/NewDescriptions.java`](reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/release/NewDescriptions.java) |
| Hierarchy / stated-relationship analysis on an ECL subset | [`qi/FullyDefinedParentsInSubHierarchy.java`](reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/qi/FullyDefinedParentsInSubHierarchy.java) |
| Attribute pattern / MRCM analysis | Anything in `reports/qi/` |
| Drug modelling | Anything in `reports/drugs/` |

Picking the right template saves hours — most "how do I access X?" questions have already been answered by someone who wrote a similar report.

---

## 3 The GraphLoader (`gl`)

`gl` is inherited from the base class and is your primary handle on the loaded SNOMED data.

```java
// All concepts in the loaded graph (International + extension merged)
Collection<Concept> all = gl.getAllConcepts();

// Single concept by SCTID
Concept c = gl.getConcept("404684003");           // throws if not found
Concept c = gl.getConceptSafely("404684003");     // returns null if not found

// Descriptions (includes language refset entries)
Description d = gl.getDescription("2462558017");
```

### What IS loaded by default

- All Concept, Description, TextDefinition, Relationship, ConcreteRelationship rows
- OWL axiom snapshot (deserialised into the concept's axioms)
- Active and inactive states (check with `c.isActiveSafely()`)
- Historical association refset members → `c.getAssociationEntries(ActiveState, historicalOnly)`
- Inactivation indicator refset members → `c.getInactivationIndicator()` (returns an enum)
- Language refset entries → `d.getLangRefsetEntries(...)`

### What is NOT loaded by default

- **Simple-type refset members** — need `getArchiveManager().setLoadOtherReferenceSets(true)` in `init()`
- **Map-type refset members** — same flag
- **MRCM, module dependency, annotation refsets** — same flag
- **"Other" refset content generally** — everything that isn't language/association/inactivation

When the flag is on, these members land in `concept.getOtherRefsetMembers()`.

### "Stub" concepts

`gl.getAllConcepts()` can contain concepts that were **referenced** (by an indicator, association, or refset-member row) but whose own concept-file row was never loaded. These "stubs" have:

- `getConceptId()` → populated
- `getFsn()`, `isActive()` → null. Calling `isActive()` throws `IllegalStateException`; `isActiveSafely()` treats null as `false` (= inactive).
- `getInactivationIndicator()`, `getAssociationEntries(...)` → typically populated, because the indicator/association file loaders use `getConcept(id, createIfRequired=true)`.
- `getDescriptions(...)` → empty. `getPreferredSynonym()` throws `IllegalStateException` with "has no descriptions".

Stubs commonly appear under the full-edition double-load scenario (see [LOCAL-SETUP.md §8](LOCAL-SETUP.md#8-known-issues)), and also for concepts inactivated in prior release cycles whose concept-file rows aren't re-emitted. For iteration over inactive concepts, `isActiveSafely()` is the right filter. For display, prefer `getFsn()` (safe field read) over `getPreferredSynonym()` (throws on stubs).

---

## 4 The Concept API (key methods)

```java
// Identity & basic fields
c.getConceptId()                   // SCTID as String
c.getFsn()                         // FSN including semtag
c.getSemTag()                      // just the semtag, e.g. "(finding)"
c.getPreferredSynonym()            // PT in US English dialect
c.getPreferredSynonym(refsetId)    // PT in a specific language refset
c.getModuleId()
c.getEffectiveTime()               // release date, empty if unreleased
c.getDefinitionStatus()

// Active / inactive
c.isActiveSafely()                 // boolean, null-safe
c.getInactivationIndicator()       // InactivationIndicator enum (or null)
c.getAssociationEntries(ActiveState.ACTIVE, true /* historicalAssocsOnly */)

// Structure
c.getDescriptions(ActiveState.ACTIVE)
c.getRelationships(...)
c.getAxioms()
c.getParents(CharacteristicType.INFERRED_RELATIONSHIP)
c.getAncestors(...)

// Refset memberships (needs setLoadOtherReferenceSets(true))
c.getOtherRefsetMembers()
```

Pass a `Concept` straight into `report(...)` and it auto-expands to three columns: `ID | FSN | SemTag`. Good when those three columns are contiguous; otherwise pass `c.getId()`, `c.getFsn()`, `c.getSemTag()` separately.

---

## 5 Parameters (`getJob()`)

Parameters are declared in `getJob()` and appear as form fields in the Reporting UI.

```java
JobParameters params = new JobParameters()
    .add(ECL).withType(Type.ECL)                                   // ECL editor widget
    .add("Module").withType(Type.CONCEPT)                           // concept picker
    .add("Include Retired").withType(Type.BOOLEAN).withDefaultValue("false")
    .add("Limit").withType(Type.STRING)                             // free text
    .add(UNPROMOTED_CHANGES_ONLY).withType(Type.BOOLEAN).withMandatory()
    .build();
```

Standard parameter name constants live on `TermServerScript` — use them rather than inventing new strings:

| Constant | Meaning |
|---|---|
| `ECL` | "ECL" — main ECL filter |
| `UNPROMOTED_CHANGES_ONLY` | "Unpromoted Changes Only" |
| `SUB_HIERARCHY` | "Subhierarchy" |
| `MODULES` | "Modules" |
| `INPUT_FILE` | "Input File" |

Read values in `init()` or `postInit()`:

```java
userECL = run.getParamValue(ECL);                          // nullable
boolean flag = run.getMandatoryParamBoolean("My Flag");    // throws if missing
String project = run.getProject();
```

---

## 6 Output (`report()`)

### Writing rows

```java
report(PRIMARY_REPORT, concept, "extra", 42, someEnum);
// → columns: Id, FSN, SemTag, extra, 42, SOMEENUM
```

- First arg is the **tab index** — constants: `PRIMARY_REPORT` (0), `SECONDARY_REPORT` (1), `TERTIARY_REPORT` (2), `QUATERNARY_REPORT` (3), etc.
- `Concept` args auto-expand to 3 columns (Id | FSN | SemTag).
- Other args become one column each (via `toString()`).
- Enums print as their declaration name, e.g. `AMBIGUOUS`.
- Pass `c.getId()` + `c.getPreferredSynonym()` separately if you want Id+PT rather than Id+FSN+SemTag.

### Tabs and column headings

Declared in `postInit()`. The number of tab names and column-heading strings must match — one entry per tab.

```java
String[] tabNames = { "Concepts", "Summary" };
String[] columnHeadings = {
    "Id, FSN, SemTag, Reason",           // columns for tab 0 (PRIMARY_REPORT)
    "Module, Count"                       // columns for tab 1 (SECONDARY_REPORT)
};
super.postInit(tabNames, columnHeadings);
```

### Issue count

`countIssue(concept)` increments the report's issue counter (unique by concept). Call it once per row you emit if you want the summary total at the end to match.

### Output destinations

By default the engine writes to **Google Sheets** (requires `google.client.*` service-account credentials — see `application.properties`). For local standalone runs you'll typically want LOCAL_FILE — see [LOCAL-SETUP.md §4](LOCAL-SETUP.md) for the temporary `main()` edits. **Do not commit those test-only params.**

---

## 7 Snapshot Loading Flags (`init()`)

Set these on `getArchiveManager()` in `init()` **before** calling `super.init(run)`:

| Flag | Purpose | When to use |
|---|---|---|
| `setEnsureSnapshotPlusDeltaLoad(true)` | Load INT + extension release archives, then apply a fresh delta exported from Snowstorm at runtime. Gives latest branch state. | Any report that needs **current** state from Snowstorm (default for most release/QI reports). |
| `setLoadOtherReferenceSets(true)` | Also load simple/map/other refset members into the graph (populates `Concept.getOtherRefsetMembers()`). | When iterating refset memberships. |
| `setRunIntegrityChecks(false)` | Skip the "Ensuring all concepts have parents and depth if required" post-load phase. | If your report doesn't rely on concept depth/ancestry AND you're hitting fatal integrity errors on extension edition double-loads. |
| `setPopulateReleasedFlag(true)` | Preserve each component's `released` flag across load. | Reports that filter by "new this cycle" (e.g. `d.isReleased() == false`). |

### Lesson: don't disable integrity checks casually

`setRunIntegrityChecks(false)` also skips the depth-population step that hierarchical ECL operators (`<`, `<<`, `>`) rely on. If your report uses a **hierarchical ECL** either in `findConcepts(ecl)` calls or as a user-supplied parameter, the ECL will silently return an incomplete set (only concepts whose ISA edges were already wired up at load time — typically just the International core). Symptoms: report runs to completion, reports 0 issues, log shows `Recovered N concepts for simple ecl from local memory` where N is much lower than expected.

If you need both (a) tolerance of the double-load integrity failure AND (b) full descendant resolution, either:
- avoid hierarchical ECL (enumerate refsets explicitly, or filter by another attribute like module id), OR
- filter at the member level (e.g. "simple refset members" = those with `getAdditionalFields().isEmpty()`), OR
- use Snowstorm REST calls (`findConcepts(ecl)` hits Snowstorm when the graph hierarchy is absent/broken — but only if you haven't already pre-resolved from local memory via the ECL cache).

### Forcing `findConcepts(ecl)` through Snowstorm

`EclCache.isSimple()` decides whether to resolve an ECL locally (against the in-memory graph) or via Snowstorm. **Simple** ECLs are served from local memory and will return incomplete results if the graph hierarchy is absent/broken (e.g. integrity checks disabled). An ECL is treated as non-simple — and therefore routed to Snowstorm — if it contains any of:

- `(`, `{`, `,`, `^`, `!`, `:`
- ` AND `, ` OR `, ` MINUS `
- more than 2 `|` pipes

Two common pitfalls:

1. **Single-clause hierarchical** like `< 446609009 |Simple type reference set|` is "simple" — resolved locally, returns only concepts whose parents are wired up.
2. **Outer parens alone won't help**: `EclCache.findConcepts` strips outer `(` / `)` before checking `isSimple()` *unless* the ECL contains `AND`/`OR`/`MINUS`. So `(< 446609009)` becomes `< 446609009` and goes local.

Reliable tricks to force Snowstorm resolution:

- Append a no-op `MINUS` against an unrelated metadata concept:
  `< 446609009 |Simple type reference set| MINUS 900000000000497000 |CTV3 simple map reference set (foundation metadata concept)|`
  (CTV3 isn't a descendant of 446609009, so the MINUS doesn't change the result set — but `MINUS` forces non-simple *and* preserves any wrapping parens.)
- Add a refinement: `<446609009 {{C moduleId = 32506021000036107}}` — the `{` / `{{...}}` marks it non-simple.
- Watch the log: `Recovering N concepts from TS matching ...` = Snowstorm; `Recovered N concepts for simple ecl from local memory` = trap.

---

## 8 Common Patterns (Cookbook)

### Iterate all concepts matching an ECL

```java
Collection<Concept> subset = findConcepts(ecl);
for (Concept c : subset) { ... }
```

### Resolve a single concept

```java
Concept c = gl.getConceptSafely(sctid);
if (c == null) { /* not loaded */ }
```

### Look up refset members where the referenced component is a concept

With `setLoadOtherReferenceSets(true)`:

```java
for (Concept c : gl.getAllConcepts()) {
    for (RefsetMember m : c.getOtherRefsetMembers()) {
        if (!m.isActiveSafely()) continue;
        String refsetId = m.getRefsetId();
        // ...
    }
}
```

### Detect simple-type refset members vs map-type

Simple refsets have only the 6 core RF2 columns; map / attribute-value / annotation / OWL refsets carry extra fields:

```java
private boolean isSimpleRefsetMember(RefsetMember m) {
    Map<String, String> fields = m.getAdditionalFields();
    return fields == null || fields.isEmpty();
}
```

### Get inactivation reason + historical associations for a concept

```java
InactivationIndicator reason = c.getInactivationIndicator();
for (AssociationEntry a : c.getAssociationEntries(ActiveState.ACTIVE, true)) {
    String assocType = SnomedUtils.getAssociationType(a);    // "REPLACED BY", "SAME AS", ...
    Concept target = gl.getConcept(a.getTargetComponentId());
}
```

### Batch Snowstorm refset-member queries

When you need refset members NOT loaded into the graph (or want to avoid loading them):

```java
for (List<String> batch : Iterables.partition(conceptIds, CLAUSE_LIMIT /* 100 */)) {
    Collection<RefsetMember> members = searchMembers(batch, refsetECL);
    for (RefsetMember m : members) { ... }
    Thread.sleep(200L);  // politeness pause — other reports do the same
}
```

`searchMembers()` hits Snowstorm at the branch path. It requires a Snowstorm connection (doesn't work when the project is a `.zip` file — no branch context).

**Caveat: the authoring-platform URL (`https://*-authoring.ihtsdotools.org/snowstorm/...`) is a proxy and in some deployments only exposes GET endpoints.** `searchMembers()` POSTs to `/members/search` and can fail with `400 Request method 'POST' is not supported`. If you hit this:

- Use the direct Snowstorm URL instead (e.g. env index 9 `prod-snowstorm.ihtsdotools.org`), if you have credentials for it. No code change needed. OR
- Iterate refsets one at a time using the GET endpoint `tsClient.findRefsetMembers(branchPath, refsetId, null)` and filter client-side.

### Per-refset GET queries (when POST is unavailable)

```java
String branchPath = project.getBranchPath();
for (String refsetId : targetRefsetIds) {
    Collection<RefsetMember> members = tsClient.findRefsetMembers(branchPath, refsetId, null);
    for (RefsetMember m : members) { ... }
    Thread.sleep(200L);
}
```

`findRefsetMembers(branchPath, refsetId, ...)` handles pagination internally and pulls **active and inactive** members. The method issues `limit=10000` on each page — bumped up from Snowstorm's default of ~100 because an 8K-member refset at 100 per page is 80 round trips and takes 30+ minutes.

### Prefer Snowstorm over `getOtherRefsetMembers()` for completeness

`concept.getOtherRefsetMembers()` only contains members from refset files that were packaged in the release archives loaded into the graph. For a multi-edition project like `MAIN/SNOMEDCT-AU`, this is frequently **incomplete** — many AU refsets aren't in the loaded extension archive. Symptoms: your local member count is an order of magnitude lower than what's in Snowstorm or SQL.

For an authoritative count, query Snowstorm directly (either `searchMembers` if POST works, or per-refset GET). Use the local graph only for concept-level data like inactivation reason and historical associations.

### Display labels for Snowstorm-resolved concepts

A concept returned by `findConcepts(...)` or `tsClient.getConcept(...)` has its FSN populated as a string field but **no attached Description objects**. Calling `getPreferredSynonym()` on it throws `IllegalStateException: Concept X has no descriptions`.

Prefer FSN for labels; wrap PT lookups in try/catch:

```java
private String bestLabel(Concept c) {
    if (c == null) return "";
    if (!StringUtils.isEmpty(c.getFsn())) return c.getFsn();
    try {
        String pt = c.getPreferredSynonym();
        if (!StringUtils.isEmpty(pt)) return pt;
    } catch (Exception e) { /* no descriptions attached */ }
    return c.getId();
}
```

### Suppress framework integrity warnings in CSV output

`TermServerScript.postInit()` unconditionally writes any accumulated `gl.getIntegrityWarnings()` to `PRIMARY_REPORT` before your data rows. For reports that don't consume MRCM — and when a full-edition double-load is producing spurious warnings — this corrupts the CSV (warning rows between the header and your data).

Clear the list before calling `super.postInit()`:

```java
@Override
public void postInit() throws TermServerScriptException {
    // ... set up columnHeadings, tabNames ...
    gl.getIntegrityWarnings().clear();
    super.postInit(tabNames, columnHeadings);
}
```

Only do this if your report doesn't depend on MRCM — the warnings are meaningful for MRCM consumers.

---

## 9 Registering the Report

The worker auto-discovers any class on the classpath that implements `JobClass` (transitively via `ReportClass` / `BatchJobClass`). See [`reporting-engine-worker/src/main/java/org/ihtsdo/termserver/job/JobManager.java`](reporting-engine-worker/src/main/java/org/ihtsdo/termserver/job/JobManager.java).

**No registry file to update.** Just put your report under `reporting-engine-worker/src/main/java/org/ihtsdo/termserver/scripting/reports/<category>/`, rebuild, and it's picked up.

---

## 10 Testing Your Report Locally

Follow [LOCAL-SETUP.md](LOCAL-SETUP.md) for the extract-and-compile workflow.

### Recommended flow

1. **First pass — daily build archive.** Pass a release ZIP as `-p` (e.g. an AU daily build). Runs on a single self-consistent edition, no Snowstorm needed, passes integrity checks cleanly. Good for verifying report *logic*.
2. **Second pass — Snowstorm branch.** Pass `-p MAIN/SNOMEDCT-AU` with a valid auth cookie. Loads INT release + EXT release + delta. Exercises the snapshot+delta path your report will see in production.
3. **Spot-check output** against a known baseline (SQL against the published snapshot, a manual ECL in the browser, another report).

### Lessons from past runs

- A full extension **edition** archive renamed to the expected `EXT_ONLY_*` filename will load, but produces many orphan references (`|null|` FSNs). See GitHub issues aehrc/reporting-engine#1 and #2.
- Memory: budget **16 GB heap** for snapshot+delta + `setLoadOtherReferenceSets(true)` on AU. `-Xmx10g` OOMs.
- If a run fails mid-load, a partial snapshot is cached under `snapshots/<project>_<env>/` and will be reused (and fail again) on the next run. Delete it.
- `findConcepts(ECL)` results are cached per branch in `EclCache`. Restarting the JVM is the easiest way to force a re-query.
- `-headless <n>` exists as a CLI flag but it only sets a JobRun parameter — it doesn't actually bypass the interactive environment prompt for the `TermServerScript.run()` path. Plan to type the environment index into stdin, or pre-set it in your main().

---

## 11 Before You Commit

- Remove any **temporary test-only** tweaks from your `main()`:
  - `params.put("ReportOutputTypes", "LOCAL_FILE")` / `"ReportFormatType"`
  - any hardcoded environments[N] URL patches
  - any hardcoded project / cookie / SCTID overrides
- Double-check `setRunIntegrityChecks(false)` is **deliberate**, not cargo-culted from another report.
- Verify your report runs against the daily-build edition AND produces the same row count against the snapshot+delta path (subject to any delta since the release).
- Add a Javadoc class comment explaining **what question the report answers** for maintainers.
- Reference your Jira/RP ticket in the class comment (see existing reports, e.g. `RP-370` on `InactiveConceptInRefset`).
- Don't silently alter output shape of existing reports — the Reporting UI stores column layouts keyed on report name.
