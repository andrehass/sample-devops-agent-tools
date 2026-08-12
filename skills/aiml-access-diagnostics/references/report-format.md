# Report Format

Structure of the rendered report, and the validation to run before delivering it.

Readability is a hard requirement: emoji markers always have a space after them, the
body text below a heading never repeats the heading's emoji, and every verdict is
traceable to the evidence that produced it.

## Required sections, in order

```
# AI/ML Access Diagnosis — <service> <action>

## Summary
## ⚠️ Incomplete Diagnosis Notice        <- only if any AgentAccessDenied occurred
## ⏳ Possible Propagation Delay          <- only if the propagation trigger fired
## Authorization Chain
## Findings
## Proposed Policy
## What This Diagnosis Cannot Tell You
## References
```

## Summary

Four elements in order.

**1. Metadata block**

```
- **Service:** <Amazon Bedrock | Amazon SageMaker>
- **Failed action:** <IAM action>
- **Principal:** <principal ARN>
- **Resource:** <resource ARN or "not specified in the error">
- **Account / Region:** <account id> / <region>
- **Cross-account:** <Yes — resource in <account id> | No>
- **Evidence:** <CloudTrail event at <ts> | user-supplied error text | both>
- **Diagnosing identity:** <the agent's own ARN>
```

The diagnosing identity matters: the report is a view from one principal's vantage, and
what it could not read is a function of that principal's permissions.

**2. Root cause line**

```
**Root cause:** <emoji> <hop name> — <one-line statement>
```

Or, when undetermined:

```
**Root cause:** ❓ Undetermined — <what was ruled out, and what could not be evaluated>
```

**3. Deny kind**, when a root cause was identified

```
**Denial type:** <Explicit deny — adding permissions will not help | Implicit deny — a scoped allow resolves this | Not applicable>
```

**4. Chain table**

```
| Hop | Check | Verdict |
|---|---|---|
| 1 | Caller action | <emoji> <short verdict> |
| 2 | PassRole | <emoji> <short verdict> |
| 3 | Role trust policy | <emoji> <short verdict> |
| 4 | Role permissions | <emoji> <short verdict> |
| 5 | Resource policy | <emoji> <short verdict> |
| 6 | Organization SCP | <emoji> <short verdict> |
```

**Verdict-to-emoji mapping:**

| Verdict | Emoji | Short form |
|---|---|---|
| `DENIED_BY` | ❌ | `Denied here` |
| `ALLOWED_BUT_UNVERIFIABLE` | ⚠️ | `Allows (unverified)` |
| `CANNOT_DETERMINE` | ❓ | `Cannot determine` |
| `NOT_APPLICABLE` | — | `Not applicable` |
| `NOT_EVALUATED` | — | `Not evaluated — denied at hop N` |

Note that ⚠️ rather than ✅ is used for the allow case. This is deliberate: no hop is
ever asserted as definitively permitting the call. Using a green check would imply a
certainty the evidence does not support.

## Findings

One `###` subsection per applicable hop, in traversal order.

**Heading:** `### <emoji> Hop <N>: <hop name>`

**Body:** the template from `finding-logic.md`, verbatim, with placeholders
substituted. Do not prepend the emoji to the body text — it belongs on the heading only.

Hops marked `NOT_APPLICABLE` or `NOT_EVALUATED` get a one-line entry, not a full
subsection.

Service-specific non-IAM findings appear after hop 6 under:

```
### <emoji> Service-specific: <cause name>
```

## Proposed Policy

Open with the banner, verbatim:

> ⚠️ **Review before applying.** This policy is a proposal derived from the evidence
> above. It has not been validated against your workload, and resource scoping should
> be narrowed to your specific resources before use. Applying IAM changes is outside
> this skill's scope — it performs read-only diagnosis.

Then the two categories from `finding-logic.md`, each with its heading and preamble,
each as a separate fenced JSON block. Never merge them into one policy document.

Omit this whole section, replacing it with a one-line explanation, when:
- the root cause is an explicit deny (state which statement to amend instead)
- the root cause is an SCP (direct to the organization administrator)
- the root cause is undetermined (say so; do not guess a policy)

## What This Diagnosis Cannot Tell You

**Mandatory. Never omit, never abbreviate.** A diagnosis without its boundaries is the
failure mode this skill exists to prevent.

Always include:

- The policy simulator does not evaluate SCPs that carry conditions, so a conditional
  SCP can deny a call this diagnosis reports as permitted.
- Session policies applied at role assumption are not visible in a role's attached
  policies and can narrow permissions further.
- AWS documents that simulator results can differ from the live environment.
- CloudTrail delivery can lag up to approximately 15 minutes, so a very recent call may
  not appear yet.

Add conditionally:

- For every `CANNOT_DETERMINE` hop: what could not be read and why.
- For cross-account: the specific checks required in the remote account.
- When simulation and CloudTrail disagree: that the cause lies outside what simulation
  models.
- When hop 4 permissions were checked: that the curated list is not exhaustive for the
  user's workload.

## References

Bulleted AWS documentation links relevant to the findings actually produced. Do not
include links unrelated to this diagnosis.

### Canonical URLs

- IAM policy evaluation logic: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- Policy simulator: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html
- `SimulatePrincipalPolicy` API: https://docs.aws.amazon.com/IAM/latest/APIReference/API_SimulatePrincipalPolicy.html
- PassRole: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html
- Role trust policies: https://docs.aws.amazon.com/IAM/latest/UserGuide/roles-managingrole-editing-console.html
- Service control policies: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- Troubleshooting access denied: https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_access-denied.html
- Bedrock model access: https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html
- Bedrock IAM: https://docs.aws.amazon.com/bedrock/latest/userguide/security-iam.html
- SageMaker roles: https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html
- SageMaker execution role: https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html#sagemaker-roles-create-execution-role

## Pre-render validation

Run all 12 checks before delivering. Do not output the validation results.

### Structure (4)

1. **Required sections present**, in order, none missing, none extra.
2. **Conditional sections gated correctly.** The incomplete-diagnosis notice appears if
   and only if an `AgentAccessDenied` occurred. The propagation section appears if and
   only if the trigger fired.
3. **Chain table complete.** All six hops present, each with a verdict or a
   not-applicable / not-evaluated marker.
4. **Findings match the table.** Every hop's finding severity agrees with its table row.
   No finding without a table row, no table row without a finding or marker.

### Verdict integrity (4)

5. **No plain "allowed" verdict anywhere.** No ✅ against a hop, and no phrasing that
   asserts a hop definitively permits the call.
6. **Root cause consistency.** If any hop is `DENIED_BY`, the root cause is the earliest
   such hop. If none is, the root cause is a service-specific cause or undetermined.
7. **Deny kind stated** whenever a root cause was identified, and the remediation
   matches it — no policy proposal for an explicit deny.
8. **Every `CANNOT_DETERMINE` names its missing evidence.** A bare "cannot determine"
   with no reason fails validation.

### Substitution and safety (3)

9. **No unsubstituted placeholders.** No `<...>` or `[...]` tokens remain outside fenced
   code blocks.
10. **No `"Resource": "*"`** in any proposed policy block.
11. **No credential material.** No secret values, access keys, or session tokens
    anywhere in the output. ARNs and aliases only.

### Completeness (1)

12. **Limitations section present and populated**, including one entry per
    `CANNOT_DETERMINE` hop.

## Artifact naming

When the runtime supports persisted artifacts:

```
aiml-access-diagnosis-<service>-<YYYY-MM-DD>.md
```

Where `<service>` is `bedrock` or `sagemaker`. See the Final Delivery Contract in
`SKILL.md` for delivery requirements.
