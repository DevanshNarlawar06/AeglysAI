# AeglysAI Contributor Qualification

GitHub Username: DevanshNarlawar06

- Public fork: https://github.com/DevanshNarlawar06/AeglysAI
- Qualification branch: `qualification/DevanshNarlawar06`
- Upstream commit tested: `48047e46aee115181dc0fd67a45105a0b7a45a45` (fix(docker): resolve NuGet-Migrations mutex initialization, 2026-11-02)

Environment: .NET 8 SDK 8.0.427 (installed per-user with the official dotnet-install script, channel 8.0), Git 2.50.1, curl; no Docker needed for this path.

OS: Windows 11 Home 10.0.26200 (x64)

Runtime Version: .NET SDK 8.0.427 / Microsoft.NETCore.App 8.0.x (net8.0)

## Setup and .NET 8 prerequisites

`global.json` pins SDK `8.0.204` with `rollForward: latestFeature`, so any 8.0.2xx+ SDK in the 8.0 band works.
I installed the .NET 8 SDK with `winget install --id Microsoft.DotNet.SDK.8 -e`, then ran from the repository root:

```sh
dotnet --info
dotnet restore
dotnet build --no-restore
dotnet test --no-build
dotnet run --project src/AeglysAI.Api -- --urls http://localhost:5080
curl http://localhost:5080/health
```

No credentials, tokens or private data were needed for restore, build, test or the health check.

## Build Result

**PASS**

`dotnet restore` then `dotnet build --no-restore`: Build succeeded, 0 Warning(s), 0 Error(s). Full log: [`DevanshNarlawar06-logs/build.txt`](DevanshNarlawar06-logs/build.txt).

## Test Result

**PASS**

`dotnet test --no-build`: 168 passed, 0 failed, 0 skipped.

| Test project | Passed |
| --- | ---: |
| AeglysAI.Application.Tests | 53 |
| AeglysAI.Domain.Tests | 14 |
| AeglysAI.ArchitectureTests | 4 |
| AeglysAI.Experiments.Tests | 4 |
| AeglysAI.IntegrationTests | 93 |

Full log: [`DevanshNarlawar06-logs/test.txt`](DevanshNarlawar06-logs/test.txt).

## Run Result

**PASS**

`dotnet run --project src/AeglysAI.Api -- --urls http://localhost:5080`, then `GET /health` returned `HTTP 200 {"status":"healthy"}` and an unauthenticated `GET /identity` returned `HTTP 401 Unauthorized`, as documented. Log: [`DevanshNarlawar06-logs/run.txt`](DevanshNarlawar06-logs/run.txt). I did not run the Docker `./dev` workflow.

## Architecture Understanding

AeglysAI V0.1 is a .NET 8 Clean Architecture solution. The dependency direction is enforced by
`tests/AeglysAI.ArchitectureTests/ProjectDependencyTests.cs`: Domain depends on nothing, Application on Domain,
Infrastructure on Application + Domain, and Api on Application + Infrastructure. Domain and Application must
also stay free of framework/package references.

- **Domain** (`src/AeglysAI.Domain/Authorization`): pure concepts. `Subject`, `Role`, `Permission`, `Resource`,
  `Action`, `AttributeSet`/`AttributeCondition`/`AbacRule`, and `RiskContext`. `RiskContext` validates that every
  signal is a defined `RiskSignal` in [0, 1] with no duplicates, and treats an absent signal as *unknown*, never zero.
- **Application** (`src/AeglysAI.Application`): decision logic behind interfaces.
  `RbacAuthorizationEngine` (Model A), `AbacEvaluator` + `ModelBAuthorizationEngine` (Model B),
  `ContextualRiskEngine` + `ModelCAuthorizationEngine` (Model C), the identity boundary (`ICurrentIdentity`) and the
  audit event contract.
- **Infrastructure** (`src/AeglysAI.Infrastructure`): in-memory providers for RBAC policy, ABAC context and risk
  context, loaded from server configuration (never from request bodies).
- **Api** (`src/AeglysAI.Api`): minimal API host. JWT bearer authentication, `GET /health` (anonymous),
  `GET /identity` and `POST /authorization/evaluate{,/model-b,/model-c}` (authenticated), configuration binding with
  strict key validation, and a logging audit sink with ActivitySource/Meter instrumentation.

### A/B/C baseline

Request flow: authenticated caller → `AuthorizationRequest(subject+issuer, resource, action)` → engine → decision → audit event.

- **Model A — RBAC.** ALLOW only if one of the caller's configured roles grants the resource/action; the default
  policy grants nothing, so the default is DENY.
- **Model B — RBAC + ABAC.** Requires the Model A grant **and** every applicable ABAC condition over server-owned
  subject/resource/environment attributes. Missing attributes or rules deny.
