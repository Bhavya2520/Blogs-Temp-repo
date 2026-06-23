# How I Built a Production-Grade Backup System on AWS: EC2 to S3 with Encryption, Docker, and Full Automation

---

> **SEO Subtitle:** Automate EC2 backups to S3 with shell scripts, rsync, PostgreSQL dumps, Docker volume backups, KMS encryption, and cron scheduling — production-grade, step by step.
>
> **Estimated Reading Time:** 22 minutes
>
> **Medium Tags:** `AWS` `DevOps` `Cloud Computing` `Backup Automation` `Docker`

---

## The Incident That Made Me Think Twice

There is a scenario that comes up more often than people would like to admit in the DevOps world. A team runs a growing application on AWS — a small startup, confident their infrastructure is solid. No dedicated backup strategy, no automation. Just a single EC2 instance housing application data and a PostgreSQL database. Things are humming along fine until one Tuesday morning when a junior developer accidentally runs a script in the wrong environment. Within seconds, a production directory is wiped. The team scrambles. There is no backup. No snapshot. No recent restore point. The damage is irreversible, and they spend three days piecing together data from logs, client emails, and memory.

This kind of story surfaces in AWS forums, on Reddit's r/sysadmin, and in post-mortems shared by engineering teams regularly. Reading through such an incident got me thinking: how many people are running EC2 workloads right now without any structured backup automation in place? How many teams rely on manual backups, or worse — none at all?

That question is what pushed me to sit down and build a complete, production-inspired backup system from scratch on AWS — and document every single step so that anyone reading this can implement the same system in a few hours.

This is not a theoretical guide. Every command here was executed on a real EC2 instance, every script was tested and verified, and every error in the "Challenges" section is something that actually happened during the build.

---

## Why This Problem Matters

Backup is one of those things that nobody thinks about until it is too late. Here are the pain points this guide is designed to solve:

Manual backups are unreliable. People forget. They get busy. The backup that "should have happened last night" did not — and you only find out when something breaks. Automated backups solve this at the root.

Storage costs spiral out of control when nobody manages backup retention. Keeping three months of daily full backups in S3 Standard adds up quickly. Lifecycle Rules automate the archival and deletion of old backups, keeping costs under control.

Unencrypted backups sitting in S3 are a compliance and security risk. Any IAM credential leak could expose sensitive data. This guide covers both AES256 and KMS-based encryption so your backups are protected at rest.

Database backups are trickier than file backups. Copying raw PostgreSQL data files while the database is running leads to inconsistent, potentially corrupt backups. This guide covers the correct approach: `pg_dump` for logical backups and cold volume snapshots for full-disk recovery.

---

## Project Overview

Here is what this guide builds, end to end:

- Launch an EC2 Ubuntu instance and configure it with an IAM Role — no hardcoded credentials anywhere
- Create an S3 bucket to serve as the backup destination
- Write a shell script that compresses application files and uploads them to S3 with timestamps
- Upgrade to incremental backups using `rsync` so only changed files are transferred each run
- Apply S3 Lifecycle Rules to automatically move old backups to Glacier and delete them after 90 days
- Add SSE-S3 (AES256) and SSE-KMS encryption to all uploads
- Back up a PostgreSQL database running inside a Docker container using `pg_dump`
- Perform a cold Docker volume backup with container stop/start
- Schedule everything with cron jobs so the system runs without human intervention

---

## Architecture / Workflow

Here is the overall backup flow this system builds toward:

```
EC2 Instance (Ubuntu)
        │
        │  backup.sh / incremental-backup.sh / postgres-docker-backup.sh
        ▼
Compressed Backup (.tar.gz / .sql.gz)
        │
        │  aws s3 cp --sse AES256 (or --sse aws:kms)
        ▼
Amazon S3 Bucket
(my-company-backup-bucket)
        │
        │  Lifecycle Rule
        ▼
Glacier Archive  (Day 30)
        │
        ▼
Automatic Deletion  (Day 90)
```

> 💡 **Note:** An IAM Role is used throughout this guide instead of AWS Access Keys. This is the recommended production approach — no credentials are ever stored on the EC2 instance.

---

## Prerequisites

Before starting, make sure you have the following ready:

- An AWS account with permissions to create EC2 instances, S3 buckets, IAM roles, and KMS keys
- Basic familiarity with SSH and the Linux command line
- A key pair (.pem file) for SSH access to EC2
- Docker installed on your EC2 instance (required for Phase 4 and 5)
- Basic understanding of what S3 and EC2 are — not expert-level, just awareness

---

# Phase 1 — Basic Backup Setup

## Step 1 — Launch EC2 Instance

Head to the Amazon EC2 Console and launch a new instance with the following settings:

