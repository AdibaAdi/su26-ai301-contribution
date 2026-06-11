# Contribution 1: Add Expression scripting support for knn_vector

**Contribution Number:** 1  
**Student:** Adiba Akter  
**Issue:** https://github.com/opensearch-project/k-NN/issues/459  
**Fork:** https://github.com/AdibaAdi/k-NN  
**Branch:** [fix-expression-scripting-knn-vector](https://github.com/AdibaAdi/k-NN/tree/fix-expression-scripting-knn-vector)  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it connects directly to vector search, OpenSearch, scripting engines and backend search infrastructure. Since I am interested in software engineering work involving data systems and AI infrastructure, this issue gives me a chance to understand how a real search engine plugin handles vector field behavior across different scripting languages.

The issue asks for Mustache and Expression scripting support for the `knn_vector` field. I noticed there is already an open PR focused on Mustache support, so my planned scope is to investigate the remaining Expression scripting behavior and understand what is missing compared with the existing painless scripting path.

---

## Understanding the Issue

### Problem Description

OpenSearch k-NN currently supports painless scripting with the `knn_vector` field type, but the issue asks for support across other OpenSearch scripting languages as well. Since Mustache support already has an open PR, the remaining useful scope appears to be understanding and potentially adding Expression scripting support for `knn_vector`.

### Expected Behavior

Users should be able to use the `knn_vector` field with supported OpenSearch scripting behavior beyond only painless scripting, specifically Expression scripting if that scope is still valid for the project.

### Current Behavior

Based on the issue, `knn_vector` scripting support is currently limited and does not fully support the other scripting languages mentioned in the issue.

### Affected Components

The likely affected components are the k-NN plugin scripting implementation, the existing painless scripting support path, script engine integration and tests around `knn_vector` script behavior.

---

## Reproduction Process

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

This is a feature-gap issue, not a crash, so "reproduce" means showing with a minimal, runnable example that an Expression script over a `knn_vector` field is rejected today, and capturing the exact behavior and the stage that produces it.

I mirrored the project's existing Painless test (`PainlessScriptScoreIT.testL2ScriptScore`) and changed only the script language, from Painless to Expression. Holding everything else constant isolates the gap to the Expression path.

1. Build the native libraries: `./gradlew buildJniLib`.
2. Confirm the Painless harness works (proves cluster, JNI load, and script-scoring path are healthy):
   ```
   ./gradlew integTest --tests "org.opensearch.knn.integ.PainlessScriptScoreIT.testL2ScriptScore"
   ```
3. Add a minimal Expression test, `ExpressionScriptScoreIT.testExpressionScriptScoreFailsOnKnnVector`, that:
   - creates a 2-dimensional `knn_vector` index with one document `[1.0, 1.0]`;
   - runs a `script_score` query with `"lang": "expression"` and source `doc['my_vector'].value` (the Expression-grammar parallel of the Painless `doc['my_vector'].value[0]` that passes today);
   - asserts HTTP 400 and logs the full error body.
4. Run only that test:
   ```
   ./gradlew integTest --tests "org.opensearch.knn.integ.ExpressionScriptScoreIT.testExpressionScriptScoreFailsOnKnnVector"
   ```

The only variable changed between the two tests is `lang`: Painless returns 200 OK; Expression returns 400.

### Reproduction Evidence

- **Painless baseline (control):** `PainlessScriptScoreIT.testL2ScriptScore` -> **HTTP 200 OK**.
- **Expression reproduction:** `ExpressionScriptScoreIT.testExpressionScriptScoreFailsOnKnnVector` -> **HTTP 400 Bad Request**.

The Expression test passes precisely because the server rejected the request with the expected 400 (`BUILD SUCCESSFUL`; JUnit `tests="1" skipped="0" failures="0" errors="0"`).

A diagnostic reproduction test was created locally for Phase II evidence and has not been committed.

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

**Expression scripting over a `knn_vector` field fails during field binding / query creation, because `knn_vector` is not a numeric, date, or geopoint field.**

Reading the error chain from the outside in:

1. `query_shard_exception: "failed to create query: link error"` (the failure happens while building the query on the shard, before any document is scored).
2. `script_exception: "link error"` on `doc['test_field'].value` (the Lucene Expression link/bind phase, where each `doc['...']` variable must resolve to a numeric value source).
3. `parse_exception: "Field [test_field] must be numeric, date, or geopoint"`. This is the root cause: Expression only binds numeric/date/geopoint fields, and `knn_vector` is none of these.

What this rules out:

- **Not a syntax problem:** `doc['my_vector'].value` is valid Expression syntax; it compiles. The failure is binding, not parsing.
- **Not a script-execution problem:** query creation fails before any per-document execution.

One architectural observation, **inferred and not yet verified at a specific line:** Painless support for `knn_vector` works because k-NN registers scoring functions into Painless via an allowlist extension (`KNNAllowlistExtension`). Lucene Expressions have no equivalent allowlist mechanism and bind only scalar numeric values, and the rejecting message appears to originate in OpenSearch core (`org.opensearch.script.expression`) rather than the k-NN plugin. If that holds, the honest Phase II finding is a documented structural constraint (Expression's one-numeric-value-per-document contract cannot represent a vector field) rather than a missing feature flag. The exact core source line still needs confirmation.

### Proposed Solution

Based on current evidence I lean toward documenting the structural constraint and, if maintainers agree, surfacing a clearer unsupported-field message for `lang: expression` on `knn_vector`, rather than forcing a feature path the engine may not support. I want to confirm this against OpenSearch core and the issue thread before committing to a direction (see the plan below).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** #459 asks for Expression scripting on `knn_vector`. Reproduction shows it is rejected at field binding because the field is non-numeric (HTTP 400, error above).

**Match:** Relevant precedents in the repo: `KNNAllowlistExtension` (how Painless support is wired), `KNNScoringScriptEngine` (k-NN's own `lang: "knn"` engine), and `PainlessScriptScoreIT` (the test pattern to mirror).

**Plan:** Two directions, pending a closer read of core and maintainer input:
1. *Document the constraint and clarify the error.* If Expression structurally cannot bind a vector field, contribute documentation and, if maintainers agree, a clearer unsupported-field message. Smallest and non-duplicative.
2. *Investigate a feature path.* Explore whether any numeric binding could be exposed. Higher uncertainty, since Expression has no allowlist mechanism and per-dimension binding may not be meaningful for vector scoring.

**Implement:** Out of scope for Phase II.

**Review:** Re-run the duplicate-work check (issue, open PRs, recent commits) before any code; match local conventions; keep the diff minimal.

**Evaluate:** Success means the chosen direction is backed by the reproduction, agreed with maintainers, and covered by a test mirroring the Painless one.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Phase II Status

* Environment validated: Java 21, `compileTestJava`, native dependencies, `buildJniLib`, and the existing Painless integration test all passed.
* Gap reproduced: the Expression test returns HTTP 400 with `Field [test_field] must be numeric, date, or geopoint`.
* Root cause located: failure is at Expression field binding / query creation, because `knn_vector` is not numeric/date/geopoint.
* No final fix implemented yet.
* No PR created yet.
* Next: confirm the producing core source line, then align with maintainers on the issue before any Phase III work.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
