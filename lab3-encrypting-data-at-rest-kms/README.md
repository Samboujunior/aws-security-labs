# Lab 3: Encrypting Data at Rest Using AWS KMS

## Overview

This lab demonstrates how to protect data stored in Amazon S3 using **AWS Key Management Service (KMS)**. It covers creating a customer-managed KMS key, using it to encrypt S3 objects with **SSE-KMS**, validating that public access is blocked on encrypted objects, and generating **pre-signed URLs** for secure temporary access. The lab also shows the operational impact of disabling a KMS key — and how re-enabling it restores access to dependent resources like EC2 instances.

---

## AWS Services Used

| Service | Purpose |
|---|---|
| AWS KMS | Create and manage customer-managed encryption keys |
| Amazon S3 | Store encrypted objects (SSE-KMS) |
| Amazon EC2 | LabInstance — demonstrates KMS key dependency at boot |
| IAM | Define key administrative and usage permissions |

---

## Architecture

```
AWS KMS
    └── Customer-Managed Key (CMK)
            ARN: arn:aws:kms:us-east-1:590183947490:key/ea0e3f5b-7d0d-42e5-baba-a91a16c998f4
            Key Administrator: voclabs (IAM Role)
                    │
                    ▼
Amazon S3 Bucket: imagebucket-xxh4lcspjk8l
    └── clock.png
            Encryption: SSE-KMS (Override bucket default)
            S3 Bucket Key: Enabled
                    │
                    ├── Public URL → ✗ AccessDenied
                    └── Pre-signed URL → ✓ Accessible (time-limited)

Amazon EC2
    └── LabInstance (i-03f0a26c390a34798)
            └── EBS volume encrypted with same KMS key
                    ├── KMS key disabled → Instance fails to start
                    └── KMS key re-enabled → Instance starts successfully
```

---

## What Was Demonstrated

### 1. KMS Customer-Managed Key Creation
- Navigated to **AWS KMS → Customer-managed keys → Create key**.
- Followed the 6-step key creation wizard:
  - **Step 1:** Configure key type and options
  - **Step 2:** Add labels (alias/description)
  - **Step 3:** Define key administrative permissions — selected **`voclabs`** IAM role as the key administrator
  - **Step 4:** Define key usage permissions
  - **Step 5:** Edit key policy
  - **Step 6:** Review and create
- The resulting key ARN: `arn:aws:kms:us-east-1:590183947490:key/ea0e3f5b-7d0d-42e5-baba-a91a16c998f4`

### 2. S3 Object Encryption with SSE-KMS
- Uploaded an object (`clock.png`) to the S3 bucket `imagebucket-xxh4lcspjk8l`.
- During upload, configured **server-side encryption**:
  - Selected **"Specify an encryption key"**
  - Chose **"Override bucket settings for default encryption"**
  - Encryption type: **SSE-KMS** (Server-side encryption with AWS KMS keys)
  - Selected the customer-managed KMS key from the dropdown
  - **S3 Bucket Key:** Enabled (reduces KMS API call costs)

### 3. Access Denied on Public URL
- Attempted to access the encrypted object (`clock.png`) directly via its public S3 URL.
- Result: **AccessDenied** — KMS-encrypted objects cannot be accessed without valid AWS credentials and KMS decrypt permissions, even if the bucket were public.
- This confirms the encryption is enforced at the object level regardless of bucket ACL settings.

### 4. Successful Access via Pre-Signed URL
- Generated a **pre-signed URL** for `clock.png` with a short expiry (300 seconds / 5 minutes).
- The URL included AWS4-HMAC-SHA256 signature parameters:
  - `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Expires`, `X-Amz-Security-Token`, `X-Amz-Signature`
- Accessing the pre-signed URL successfully retrieved the object — demonstrating that temporary, scoped access can be granted without making the object public or changing its encryption.

### 5. KMS Key Disabled → EC2 Fails to Start
- The **LabInstance** EC2 instance (`i-03f0a26c390a34798`) had its EBS volume encrypted with the same KMS key.
- When the KMS key was **disabled**, the instance could not start — AWS cannot decrypt the EBS volume without access to the key.
- This demonstrated the critical dependency EC2 instances have on KMS keys when their storage is encrypted.