- **Model C — deterministic contextual risk.** Runs only after Model B allows (earlier DENY is final and the risk
  provider is not even called). `ContextualRiskEngine` computes a fixed weighted sum of five normalized signals
  (AuthenticationWeakness 0.25, ResourceSensitivity 0.25, OperationSensitivity 0.20, NetworkExposure 0.15,
  DeviceExposure 0.15). Any missing signal → `Unknown` → DENY. Otherwise score < 0.25 ALLOW, < 0.50 STEP_UP
  (obligation: stronger authentication + re-evaluate), < 0.75 LIMIT (obligation: 10 requests/minute),
  else DENY. Each decision returns per-signal contributions, so it is explainable. STEP_UP and LIMIT are
  recommendations with obligations; there is no enforcement yet.

Model D (behavioral/ML evidence under policy control, ADR-006) is planned and not required here.

## Technical Observation

**Model C's score is fully compensatory, so a single maximal signal can never escalate a request on its own.**

Because the score is a weighted *sum* and the largest weight is 0.25, one signal at its maximum value (1.0) with
all others at 0 produces at most 0.25:

| Signal at 1.0 (others 0) | Score | RiskLevel | Model C outcome |
| --- | ---: | --- | --- |
| AuthenticationWeakness | 0.25 | Medium | STEP_UP |
| ResourceSensitivity | 0.25 | Medium | STEP_UP |
| OperationSensitivity | 0.20 | Low | ALLOW |
| NetworkExposure | 0.15 | Low | ALLOW |
| DeviceExposure | 0.15 | Low | ALLOW |

So a fully exposed device (DeviceExposure = 1.0) or a fully exposed network on an otherwise low-risk request is
**ALLOW**, and even maximal authentication weakness alone only reaches STEP_UP. A high signal can also be
"diluted" by low values elsewhere. The existing tests in
`tests/AeglysAI.Application.Tests/ContextualRiskTests.cs` check exact boundaries only with *uniform* contexts
(all five signals equal), so this single-signal behavior is currently untested and undocumented as an intended
property.

How I tested it: I read `ContextualRiskEngine.cs` and `ModelCAuthorizationEngine.cs`, then wrote a temporary local xUnit test that builds a `RiskContext` with one signal at 1.0 and the rest at 0, calls `ContextualRiskEngine.Assess`, and maps the score through the same thresholds as `ModelCAuthorizationEngine`. The table above is its actual output. I removed the temporary test afterwards; it is not part of this branch.

Limitations: this is a property of the documented V0.1 heuristic, not a bug claim. Whether one exposed signal
*should* escalate is a policy choice that SPEC-007/ADR-004 would need to decide. Signal values are synthetic,
normalized indicators, not real telemetry.

## Improvement Idea

Add an explicit, versioned **per-signal escalation floor** to Model C, applied after the weighted sum:
if any single configured signal is at or above a declared threshold (for example `>= 0.9`), the outcome is at
least STEP_UP, regardless of the sum. Keep it deterministic and explainable by adding a reason such as
`SingleSignalFloor:DeviceExposure` to the decision, and bump `ContextualRiskEngine.Version`
(e.g. `contextual-weighted-v2`) so audit records stay traceable. Deny precedence, missing-context DENY and
the existing threshold bands do not change.

## Why It Matters

Compensatory scoring lets a strong, independent indicator of compromise (an unmanaged or exposed device) be
hidden by otherwise benign context. A floor is a common defence-in-depth pattern for risk-based authentication.
It also gives Model D a clearer target: behavioral evidence should be compared against a Model C that does not
silently average away a single strong signal, which matters for RQ1 (security effectiveness) and RQ2
(avoiding inappropriate actions).

## Validation Approach

1. **Unit tests (Application.Tests):** a `[Theory]` over all five signals at 1.0 with others at 0, asserting at least
   STEP_UP and the new reason code; the same with the signal at 0.89 asserting the v1 outcome is unchanged
   (floor not triggered); a test that existing uniform boundary cases (0, 0.2499, 0.25, …, 1) give identical
   outcomes, so the change is additive.
2. **Invariants kept:** existing tests for Baseline-denied → DENY (risk provider not called), missing signal → DENY,
   and provider failure → error must all still pass unchanged.
3. **Integration (IntegrationTests):** a Model C endpoint test with one exposed signal in configuration, asserting the
   response outcome, reason and audit event algorithm version.
4. **Experiment comparison:** rerun the synthetic A–C experiment runner (`research/README.md`) before and after, and
   report how many scenarios move from ALLOW to STEP_UP. The goal is a small, explained change, not more DENYs.
   Report unchanged or adverse results honestly.

## Sanitized evidence

Build/test/run logs: [`community/qualification/DevanshNarlawar06-logs/`](DevanshNarlawar06-logs/) on this branch (repository paths replaced with `<repo>`).

No credentials, tokens, machine usernames, private paths or personal data are included in this document or the logs.

## AI assistance disclosure

I used an AI coding assistant to help read the codebase and draft this document. The commands were run on my own machine, and I checked the observation, numbers and results against the actual source and test output.
