# Service Specifics — Amazon SageMaker AI

Load only when the failing call is a SageMaker operation.

SageMaker is the service where the **two-hop problem** is most pronounced. A caller with
correct permissions still fails if the execution role cannot reach S3, ECR, or
CloudWatch — and the error surfaces at the caller as a generic `AccessDenied` with no
indication that the execution role is at fault.

## Applicable hops

| Call | Hop 1 | Hop 2 | Hop 3 | Hop 4 | Hop 5 | Hop 6 |
|---|---|---|---|---|---|---|
| `CreateTrainingJob` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `CreateProcessingJob`, `CreateTransformJob` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `CreateModel`, `CreateEndpointConfig`, `CreateEndpoint` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `CreatePipeline`, `StartPipelineExecution` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Execution role failing at runtime | n/a | n/a | n/a | ✓ | ✓ | ✓ |

Every job-creating call passes an execution role, so the full chain applies. This is the
opposite of Bedrock invocation, where hops 2–4 do not exist.

## The four failure modes that look identical

This table is the core of SageMaker access diagnosis. All four produce an
`AccessDenied` that a customer will describe the same way.

| # | Hop | What is missing | Observed `errorCode` | Who needs the fix |
|---|---|---|---|---|
| 1 | 1 | Caller lacks `sagemaker:CreateTrainingJob` | `AccessDeniedException` | Caller's policy |
| 2 | 2 | Caller lacks `iam:PassRole` for the execution role | `AccessDeniedException` | Caller's policy |
| 3 | 3 | Execution role's trust policy omits `sagemaker.amazonaws.com` | **`ValidationException`** — "Could not assume role" | Execution role trust policy |
| 4 | 4 | Execution role cannot reach S3, ECR, CloudWatch, or KMS | `AccessDenied` at runtime, or **`ValidationException`** — "No S3 objects found under S3 URL" at create-time | Execution role's permissions |

**The error codes are the fastest discriminator, and two of them are not access codes.**
Modes 1 and 2 both return `AccessDeniedException` and must be separated by reading the
policies. Modes 3 and 4, however, return `ValidationException` — so a diagnosis that
filters CloudTrail for `AccessDenied` will not find them at all and will wrongly conclude
that no denial occurred.

Mode 4 at create-time is the most deceptive: SageMaker validates the S3 input path using
the **execution role**, not the caller. If the execution role lacks `s3:ListBucket` on the
input prefix, `CreateTrainingJob` fails with "No S3 objects found under S3 URL", which
reads as missing data. The objects may exist and be readable by the caller. Verify the
objects independently — with `s3:ListBucket` as the agent, or by asking the user — before
reporting a data problem. If they exist, the cause is the execution role's S3 permissions.

Modes 2 and 3 are the pair most often conflated. Both are "PassRole problems" in casual
description, but the fix locations are different — one is the caller's identity policy,
the other is the role's trust policy. Adding `iam:PassRole` will not fix a trust-policy
gap, and amending the trust policy will not fix a missing `iam:PassRole`. Name which one.

Mode 4 is the most common in practice and the most likely to be misdiagnosed, because
hops 1–3 all pass and the natural conclusion is that permissions are fine.

## Identifying the execution role

- From the CloudTrail `requestParameters` on the failed create call: the `RoleArn` field.
- For an existing job or endpoint: `sagemaker:DescribeTrainingJob` or
  `sagemaker:DescribeEndpointConfig` then `DescribeModel`, and read `RoleArn` or
  `ExecutionRoleArn`.
- For Studio: `sagemaker:DescribeDomain` and `DescribeUserProfile` — a user profile can
  override the domain's default execution role, so check the profile before assuming the
  domain role is in play.

If the execution role cannot be identified, hops 2–4 are `CANNOT_DETERMINE`. Say so
rather than diagnosing hop 1 alone and implying the chain is clear.

## Hop 2 — PassRole specifics

The caller needs `iam:PassRole` with the execution role in `Resource`. A frequent
authoring pattern scopes it with a condition on the passing service:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::<account>:role/<execution-role>",
  "Condition": { "StringEquals": { "iam:PassedToService": "sagemaker.amazonaws.com" } }
}
```

Two things to check when hop 2 denies:
- Is the `Resource` the actual execution role ARN, or a different role or a wildcard that
  does not match?
- Is there an `iam:PassedToService` condition whose value does not match the service
  actually receiving the role? A condition naming a different service denies the pass
  while looking correctly configured.

## Hop 3 — Trust policy specifics

The execution role's trust policy must allow `sagemaker.amazonaws.com` to assume it:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "sagemaker.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Check for conditions that can deny assumption while appearing correct:
- `aws:SourceArn` or `aws:SourceAccount` scoped to a different job, account, or resource
- `sts:ExternalId` present but not supplied by the service

Note that some newer SageMaker features use different service principals. If the trust
policy names `sagemaker.amazonaws.com` and the call still fails at hop 3, check which
principal the specific feature requires before concluding the trust policy is correct.

## Hop 4 — Execution role downstream permissions

The four permission groups an execution role needs, per AWS: access to Amazon S3, Amazon
ECR, Amazon CloudWatch, and Amazon EC2.

| Group | Actions | Resource scoping | Purpose |
|---|---|---|---|
| S3 input | `s3:GetObject`, `s3:ListBucket` | Input bucket and prefix | Read training data |
| S3 output | `s3:PutObject` | Output prefix | Write model artifacts |
| ECR auth | `ecr:GetAuthorizationToken` | Must be `*` — this action does not support resource-level permissions | Authenticate to the registry |
| ECR pull | `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage` | Repository ARN | Pull the training or inference image |
| CloudWatch Logs | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, `logs:DescribeLogStreams` | Log group ARN | Emit job logs |
| CloudWatch Metrics | `cloudwatch:PutMetricData` | `*` with a namespace condition | Emit job metrics |
| KMS | `kms:Decrypt`, `kms:GenerateDataKey` | Key ARN | Encrypted volumes, S3 objects, or output |
| EC2 (VPC mode only) | `ec2:CreateNetworkInterface`, `ec2:CreateNetworkInterfacePermission`, `ec2:DeleteNetworkInterface`, `ec2:DescribeNetworkInterfaces`, `ec2:DescribeVpcs`, `ec2:DescribeSubnets`, `ec2:DescribeSecurityGroups` | `*` | Attach the job to the VPC |

**`ecr:GetAuthorizationToken` is the classic gap.** It does not support resource-level
permissions, so a policy that scopes all ECR actions to a repository ARN silently omits
it and image pulls fail. When hop 4 denies on an ECR action, check whether
`GetAuthorizationToken` is scoped to a repository rather than `*`.

**VPC mode changes the answer.** If the job specifies `VpcConfig`, the EC2 network
interface actions are required and their absence produces a failure that looks unrelated
to networking. Check `requestParameters.VpcConfig` on the create call before concluding
the EC2 group is unnecessary.

## Curated hop-4 permissions for Category B

For the proposed policy's "commonly required — not observed" section. Include only the
groups relevant to the operation that failed:

| Operation | Groups to include |
|---|---|
| `CreateTrainingJob` | S3 input, S3 output, ECR auth, ECR pull, Logs, Metrics, KMS if encrypted, EC2 if `VpcConfig` present |
| `CreateEndpoint` | S3 (model artifact read), ECR auth, ECR pull, Logs, Metrics, KMS if encrypted, EC2 if `VpcConfig` present |
| `CreateProcessingJob` | Same as training |
| Studio / notebook | S3, ECR, Logs, plus `sagemaker:*` scoped to the domain as appropriate |

Always label these "commonly required — not observed; verify against your workload" and
narrow the resource ARNs. AWS also publishes managed policies for job execution roles
covering these groups; referencing one is often a better recommendation than a
hand-built policy, and is worth surfacing as an option.

## S3 cross-account and bucket policies

A common real-world shape: the execution role is in account A and the data bucket is in
account B. Both an identity-based allow on the role and a bucket-policy allow in account
B are required. If the bucket is in another account, hop 5 is `CANNOT_DETERMINE` for the
bucket policy — follow the cross-account handling in `finding-logic.md` and name what
must be checked in account B.

Also check for a bucket policy with `aws:SecureTransport` or `s3:x-amz-server-side-encryption`
conditions that the job does not satisfy — these deny while the permissions themselves
look correct.

## Diagnostic order for SageMaker

1. Identify the execution role from `requestParameters` or the Describe call
2. Hop 1 — caller's `sagemaker:<Action>`
3. Hop 2 — caller's `iam:PassRole` for that specific role, including any
   `iam:PassedToService` condition
4. Hop 3 — trust policy principal and conditions
5. Hop 4 — execution role's downstream groups, selected by operation and by whether
   `VpcConfig` is present
6. Hop 5 — S3 bucket policy, KMS key policy, ECR repository policy
7. Hop 6 — SCP

Do not stop at a passing hop 1. In this service that is the beginning of the
investigation, not the end.

## References

- SageMaker roles: https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html
- API permissions reference: https://docs.aws.amazon.com/sagemaker/latest/dg/api-permissions-reference.html
- Managed policies for job execution roles: https://docs.aws.amazon.com/sagemaker/latest/dg/security-iam-awsmanpol-jobs.html
- ML activity reference: https://docs.aws.amazon.com/sagemaker/latest/dg/role-manager-ml-activities.html
- PassRole for pipelines: https://docs.aws.amazon.com/sagemaker/latest/dg/build-and-manage-access.html
- ECR service authorization reference: https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonelasticcontainerregistry.html
- PassRole: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html
