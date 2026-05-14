# AWS Security Labs

I built these labs as part of my AWS security training. Each one was done in a live AWS environment — these are my notes, screenshots and configs from working through them.

---

## Labs

### Lab 1 — IAM Roles & S3 Bucket Access Control](./lab1-iam-s3-access-control/README.md
In this lab I explored how AWS IAM controls access to S3. I tested what happens when a user has no S3 permissions, then switched to an IAM role that had a bucket policy granting access — and it worked. Good way to see the difference between identity-based and resource-based policies in action.

**Services:** IAM · Amazon S3 · EC2

**Key topics:** Identity vs resource-based policies · IAM role assumption · Explicit allow/deny · Least privilege

---

### Lab 2 — Securing VPC Resources Using Security Groups](./lab2-securing-vpc-resources/README.md
In this lab I set up network security inside a VPC. I started with a wide-open security group and progressively locked it down — first with a Network ACL, then restricting to a specific IP, then using security group referencing. I also tried both ways of getting into a private EC2 instance: the Bastion Host SSH method and AWS Session Manager.

**Services:** VPC · EC2 · Security Groups · Network ACLs · Systems Manager · NAT Gateway

**Key topics:** Stateful vs stateless firewalls · SG referencing · Bastion host pattern · Session Manager · Private subnet routing

---

### Lab 3 — Encrypting Data at Rest Using AWS KMS](./lab3-encrypting-data-at-rest-kms/README.md
In this lab I created my own KMS key and used it to encrypt an S3 object. I confirmed that the public URL gets blocked, then used a pre-signed URL to access it temporarily. I also disabled the KMS key to see what happens — the EC2 instance couldn't start until I re-enabled it. That part really showed how much AWS services depend on KMS keys.

**Services:** AWS KMS · Amazon S3 · EC2 · IAM

**Key topics:** SSE-KMS · Customer-managed keys · Pre-signed URLs · S3 Bucket Keys · Key disable impact

---

## Repo Structure

```
aws-security-labs/
├── README.md
├── lab1-iam-s3-access-control/
│   ├── README.md
│   └── screenshots/
├── lab2-securing-vpc-resources/
│   ├── README.md
│   └── screenshots/
└── lab3-encrypting-data-at-rest-kms/
    ├── README.md
    └── screenshots/
```

---

## Skills Demonstrated

- Configuring and troubleshooting IAM policies, roles, and group permissions
- Designing secure VPC architectures with public and private subnets
- Applying defence-in-depth using Security Groups and Network ACLs
- Managing encryption keys with AWS KMS and applying SSE-KMS to S3
- Using AWS Session Manager as a secure, keyless alternative to SSH
- Reading and interpreting AWS access denied errors and IAM policy evaluation logic
