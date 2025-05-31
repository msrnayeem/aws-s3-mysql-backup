````markdown
# AWS S3 Backup Setup Documentation

## Overview
This document describes how to set up automated daily backups of a MySQL database to an AWS S3 bucket from an Ubuntu EC2 instance.

---

## Prerequisites
- An AWS account with an S3 bucket created (e.g., bucket name: `hrsoft`).
- An EC2 instance running Ubuntu with IAM role attached for S3 access.
- AWS CLI installed and configured with proper permissions.
- MySQL database running on the EC2 instance or accessible from it.

---

## Step 1: Create S3 Bucket
- Create an S3 bucket in your preferred AWS Region (e.g., Asia Pacific (Sydney) `ap-southeast-2`).
- Use a unique bucket name (e.g., `hrsoft`).
- Enable **Block all public access** to keep the bucket secure.
- Enable bucket versioning (optional but recommended).

---

## Step 2: Create IAM Role and Policy for EC2

- Create an IAM Role (e.g., `EC2S3BackupUploadRole`) with trusted entity type **EC2**.
- Attach a custom policy (`S3BackupUploadPolicy`) to allow these actions on your bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<s3-bucket>",
                "arn:aws:s3:::<s3-bucket>/*"
            ]
        }
    ]
}
````

* Attach this role to your EC2 instance.

---

## Step 3: Install and Configure AWS CLI on EC2

* Install AWS CLI (v2 recommended).
* Verify installation by running:

```bash
aws --version
```

* Since the EC2 instance uses an IAM role with S3 permissions, no manual credential configuration is needed.

Try uploading a test file to confirm everything works:

```bash
echo "test file content" > testfile.txt
aws s3 cp testfile.txt s3://<s3-bucket>/testfile.txt
```

Check if the file uploaded successfully:

```bash
aws s3 ls s3://<s3-bucket>/
```

You should see something like:

```
2025-05-31  XX:XX:XX      <size> testfile.txt
```

---

## Step 4: Create Backup Script

Create a shell script file (e.g., `/home/ubuntu/backup_to_s3.sh`) with the following content:

```bash
#!/bin/bash

# Variables
DATE=$(date +'%Y-%m-%d-%H%M')
BACKUP_DIR="/home/ubuntu/db_backups"
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.sql"
BUCKET_NAME="<s3-bucket>"

# MySQL credentials
DB_USER="your_db_username"
DB_PASS="your_db_password"
DB_NAME="your_database_name"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Create database backup
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME > $BACKUP_FILE

# Upload backup to S3 bucket
aws s3 cp $BACKUP_FILE s3://$BUCKET_NAME/

# Remove local backup file to save space
rm $BACKUP_FILE
```

Make the script executable:

```bash
chmod +x /home/ubuntu/backup_to_s3.sh
```

---

## Step 5: Schedule Daily Backup Using Cron

Edit the crontab for the `ubuntu` user:

```bash
crontab -e
```

Choose the nano editor (option 1) if prompted.

Add the following line to run the backup every day at 2:00 AM:

```cron
0 2 * * * /home/ubuntu/backup_to_s3.sh >> /home/ubuntu/backup.log 2>&1
```

Save and exit the editor.

---

## Step 6: Verify Setup

* Wait for the cron job to trigger or run the backup script manually:

```bash
/home/ubuntu/backup_to_s3.sh
```

* Check the backup files in your S3 bucket:

```bash
aws s3 ls s3://<s3-bucket>/
```

* Review the log file for any errors or status messages:

```bash
cat /home/ubuntu/backup.log
```

---

## Notes

* Ensure your database credentials in the script are correct and kept secure.
* Customize the backup schedule and retention policy as needed.
* Monitor your S3 bucket storage usage and costs regularly.

---

## References

* [AWS S3 Documentation](https://docs.aws.amazon.com/s3/index.html)
* [AWS IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
* [AWS CLI User Guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
* [mysqldump Documentation](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)

---

*End of Document*

