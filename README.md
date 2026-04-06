# AWS IAM Cross-Account Access — Cloud Security Project

![AWS](https://img.shields.io/badge/AWS-IAM%20%7C%20STS%20%7C%20CloudTrail-orange?logo=amazonaws)
![Security](https://img.shields.io/badge/Security-Cloud%20Security-blue)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

## Overview

This project implements **production-grade IAM cross-account access** on AWS, demonstrating how real organizations securely manage permissions across multiple AWS accounts without long-lived credentials.

The setup follows the **hub-and-spoke model**: a central Security Account that assumes roles into Workload Accounts — with MFA enforcement, ExternalId conditions, and full CloudTrail auditing.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      AWS ORGANIZATION                        │
│                                                              │
│  ┌─────────────────────┐        ┌──────────────────────────┐ │
│  │  SECURITY ACCOUNT   │        │    WORKLOAD ACCOUNT      │ │
│  │  (Account A)        │        │    (Account B)           │ │
│  │                     │        │                          │ │
│  │  IAM User           │──STS──▶│  SecurityAuditRole       │ │
│  │  "SecurityAdmin"    │AssumeRole  (read-only audit)      │ │
│  │                     │        │                          │ │
│  │  + AssumeRole       │──STS──▶│  IncidentResponseRole    │ │
│  │    Permission       │AssumeRole  (incident handling)    │ │
│  │  + MFA Enforced     │        │                          │ │
│  └─────────────────────┘        │  Trust Policy:           │ │
│                                 │  - Principal: Account A  │ │
│                                 │  - Requires ExternalId   │ │
│                                 │  - Requires MFA          │ │
│                                 └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## What Was Built

### Roles Implemented

| Role | Account | Purpose | Permissions |
|------|---------|---------|-------------|
| `SecurityAuditRole` | Workload (B) | Read-only security review | `SecurityAudit`, `ViewOnlyAccess` |
| `IncidentResponseRole` | Workload (B) | Active incident handling | EC2, VPC, IAM read (scoped) |

### Security Controls Applied

- **MFA required** on every `AssumeRole` call
- **ExternalId condition** to prevent confused deputy attacks
- **Least privilege permissions** — no wildcards, only exact actions needed
- **Temporary credentials** (max 1 hour) — no long-lived secrets
- **CloudTrail logging** — every role assumption is recorded with timestamp, source IP, and session name

---

## How It Works — Role Assumption Flow

```
Security Account                          Workload Account
════════════════                          ════════════════

1. SecurityAdmin authenticates
   with MFA (TOTP)
         │
         ▼
2. Calls STS AssumeRole ──────────────▶  3. Trust Policy evaluated:
   with:                                    ✓ Principal = Account A
   - Role ARN                               ✓ ExternalId matches
   - ExternalId                             ✓ MFA present
   - MFA token
         │
         ◀──────────────────────────────  4. Temporary credentials returned
         │                                   (AccessKeyId + SecretKey + Token)
         ▼
5. Uses credentials to access
   resources in Workload Account
   (expires in max 1 hour)
```

---

## Implementation Details

### Trust Policy (on each role in Account B)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_A_ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UniqueSecretValue"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

### AssumeRole Permission Policy (on SecurityAdmin in Account A)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::ACCOUNT_B_ID:role/SecurityAuditRole",
        "arn:aws:iam::ACCOUNT_B_ID:role/IncidentResponseRole"
      ]
    }
  ]
}
```


### Assuming a Role (CLI)

```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::ACCOUNT_B:role/SecurityAuditRole" \
  --role-session-name "AuditSession" \
  --external-id "YourExternalId" \
  --serial-number "arn:aws:iam::ACCOUNT_A:mfa/username" \
  --token-code "123456" \
  --profile security-admin
```

---

## Design Decisions

### Why role assumption instead of long-lived credentials?

| Long-Lived Keys | Role Assumption |
|----------------|-----------------|
| Never expire | Credentials expire in max 1 hour |
| Can be leaked and reused forever | Stolen credentials become useless quickly |
| Hard to audit precisely | CloudTrail logs every single assumption |
| No MFA support | MFA can be enforced as a condition |
| Shared secrets to manage | No secrets — identity-based |

### Why ExternalId?

ExternalId prevents the **confused deputy problem**: if a third party tricks AWS into assuming your role on their behalf, they cannot do so without knowing your secret ExternalId. It's an extra layer of defense even if the role ARN is discovered.

### Why least privilege permissions?

The difference in blast radius if a role is compromised:

```
Over-privileged role                  Properly scoped role
════════════════════                  ════════════════════
Attacker can:                         Attacker can:
✗ Delete all resources                ✓ Read EC2 metadata
✗ Exfiltrate all data                 ✓ View security groups
✗ Create backdoor users               ✓ List IAM roles
✗ Disable logging                     Nothing else.
✗ Mine crypto
Impact: CATASTROPHIC                  Impact: LOW
```

---

## Scaling to 100+ Accounts

For large organizations, this pattern scales using:

- **AWS Organizations** — manage all accounts from a single pane
- **Service Control Policies (SCPs)** — enforce guardrails at the org level
- **CloudFormation StackSets** — deploy identical roles across all accounts automatically
- **AWS IAM Identity Center** — centralized SSO with permission sets replacing individual role management

---

## Monitoring & Detection

CloudTrail events to alert on:

- `AssumeRole` from unexpected source IPs
- `AssumeRole` failures (potential brute force)
- Role assumptions outside business hours
- Changes to trust policies (`UpdateAssumeRolePolicy`)
- Cross-account access from unapproved accounts

---

## Deliverables Checklist

- [x] `SecurityAuditRole` with Trust Policy + ExternalId + MFA condition
- [x] `IncidentResponseRole` with least privilege permission policy
- [x] `SecurityAdmin` user with scoped AssumeRole permission
- [x] MFA enforced on the assuming identity
- [x] Successful cross-account role assumption verified via CLI
- [x] CloudTrail enabled and AssumeRole events confirmed
- [x] Architecture documented with design rationale

---

## Technologies

- **AWS IAM** — Identity and Access Management
- **AWS STS** — Security Token Service (temporary credentials)
- **AWS CloudTrail** — API audit logging
- **AWS CLI** — Command-line testing and verification

---
