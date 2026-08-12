# Changelog

All notable changes to this skill are documented here. New entries go at the top.

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