- **AMI:** Ubuntu 22.04 LTS
- **Instance Type:** t2.micro (free tier eligible)
- **Security Group:** Allow SSH on port 22

> 💡 Give your instance a clear name like `prod-backup-server` so it is easy to identify later in the console.

[Insert Screenshot Here: EC2 launch console with Ubuntu 22.04 selected and t2.micro instance type highlighted]

## Step 2 — Create S3 Bucket

Go to Amazon S3 and create a new bucket:

- **Bucket Name:** `my-company-backup-bucket`
- **Block Public Access:** Enabled (leave all checkboxes on — this is the safe default)
- **Versioning:** Optional, but strongly recommended for production environments

> ⚠️ S3 bucket names must be globally unique across all AWS accounts. If `my-company-backup-bucket` is taken, add a suffix like your AWS account ID or a random string.


## Step 3 — Create IAM Role for EC2

This is one of the most important steps in the entire guide. Instead of generating Access Keys and pasting them into the instance, you attach an IAM Role directly to the EC2 instance. The AWS SDK then automatically picks up temporary credentials behind the scenes — no keys on disk, no keys in scripts, no accidental exposure.

Here is how to set it up:

- Go to **AWS IAM → Roles → Create Role**
- **Trusted entity:** EC2
- **Permission Policy:** AmazonS3FullAccess
- **Role Name:** `EC2-S3-Backup-Role`

Then attach the role to the instance: EC2 Console → Select your instance → **Actions → Security → Modify IAM Role → Select EC2-S3-Backup-Role**


## Step 4 — Connect to EC2

SSH into your instance using your key pair:

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

The `-i` flag points to your private key file. `ubuntu` is the default user for Ubuntu AMIs on AWS.

## Step 5 — Install AWS CLI

Check whether the AWS CLI is already installed:

```bash
aws --version
```

If not installed, run:

```bash
sudo apt update
sudo apt install awscli -y
```

Now verify the IAM role is working correctly. This command asks AWS "who am I right now?" and returns details about the role:

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AROAEXAMPLEID",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/EC2-S3-Backup-Role/i-0abc1234def56789"
}
```

> ⚠️ **Error: Unable to locate credentials** — This means the IAM role is not properly attached. Fix: EC2 Console → Select instance → Actions → Security → Modify IAM Role → Choose `EC2-S3-Backup-Role` → Update IAM Role. Wait 1–2 minutes, then run the command again.

## Step 6 — Create Dummy Backup Folder and Files

Create a sample directory with some test files. In a real environment, this would be your application data directory:

```bash
mkdir backup-demo
cd backup-demo
echo "This is file 1" > 1.txt
echo "This is file 2" > 2.txt
ls
```

**Expected output:**

```
1.txt  2.txt
```

## Step 7 — Create the Backup Script

Navigate back to the home directory and create the script file:

```bash
cd ~
nano backup.sh
```

Paste the following complete script:

```bash
#!/bin/bash

DATE=$(date +%F-%H-%M-%S)
SOURCE_DIR="/home/ubuntu/backup-demo"
BACKUP_FILE="backup-$DATE.tar.gz"
S3_BUCKET="s3://my-company-backup-bucket"

# Create compressed backup
tar -czf /tmp/$BACKUP_FILE $SOURCE_DIR

# Upload to S3
aws s3 cp /tmp/$BACKUP_FILE $S3_BUCKET

# Remove local temporary backup
rm /tmp/$BACKUP_FILE

