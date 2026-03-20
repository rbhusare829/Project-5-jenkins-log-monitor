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
├── screenshots/              # All proof screenshots
├── monitor_log.sh            # Shell script to monitor log size
├── Jenkinsfile               # Jenkins job script
└── README.md                 # This file
```

---

## ⚙️ Step-by-Step Implementation

### Step 1: Launch EC2 Instance
- Instance Name: `JENKINS`
- OS: Ubuntu 22.04 LTS | Type: t2.micro
- Region: ap-south-1 (Mumbai) | IP: 13.126.61.225
- Ports Open: 22 (SSH), 8080 (Jenkins)

### 📸 Screenshot 1 - EC2 Instance Running
![EC2 Instance](screenshots/Screenshot_2026-03-20_162232.png)

---

### Step 2: Install Java & Jenkins

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [trusted=yes] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update && sudo apt install jenkins -y
sudo systemctl start jenkins
```

---

### Step 3: Install & Configure AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws configure   # region: ap-south-1

# Copy credentials to Jenkins user
sudo mkdir -p /var/lib/jenkins/.aws
sudo cp ~/.aws/credentials /var/lib/jenkins/.aws/
sudo cp ~/.aws/config /var/lib/jenkins/.aws/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.aws
```

### 📸 Screenshot 2 - AWS Configure & Script Execution
![AWS and Script](screenshots/Screenshot_2026-03-20_164435.png)

---

### Step 4: Create S3 Bucket

```bash
aws s3 mb s3://devops-log-backup-rohit --region ap-south-1
aws s3 ls   # Verify bucket exists
```

---

### Step 5: Jenkins Job - Parameter Setup

- Job Name: `upload-log-to-s3` | Type: Freestyle
- Parameter: `LOG_FILE_PATH` = `/var/log/httpd/access.log`

### 📸 Screenshot 3 - Jenkins Job Parameter Configuration
![Job Parameter](screenshots/Screenshot_2026-03-20_154803.png)

---

### Step 6: Jenkins Job Created

### 📸 Screenshot 4 - Jenkins Job Status Page
![Job Status](screenshots/Screenshot_2026-03-20_155405.png)

---

### Step 7: Jenkins API Token

### 📸 Screenshot 5 - API Token Generated (monitor-token)
![API Token](screenshots/Screenshot_2026-03-20_160840.png)

---

### Step 8: Cron Job Setup

```bash
crontab -e
# Added: */5 * * * * /opt/scripts/monitor_log.sh
crontab -l   # Verify
```

### 📸 Screenshot 6 - Crontab Nano Editor
![Crontab Editor](screenshots/Screenshot_2026-03-20_161954.png)

---

### Step 9: Monitoring Script (`monitor_log.sh`)

```bash
#!/bin/bash
LOG_FILE="/var/log/httpd/access.log"
SIZE_LIMIT=$((1 * 1024 * 1024 * 1024))
JENKINS_URL="http://localhost:8080"
JENKINS_JOB="upload-log-to-s3"
JENKINS_USER="rohit"
JENKINS_TOKEN="<api_token>"
SCRIPT_LOG="/var/log/monitor_log.log"

log_msg() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$SCRIPT_LOG"
}

FILE_SIZE=$(stat -c%s "$LOG_FILE")
log_msg "Current size: $FILE_SIZE bytes"

if [ "$FILE_SIZE" -ge "$SIZE_LIMIT" ]; then
    log_msg "ALERT: File exceeds 1GB! Triggering Jenkins..."
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
        -X POST --user "$JENKINS_USER:$JENKINS_TOKEN" \
        "$JENKINS_URL/job/$JENKINS_JOB/build")
    log_msg "Jenkins response: $HTTP_CODE"
else
    log_msg "INFO: File size OK. No action needed."
fi
```

### 📸 Screenshot 7 - Crontab -l & Script Running
![Script Running](screenshots/Screenshot_2026-03-20_164312.png)

---

## ✅ Results & Proof

### 📸 Screenshot 8 - Jenkins Build #2 Console - Upload SUCCESS
![Jenkins Console Success](screenshots/Screenshot_2026-03-20_162520.png)

---

### 📸 Screenshot 9 - S3 Bucket - File Uploaded (1.1 GB)
![S3 Upload Proof](screenshots/Screenshot_2026-03-20_162654.png)

---

### 📸 Screenshot 10 - Jenkins Job Final Status (Build #2 ✅)
![Jenkins Final Status](screenshots/Screenshot_2026-03-20_162729.png)

---

## 📋 Deliverables Summary

| # | Deliverable | Status |
|---|-------------|--------|
| 1 | `monitor_log.sh` - Shell script | ✅ |
| 2 | Jenkins Freestyle Job | ✅ |
| 3 | S3 Upload Proof (1.1 GB) | ✅ |
| 4 | Log File Cleared | ✅ |
| 5 | Cron Job `*/5 * * * *` | ✅ |
| 6 | Jenkins API Token | ✅ |
| 7 | AWS CLI v2 Configured | ✅ |
| 8 | README.md Documentation | ✅ |

---

*✅ Project 5 Completed Successfully — March 20, 2026 — Rohit Bhusare* 🚀
