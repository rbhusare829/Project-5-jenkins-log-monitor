# Project 5 - Automated Jenkins Job Triggered by Access Log Size

## 👤 Student: Rohit Bhusare

---

## 📌 Objective

Create a system where a Jenkins job is triggered **automatically** when an access log file exceeds **1GB** in size. Once triggered, the Jenkins job:
1. Transfers the log file to a specific **Amazon S3 bucket**
2. Clears the contents of the original access log file after successful transfer

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| AWS EC2 (Ubuntu 22.04) | Jenkins Server |
| Jenkins 2.541.3 | CI/CD Automation |
| AWS CLI v2 | S3 Upload |
| Amazon S3 | Log File Storage |
| Bash Script | Log Size Monitoring |
| Cron | Scheduling (every 5 min) |

---

## 📁 Project Structure

```
project-5/
├── monitor_log.sh        # Shell script to monitor log size & trigger Jenkins
├── Jenkinsfile           # Jenkins pipeline/freestyle job script
└── README.md             # Project documentation
```

---

## ⚙️ Step-by-Step Implementation

### Step 1: Launch EC2 Instance
- Launched **Ubuntu 22.04 LTS** on **t2.micro** (Free Tier)
- Instance Name: `JENKINS`
- Region: `ap-south-1` (Mumbai)
- Public IP: `13.126.61.225`
- Security Group: Port **8080** open for Jenkins, Port **22** for SSH

---

### Step 2: Install Java (Jenkins Prerequisite)
```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
# Output: openjdk version "17.0.18" 2026-01-20
```

---

### Step 3: Install Jenkins
```bash
# Add Jenkins GPG Key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins Repository
echo "deb [trusted=yes] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
# Output: active (running)
```

Jenkins accessible at: `http://13.126.61.225:8080`

---

### Step 4: Install AWS CLI v2
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y && unzip awscliv2.zip
sudo ./aws/install
aws --version
# Output: aws-cli/2.34.13 Python/3.14.3 Linux/6.17.0-1007-aws
```

---

### Step 5: Configure AWS Credentials
```bash
aws configure
# AWS Access Key ID: ********************
# AWS Secret Access Key: ********************
# Default region name: ap-south-1
# Default output format: json

# Copy credentials for Jenkins user
sudo mkdir -p /var/lib/jenkins/.aws
sudo cp ~/.aws/credentials /var/lib/jenkins/.aws/credentials
sudo cp ~/.aws/config /var/lib/jenkins/.aws/config
sudo chown -R jenkins:jenkins /var/lib/jenkins/.aws
```

---

### Step 6: Create S3 Bucket
```bash
aws s3 mb s3://devops-log-backup-rohit --region ap-south-1
aws s3 ls
# Output: 2026-03-16 08:13:19 devops-log-backup-rohit
```

**S3 Bucket Name:** `devops-log-backup-rohit`

---

### Step 7: Create Jenkins Job

- Job Name: `upload-log-to-s3`
- Type: **Freestyle Project**
- Parameter: `LOG_FILE_PATH` (String) = `/var/log/httpd/access.log`
- Build Step: **Execute Shell**

```bash
#!/bin/bash
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
S3_BUCKET="devops-log-backup-rohit"
AWS_REGION="ap-south-1"
LOG_FILE="${LOG_FILE_PATH}"

echo "Uploading $LOG_FILE to S3..."
aws s3 cp "$LOG_FILE" "s3://$S3_BUCKET/access-logs/access_log_${TIMESTAMP}.log" --region $AWS_REGION

if [ $? -eq 0 ]; then
    echo "Upload successful!"
    sudo truncate -s 0 "$LOG_FILE"
    echo "Log file cleared!"
else
    echo "Upload FAILED!"
    exit 1
fi
```

---

### Step 8: Create Monitoring Script

```bash
sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/monitor_log.sh
sudo chmod +x /opt/scripts/monitor_log.sh
```

**Script Content (`monitor_log.sh`):**

```bash
#!/bin/bash
LOG_FILE="/var/log/httpd/access.log"
SIZE_LIMIT=$((1 * 1024 * 1024 * 1024))
JENKINS_URL="http://localhost:8080"
JENKINS_JOB="upload-log-to-s3"
JENKINS_USER="rohit"
JENKINS_TOKEN="1127f552c6fea6bfe5da3eb948b6f53a71"
SCRIPT_LOG="/var/log/monitor_log.log"

log_msg() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$SCRIPT_LOG"
}

FILE_SIZE=$(stat -c%s "$LOG_FILE")
log_msg "Current size: $FILE_SIZE bytes"

if [ "$FILE_SIZE" -ge "$SIZE_LIMIT" ]; then
    log_msg "ALERT: File exceeds 1GB! Triggering Jenkins..."
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
        -X POST \
        --user "$JENKINS_USER:$JENKINS_TOKEN" \
        "$JENKINS_URL/job/$JENKINS_JOB/build")
    log_msg "Jenkins response: $HTTP_CODE"
else
    log_msg "INFO: File size OK. No action needed."
fi
```

---

### Step 9: Create Test Log File (1.1 GB)
```bash
sudo mkdir -p /var/log/httpd
sudo dd if=/dev/zero of=/var/log/httpd/access.log bs=1M count=1100
# Output: 1153433600 bytes (1.2 GB) copied
```

---

### Step 10: Test the Script
```bash
sudo /opt/scripts/monitor_log.sh
```

**Output:**
```
[2026-03-20 07:12:57] Current size: 1153433600 bytes
[2026-03-20 07:12:57] ALERT: File exceeds 1GB! Triggering Jenkins...
[2026-03-20 07:12:58] Jenkins response: 201
```

---

### Step 11: Setup Cron Job (Every 5 Minutes)
```bash
crontab -e
```

Added line:
```
*/5 * * * * /opt/scripts/monitor_log.sh
```

Verified:
```bash
crontab -l
# Output: */5 * * * * /opt/scripts/monitor_log.sh
```

---

## ✅ Results / Proof

### Jenkins Build #2 - SUCCESS
- Upload: `access.log` → `s3://devops-log-backup-rohit/access-logs/access_log_20260320_071712.log`
- Log File Cleared: ✅
- Status: **Finished: SUCCESS** ✅

### S3 Bucket Verification
- File: `access_log_20260320_071712.log`
- Size: **1.1 GB**
- Date: March 20, 2026

---

## 📋 Deliverables Summary

| # | Deliverable | Status |
|---|-------------|--------|
| 1 | Shell script (`monitor_log.sh`) | ✅ Done |
| 2 | Jenkins Job (Freestyle) | ✅ Done |
| 3 | S3 Upload Proof (Screenshot) | ✅ Done |
| 4 | Log File Cleared Proof | ✅ Done |
| 5 | Cron Job Setup | ✅ Done |
| 6 | Documentation (README.md) | ✅ Done |

---

## 📸 Screenshots

1. EC2 Instance Running
2. Jenkins Dashboard
3. Jenkins Job Configuration
4. Jenkins Build #2 - Console Output (SUCCESS)
5. S3 Bucket - File Uploaded (1.1 GB)
6. Crontab Setup

---

*Project 5 Completed Successfully on March 20, 2026*
