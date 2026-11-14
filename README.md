# AWS re/Start: My Lab Journal

My own hands-on work from the AWS re/Start program (Batch 33). Each lab folder has my screenshots, notes, the problems I ran into, and how I solved them.

> The original lab instructions come from my instructor's repo: https://github.com/jjrs07/restart-batch33
> This repo contains my own work and write-ups, not a copy of the instructions.

## Labs

| # | Lab | Date | Status | Biggest challenge |
|---|-----|------|--------|-------------------|
| 1 | [Amazon S3](my-s3-lab/README.md) | 2026-11-14 | Done | Bucket creation blocked by a service control policy |

## What I'm learning

- S3 buckets are private by default, and Block Public Access keeps them that way
- Versioning, delete markers, and lifecycle rules for controlling storage cost
- Encrypting data with SSE-KMS and restricting access with bucket policies
- Reading console error messages carefully (Regions, exact resource names, explicit denies)

## Challenges log (all labs)

| Date | Lab | Problem | Fix | Lesson |
|------|-----|---------|-----|--------|
| 2026-11-14 | S3 | Create bucket denied in us-east-2 by an SCP | Created it in another allowed Region | SCPs override IAM and can restrict Regions |
| 2026-11-14 | S3 | Bucket policy failed with "Policy has invalid resource" | Used the exact full bucket name in both ARNs | Copy the name instead of retyping it |

## Repo layout

```
.
├── README.md
├── LAB-TEMPLATE.md        <- copy this for each new lab
└── my-s3-lab/
    ├── README.md          <- my write-up
    └── screenshots/
```

## Safety note

No AWS account IDs, access keys, or credentials appear in this repo. Screenshots are cropped or covered where needed.