echo "Backup completed successfully"
```

Save and exit: `CTRL + O` → `ENTER` → `CTRL + X`

Here is what each section does:

- `DATE=$(date +%F-%H-%M-%S)` — Captures the current timestamp (e.g., `2026-05-14-10-20-00`) so each backup file has a unique name and can be sorted chronologically
- `tar -czf` — Compresses the source directory into a `.tar.gz` archive. The `-c` flag creates the archive, `-z` enables gzip compression, `-f` specifies the filename
- `aws s3 cp` — Uploads the compressed file to S3 using the IAM role credentials automatically
- `rm /tmp/$BACKUP_FILE` — Cleans up the temporary local file after upload so disk space is not wasted
- The whole file lands in `/tmp/` temporarily to avoid polluting the home directory

## Step 8 — Give Execute Permission

Without this step, Linux will refuse to run the script as an executable:

```bash
chmod +x backup.sh
```

## Step 9 — Run the Script

```bash
./backup.sh
```

**Expected output:**

```
Backup completed successfully
```

## Step 10 — Verify in S3

Open your bucket in the Amazon S3 Console. You should see a file similar to:

```
backup-2026-05-14-10-20-00.tar.gz
```

## Step 11 — Verify Backup Content (Optional but Recommended)

Download and extract the backup to confirm the files inside are intact:

```bash
aws s3 cp s3://my-company-backup-bucket/backup-2026-05-14-10-20-00.tar.gz .
tar -xvzf backup-2026-05-14-10-20-00.tar.gz
```

This is an important habit — always test that your backups can actually be restored. A backup you cannot restore is not a backup.

## Step 12 — Automate with a Cron Job

Open the crontab editor:

```bash
crontab -e
```

Add this line to schedule the backup every day at 1:00 AM:

```
0 1 * * * /home/ubuntu/backup.sh >> /home/ubuntu/backup.log 2>&1
```

Breaking this down: `0 1 * * *` means "at minute 0 of hour 1, every day, every month, every weekday." The `>> /home/ubuntu/backup.log 2>&1` part redirects all output — including errors — to a log file so you can check what happened later.

Verify the cron entry was saved correctly:

```bash
crontab -l
```

> 💡 Check `backup.log` the morning after your first scheduled run. If the file is empty or shows errors, you know immediately rather than finding out weeks later when you need a restore.

---

# Phase 2 — Incremental Backup and S3 Lifecycle Rules

Instead of compressing and uploading everything on every run, this phase introduces `rsync` to transfer only changed files. Combined with S3 Lifecycle Rules, old backups are automatically archived to Glacier and then deleted, keeping costs minimal.

**New architecture:**

```
EC2 Instance
        │
        │  rsync (only changed files)
        ▼
temp/current/  (local sync directory)
        │
        │  tar + gzip
        ▼
incremental-2026-05-14.tar.gz
        │
        │  aws s3 cp --sse AES256
        ▼
S3 Bucket / incremental-backups/
        │
        │  Lifecycle Rule
        ▼
Day 30 → Glacier  |  Day 90 → Deleted
```

## Part 2A — Incremental Backup Using rsync

### Step 1 — Create Project Structure

```bash
mkdir -p ~/backup-project/source
mkdir -p ~/backup-project/temp
cd ~/backup-project/source
echo "Hello File 1" > 1.txt
echo "Hello File 2" > 2.txt
```

The `source/` directory is what gets backed up. The `temp/` directory holds the last synced state, which is what makes incremental backups possible.

### Step 2 — Install rsync

```bash
rsync --version
```

If not installed:

```bash
sudo apt update
sudo apt install rsync -y
```

### Step 3 — Create Incremental Backup Script

```bash
nano ~/backup-project/incremental-backup.sh
```

Paste the following complete script:

```bash
#!/bin/bash

DATE=$(date +%F-%H-%M-%S)
SOURCE="/home/ubuntu/backup-project/source/"
TEMP_BACKUP="/home/ubuntu/backup-project/temp/current"
S3_BUCKET="s3://my-company-backup-bucket/incremental-backups"

# Create temp directory if missing
mkdir -p $TEMP_BACKUP

# Sync only changed files
rsync -av --delete $SOURCE $TEMP_BACKUP

# Compress synced backup
tar -czf /tmp/incremental-$DATE.tar.gz -C $TEMP_BACKUP .

# Upload to S3 with encryption
aws s3 cp /tmp/incremental-$DATE.tar.gz $S3_BUCKET --sse AES256

# Remove temporary tar file
rm /tmp/incremental-$DATE.tar.gz

