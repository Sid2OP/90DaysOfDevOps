# Day 08 – Cloud Server Setup:  Nginx & Web Deployment

---

## Objective

Deploy a real web server on the cloud and understand practical server management tasks used in production DevOps environments.

---

## Part 1: Launch Cloud Instance & SSH Access

### Step 1: Create a Cloud Instance

- Launched an EC2 instance using **Amazon Linux / Ubuntu** (Free Tier).
- Selected a public subnet and enabled auto-assign public IP.
- Created a key pair for SSH access.

### Step 2: Connect via SSH

```bash
ssh -i your-key.pem ubuntu@<your-instance-ip>

```

**Why:** Securely connects to the remote cloud server for management.


---

---

### Step 3: Install Nginx

```bash
sudo apt install nginx -y

```

Start and enable Nginx:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx

```

Verify Nginx status:

```bash
systemctl status nginx

```

**Why:** Confirms that the web server is running.

---

## Part 3: Security Group Configuration

- Allowed inbound traffic:
    - **Port 22 (SSH)** – My IP
    - **Port 80 (HTTP)** – Anywhere (0.0.0.0/0)

### Test Web Access

Open browser:

```
http://44.246.130.20

```

✅ Nginx welcome page displayed successfully.

📸 Screenshot: `nginx-webpage.png`
<img width="1918" height="1027" alt="Nginx" src="https://github.com/user-attachments/assets/7863d868-7568-4d64-b3cd-87cb2a508e90" />


---

## Part 4: Extract Nginx Logs

### Step 1: View Nginx Logs

```bash
sudo tail -n 50 /var/log/nginx/access.log
sudo tail -n 50 /var/log/nginx/error.log

```

**Why:** Access and error logs help debug web traffic and issues.

---

### Step 2: Save Logs to File

```bash
sudo cat /var/log/nginx/access.log > ~/nginx-logs.txt

```

Verify:

```bash
cat ~/nginx-logs.txt

```
<img width="1918" height="223" alt="Nginx_logs" src="https://github.com/user-attachments/assets/4af08eb4-9eee-468d-9df7-0c9411c94101" />

---

### Step 3: Download Log File to Local Machine

**AWS:**

```bash
scp -i your-key.pem ubuntu@<your-instance-ip>:~/nginx-logs.txt .

```

📸 Screenshot: `nginx.png`

<img width="1887" height="237" alt="Logs_txt" src="https://github.com/user-attachments/assets/da4d16b5-cd16-431d-bbc8-2d172320e96e" />


---

## Commands Used

- ssh
- apt update / upgrade
- apt install nginx docker.io
- systemctl start / enable / status
- tail, cat
- scp

---

## Challenges Faced

- Nginx page not loading initially due to missing port 80 rule.
- Resolved by updating the security group to allow HTTP traffic.

---

## What I Learned

- How to launch and access a cloud server
- Installing and managing services using systemctl
- Configuring security groups for web access
- Viewing and extracting service logs
- Verifying deployments from a public browser
