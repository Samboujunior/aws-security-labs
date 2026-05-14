# Lab 1: AWS IAM Roles & S3 Bucket Access Control

## Overview

This lab explores how AWS Identity and Access Management (IAM) controls access to Amazon S3 resources. It demonstrates the difference between **identity-based policies** (attached to users/groups) and **resource-based policies** (attached to S3 buckets), and shows how IAM role assumption can grant elevated, scoped permissions that a standard user does not have.

---

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon S3 | Object storage — two buckets with different access configurations |
| AWS IAM | Users, groups, roles, and policies |
| EC2 (referenced) | Tested as a service the `devuser` cannot access |

---

## Architecture

```
IAM User: devuser
    └── Member of: DeveloperGroup
            └── DeveloperGroupPolicy (Customer Inline)
                    ├── cloudformation:Describe*, Get*, List*
                    ├── iam:Describe*, Get*, List*
                    └── logs:Desc*, Get*, List*
                    (No S3 or EC2 permissions)

IAM Role: BucketsAccessRole
    └── Bucket Policy on bucket2 grants:
            ├── s3:GetObject   ──► bucket2/*
            ├── s3:PutObject   ──► bucket2/*
            └── s3:ListBucket  ──► bucket2

Two S3 Buckets:
    ├── bucket1  (no resource policy granting BucketsAccessRole)
    └── bucket2  (resource/bucket policy grants BucketsAccessRole read/write)
```

---

## What Was Demonstrated

### 1. Access Denied — Bucket 1 (No Policy)
- Attempted to access an object (`Image2.jpg`) in **bucket1** directly via its S3 URL.
- Result: **AccessDenied** — the `devuser` has no identity-based policy allowing `s3:GetObject`, and bucket1 has no resource-based policy granting access.

### 2. DeveloperGroup IAM Policy
- Inspected the **DeveloperGroupPolicy** (Customer Inline) attached to `DeveloperGroup`.
- The policy grants read-only access to CloudFormation, IAM, and CloudWatch Logs — but **no S3 or EC2 permissions**.

### 3. EC2 Permission Denied
- Navigated to the EC2 console as `devuser`.
- Result: **Not authorized** to perform `ec2:DescribeInstances` — confirming the group policy does not include EC2 access.

### 4. Upload Access Denied (bucket1 / tg1312)
- Attempted to upload `DeveloperGroupPolicy.json` to an S3 bucket as `devuser`.
- Result: **Upload failed — Access Denied** — no `s3:PutObject` permission exists on the user or group.

### 5. Insufficient Permissions on Bucket 1 Access Points
- Navigated to the Access Points tab on bucket1 as `devuser`.
- Result: **Insufficient permissions to list access points** — missing `s3:ListAccessPoints`.

### 6. Role Switch to BucketsAccessRole
- Switched IAM role to **BucketsAccessRole** using the AWS Console role switcher.
- The role history in the account menu confirmed the active session changed to `BucketsAccessRole`.

### 7. Bucket 2 Resource Policy Inspection
- Viewed the **bucket policy on bucket2**, which explicitly grants `BucketsAccessRole`:
  - `s3:GetObject` and `s3:PutObject` on all objects (`bucket2/*`)
  - `s3:ListBucket` on the bucket itself

### 8. Successful Download via Role
- While assuming **BucketsAccessRole**, successfully accessed and downloaded `Image2.jpg` from **bucket1**.
- Result: **Download succeeded** — demonstrating that the role has the necessary trust and the bucket policy grants the required access.

---

## Key Concepts Learned

**Identity-based vs Resource-based Policies**
IAM identity-based policies are attached to users, groups, or roles. Resource-based policies (like S3 bucket policies) are attached directly to the resource and specify who (Principal) can perform which actions.

**Principle of Least Privilege**
The `devuser` was intentionally limited — they could inspect IAM and CloudFormation metadata but could not interact with S3 or EC2. This reflects the security principle of granting only the permissions needed for a specific job function.

**IAM Role Assumption**
By switching to `BucketsAccessRole`, the session temporarily gained the permissions defined for that role. This is how AWS recommends granting elevated or cross-service access — not by adding permissions directly to a user.

**Explicit Allow Required**
AWS IAM denies all actions by default. An explicit `Allow` must exist in either an identity-based policy or a resource-based policy (or both, depending on the scenario) for access to succeed.

---

## IAM Policy Snippets

### DeveloperGroupPolicy (Inline — partial)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "cloudformation:Describe*",
        "cloudformation:Get*",
        "cloudformation:List*",
        "iam:Describe*",
        "iam:GetAccountAuthorizationDetails",
        "iam:GetGroup",
        "iam:GetGroupPolicy",
        "iam:GetPolicy",
        "iam:GetRole",
        "iam:GetRolePolicy",
        "iam:GetUser",
        "iam:GetUserPolicy",
        "iam:List*",
        "logs:Desc*",
        "logs:Get*",
        "logs:List*"
      ]
    }
  ]
}
```

### Bucket 2 Resource Policy (partial)
```json
{
  "Statement": [
    {
      "Sid": "S3Write",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::058264216884:role/BucketsAccessRole"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::bucket2/*"
    },
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::058264216884:role/BucketsAccessRole"
      },
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::bucket2"
    }
  ]
}
```

---

## Region

**US East (N. Virginia) — us-east-1**

---

## Screenshots

| Screenshot | Description |
|---|---|
| `bucket1-download-denied.PNG` | AccessDenied error accessing bucket1 object as devuser |
| `developer-group-policy.PNG` | DeveloperGroupPolicy inline policy in IAM |
| `ec2-permission-denied.PNG` | devuser denied ec2:DescribeInstances |
| `upload-access-denied.PNG` | Upload to S3 failed — Access Denied |
| `s3-insufficient-permissions.PNG` | Insufficient permissions to list S3 Access Points |
| `role-switch-bucketsaccessrole.PNG` | Switching to BucketsAccessRole in the console |
| `bucket2-resource-policy.PNG` | Bucket policy on bucket2 granting BucketsAccessRole |
| `download-success-role.PNG` | Successful object access after assuming BucketsAccessRole |

---

## Summary

This lab reinforced that in AWS, **access is denied by default**. Permissions must be explicitly granted through identity-based policies (on users/groups/roles) or resource-based policies (on the resource itself). IAM role assumption is a secure and auditable way to grant elevated access without permanently modifying a user's permissions. The contrast between `devuser`'s restricted access and the elevated access available through `BucketsAccessRole` clearly illustrated how scoped, role-based access control works in a real AWS environment.
