# AWS Least-Privilege IAM Design Lab

Least-privilege AWS IAM role design lab — scoped roles for app runtime, CI/CD deploy (GitHub OIDC), and developer debug access.

## Overview

This project demonstrates the design, implementation, and verification of least-privilege IAM roles for a 3-tier AWS web application. Each identity in the system — the running application, the CI/CD deployment pipeline, and a human developer — is granted only the permissions required for its specific function, with no broader access.

## Scenario

A web application running on AWS with the following components:

- An **EC2-based application** that reads/writes user-uploaded files to an S3 bucket and retrieves a database credential from Secrets Manager
- A **CI/CD pipeline** (GitHub Actions) that deploys new application code
- A **developer** who needs visibility into logs and bucket contents for debugging, without direct access to production data

## Roles

| Role | Purpose | Trust |
|---|---|---|
| `app-runtime-role` | EC2 instance profile for the running application | AWS service (EC2) |
| `deploy-pipeline-role` | CI/CD deployment via GitHub Actions | Web identity (GitHub OIDC), restricted to `main` branch |
| `developer-role` | Read-only debug access for a human developer | AWS account principal, MFA required |

Full permission tables and justifications are in [WRITEUP.md](./WRITEUP.md).

## Repository Contents

```
.
├── README.md
├── WRITEUP.md
└── policies/
    ├── app-runtime-role/
    │   ├── trust-policy.json
    │   └── permissions-policy.json
    ├── deploy-pipeline-role/
    │   ├── trust-policy.json
    │   └── permissions-policy.json
    └── developer-role/
        ├── trust-policy.json
        └── permissions-policy.json
```

## Verification

Each role was tested with `aws iam simulate-principal-policy` to confirm that intended actions are allowed and out-of-scope actions return `implicitDeny`. Results are documented in [WRITEUP.md](./WRITEUP.md#verification).

## License

MIT — see [LICENSE](./LICENSE).
