# Cloud Security Home Lab

**Author:** Andrew Mwine  
**Goal:** Build hands-on cloud security skills to demonstrate to employers  
**Platform:** AWS (Amazon Web Services)  
**Status:** 🟢 In progress

---

## What this repo is

This is a documented record of every cloud security lab I complete as I transition from software development into cloud security. Each folder contains:
- What the vulnerability or technique is
- Exact steps to reproduce it
- What it looks like from an attacker's perspective
- How to detect it
- How to fix it

This is real work done in a live AWS account, not theory.

---

## Lab Index

| # | Lab | Technique | Status |
|---|-----|-----------|--------|
| 01 | [IAM Least Privilege + S3 Misconfiguration](#lab-01) | IAM, S3 Public Access | ✅ Complete |
| 02 | flaws.cloud | S3 enumeration, IAM abuse | 🔜 Coming soon |
| 03 | CloudGoat — IAM Privilege Escalation | Privilege escalation | 🔜 Coming soon |
| 04 | CloudGoat — SSRF to Credential Theft | SSRF, metadata endpoint | 🔜 Coming soon |
| 05 | CloudTrail + GuardDuty Detection | Threat detection | 🔜 Coming soon |

---

## Lab 01

# Lab 01 — IAM Least Privilege and S3 Public Access Misconfiguration

**Date:** 3 June 2026  
**Platform:** AWS Console  
**Services used:** IAM, S3  
**Time taken:** ~1 hour  

---

### What I learned

Two of the most common causes of real cloud breaches:

1. **Overly permissive IAM users** — giving users more access than they need
2. **Public S3 buckets** — storage buckets accidentally exposed to the entire internet

Real-world examples of S3 bucket breaches include Capital One (2019), GoDaddy, and Twitch. All involved misconfigured cloud access controls.

---

### Part 1 — IAM Least Privilege

#### What is IAM?

IAM stands for Identity and Access Management. It controls who is allowed to do what inside an AWS account. Every action in AWS — creating a server, reading a file, deleting a database — requires an IAM permission. If those permissions are too broad, an attacker who gets hold of credentials can do enormous damage.

#### The principle of least privilege

A user should only have the minimum permissions they need to do their job — nothing more. This is the single most important rule in cloud security.

#### What I did

**Step 1 — Created an IAM user called `lab-user` with zero permissions**

This simulates a freshly created service account or developer account with no access yet.

![lab-user created with no permissions](screenshots/01-lab-user-no-permissions.png)

*What you can see: lab-user exists, Console access is Disabled, and the Permissions summary shows 0 policies attached. This user cannot do anything in AWS.*

---

**Step 2 — Attached the AmazonS3ReadOnlyAccess policy**

This gives lab-user the ability to list and read S3 buckets — but not create, modify, or delete anything.

![S3 read-only policy attached to lab-user](screenshots/03-s3-readonly-policy-attached.png)

*What you can see: AmazonS3ReadOnlyAccess is attached directly to lab-user. The user can now read S3 but has no other permissions in the entire AWS account.*

#### Why this matters

If an attacker steals this user's credentials, they can only read S3 — they cannot spin up servers, create admin users, access databases, or exfiltrate from other services. Least privilege limits the blast radius of a breach.

---

### Part 2 — S3 Public Access Misconfiguration

#### What is an S3 bucket?

An S3 bucket is cloud file storage — like a Google Drive folder that developers use to store files, backups, application assets, and data. By default, S3 buckets are private. The misconfiguration happens when "Block Public Access" is turned off, making everything in the bucket readable by anyone on the internet.

#### What I did

**Step 1 — Created a bucket called `lab-bucket-andrewmwine`**

A fresh bucket is private by default — this is the correct, secure state.

---

**Step 2 — Turned off Block Public Access (the misconfiguration)**

I went to the bucket → Permissions → Block public access → Edit → unticked all four checkboxes → saved.

![S3 bucket with public access turned on](screenshots/04-s3-bucket-public-misconfigured.png)

*What you can see: "Block all public access" is set to OFF with a warning triangle. This bucket is now exposed to the internet. Any file uploaded here would be publicly readable by anyone with the URL.*

**This is the misconfiguration that caused the Capital One breach in 2019.**

---

**Step 3 — Fixed the misconfiguration**

I went back to Permissions → Edit → ticked "Block all public access" → saved.

![S3 bucket with public access fixed](screenshots/05-s3-bucket-public-fixed.png)

*What you can see: "Block all public access" is back ON. The bucket is now private and secure.*

---

**Step 4 — Deleted the bucket**

After the lab, I deleted the bucket entirely to avoid any ongoing storage costs and to keep the AWS account clean. This is standard practice after every lab session.

---

### Detection — how would you catch this in a real environment?

If this misconfiguration happened in a production AWS account, here is how a security team would detect it:

**CloudTrail** would log the API call:
```
eventName: PutBucketAcl
requestParameters: {
  bucketName: "lab-bucket-andrewmwine",
  AccessControlPolicy: { ... public access granted ... }
}
```

**AWS Config** has a managed rule called `s3-bucket-public-read-prohibited` that fires an alert the moment any bucket is made public.

**AWS Security Hub** would flag this as a critical finding under the CIS AWS Foundations Benchmark.

---

### Prevention — how do you stop this from happening?

1. **Enable S3 Block Public Access at the account level** — this prevents any bucket in the entire account from ever being made public, regardless of individual bucket settings.

2. **Use AWS Config rules** to automatically detect and alert on public buckets.

3. **Use a Lambda function** to automatically re-enable Block Public Access the moment it is turned off (automated remediation — covered in Lab 05).

4. **Use Checkov or tfsec** to scan Terraform/CloudFormation code before deployment and catch this misconfiguration before it ever reaches production.

---

### Key takeaways

| Concept | What I learned |
|---------|---------------|
| Least privilege | Users should have only the permissions they need — tested by creating a read-only IAM user |
| S3 public access | A single checkbox can expose an entire bucket to the internet — this is how real breaches happen |
| Remediation | Turning Block Public Access back on immediately closes the exposure |
| Detection | CloudTrail, AWS Config, and Security Hub all provide visibility into this misconfiguration |
| Clean-up | Always delete lab resources after each session to avoid costs |

---

### Resources used

- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)
- [AWS S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Capital One breach analysis](https://krebsonsecurity.com/2019/07/capital-one-data-theft-impacts-106m-people/)
- [flaws.cloud](http://flaws.cloud) — next lab

---

*This lab was completed as part of a self-directed cloud security home lab. All work was done in a personal AWS account using free tier resources. No real data was exposed at any point.*
