# Changelog

All notable changes to this skill are documented here. New entries go at the top.

## [1.1.0] - 2026-08-12

Corrections from end-to-end validation against live Bedrock and SageMaker denials. Each
item below is a case the 1.0.0 logic would have diagnosed incorrectly.

### Fixed

- CloudTrail event selection no longer filters on `AccessDenied` alone. Two of the four
  SageMaker failure modes return `ValidationException`, so an `AccessDenied`-only query
  found neither and would have concluded that no denial occurred.
- Hop 3 (role trust policy) now documents its real signal: `ValidationException` with
  "Could not assume role", not an access error code. Added as evidence guidance on the
  finding, and to the activation description so the skill triggers on it.
- Hop 4 create-time S3 failures now documented as `ValidationException` with "No S3
  objects found under S3 URL". SageMaker validates the input path using the *execution
  role*, so a role lacking `s3:ListBucket` causes an object that exists to be reported as
  absent. Verified with a control job differing only in S3 permissions. Previously this
  would have been diagnosed as missing data.
- `cloudtrail:LookupEvents` documented as region-scoped. It returns only events from the
  queried region even with a multi-region trail, so the region-mismatch cause this skill
  claims to diagnose was invisible when querying the default region alone.
- Model deprecation added as a non-IAM cause. A legacy model returns
  `ResourceNotFoundException` whose message begins "Access denied", which must not be
  diagnosed as a permissions gap.
- `iam:PassRole` simulation now requires an `iam:PassedToService` context entry. Without
  it the condition in AWS's own recommended scoping pattern cannot be satisfied, and the
  simulator returns `implicitDeny` for a correctly configured caller. Verified both ways
  against a live role. This was the most damaging defect found: it would have sent
  customers to add a permission they already held while the real cause at hop 3 went
  unreported. Hop 2 now refuses to emit a denial when condition keys were missing, and
  returns `CANNOT_DETERMINE` instead.
- `ResourceArns` documented as mandatory, with evidence that omitting it errs in both
  directions — an `Allow` on `*` with a resource-specific `Deny` simulates as `allowed`
  (false negative), while a region-scoped `Allow` simulates as `implicitDeny` (false
  positive, blaming hop 1).
- A denial carrying non-empty `MissingContextValues` is no longer treated as evidence of a
  permission gap anywhere in the finding logic.
- Frontmatter `description` condensed to fit the DevOps Agent upload limit of 1024
  characters, which rejects the skill outright when exceeded. All trigger and exclusion
  phrases were preserved.

## [1.0.0] - 2026-08-11

### Added

- Initial release. Read-only diagnosis of IAM and access failures for Amazon Bedrock and
  Amazon SageMaker calls.
- Six-hop authorization chain traversal: caller action, `iam:PassRole`, role trust policy,
  role permissions, resource policy, and organization SCP, with a fixed evaluation order
  and documented precedence rules.
- Three-state verdict model — `DENIED_BY`, `ALLOWED_BUT_UNVERIFIABLE`,
  `CANNOT_DETERMINE`. There is deliberately no verdict asserting a hop permits the call,
  since simulation is a model of the policies rather than proof of live behavior.
- Implicit versus explicit deny distinction, carried into the remediation: an explicit
  deny cannot be resolved by adding a permission.
- Separation of the two `iam:PassRole` failure modes — the caller's missing permission
  versus the role's trust policy — which present with nearly identical symptoms.
- Bedrock non-IAM denial causes: model access not enabled, AWS Marketplace permissions
  for third-party models, propagation delay, and region mismatch.
- Cross-region inference profile handling, including the requirement to permit both the
  inference profile and the underlying foundation models in every destination region, and
  the case where an SCP blocking a single destination region fails the entire request.
- Propagation-delay detection by correlating `PutFoundationModelEntitlement`,
  `PutUseCaseForModelAccess`, `CreateFoundationModelAgreement`, Marketplace `Subscribe`,
  and IAM policy-attachment events against the denial timestamp. Emitted as an additive
  finding that never suppresses the rest of the diagnosis.
- SageMaker execution-role coverage: the four downstream permission groups, the
  `ecr:GetAuthorizationToken` resource-scoping constraint, and the VPC-mode EC2 network
  interface requirements.
- Cross-account partial diagnosis: the caller side is verified and attributed by policy
  type, while the remote resource policy is reported as undeterminable with named checks
  for the remote account.
- Proposed policy output in two labelled categories — permissions derived from the
  observed failure, kept separate from permissions commonly required but not observed.
  Wildcard resources are never emitted.
- Distinction between an agent-side permission gap and a customer-side finding, so a read
  the agent could not perform is never reported as an absent configuration.
- Fourteen pre-render validation checks, including one that fails the report if any hop is
  asserted as definitively allowed, and one that fails it if a computed value was
  approximated rather than calculated.
- Mandatory AI-generated banner on every report, required because the output proposes IAM
  policy changes.
- Output discipline rules: no narration of API calls, plans, or reasoning, and no
  post-delivery summary that could be read in place of the report.
- User-facing error handling table with graceful degradation on every condition — a single
  failed read marks its hop and continues rather than aborting the diagnosis.
- CloudFormation grant for `iam:SimulatePrincipalPolicy`, which is not part of the
  `AIDevOpsAgentAccessPolicy` managed policy.

### Known limitations at release

- Bedrock and SageMaker only. Other AI/ML services are reported as unsupported rather
  than diagnosed generically.
- Service control policies carrying conditions are not evaluated by the IAM policy
  simulator, so a conditional SCP can deny a call this skill reports as permitted.
- Session policies applied at role assumption are not visible in a role's attached
  policies.
- Resource policies and SCPs in remote accounts are not readable; cross-account
  diagnosis covers the caller side only.
