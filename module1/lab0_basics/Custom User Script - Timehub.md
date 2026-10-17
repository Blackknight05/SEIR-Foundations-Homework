# TimeHub — AWS EC2 Deployment Guide

This guide shows how to launch an EC2 Linux instance, use EC2 User Data to install Apache, SSH into the instance, and finish publishing the TimeHub website.

TimeHub includes a live Eastern Time clock, a time zone converter, and a calculator.

## 1. Prerequisites

You need:
- An AWS account
- An EC2 instance running Amazon Linux 2023 (recommended for this guide)
- A key pair if connecting over SSH
- The instance's public IPv4 address if you want to access it directly from the internet

> **Important:** User Data normally runs automatically during the instance's first boot. If you change User Data after the instance has already launched, it generally will not rerun automatically on that existing boot. The simplest approach for a fresh setup is to paste the script before launching a new instance.

## 2. Add the EC2 User Data script

When launching the instance:

1. Open **AWS Console → EC2 → Launch instance**.
2. Choose an Amazon Linux 2023 AMI.
3. Configure the instance and key pair as needed.
4. Expand **Advanced details**.
5. Find **User data**.
6. Paste the script below.
7. Launch the instance and wait for it to finish initializing.

### User Data script

This script installs Apache (`httpd`), enables it to start at boot, starts it, and writes setup progress to a log. It intentionally does not create the final TimeHub page; you will SSH into the instance and complete the remaining steps in the next sections.

```bash
#!/bin/bash
set -euxo pipefail

exec > >(tee -a /var/log/timehub-user-data.log | logger -t timehub-user-data -s 2>/dev/console) 2>&1

echo "Starting TimeHub web server setup..."

# Install Apache on Amazon Linux 2023
dnf install -y httpd

# Make Apache start automatically and start it now
systemctl enable httpd
systemctl start httpd

# Ensure the default Apache web directory exists
mkdir -p /var/www/html
chmod 755 /var/www
chmod 755 /var/www/html

echo "Apache setup complete."
systemctl --no-pager status httpd
```

## 3. SSH into the EC2 instance

After the instance is running, copy its public IPv4 address from the EC2 console.

From your local terminal, run the following commands. Replace the key path and IP address with your own:

```bash
chmod 400 /path/to/your-key.pem
ssh -i /path/to/your-key.pem ec2-user@YOUR-EC2-PUBLIC-IP
```

If you use the AWS browser-based connection instead, select the instance and click **Connect**.

Once connected, you should see a prompt similar to:

```text
[ec2-user@ip-172-31-7-176 ~]$
```

The following commands are run **inside the EC2 SSH session**, not on your local computer.

## 4. Check that Apache is running

Run:

```bash
sudo systemctl status httpd
```

Look for:

```text
Active: active (running)
```

Then test Apache locally:

```bash
curl -I http://localhost/
```

If Apache is running but the response is `403 Forbidden`, continue to the next step. This can happen when the web directory does not contain an index page.

If the service is missing, check your operating system and install Apache:

```bash
cat /etc/os-release
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

## 5. Check the website directory

Run these commands inside SSH:

```bash
sudo ls -la /var/www/html/
sudo ls -la /var/www/
```

The main website page should be located at:

```text
/var/www/html/index.html
```

If `/var/www/html/` is empty, Apache has no homepage to serve. Do not try to execute the file path as a command. For example, typing `/var/www/html/index.html` directly will produce “No such file or directory” if the file does not exist.

## 6. Create the TimeHub homepage

Open a new file:

```bash
sudo nano /var/www/html/index.html
```

Paste the **complete TimeHub HTML** into the editor. The page should contain the HTML, CSS, and JavaScript for the clock, time zone converter, and calculator.

Then save and exit:
1. Press `Ctrl + O`.
2. Press `Enter` to confirm the filename.
3. Press `Ctrl + X` to exit.

Set the permissions:

```bash
sudo chmod 755 /var/www
sudo chmod 755 /var/www/html
sudo chmod 644 /var/www/html/index.html
```

Restart Apache:

```bash
sudo systemctl restart httpd
```

> If you need the complete TimeHub HTML, use the full HTML page supplied separately in your setup conversation. The User Data script above only installs and starts Apache; it does not include the full website source.

## 7. Test the website from inside EC2

Run:

```bash
curl -I http://localhost/
```

A successful response should start with:

```text
HTTP/1.1 200 OK
```

To inspect the beginning of the page:

```bash
curl -s http://localhost/ | head -n 15
```

If the response is still `403 Forbidden`, verify that `index.html` exists and that the file permissions are correct:

```bash
sudo ls -l /var/www/html/index.html
sudo tail -n 100 /var/log/httpd/error_log
```

## 8. Open TimeHub in your browser

In your browser, visit:

```text
http://YOUR-EC2-PUBLIC-IP/
```

Replace `YOUR-EC2-PUBLIC-IP` with the instance's public IPv4 address.

If `curl` works locally but the public page does not load, check:
- The security group allows inbound TCP port 80.
- The instance has a public IPv4 address.
- The subnet route table and network ACLs permit the required traffic.
- You are using `http://` unless HTTPS has been configured.

## 9. Useful commands

| Task | Command |
|---|---|
| Check Apache | `sudo systemctl status httpd` |
| Start Apache | `sudo systemctl start httpd` |
| Restart Apache | `sudo systemctl restart httpd` |
| Enable Apache at boot | `sudo systemctl enable httpd` |
| Test local HTTP response | `curl -I http://localhost/` |
| List site files | `sudo ls -la /var/www/html/` |
| View Apache errors | `sudo tail -n 100 /var/log/httpd/error_log` |
| View User Data log | `sudo cat /var/log/timehub-user-data.log` |

## Deployment checklist

- [ ] User Data script was added before launching the EC2 instance.
- [ ] Apache is active and running.
- [ ] SSH connection to the instance works.
- [ ] `/var/www/html/index.html` contains the complete TimeHub website.
- [ ] `curl -I http://localhost/` returns `200 OK`.
- [ ] The security group permits the intended inbound HTTP traffic.
- [ ] TimeHub loads at `http://YOUR-EC2-PUBLIC-IP/`.

## Security note

This guide uses plain HTTP, which does not encrypt traffic. For a production website, configure a domain name and HTTPS with a valid TLS certificate. Keep SSH restricted to trusted IP addresses.