### 6. KMS Key Re-enabled → EC2 Starts Successfully
- Re-enabled the KMS key in the KMS console.
- Started the **LabInstance** — the console confirmed: *"Successfully initiated starting of i-03f0a26c390a34798"*
- Instance state: **Running** | Type: `t2.micro` | AZ: `us-east-1e`
- This shows that re-enabling a KMS key immediately restores access to all resources that depend on it.

---

## Key Concepts Learned

**SSE-KMS vs SSE-S3**

| Feature | SSE-S3 | SSE-KMS |
|---|---|---|
| Key management | AWS managed, no visibility | Customer visibility and control |
| Access audit | Not available | CloudTrail logs every key use |
| Key rotation | Automatic | Configurable (automatic or manual) |
| Disable/delete key | Not possible | Possible — blocks all dependent access |
| Cost | Included | KMS API call charges apply |

**S3 Bucket Key**
Enabling the S3 Bucket Key reduces the number of direct calls made to KMS by generating a short-lived data key at the bucket level. This significantly lowers KMS costs for buckets with high object counts while maintaining the same encryption strength.

**Pre-Signed URLs**
A pre-signed URL embeds temporary AWS credentials and a request signature into the URL itself. Anyone with the URL can access the object for the duration of the expiry window — without needing an AWS account or KMS permissions of their own. The signing entity's permissions are used at the time of signing, not at the time of access.

**KMS Key Administrators vs Key Users**
- **Key Administrators** (e.g., `voclabs` role) can manage the key — enable, disable, rotate, delete, and edit the key policy — but cannot necessarily use it for encryption/decryption.
- **Key Users** can call `kms:Encrypt`, `kms:Decrypt`, and `kms:GenerateDataKey` — used by services like S3 and EC2 to perform actual cryptographic operations.

**Operational Impact of Key Disabling**
Disabling a KMS key is not just a security measure — it has immediate operational consequences. Any EC2 instance with an EBS volume encrypted by that key will fail to start. Any S3 objects encrypted with that key become inaccessible. This makes KMS key management a critical operational responsibility.

---

## Configuration Summary

| Resource | Value |
|---|---|
| KMS Key ARN | `arn:aws:kms:us-east-1:590183947490:key/ea0e3f5b-7d0d-42e5-baba-a91a16c998f4` |
| Key Administrator | `voclabs` (IAM Role) |
| S3 Bucket | `c193177a4966730l14291483t1w59018394749-imagebucket-xxh4lcspjk8l` |
| Encrypted Object | `clock.png` |
| Encryption Type | SSE-KMS |
| S3 Bucket Key | Enabled |
| Pre-signed URL Expiry | 300 seconds (5 minutes) |
| EC2 Instance | `LabInstance` (`i-03f0a26c390a34798`) |
| Instance Type | `t2.micro` |
| Availability Zone | `us-east-1e` |
| Region | US East (N. Virginia) — us-east-1 |

---

## Screenshots

| Screenshot | Description |
|---|---|
| `KMS_Key_Creation.PNG` | KMS key creation wizard — Step 3: assigning voclabs as key administrator |
| `2__S3_Object_Encryption_Settings.PNG` | S3 upload — SSE-KMS encryption settings with customer-managed key selected |
| `3__Access_Denied_Error_when_trying_to_access_the_encrypted_object_publicly.PNG` | AccessDenied error when accessing encrypted object via public URL |
| `4__Successful_Decryption_via_Signed_URL.PNG` | Pre-signed URL with HMAC-SHA256 signature for temporary object access |
| `5__Re-enabling_the_KMS_Key_-_the_EC2_instance_starting_successfully_.PNG` | LabInstance running successfully after KMS key was re-enabled |

---

## Summary

This lab illustrated end-to-end data encryption at rest using AWS KMS. A customer-managed key was created with controlled administrative permissions, then used to encrypt an S3 object with SSE-KMS. Direct public access to the encrypted object was confirmed to be blocked, while temporary access was granted cleanly through a pre-signed URL. The lab also demonstrated the operational consequences of disabling a KMS key — blocking EC2 from starting — and confirmed that re-enabling the key immediately restored normal operation. Together, these steps highlight how KMS provides both strong encryption and fine-grained, auditable control over who can access encrypted data and when.
