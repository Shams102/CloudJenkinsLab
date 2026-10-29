# FestPass Jenkins CI/CD Assignment — Final Evidence Report

## Executive Summary

| Item | Details |
|---|---|
| **Student Alias** | `Shams` (derived from checked-out Git commit author) |
| **Class Code** | `001` |
| **Challenge Code** | `00647` |
| **Installation ID** | `2d7ba83f-4747-4e9e-af35-2377594fbdd0` |
| **Machine Label Hash** | `6b0a83ce392cc3f182fca6185222a4c8cb0ddd571faa3c6b021c91a6d3dad2b1` |
| **Jenkins Job** | `festpass-ci` (Pipeline from SCM: `git://127.0.0.1:9418/festpass-java`, branch `*/main`) |
| **Agent** | `student-laptop` (1 executor, exclusive usage) |

---

## Complete Build History

```
Build 1 (Bootstrap)  ──>  Triggered by user 'Admin' (Build Now)
                          Revision 0c2a3df (Configure submission codes & trigger)
                          Result: FAILURE (25 pass, 3 fail - seeded defect present)
                          Product stages skipped; submission-evidence.csv archived

Build 2 (Red Build)   ──>  Triggered by 'Started by an SCM change' (Automatic Polling)
                          Revision b11f54e (Personalize my help introduction)
                          Result: FAILURE (25 pass, 3 fail)
                          JAR Gating: NOT_PUBLISHED; submission-evidence.csv archived

Build 3 (Green Build) ──>  Triggered by 'Started by an SCM change' (Automatic Polling)
                          Revision 0be93a1 (Fix booking cancellation)
                          Result: SUCCESS (28 pass, 0 fail)
                          JAR built & smoke-tested; festpass.jar & evidence archived
```

---

## 1. Red Build (Build #2) Details

- **Build Number**: 2
- **Build URL**: http://127.0.0.1:8080/job/festpass-ci/2/
- **Trigger Cause**: `Started by an SCM change` (Automatic Git SCM polling at `* * * * *`)
- **Git Commit**: `b11f54e7d949c695935d1649182169570ac28532`
- **Commit Message**: `Personalize my help introduction`
- **Author**: `Shams`
- **Result**: `FAILURE`
- **Test Summary**: 28 total tests, 25 passed, 3 failed, 0 errors, 0 skipped
- **Failing Tests**:
  1. `cancellationFreesSeat` (`java.lang.IllegalArgumentException: SOLD OUT. Even the developer needs a ticket.`)
  2. `cancellationsSurviveReload` (`java.lang.AssertionError: Cancellation was lost`)
  3. `cliWorkflowPersistsAcrossCommands` (`java.lang.AssertionError: List does not reflect persisted cancellation`)
- **Product Stage Gating**: `Build JAR`, `Smoke test JAR`, and `Save artifact` stages were **skipped**.
- **Archived Artifacts**: `submission-evidence.csv` only (`JarSha256` = `NOT_PUBLISHED`).

---

## 2. Root Cause & Minimal Repair

### Diagnostic
In `BookingService.java`, the `cancel(String eventId, String studentId)` method validated arguments and confirmed the booking existed in `event.students`, but omitted removing the student. Consequently, the student remained on the roster, capacity was never restored, and the stale booking persisted to TSV storage.

### Applied Patch
A single line was added to `BookingService.java`:

```diff
diff --git a/src/main/java/BookingService.java b/src/main/java/BookingService.java
index 0de19f2..7e8823d 100644
--- a/src/main/java/BookingService.java
+++ b/src/main/java/BookingService.java
@@ -68,6 +68,7 @@ public final class BookingService {
         if (!event.students.contains(studentId)) {
             throw new IllegalArgumentException("No booking for student: " + studentId);
         }
+        event.students.remove(studentId);
     }
```

- **Tests Integrity**: All 28 tests in `FestPassTest.java` remained completely unchanged (`TestSourceSha256` = `d203243b893e64ada9a4752b4b686ee1012326b90b02cc481547679f3af98a34` in both runs).
- **Pipeline Integrity**: `Jenkinsfile` was not modified during repair (`PipelineSha256` = `828cfd6eaf1d108049cd8b444272da2bac58e7931dd1e0ebed3e0f7ad4ab6c2f` in both runs).

---

## 3. Green Build (Build #3) Details

- **Build Number**: 3
- **Build URL**: http://127.0.0.1:8080/job/festpass-ci/3/
- **Trigger Cause**: `Started by an SCM change` (Automatic Git SCM polling at `* * * * *`)
- **Git Commit**: `0be93a120c455fab930bcc06a593366ba7ba6b20`
- **Commit Message**: `Fix booking cancellation`
- **Author**: `Shams`
- **Result**: `SUCCESS`
- **Test Summary**: 28 total tests, 28 passed, 0 failures, 0 errors, 0 skipped
- **Smoke Test**: Passed all 7 CLI scenarios (`PASS: seven real-JAR commands; persistence, sold-out rejection, cancellation, and seat reuse verified.`)
- **Archived Artifacts**:
  - `festpass.jar` (`JarSha256` = `87a51ef04df64cd3cad6f942c4ee8feb12689b83e731a4ceb7ce255d7e1f3158`)
  - `submission-evidence.csv`
  - `build-info.txt`
  - `smoke-test.txt`

---

## 4. Executable JAR Verification

Executing `java -jar festpass.jar help` confirms personalized branding:
```text
FestPass - campus event registration (author: Shams)
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
  add-event ID "TITLE" CAPACITY
  list
  book EVENT_ID STUDENT_ID
  cancel EVENT_ID STUDENT_ID
  roster EVENT_ID
  summary
  help
Default data file: data/festpass.tsv
IDs are case-sensitive. Use synthetic student IDs, not personal data.
Exit codes: 0 success, 2 invalid request, 3 storage failure.
```

---

## 5. Submission Deliverables File Paths

All deliverables have been extracted from Jenkins and isolated in separate folders:

```
C:\CloudJenkinsLab\submission-deliverables\
├── red-build\
│   ├── submission-evidence.csv   (From Build #2, result FAILURE, JarSha256: NOT_PUBLISHED)
│   └── console-log.txt           (Full Jenkins console log proving SCM polling trigger)
└── green-build\
    ├── festpass.jar              (Executable application JAR, SHA-256: 87a51ef04df6...)
    ├── submission-evidence.csv   (From Build #3, result SUCCESS, 28/28 tests passed)
    ├── build-info.txt            (Jenkins build metadata)
    ├── smoke-test.txt            (Smoke test verification report)
    └── console-log.txt           (Full Jenkins console log proving SCM polling trigger)
```
