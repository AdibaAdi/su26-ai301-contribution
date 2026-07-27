# Contribution 1: Document Expression scripting behavior for knn_vector

**Contribution Number:** 1  
**Student:** Adiba Akter  
**Issue:** https://github.com/opensearch-project/k-NN/issues/459  
**Fork:** https://github.com/AdibaAdi/k-NN  
**Branch:** [fix-expression-scripting-knn-vector](https://github.com/AdibaAdi/k-NN/tree/fix-expression-scripting-knn-vector)  
**Upstream PR:** https://github.com/opensearch-project/k-NN/pull/3364  
**Status:** Phase IV — upstream PR opened, awaiting review

---

## Contribution Summary

- Upstream PR [#3364](https://github.com/opensearch-project/k-NN/pull/3364) is **open and awaiting maintainer review**.
- The contribution adds **one integration test** to the OpenSearch k-NN plugin.
- The test **documents and guards the current unsupported Expression scripting behavior** for the `knn_vector` field: it asserts that an Expression `script_score` query over a `knn_vector` field is rejected with HTTP 400.
- The broader feature request in **issue [#459](https://github.com/opensearch-project/k-NN/issues/459) remains open.** Mustache and full Expression scripting support are broader than this PR, so this work is scoped as unsupported-behavior coverage (Refs #459) rather than a fix that closes the issue.

---

## Why I Chose This Issue

I chose this issue because it connects directly to vector search, OpenSearch, scripting engines, and backend search infrastructure. I am interested in software engineering work involving data systems and AI infrastructure, and this issue offered a chance to understand how a production search-engine plugin handles vector-field behavior across different scripting languages.

Issue #459 asks for Mustache and Expression scripting support for the `knn_vector` field. There is already an open PR focused on Mustache support, so I scoped my work to the remaining Expression scripting behavior and to understanding what differs from the existing Painless scripting path.

---

## Understanding the Issue

### Problem Description

OpenSearch k-NN supports Painless scripting with the `knn_vector` field type, but issue #459 asks for support across additional OpenSearch scripting languages. With Mustache support already covered by an open PR, the remaining useful scope is the Expression scripting behavior for `knn_vector`.

### Expected Behavior (per the issue)

Users should be able to use the `knn_vector` field with OpenSearch scripting languages beyond Painless, specifically Expression scripting, where that scope is still valid for the project.

### Current Behavior

Based on the issue and this reproduction, `knn_vector` scripting support is limited to Painless and does not extend to Expression scripting.

### Affected Components

The relevant components are the k-NN plugin scripting path, the existing Painless scripting integration, the OpenSearch scripting-language binding layer, and the integration tests around `knn_vector` script behavior.

---

## Phase II — Investigation & Reproduction

### Environment Setup

| Item | Value |
| --- | --- |
| OS | macOS, aarch64 (Apple Silicon) |
| Java toolchain | OpenJDK 21.0.11 (Temurin); `JAVA21_HOME` exported |
| Gradle wrapper | 9.4.1 |
| Native toolchain | cmake 4.3.3, openblas, libomp, gflags, gcc (Homebrew) |
| Submodules | `jni/external/faiss`, `jni/external/nmslib` at pinned commits |

Validation, in order (all passed):

1. **Java 21 configured:** `java`/`javac` report 21.0.11; `JAVA21_HOME` set.
2. **`./gradlew compileTestJava`:** test sources (including the `KNNRestTestCase` fixtures the reproduction extends) compile under JDK 21.
3. **Native dependencies installed:** cmake, openblas, libomp, gflags, gcc.
4. **`./gradlew buildJniLib`:** produced `jni/build/release/libopensearchknn_{faiss,nmslib,common,simd,util}.jnilib`.

> **Environment note:** `buildJniLib` first failed at CMake configure because the repo lived under a path containing spaces. The k-NN CMake scripts pass the patch-file path unquoted, so the shell split it. Moving the repo to a space-free path fixed it with no source change. This is an environment issue, not a defect in #459.

### Reproduction Method

This is a feature-gap issue, not a crash, so "reproduce" means showing with a minimal, runnable example that an Expression script over a `knn_vector` field is rejected today, and capturing the exact behavior and the stage that produces it.

I mirrored the project's existing Painless test (`PainlessScriptScoreIT.testL2ScriptScore`) and changed only the script language, from Painless to Expression. Holding everything else constant isolates the difference to the Expression path.

1. Build the native libraries: `./gradlew buildJniLib`.
2. Confirm the Painless harness works (proves the cluster, JNI load, and script-scoring path are healthy):
   ```
   ./gradlew integTest --tests "org.opensearch.knn.integ.PainlessScriptScoreIT.testL2ScriptScore"
   ```
3. Add a minimal Expression test that:
   - creates a 2-dimensional `knn_vector` index with one document `[1.0, 1.0]`;
   - runs a `script_score` query with `"lang": "expression"` and source `doc['my_vector'].value` (the Expression-grammar parallel of the Painless `doc['my_vector'].value[0]` that passes today);
   - asserts HTTP 400 and logs the full error body.
4. Run only that test:
   ```
   ./gradlew integTest --tests "org.opensearch.knn.integ.ExpressionScriptScoreIT.testExpressionScriptingUnsupportedOnKnnVector"
   ```

The only variable changed between the two tests is `lang`: Painless returns 200 OK; Expression returns 400.

### Reproduction Evidence

- **Painless baseline (control):** `PainlessScriptScoreIT.testL2ScriptScore` → **HTTP 200 OK**.
- **Expression reproduction:** Expression `script_score` over `knn_vector` → **HTTP 400 Bad Request**.

The Expression test passes precisely because the server rejects the request with the expected 400 (`BUILD SUCCESSFUL`; JUnit `tests="1" skipped="0" failures="0" errors="0"`).

**Response body captured by the test** (`test_field`/`test_index` are the base-class name constants):

```json
{
  "error": {
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "failed_shards": [
      {
        "shard": 0,
        "index": "test_index",
        "reason": {
          "type": "query_shard_exception",
          "reason": "failed to create query: link error",
          "caused_by": {
            "type": "script_exception",
            "reason": "link error",
            "script": "doc['test_field'].value",
            "lang": "expression",
            "caused_by": {
              "type": "parse_exception",
              "reason": "Field [test_field] must be numeric, date, or geopoint"
            }
          }
        }
      }
    ]
  },
  "status": 400
}
```

**Key error:**
> `Field [test_field] must be numeric, date, or geopoint`

---

## Solution Approach

### Analysis

The reproduction shows that **Expression scripting over a `knn_vector` field is rejected during query creation / field binding**, because `knn_vector` is not a numeric, date, or geopoint field.

Reading the error chain from the outside in:

1. `query_shard_exception: "failed to create query: link error"` — the failure happens while building the query on the shard, before any document is scored.
2. `script_exception: "link error"` on `doc['test_field'].value` — the Lucene Expression link/bind phase, where each `doc['...']` variable must resolve to a numeric value source.
3. `parse_exception: "Field [test_field] must be numeric, date, or geopoint"` — the root cause: Expression binds only numeric/date/geopoint fields, and `knn_vector` is none of these.

What this rules out:

- **Not a syntax problem:** `doc['my_vector'].value` is valid Expression syntax and compiles. The failure is binding, not parsing.
- **Not a script-execution problem:** query creation fails before any per-document execution.

Based on this reproduction, Painless support for `knn_vector` appears to work because k-NN registers scoring functions into Painless through an allowlist/extension path, whereas Lucene Expressions bind only scalar numeric values and have no comparable allowlist mechanism. The rejecting message appears to originate in OpenSearch core (the Expression binding layer) rather than the k-NN plugin. The exact core source line was not confirmed, so this is described as an observation from the reproduction rather than a verified internal fact.

### Decision: scoped guard test over speculative feature work

Because full Expression support would likely require deeper, possibly core-level changes — and because Expression's one-numeric-value-per-document binding model does not obviously map onto a vector field — I chose a **scoped guard test** that documents the current unsupported behavior, rather than a speculative feature implementation. This produces a small, non-duplicative, verifiable artifact tied to #459 and leaves the broader feature direction to maintainer input.

---

## Phase III — Implementation, Test & Commit

The Phase III change is an **integration test that guards the current unsupported behavior** of Expression scripting over a `knn_vector` field. It pins down and documents the existing rejection; it is **not** an implementation of full Expression scripting support. The test asserts that Expression scripting is rejected for `knn_vector` and will fail if that behavior ever silently changes.

### Integration Test

- `org.opensearch.knn.integ.ExpressionScriptScoreIT.testExpressionScriptingUnsupportedOnKnnVector` — creates a `knn_vector` index, issues a `script_score` query with `"lang": "expression"`, and asserts that OpenSearch rejects it (HTTP 400) because the field is not numeric/date/geopoint. This locks in the unsupported-behavior contract as a regression guard.

### Unit Tests

- None added. The behavior under test is a request-time field-binding rejection that only surfaces against a running cluster, so it is exercised through an integration test rather than a unit test.

### Manual Testing

```
./gradlew integTest --tests "org.opensearch.knn.integ.ExpressionScriptScoreIT.testExpressionScriptingUnsupportedOnKnnVector"
```
Result: `BUILD SUCCESSFUL`, JUnit `tests="1" skipped="0" failures="0" errors="0"`.

### Code Changes

- **File added:** `src/test/java/org/opensearch/knn/integ/ExpressionScriptScoreIT.java` (in the `opensearch-project/k-NN` repo, on the fork branch).
- **Test added:** `testExpressionScriptingUnsupportedOnKnnVector`.
- **Signed-off commit:** `8348e45d Add expression scripting unsupported integration test` on branch `fix-expression-scripting-knn-vector`.
- **Scope:** an unsupported-behavior regression guard. Full Expression feature support is not implemented; a scoped PR was opened for unsupported-behavior coverage.

---

## Phase IV — Pull Request

**PR Link:** https://github.com/opensearch-project/k-NN/pull/3364

**Summary:** The PR (Refs #459) adds a focused integration test, `ExpressionScriptScoreIT.testExpressionScriptingUnsupportedOnKnnVector`, that captures the observable unsupported behavior of Expression scripting over a `knn_vector` field. The test creates a `knn_vector` index, issues a `script_score` query with `"lang": "expression"`, and asserts that OpenSearch rejects it (HTTP 400) because `knn_vector` is not numeric/date/geopoint. Based on the reproduction, the rejection appears to surface during Expression field binding, and — unlike Painless, which is wired through a plugin allowlist/extension — Expression does not appear to have a comparable mechanism for binding a vector field. Rather than ship a speculative feature path, the PR contributes a verifiable regression guard tied to #459 and asks maintainers whether further work belongs in k-NN, in documentation, or in OpenSearch core, including whether a clearer unsupported-field error message is desired.

- **Signed-off commit:** `8348e45d`
- **Scope:** one integration test documenting unsupported Expression behavior for `knn_vector` (Refs #459). The PR does not close #459; the broader Mustache and full Expression support remain open.

**Maintainer Feedback:** None yet — awaiting review.

**Status:** Awaiting review.

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how OpenSearch k-NN integration tests are structured (the `KNNRestTestCase` harness and the `org.opensearch.knn.integ` test patterns).
- Learned how Painless script support differs from Expression behavior for `knn_vector`, including that Painless reaches k-NN scoring through an allowlist/extension path that Expression does not have.
- Learned how to isolate a scripting-language behavior by mirroring an existing Painless test and changing only the script language, holding everything else constant.
- Learned how to use targeted Gradle integration test runs (`./gradlew integTest --tests ...`) and how to interpret HTTP 400 responses and script field-binding failures.

### Challenges Overcome

- **Native plugin setup on macOS Apple Silicon:** stood up the Java/C++ k-NN toolchain (Java 21, cmake, openblas, libomp, gflags, gcc) and built the JNI libraries.
- **Path-with-spaces build failure:** diagnosed a CMake configure failure caused by an unquoted patch-file path and fixed it by moving the repo to a space-free path, with no source change.
- **Avoiding over-scoping:** recognized that full Expression support may require deeper, core-level changes, and resisted forcing a speculative feature path that the binding model may not support.
- **Keeping the PR review-ready:** kept the change minimal, signed-off, and focused on a single verifiable artifact tied to the issue.

### What I'd Do Differently Next Time

- Check for duplicate and related PRs earlier in the process.
- Confirm core source-line ownership earlier, before planning any feature implementation.
- Keep README sections updated continuously instead of filling placeholders near the end.
- Open a maintainer-scoping comment earlier when the issue appears larger than a plugin-level change.

---

## Resources Used

- OpenSearch k-NN issue #459 — https://github.com/opensearch-project/k-NN/issues/459
- OpenSearch k-NN PR #3364 — https://github.com/opensearch-project/k-NN/pull/3364
- OpenSearch k-NN repository — https://github.com/opensearch-project/k-NN
- OpenSearch `knn_vector` field documentation — https://docs.opensearch.org/latest/mappings/supported-field-types/knn-vector/
- OpenSearch exact k-NN search with scoring script — https://docs.opensearch.org/latest/vector-search/vector-search-techniques/knn-score-script/
- OpenSearch Get Script Languages API — https://docs.opensearch.org/latest/api-reference/script-apis/get-script-language/
- OpenSearch k-NN query documentation — https://docs.opensearch.org/latest/query-dsl/specialized/k-nn/
- Gradle — Testing in Java projects / filtering with `--tests` — https://docs.gradle.org/current/userguide/java_testing.html
- Gradle — Command-Line Interface reference — https://docs.gradle.org/current/userguide/command_line_interface.html
- GitHub — Creating a pull request from a fork — https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork
