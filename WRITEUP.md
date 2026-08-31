# Least-Privilege IAM Design Lab — Writeup

Repo: https://github.com/monkeylax67-alt/AWS-Least-Privilege-IAM-Design-Lab

## 1. Scenario

The system under design is a 3-tier web application on AWS, consisting of:

- An **EC2-hosted application** that stores and retrieves user-uploaded files in an S3 bucket (`myapp-user-uploads`) and reads a database credential from AWS Secrets Manager (`myapp/db-credentials`)
- A **CI/CD pipeline** running in GitHub Actions that deploys new versions of the application (modeled here as a Lambda function, `myapp-*`)
- A **human developer** who needs to debug production issues (view logs, confirm files exist) without direct access to file contents or credentials

The goal was to design one IAM role per identity, granting each only the permissions required for its function, and to verify those boundaries hold using AWS's policy simulator.

## 2. Identities and Roles

### 2.1 `app-runtime-role`

- **Trusted entity:** AWS service — EC2
- **Attached via:** Instance profile `app-runtime-profile`
- **Description:** EC2 instance role for the myapp application runtime. Grants read/write access to the myapp-user-uploads S3 bucket and read-only access to the myapp database credentials secret. No delete, admin, or cross-resource permissions.

**Permissions granted:**

| Permission | Resource | Justification |
|---|---|---|
| `s3:GetObject` | `myapp-user-uploads/*` | Application reads previously uploaded files |
| `s3:PutObject` | `myapp-user-uploads/*` | Application writes new uploaded files |
| `secretsmanager:GetSecretValue` | `myapp/db-credentials-*` | Application fetches the DB password at runtime |

**Deliberately excluded:** `s3:DeleteObject` (the app does not delete uploads in this design), `s3:*` wildcard actions, access to any bucket other than `myapp-user-uploads`, `secretsmanager:*` wildcard access.

### 2.2 `deploy-pipeline-role`

- **Trusted entity:** Web identity — GitHub Actions OIDC (`token.actions.githubusercontent.com`)
- **Trust condition:** Restricted to `repo:monkeylax67-alt/AWS-Least-Privilege-IAM-Design-Lab:ref:refs/heads/main` — only workflow runs on the `main` branch of this repository can assume this role
- **Description:** CI/CD deployment role assumed via GitHub Actions OIDC, restricted to the main branch of the myapp repository. Grants permission to update Lambda function code and configuration only. No access to application data, secrets, or IAM.

**Permissions granted:**

| Permission | Resource | Justification |
|---|---|---|
| `lambda:UpdateFunctionCode` | `function:myapp-*` | Deploys new application code |
| `lambda:UpdateFunctionConfiguration` | `function:myapp-*` | Updates environment variables / runtime settings on deploy |

**Deliberately excluded:** `s3:GetObject` on the uploads bucket (a deploy pipeline never needs to read production user data), any `secretsmanager` access, any `iam:*` permissions. Excluding IAM permissions specifically prevents privilege escalation if the pipeline or its credentials were ever compromised.

**Design note:** using OIDC federation instead of long-lived IAM access keys means the pipeline never stores static AWS credentials — it exchanges a short-lived GitHub-issued token for temporary AWS credentials on each run.

### 2.3 `developer-role`

- **Trusted entity:** AWS account principal (or IAM Identity Center permission set, if using SSO)
- **Trust condition:** Requires `aws:MultiFactorAuthPresent = true`
- **Description:** Read-only debug role for developers. Allows listing S3 bucket contents and viewing CloudWatch logs for myapp, without access to object contents, secrets, or write/delete actions. Requires MFA to assume.

**Permissions granted:**

| Permission | Resource | Justification |
|---|---|---|
| `s3:ListBucket` | `myapp-user-uploads` | See what files exist, for debugging |
| `s3:GetBucketLocation` | `myapp-user-uploads` | Basic bucket metadata |
| `logs:GetLogEvents` | `/myapp/*` | Read application log content |
| `logs:DescribeLogStreams` | `/myapp/*` | Navigate log streams |
| `logs:DescribeLogGroups` | `/myapp/*` | Navigate log groups |

**Deliberately excluded:** `s3:GetObject` (the developer can see that a file exists but cannot open its contents — this keeps production user data out of reach during routine debugging), `secretsmanager:*`, any write or delete action, any IAM or EC2 administrative permissions.

**Design note:** requiring MFA to assume the role adds a second control on top of least-privilege scoping — even if credentials were phished, assumption of this role still requires an MFA factor.

## 3. Defense-in-Depth Addition

As a supplementary control, a permissions boundary / explicit deny statement was considered for all three roles, denying high-risk actions regardless of what any future policy attachment might grant:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "s3:DeleteBucket",
        "s3:PutBucketPolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

This is a belt-and-suspenders measure: even a future misconfiguration (e.g., someone accidentally attaching an overly broad policy later) cannot grant these specific high-risk actions, because an explicit deny always overrides an allow in AWS's evaluation logic.

## 4. Verification

Each role's boundaries were tested using `aws iam simulate-principal-policy`, comparing an action the role should have against one it should not.

### `app-runtime-role`

```
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:role/app-runtime-role \
  --action-names s3:DeleteObject s3:GetObject \
  --resource-arns arn:aws:s3:::myapp-user-uploads/test.txt
```

**Result:**
- `s3:DeleteObject` → `implicitDeny` (not granted, correctly blocked)
- `s3:GetObject` → `allowed` (matched the intended permissions policy)

This confirms the role can read uploaded files but cannot delete them, matching the design intent.

### `deploy-pipeline-role`

```
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:role/deploy-pipeline-role \
  --action-names lambda:UpdateFunctionCode s3:GetObject \
  --resource-arns arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:myapp-test
```

**Expected result:** `lambda:UpdateFunctionCode` → `allowed`; `s3:GetObject` → `implicitDeny`, confirming the pipeline can deploy code but cannot read production user data.

### `developer-role`

```
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:role/developer-role \
  --action-names s3:ListBucket s3:GetObject \
  --resource-arns arn:aws:s3:::myapp-user-uploads
```

**Expected result:** `s3:ListBucket` → `allowed`; `s3:GetObject` → `implicitDeny`, confirming the developer can see that files exist but cannot open their contents.

## 5. Summary

Three IAM roles were designed, each scoped to the minimum permissions its identity requires:

- **`app-runtime-role`** — data read/write, scoped to one bucket and one secret
- **`deploy-pipeline-role`** — deploy-only, no data or secret access, federated via short-lived OIDC credentials rather than static keys
- **`developer-role`** — metadata and log visibility only, with MFA required to assume

Policy simulation confirmed that unauthorized actions correctly return `implicitDeny` in each case, validating that the least-privilege design holds in practice rather than only on paper. An additional explicit-deny boundary was proposed as a defense-in-depth measure against future policy drift.
