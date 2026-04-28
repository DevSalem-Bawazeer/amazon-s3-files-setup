# 🗂️ Amazon S3 Files — Setup Guide

> Mount your S3 bucket as a shared file system in minutes.  
> No data duplication. No sync pipelines. No code changes.

**By [Salem Bawazeer](https://github.com/DevSalem-Bawazeer) · April 2026**

---

## 📋 Table of Contents

- [What is S3 Files?](#what-is-s3-files)
- [Prerequisites](#prerequisites)
- [Step 1 — Create the S3 File System](#step-1--create-the-s3-file-system)
- [Step 2 — Create a Mount Target](#step-2--create-a-mount-target)
- [Step 3 — Mount on EC2](#step-3--mount-on-ec2)
- [Step 4 — Mount on EKS / ECS](#step-4--mount-on-eks--ecs)
- [Verify It Works](#verify-it-works)
- [Cost Breakdown](#cost-breakdown)
- [Important Caveats](#important-caveats)
- [Useful Links](#useful-links)

---

## What is S3 Files?

Amazon S3 Files makes your S3 bucket directly accessible as a **shared NFS file system** — without moving or duplicating your data. Built on Amazon EFS, it translates POSIX file system operations into S3 API calls on your behalf.

```
Your App (POSIX file ops)
        │
        ▼
  S3 File System  ◄──── NFS v4.1/4.2
        │
        ▼
  Your S3 Bucket  (data never leaves S3)
```

---

## Prerequisites

Before you begin, make sure you have:

- [ ] An existing **S3 bucket** (general purpose bucket, not directory/table/vector)
- [ ] A **VPC** with at least one subnet
- [ ] An **EC2 instance / EKS cluster / Lambda** in the same VPC
- [ ] IAM permissions: `elasticfilesystem:*` and `s3:*` on your bucket
- [ ] AWS CLI v2 installed and configured
- [ ] `amazon-efs-utils` installed on your EC2 instance

### Install EFS utils on EC2

```bash
# Amazon Linux 2 / AL2023
sudo yum install -y amazon-efs-utils

# Ubuntu / Debian
sudo apt-get install -y amazon-efs-utils

# Or via pip
pip install botocore
```

---

## Step 1 — Create the S3 File System

### Option A: AWS Console

1. Go to **S3 → your bucket → File system tab**
2. Click **Create file system**
3. Select your bucket (or a prefix within it)
4. Configure the **file size threshold** (default: 128 KB — files below this are cached on high-performance storage)
5. Configure the **data expiration window** (default: 30 days)
6. Click **Create**

### Option B: AWS CLI

```bash
# Create the file system linked to your bucket
aws s3files create-file-system \
  --bucket-name YOUR_BUCKET_NAME \
  --file-size-threshold-in-kb 128 \
  --data-expiration-in-days 30 \
  --region YOUR_REGION

# Save the file system ID from the output
export FS_ID=fs-xxxxxxxxxxxxxxxxx
```

### Option C: Terraform

```hcl
resource "aws_s3_file_system" "example" {
  bucket_name                  = aws_s3_bucket.example.id
  file_size_threshold_in_kb    = 128
  data_expiration_in_days      = 30
}
```

---

## Step 2 — Create a Mount Target

You need one mount target per Availability Zone where your compute runs.

```bash
# Get your subnet ID and security group
export SUBNET_ID=subnet-xxxxxxxxxxxxxxxxx
export SG_ID=sg-xxxxxxxxxxxxxxxxx

# Create a mount target
aws s3files create-mount-target \
  --file-system-id $FS_ID \
  --subnet-id $SUBNET_ID \
  --security-groups $SG_ID \
  --region YOUR_REGION

# Note the mount target IP or DNS name
```

> **Security Group rules required:**
> - Inbound: NFS (port 2049) from your compute security group
> - Outbound: All traffic (or at minimum port 443 to S3)

---

## Step 3 — Mount on EC2

```bash
# Create a mount directory
sudo mkdir -p /mnt/s3data

# Mount using NFS v4.1
sudo mount -t nfs4 \
  -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
  YOUR_MOUNT_TARGET_DNS:/ \
  /mnt/s3data

# Verify
df -h /mnt/s3data
ls /mnt/s3data  # You should see your S3 objects as files!
```

### Make it persistent (auto-mount on reboot)

Add to `/etc/fstab`:

```
YOUR_MOUNT_TARGET_DNS:/ /mnt/s3data nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,_netdev 0 0
```

---

## Step 4 — Mount on EKS / ECS

### EKS — using a PersistentVolume

```yaml
# pv-s3files.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3files-pv
spec:
  capacity:
    storage: 1Ti
  accessModes:
    - ReadWriteMany
  nfs:
    server: YOUR_MOUNT_TARGET_DNS
    path: "/"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3files-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Ti
  volumeName: s3files-pv
```

```bash
kubectl apply -f pv-s3files.yaml
```

Use in your pod:

```yaml
volumes:
  - name: s3data
    persistentVolumeClaim:
      claimName: s3files-pvc
containers:
  - name: my-app
    volumeMounts:
      - mountPath: /mnt/s3data
        name: s3data
```

### ECS — using an EFS volume config

```json
{
  "volumes": [{
    "name": "s3files-volume",
    "efsVolumeConfiguration": {
      "fileSystemId": "YOUR_FS_ID",
      "rootDirectory": "/",
      "transitEncryption": "ENABLED"
    }
  }],
  "mountPoints": [{
    "sourceVolume": "s3files-volume",
    "containerPath": "/mnt/s3data",
    "readOnly": false
  }]
}
```

---

## Verify It Works

```bash
# List files — should show your S3 objects
ls -la /mnt/s3data

# Read a file directly
cat /mnt/s3data/your-file.txt

# Write a file — it syncs back to S3 automatically
echo "Hello from file system!" > /mnt/s3data/test.txt

# Confirm it appeared in S3
aws s3 ls s3://YOUR_BUCKET_NAME/test.txt

# Check performance
time dd if=/mnt/s3data/large-file.bin of=/dev/null bs=1M
```

---

## Cost Breakdown

| Component | Price |
|-----------|-------|
| High-performance storage | **$0.30 / GB-month** (active small files only) |
| Write to file system | **$0.06 / GB** |
| Write sync back to S3 | **$0.03 / GB** |
| Small file reads (< 128 KB) | **$0.03 / GB** |
| Large file reads (≥ 128 KB) | **FREE** (served direct from S3) |
| Existing S3 storage | Standard S3 rates apply as before |

### Real-world example (from AWS docs)
- 100 GB bucket, app reads 10 GB (94% large files), writes 1 GB
- **Total S3 Files monthly charge: ~$0.38**

> 💡 Cost is proportional to your *active working set*, not total data. Unused data auto-expires from high-performance storage after 30 days (configurable).

---

## Important Caveats

> ⚠️ **Things to know before going to production**

- **Not zero-setup** — you need to create a file system resource and mount targets per AZ. Your data stays in S3, but there is a setup step.
- **Supported compute only** — EC2, EKS, ECS, and Lambda. Not supported outside of AWS (no on-prem NFS mounts).
- **NFS v4.1 / v4.2 only** — older NFS versions are not supported.
- **Eventual sync** — writes go to high-performance storage first, then sync back to S3. Very recent writes may not immediately appear via `aws s3 ls`.
- **Not a replacement for all EFS use cases** — if you need pure POSIX file storage with no S3 backing, EFS is still the right choice.
- **Your existing S3 costs continue** — S3 Files is an *additional* charge on top of standard S3 storage and request pricing.

---

## Useful Links

- 📖 [Official AWS Docs — S3 Files](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files.html)
- 💰 [S3 Files Pricing](https://aws.amazon.com/s3/pricing/)
- 🚀 [Getting Started Tutorial](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-getting-started.html)
- 📣 [AWS Announcement Blog](https://aws.amazon.com/blogs/aws/launching-s3-files-making-s3-buckets-accessible-as-file-systems)
- 🔧 [AWS Pricing Calculator](https://calculator.aws/)

---

> **Found this useful?** Give it a ⭐ and share it with your team!  
> Questions or corrections? Open an issue or PR.

**Author:** [Salem Bawazeer](https://github.com/DevSalem-Bawazeer) — DevOps Engineer