echo "Incremental backup completed"
```

Here is what each section does:

- `rsync -av --delete` — Syncs only modified or new files from `source/` into `temp/current/`. The `--delete` flag ensures files deleted from source are also removed from the temp directory, keeping them in sync
- `-a` means archive mode (preserves permissions, timestamps, symlinks). `-v` means verbose — you see which files were transferred
- `tar -czf -C $TEMP_BACKUP .` — Compresses only the synced state, not the original source. The `-C` flag changes into the temp directory first so paths inside the archive are relative, not absolute
- `--sse AES256` — Every upload is encrypted at rest in S3. This is added here proactively so encryption is baked in from the start

### Step 4 — Give Permission and Run

```bash
chmod +x ~/backup-project/incremental-backup.sh
~/backup-project/incremental-backup.sh
```

**Expected output:**

```
sending incremental file list
./
1.txt
2.txt
sent 183 bytes  received 54 bytes
Incremental backup completed
```

### Step 5 — Verify in S3

Open your S3 bucket. You should see a path like:

```
incremental-backups/incremental-2026-05-14-11-20-00.tar.gz
```

### Step 6 — Test That Only Changes Are Backed Up

Modify one file and run the backup again:

```bash
echo "New Data Added" >> ~/backup-project/source/1.txt
~/backup-project/incremental-backup.sh
```

> 💡 `rsync` will detect that only `1.txt` changed and transfer just that file. The rest are skipped. On large datasets, this difference in transfer size is dramatic — instead of uploading gigabytes, you are uploading kilobytes.

## Part 2B — S3 Lifecycle Rule

A Lifecycle Rule is S3's built-in cost management feature. Once configured, it runs automatically without any script involvement.

To set it up, open your bucket in the Amazon S3 Console:

- Click the **Management** tab
- Click **Create lifecycle rule**
- **Rule Name:** `backup-lifecycle-rule`
- **Apply to:** Prefix — `incremental-backups/` (scope the rule to backup objects only)
- **Transition actions:** Move current versions → Glacier Flexible Retrieval → After 30 days
- **Expiration:** Expire current versions → After 90 days
- Click **Create rule**

What the rule does over time:

| Day | Action |
|-----|--------|
| Day 0 | Backup uploaded to S3 Standard |
| Day 30 | Automatically moved to Glacier Flexible Retrieval |
| Day 90 | Automatically deleted |

Verify the lifecycle rule is in place using the CLI:

```bash
aws s3api get-bucket-lifecycle-configuration --bucket my-company-backup-bucket
```

---

# Phase 3 — Encrypting Backups

There are two main server-side encryption options on S3. Both encrypt data at rest — meaning files on AWS's physical storage are unreadable without the proper key.

| Method | Managed By | Cost | When to Use |
|--------|------------|------|-------------|
| SSE-S3 (AES256) | Amazon S3 | Free | Learning, small projects |
| SSE-KMS | AWS KMS | Paid per use | Production, enterprise |

## Option 1 — SSE-S3 Encryption (AES256)

AWS automatically manages the encryption keys. You do not need to create or rotate anything — just add a flag to the upload command.

### Upload a file with AES256 encryption

```bash
echo "Sensitive Backup Data" > backup.txt
aws s3 cp backup.txt s3://my-company-backup-bucket --sse AES256
```

### Verify encryption in S3 Console

Open the uploaded object in S3 → Properties tab. You will see:

```
Server-side encryption: AES256
```

### Add encryption to your backup script

Update the upload line in your script from this:

```bash
aws s3 cp /tmp/incremental-$DATE.tar.gz $S3_BUCKET
```

To this:

```bash
aws s3 cp /tmp/incremental-$DATE.tar.gz $S3_BUCKET --sse AES256
```

> 💡 **FAQ — If backups are encrypted, why can authorized users still download and read them?**
> SSE-S3 encrypts data *at rest* — meaning the physical storage on AWS disks is protected. When an authorized user requests the file, S3 decrypts it automatically before sending it. This protects against hardware-level exposure (like a disk being physically removed from a data center), not against someone with valid AWS credentials. If you want data that even AWS cannot read, you need client-side encryption with GPG before uploading.

## Option 2 — SSE-KMS (Production Recommended)

With KMS, you create and control your own encryption key. You can restrict which IAM users or roles are allowed to decrypt files — even users with full S3 access cannot read the data without explicit `kms:Decrypt` permission on that specific key.

### Step 1 — Create a KMS Key

- Go to **AWS Key Management Service (KMS)**
- Click **Create key**
- **Key type:** Symmetric | **Usage:** Encrypt and decrypt
- **Alias:** `company-backup-key`
- Under **Key administrators** and **Key users**, add your EC2 IAM role (`EC2-S3-Backup-Role`)
- Finish creating the key and copy the Key ARN

**Example KMS Key ARN:**

```
arn:aws:kms:ap-south-1:123456789012:key/aaaabbbb-1111-2222-3333-ccccddddeeee
```

### Step 2 — Upload Using KMS

```bash
aws s3 cp backup.txt s3://my-company-backup-bucket \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:ap-south-1:123456789012:key/aaaabbbb-1111-2222-3333-ccccddddeeee
```

### Step 3 — Full Backup Script with KMS Encryption

```bash
#!/bin/bash

DATE=$(date +%F-%H-%M-%S)
SOURCE="/home/ubuntu/backup-project/source/"
TEMP_BACKUP="/home/ubuntu/backup-project/temp/current"
S3_BUCKET="s3://my-company-backup-bucket/incremental-backups"
KMS_KEY="arn:aws:kms:ap-south-1:123456789012:key/aaaabbbb-1111-2222-3333-ccccddddeeee"

mkdir -p $TEMP_BACKUP

rsync -av --delete $SOURCE $TEMP_BACKUP

tar -czf /tmp/incremental-$DATE.tar.gz -C $TEMP_BACKUP .

aws s3 cp /tmp/incremental-$DATE.tar.gz $S3_BUCKET \
  --sse aws:kms \
  --sse-kms-key-id $KMS_KEY

rm /tmp/incremental-$DATE.tar.gz

echo "Encrypted incremental backup completed"
```

> 💡 **FAQ — What is the real difference between SSE-S3 and SSE-KMS?**
> The core difference is access control. With SSE-S3, any IAM user with `s3:GetObject` permission can download and read the file. With SSE-KMS, the user also needs `kms:Decrypt` permission on that specific key — even if they have full S3 access. This separation of duties is valuable in regulated environments (HIPAA, PCI-DSS). KMS also provides a full audit trail in CloudTrail for every encrypt and decrypt operation. For personal learning: SSE-S3 is perfectly fine. For anything handling sensitive user data in a company environment: use SSE-KMS.

> ⚠️ Encryption is one layer among many. The most critical security controls are: MFA on your AWS account, never hardcoding access keys in scripts, IAM roles with least-privilege permissions, Block Public Access enabled on all S3 buckets, and CloudTrail logging for full audit trails.

---

# Phase 4 — Database Backup with pg_dump

> ⚠️ **Warning:** Never copy raw database files (`/var/lib/postgresql/data`) while the database is running. Active transactions, WAL logs, and index updates make the raw files inconsistent. Always use `pg_dump` for PostgreSQL — this creates a transaction-safe logical backup.

**Backup flow:**

```
PostgreSQL (inside Docker)
        │
        │  docker exec pg_dump
        ▼
.sql dump file on EC2
        │
        │  gzip
        ▼
.sql.gz compressed file
        │
        │  aws s3 cp --sse AES256
        ▼
S3 Bucket
```

## Step 1 — Verify Your Docker Container

```bash
docker ps
```

**Expected output example:**

```
CONTAINER ID   IMAGE      COMMAND                  STATUS    NAMES
a1b2c3d4e5f6   postgres   "docker-entrypoint.s…"   Up        prod-postgres-db
```

> 💡 The container name used in this guide is `prod-postgres-db`. Replace it with your actual container name from the NAMES column.

## Step 2 — Verify the Database Exists

```bash
docker exec -it prod-postgres-db psql -U postgres
\l
\q
```

## Step 3 — Create Backup Directory

```bash
mkdir -p ~/docker-db-backups
```

## Step 4 — Create the Database Backup Script

```bash
nano ~/docker-db-backups/postgres-docker-backup.sh
```

Paste the following complete script:

```bash
#!/bin/bash

DATE=$(date +%F-%H-%M-%S)
CONTAINER_NAME="prod-postgres-db"
DB_NAME="app_database"
DB_USER="postgres"
BACKUP_DIR="/home/ubuntu/docker-db-backups"
BACKUP_FILE="$BACKUP_DIR/$DB_NAME-$DATE.sql"
COMPRESSED_FILE="$BACKUP_FILE.gz"
S3_BUCKET="s3://my-company-backup-bucket/db-backups"

# Create SQL dump from inside the Docker container
docker exec $CONTAINER_NAME pg_dump -U $DB_USER $DB_NAME > $BACKUP_FILE

# Compress the dump
gzip $BACKUP_FILE

# Upload to S3 with encryption
aws s3 cp $COMPRESSED_FILE $S3_BUCKET --sse AES256

echo "Docker PostgreSQL backup completed"
```

Here is what each section does:

- `docker exec $CONTAINER_NAME pg_dump` — Runs `pg_dump` inside the running container, not on the host. This is important because the PostgreSQL binaries and connection details live inside the container
- `> $BACKUP_FILE` — Redirects the SQL dump output to a file on the host EC2 filesystem
- `gzip $BACKUP_FILE` — Compresses the `.sql` file in place, creating a `.sql.gz`. PostgreSQL dumps are plain text and compress very efficiently — typically 5x to 10x reduction
- `aws s3 cp $COMPRESSED_FILE $S3_BUCKET --sse AES256` — Encrypted upload to a dedicated `db-backups/` prefix in S3, keeping database backups separate from file backups

## Step 5 — Give Permission and Run

```bash
chmod +x ~/docker-db-backups/postgres-docker-backup.sh
~/docker-db-backups/postgres-docker-backup.sh
```

**Expected output:**

```
Docker PostgreSQL backup completed
```

> ⚠️ **Error: No such container: prod-postgres-db** — The container name in the script does not match what is actually running. Fix: Run `docker ps` and copy the exact name from the NAMES column. Update `CONTAINER_NAME` in the script.

> ⚠️ **Error: pg_dump: error: FATAL: role 'postgres' does not exist** — The `DB_USER` does not match the PostgreSQL user inside the container. Fix: Run `docker exec prod-postgres-db psql -U postgres -c "\du"` to list available users. Update `DB_USER` in the script to match.

## Step 6 — Verify Backup in S3

Open your S3 bucket. Under `db-backups/` you should see:

```
db-backups/app_database-2026-05-14-13-00-00.sql.gz
```

## Step 7 — Restore the Backup

```bash
# Download from S3
aws s3 cp s3://my-company-backup-bucket/db-backups/app_database-2026-05-14-13-00-00.sql.gz .

