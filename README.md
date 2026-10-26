# TaskFlow API Contract

## 1. Overview
Rapid, AI-assisted development makes API contract changes frequent, increasing the risk of introducing modifications that break existing API consumers. Catching contract incompatibilities at pull-request (PR) time prevents broken client integrations, failed deployments, and runtime service disruptions before changes reach production.

In this workflow, a separate API Compatibility Analyzer integrates into a Jenkins CI pipeline as an automated compatibility gate. Jenkins compares the OpenAPI contract proposed in a pull request against the baseline contract on the target branch (`main`). If breaking changes are detected, Jenkins fails the build before downstream integration testing or merging can take place.

> **Note:** This repository contains only the version-controlled OpenAPI contract (`openapi.yml`) and this `README.md`. The compatibility analyzer, Jenkins pipeline definitions, and backend service implementations reside in separate repositories.

---

## 2. Workflow
The automated contract validation follows these steps:

1. A developer modifies the contract on a feature branch.
2. The developer commits the updated `openapi.yml`.
3. A pull request is opened against `main`.
4. Jenkins retrieves the baseline contract from the target branch (`main`) and the modified contract from the PR head commit.
5. Jenkins submits both contracts to the separate compatibility analyzer.
6. The analyzer evaluates differences across operations, parameters, schemas, and security requirements, returning a compatibility report.
7. Jenkins fails the compatibility gate if breaking changes are detected.
8. If the gate passes, the pipeline continues to integration testing and the rest of the workflow.

*In a real production workflow, an intentional breaking change would require an approved exception, a coordinated consumer migration plan, or an API versioning strategy (e.g., releasing `/api/v2`).*

---

## 3. Main Demo: Comprehensive Breaking-Change PR
This scenario demonstrates how Jenkins and the analyzer catch multiple breaking changes across different parts of the specification within a single PR.

### Branch
Create branch `demo/comprehensive-breaking-changes` from the unchanged `main` branch.

### Steps
1. Create and check out `demo/comprehensive-breaking-changes` from `main`.
2. Apply all five changes (A through E) detailed below to `openapi.yml`.
3. Validate the modified YAML syntax.
4. Commit and push the branch to the remote repository.
5. Open a pull request targeting `main`.
6. Wait for the Jenkins pipeline to run.
7. Inspect the analyzer's actual report output and the Jenkins gate result.

### Modifications
- **Change A (Operation removal):** Remove the `DELETE /tasks/{taskId}` path item and operation entirely.
- **Change B (Parameter type change):** Change the `limit` query parameter on `GET /tasks` from `type: integer` to `type: string`, keeping parameter name and location unchanged.
- **Change C (Enum restriction):** In the `GET /tasks` query parameter `status`, remove `completed` from the `enum` list (leaving only `pending` and `in_progress`). Leave `Task.status` and `UpdateTaskRequest.status` untouched.
- **Change D (Response property removal):** Remove the `description` property from the `Task` response schema in `components.schemas`. Keep `description` present in `CreateTaskRequest` and `UpdateTaskRequest`.
- **Change E (Global security change):** Add a global `security` requirement enforcing bearer auth, and add `components.securitySchemes` defining an HTTP bearer scheme (`type: http`, `scheme: bearer`, `bearerFormat: JWT`).

### Intended Findings (Not Verified Results)

| Modification | Detection area | Intended impact |
|---|---|---|
| Remove `DELETE /tasks/{taskId}` | Operation comparison | Breaking |
| Change `limit` from integer to string | Parameter comparison | Breaking |
| Remove `completed` from the query enum | Enum comparison | Breaking |
| Remove `Task.description` from the response schema | Schema comparison | Breaking |
| Add a global bearer authentication requirement | Security comparison | Breaking |

> **Note:** The actual analyzer report may contain a different number of findings or classifications depending on its rule engine and diff heuristics.

---

## 4. Secondary Demo: Backward-Compatible PR
This scenario demonstrates a safe contract change that expands functionality without breaking existing consumers.

### Branch
Create branch `demo/compatible-limit` from the original `main` branch.

### Steps & Expected Result
1. Create and check out `demo/compatible-limit` from the unchanged `main`.
2. Modify only `limit.maximum` on `GET /tasks`, raising it from `50` to `100`.
3. Commit, push, and open a separate pull request against `main`.
4. Inspect the Jenkins execution and analyzer report.

**Expected Impact:** Existing consumers sending integer values `1` through `50` remain fully valid, so the analyzer should ideally classify this change as non-breaking, allowing the Jenkins gate to pass. This behavior must be confirmed against the real analyzer.

---

## 5. Optional Demo: No Semantic Change
This scenario verifies that purely cosmetic or documentation edits do not trigger false positives.

### Branch
Create branch `demo/no-semantic-change` from `main`.

### Steps & Expected Result
1. Create and check out `demo/no-semantic-change` from `main`.
2. Edit only the wording of one operation's `description` field (e.g., updating the description of `GET /tasks`).
3. Commit, push, and open a pull request against `main`.
4. Inspect the result.

**Expected Impact:** Non-structural text changes do not affect contract semantics; the analyzer should report no breaking changes and Jenkins should pass. This must be confirmed against the real analyzer.

---

## 6. Running Scenarios Independently
Every scenario must be evaluated against the original, non-breaking baseline:
- Always create each scenario branch directly from the unchanged `main` branch.
- Do not merge the breaking-change PR into `main` before running the compatible demo, or reset `main` to the baseline between demos.
- Never combine breaking changes and compatible changes into the same pull request.

---

## 7. Limitations
- **Documentation vs. Implementation:** The analyzer inspects only the documented OpenAPI specification; it does not verify whether the backend code or mock server actually obeys the contract.
- **Rule Engine Dependencies:** Results and classifications depend entirely on the specific diff engine and rules configured in the compatibility analyzer.
- **Intentional Breaking Changes:** A breaking classification does not necessarily indicate a bug—intentional breaking changes require coordinated deprecation notices, consumer migration plans, or a major version bump.