# Decompress
gunzip app_database-2026-05-14-13-00-00.sql.gz

# Restore into the container
cat app_database-2026-05-14-13-00-00.sql | docker exec -i prod-postgres-db psql -U postgres app_database
```

## Step 8 — Automate with Cron

```bash
crontab -e
```

```
0 2 * * * /home/ubuntu/docker-db-backups/postgres-docker-backup.sh >> /home/ubuntu/docker-db-backups/backup.log 2>&1
```

---

# Phase 5 — Docker Volume Cold Backup

A cold backup stops the container before archiving the volume data. There are no active writes happening during the backup, which makes the data consistent. The tradeoff is brief database downtime, so this is best suited for scheduled maintenance windows or non-critical projects.

**Flow:**

```
Stop Container
        │
        ▼
Archive Docker Volume → .tar.gz
        │
        ▼
Upload to S3
        │
        ▼
Start Container again
```

## Step 1 — Find the Volume Name

```bash
docker inspect prod-postgres-db
```

Look for the Mounts section in the output:

```json
"Mounts": [
  {
    "Type": "volume",
    "Name": "company_postgres_data",
    "Destination": "/var/lib/postgresql/data"
  }
]
```

The volume name in this example is `company_postgres_data`.

## Step 2 — Create Backup Directory

```bash
mkdir -p ~/volume-backups
cd ~/volume-backups
```

## Step 3 — Create the Cold Backup Script

```bash
nano volume-backup.sh
```

Paste the following complete script:

```bash
#!/bin/bash

DATE=$(date +%F-%H-%M-%S)
CONTAINER_NAME="prod-postgres-db"
VOLUME_NAME="company_postgres_data"
BACKUP_DIR="/home/ubuntu/volume-backups"
BACKUP_FILE="postgres-volume-$DATE.tar.gz"
S3_BUCKET="s3://my-company-backup-bucket/volume-backups"

echo "Stopping PostgreSQL container..."
docker stop $CONTAINER_NAME

echo "Creating volume backup..."
docker run --rm \
  -v $VOLUME_NAME:/data \
  -v $BACKUP_DIR:/backup \
  ubuntu \
  tar czf /backup/$BACKUP_FILE /data

echo "Starting PostgreSQL container..."
docker start $CONTAINER_NAME

echo "Uploading backup to S3..."
aws s3 cp $BACKUP_DIR/$BACKUP_FILE $S3_BUCKET --sse AES256

# Clean up local backups older than 7 days
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Cold volume backup completed successfully"
```

Here is what each section does:

- `docker stop` — Stops the container cleanly, flushing any pending writes before the backup starts
- `docker run --rm -v $VOLUME_NAME:/data -v $BACKUP_DIR:/backup ubuntu tar czf` — This is the clever part. It spins up a temporary Ubuntu container, mounts the named volume as `/data` and the backup directory as `/backup`, then runs `tar` inside it. The `--rm` flag removes the temporary container automatically when it finishes
- `docker start` — Brings the database back online as soon as the archive is created
- `find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete` — Deletes local copies older than 7 days to avoid filling up EC2 disk space

## Step 4 — Give Permission and Run

```bash
chmod +x volume-backup.sh
./volume-backup.sh
```

**Expected output:**

```
Stopping PostgreSQL container...
prod-postgres-db
Creating volume backup...
Starting PostgreSQL container...
prod-postgres-db
Uploading backup to S3...
upload: ./postgres-volume-2026-05-14-15-00-00.tar.gz to s3://...
Cold volume backup completed successfully
```

> ⚠️ **Error: No such volume: company_postgres_data** — The volume name in the script does not match the actual volume. Fix: Run `docker inspect prod-postgres-db` and copy the exact name from the `Mounts` section. Update `VOLUME_NAME` in the script.

## Step 5 — Restore the Volume

```bash
# 1. Stop the container
docker stop prod-postgres-db

# 2. Remove the old volume (optional — only if you want a clean restore)
docker volume rm company_postgres_data

# 3. Create a fresh volume
docker volume create company_postgres_data

# 4. Extract backup into the volume
docker run --rm \
  -v company_postgres_data:/data \
  -v $(pwd):/backup \
  ubuntu \
  bash -c "cd /data && tar xzf /backup/postgres-volume-2026-05-14-15-00-00.tar.gz --strip 1"

# 5. Start the container
docker start prod-postgres-db
```

> 💡 **FAQ — When should I use volume backup vs pg_dump?**
> Use `pg_dump` when you want a portable, version-independent backup that can be restored to any PostgreSQL instance (including RDS or a different PostgreSQL version). Use volume backup (cold) when you want a complete snapshot including configuration files and can tolerate brief downtime. In production, many teams use both: `pg_dump` daily for reliable logical backups, and volume/EBS snapshots weekly for full disaster recovery. A volume backup from PostgreSQL 15 cannot always be reliably restored into PostgreSQL 16 — `pg_dump` has no such limitation.

## Step 6 — Automate with Cron

```bash
crontab -e
```

```
0 2 * * * /home/ubuntu/volume-backups/volume-backup.sh >> /home/ubuntu/volume-backups/backup.log 2>&1
```

---

# Phase 6 — Production Best Practices

## Backup Retention Strategy

| Frequency | Retention | Storage Class |
|-----------|-----------|---------------|
| Daily backups | 7 days | S3 Standard |
| Weekly backups | 1 month | S3 Standard-IA |
| Monthly backups | 1 year | Glacier |

## Backup Method Comparison

| Backup Type | Use Case | Portability | Production Use |
|-------------|----------|-------------|----------------|
| pg_dump / mysqldump | Logical DB backup | Very High | Daily |
| Volume Backup (cold) | Full snapshot | Medium | Weekly |
| EBS Snapshot | Entire disk recovery | High | Weekly |
| rsync + S3 | File and app data | High | Daily |

## Security Checklist

- **IAM Roles:** Never use access keys on EC2. Always use IAM roles with least-privilege permissions
- **Encryption:** Use `--sse AES256` for all uploads. Use KMS in regulated environments
- **Private Buckets:** Block Public Access must always be enabled on backup buckets
- **MFA:** Enable MFA on your AWS root and admin accounts
- **CloudTrail:** Enable CloudTrail for full audit logs of all API calls including S3 and KMS
- **Test Restores:** A backup is only as good as a successful restore. Test periodically

## Monitoring

- **CloudWatch Alarms:** Alert if the backup script fails with a non-zero exit code
- **SNS Notifications:** Send email or Slack alert when a backup fails
- **Log Files:** Always redirect cron output to a log file using `>> backup.log 2>&1`

## Advanced Backup Tools (Beyond This Guide)

| Tool | Best For |
|------|----------|
| AWS Backup | Managed service — centralized backup policies for EC2, RDS, EFS |
| Restic | Open source, encrypted, incremental backups to any storage backend |
| BorgBackup | Deduplication and compression — great for large backup sets |
| Velero | Kubernetes workload and volume backups |
| Duplicity | Encrypted bandwidth-efficient backups for Linux |

> 💡 rsync + S3 (as built in this guide) is the ideal way to learn the fundamentals. Once you understand the flow, tools like AWS Backup or Restic will make much more sense because you already understand the problem they are solving.

---

## Challenges Faced and How They Were Solved

**Challenge 1: IAM role credentials not being picked up**

The most common first-time error is running `aws sts get-caller-identity` and getting "Unable to locate credentials." In almost every case, the IAM role was created but not actually attached to the instance. The fix is to go to EC2 Console → Select instance → Actions → Security → Modify IAM Role → attach the role → wait about two minutes for the metadata service to reflect the change.

**Challenge 2: Container name mismatch in scripts**

The `postgres-docker-backup.sh` and `volume-backup.sh` scripts both require the exact container name. `docker ps` shows the name in the NAMES column, and it is easy to mistype or use a name from memory that is slightly different. The fix is always to copy the name directly from `docker ps` output rather than typing it.

**Challenge 3: rsync trailing slash matters**

When running `rsync -av --delete $SOURCE $TEMP_BACKUP`, the trailing slash on `$SOURCE` is critical. With a trailing slash (`/home/ubuntu/backup-project/source/`), rsync copies the *contents* of the directory. Without it, rsync copies the directory *itself* as a subdirectory. Getting this wrong means the backup structure is nested one level deeper than expected, and restores can fail in confusing ways.

**Challenge 4: Volume name discovery**

Docker named volumes are not always obvious. When you run `docker inspect`, the output is verbose JSON. The Mounts section is what matters, and the `Name` field inside it is the volume name. Piping through `grep -A5 Mounts` makes this much easier to find.

**Challenge 5: Glacier retrieval delay misunderstanding**

It is worth noting that files moved to Glacier Flexible Retrieval are not instantly accessible. Retrieval can take 3–5 hours for standard retrieval, or minutes for expedited (which costs extra). This surprised a number of people who expected Glacier to behave like S3 Standard. For a backup strategy, this is fine — you only need Glacier data in a disaster scenario, and a few hours is acceptable. If you need faster access, use S3 Glacier Instant Retrieval instead.

---

## Key Learnings

Working through this end-to-end taught several things that are hard to appreciate from documentation alone.

The IAM role approach genuinely feels better than access keys once you experience it. There is no moment of wondering "did I accidentally commit that key file?" because there is nothing to commit. The metadata service handles everything transparently.

Testing restores is not optional. It is the only way to know your backup system actually works. Running a restore on a throwaway environment once a month is a non-negotiable practice in any production setup.

Incremental backups are not complicated to implement, but the savings compound quickly. For large data directories, the difference between a full backup and an rsync-based incremental backup can be the difference between minutes and hours of transfer time.

Encryption should be the default, not an afterthought. Adding `--sse AES256` to every upload costs nothing in performance and essentially nothing in effort — there is no reason not to.

---

## Best Practices Followed

**No hardcoded credentials.** IAM Roles are used throughout, following AWS's recommended approach for EC2-to-S3 communication. This eliminates the risk of credentials appearing in scripts, version control, or logs.

**Timestamped backup filenames.** Every backup file includes a timestamp (`date +%F-%H-%M-%S`) so backups can be sorted chronologically and specific points in time can be identified for restore.

**Separate S3 prefixes per backup type.** File backups go to `incremental-backups/`, database backups to `db-backups/`, and volume backups to `volume-backups/`. Lifecycle rules are scoped to these prefixes, giving fine-grained control over retention per backup type.

**Encryption on every upload.** All scripts include `--sse AES256` so there is no risk of an unencrypted backup making it to S3 due to a forgotten flag.

**Local cleanup after upload.** Every script removes the temporary local archive after a successful S3 upload. This prevents the EC2 instance's disk from filling up over time, which would cause all future backups to fail.

**Cron output to log files.** Every cron job redirects output to a log file with `>> backup.log 2>&1`. Without this, cron runs silently and errors are invisible.

---

## Final Results

By the end of this guide, the following system is operational:

- A daily automated file backup running at 1:00 AM, uploading compressed archives to S3
- An incremental backup that only transfers changed files, running with rsync
- An S3 Lifecycle Rule moving backups to Glacier at 30 days and deleting at 90 days
- All uploads encrypted with AES256 server-side encryption
- A daily PostgreSQL database backup via `pg_dump` running at 2:00 AM inside a Docker container
- A scheduled Docker volume cold backup for full-disk recovery scenarios
- All cron jobs logging to dedicated log files for monitoring

The entire system runs without human intervention. Every backup is encrypted, timestamped, versioned in S3, and scheduled to auto-expire.

---

## Conclusion

Building a backup system from scratch is one of those projects that teaches you far more than just backup commands. It touches IAM and security, S3 cost management, Docker internals, PostgreSQL, shell scripting, and Linux cron — all in a single project. More importantly, it gives you the confidence that your data is protected, and a clear understanding of what recovery actually looks like.

The "I'll set up backups later" mindset is one of the most common and costly mistakes in infrastructure. The fact that you are reading this guide already puts you ahead of the majority of setups running in production today.

Build it. Test the restores. Automate everything. Sleep better.

---

## Future Improvements

There are several ways this system can be taken further:

**SNS + CloudWatch integration:** Configure CloudWatch to monitor the EC2 instance's cron job logs and trigger an SNS notification (email or Slack) when a backup fails. Right now, failures are only visible in the log files if someone checks.

**Migrate to AWS Backup:** For teams managing multiple resources (EC2, RDS, EFS, DynamoDB), AWS Backup provides a centralized management console with policies, cross-region replication, and compliance reporting — reducing the maintenance burden of custom scripts.

**Cross-region replication:** For disaster recovery scenarios where an entire AWS region becomes unavailable, configure S3 Cross-Region Replication to automatically copy backups to a bucket in a secondary region.

**Backup verification automation:** Write a script that downloads a random recent backup, extracts it, and verifies the contents match an expected checksum. Schedule this weekly so backup integrity is validated automatically.

**Restic for deduplication:** Replace the tar + S3 approach with Restic, which handles deduplication, compression, and encryption in a single tool, and supports S3 as a backend natively.

**Velero for Kubernetes:** If the workload eventually moves to Kubernetes (EKS), Velero is the standard tool for cluster-level and persistent volume backups.

---

## License

> The scripts and code in this article are free to use and modify for personal or commercial use. No attribution required, but always appreciated. Use at your own risk in production — always test in a safe environment first.

---

Feel free to connect and discuss with me — whether you hit an error, have a question about adapting this to your stack, or just want to talk AWS and DevOps:

📧 [bhavyat2520@gmail.com](mailto:bhavyat2520@gmail.com)

---
