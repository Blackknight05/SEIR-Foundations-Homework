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

Paste the entire HTML code below into the editor. It includes the HTML, CSS, and JavaScript for the clock, time zone converter, and calculator.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>TimeHub | Clock, Time Zones & Calculator</title>
  <style>
    * { box-sizing: border-box; }
    body {
      margin: 0;
      font-family: Arial, sans-serif;
      background: #f4f7fb;
      color: #172033;
    }
    header {
      background: #243b67;
      color: white;
      padding: 24px;
      text-align: center;
    }
    header h1 { margin: 0 0 8px; }
    main {
      max-width: 900px;
      margin: 28px auto;
      padding: 0 16px;
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
      gap: 20px;
    }
    section {
      background: white;
      border-radius: 14px;
      padding: 24px;
      box-shadow: 0 4px 16px #17203312;
    }
    h2 { margin-top: 0; color: #243b67; }
    #clock {
      font-size: 2rem;
      font-weight: bold;
      margin: 18px 0 8px;
      overflow-wrap: anywhere;
    }
    input, select, button {
      width: 100%;
      padding: 11px;
      margin: 7px 0;
      border: 1px solid #ccd4e0;
      border-radius: 8px;
      font-size: 1rem;
    }
    button {
      background: #3159a6;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover { background: #243b67; }
    .result { font-weight: bold; overflow-wrap: anywhere; }
    footer { text-align: center; padding: 24px; color: #657087; }
  </style>
</head>
<body>
  <header>
    <h1>TimeHub</h1>
    <p>Your simple hub for time, time zones, and calculations.</p>
  </header>

  <main>
    <section>
      <h2>Live Eastern Time</h2>
      <div id="clock">Loading...</div>
      <div id="today"></div>
      <p>Timezone: America/New_York</p>
    </section>

    <section>
      <h2>Time Zone Converter</h2>
      <label for="datetime">Date and time</label>
      <input id="datetime" type="datetime-local">
      <label for="fromZone">Convert from</label>
      <select id="fromZone"></select>
      <label for="toZone">Convert to</label>
      <select id="toZone"></select>
      <button id="convertBtn">Convert Time</button>
      <p class="result" id="conversion">Choose your time and time zones.</p>
    </section>

    <section>
      <h2>Quick Calculator</h2>
      <input id="num1" type="number" placeholder="First number">
      <select id="operator">
        <option value="+">Add (+)</option>
        <option value="-">Subtract (−)</option>
        <option value="*">Multiply (×)</option>
        <option value="/">Divide (÷)</option>
        <option value="%">Remainder (%)</option>
      </select>
      <input id="num2" type="number" placeholder="Second number">
      <button id="calculateBtn">Calculate</button>
      <p class="result" id="answer">Your result will appear here.</p>
    </section>
  </main>

  <footer>TimeHub — useful tools, all in one place.</footer>

  <script>
    const zones = [
      ["New York / Eastern", "America/New_York"],
      ["Chicago / Central", "America/Chicago"],
      ["Denver / Mountain", "America/Denver"],
      ["Los Angeles / Pacific", "America/Los_Angeles"],
      ["Toronto", "America/Toronto"],
      ["Vancouver", "America/Vancouver"],
      ["London", "Europe/London"],
      ["Paris", "Europe/Paris"],
      ["Dubai", "Asia/Dubai"],
      ["Karachi", "Asia/Karachi"],
      ["Kolkata", "Asia/Kolkata"],
      ["Singapore", "Asia/Singapore"],
      ["Tokyo", "Asia/Tokyo"],
      ["Sydney", "Australia/Sydney"],
      ["UTC", "UTC"]
    ];

    const clockFormatter = new Intl.DateTimeFormat("en-US", {
      timeZone: "America/New_York",
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
      hour12: true
    });

    function updateClock() {
      const now = new Date();
      document.getElementById("clock").textContent = clockFormatter.format(now);
      document.getElementById("today").textContent =
        new Intl.DateTimeFormat("en-US", {
          timeZone: "America/New_York",
          weekday: "long",
          year: "numeric",
          month: "long",
          day: "numeric"
        }).format(now);
    }

    setInterval(updateClock, 1000);
    updateClock();

    const fromZone = document.getElementById("fromZone");
    const toZone = document.getElementById("toZone");

    zones.forEach(([label, value]) => {
      fromZone.add(new Option(label, value));
      toZone.add(new Option(label, value));
    });

    fromZone.value = "America/New_York";
    toZone.value = "Europe/London";

    const dateInput = document.getElementById("datetime");
    const pad = n => String(n).padStart(2, "0");
    const now = new Date();

    dateInput.value =
      `${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())}T${pad(now.getHours())}:${pad(now.getMinutes())}`;

    function zonedParts(date, zone) {
      const parts = new Intl.DateTimeFormat("en-CA", {
        timeZone: zone,
        year: "numeric",
        month: "2-digit",
        day: "2-digit",
        hour: "2-digit",
        minute: "2-digit",
        second: "2-digit",
        hourCycle: "h23"
      }).formatToParts(date);

      return Object.fromEntries(
        parts.filter(p => p.type !== "literal")
          .map(p => [p.type, Number(p.value)])
      );
    }

    function localTimeToDate(value, zone) {
      const [datePart, timePart] = value.split("T");
      if (!datePart || !timePart) {
        throw new Error("Please select a date and time.");
      }

      const [year, month, day] = datePart.split("-").map(Number);
      const [hour, minute] = timePart.split(":").map(Number);
      const desired = Date.UTC(year, month - 1, day, hour, minute);
      let guess = desired;

      for (let i = 0; i < 4; i++) {
        const p = zonedParts(new Date(guess), zone);
        const represented = Date.UTC(
          p.year, p.month - 1, p.day, p.hour, p.minute, p.second
        );
        guess += desired - represented;
      }

      return new Date(guess);
    }

    document.getElementById("convertBtn").addEventListener("click", () => {
      try {
        const instant = localTimeToDate(dateInput.value, fromZone.value);
        const formatted = new Intl.DateTimeFormat("en-US", {
          timeZone: toZone.value,
          dateStyle: "full",
          timeStyle: "long"
        }).format(instant);

        document.getElementById("conversion").textContent =
          `${formatted} (${toZone.value})`;
      } catch (error) {
        document.getElementById("conversion").textContent = error.message;
      }
    });

    document.getElementById("calculateBtn").addEventListener("click", () => {
      const first = document.getElementById("num1").value;
      const second = document.getElementById("num2").value;
      const a = Number(first);
      const b = Number(second);
      const op = document.getElementById("operator").value;

      if (first === "" || second === "") {
        document.getElementById("answer").textContent = "Enter both numbers.";
        return;
      }

      if ((op === "/" || op === "%") && b === 0) {
        document.getElementById("answer").textContent =
          "Cannot divide by zero.";
        return;
      }

      let result;
      switch (op) {
        case "+": result = a + b; break;
        case "-": result = a - b; break;
        case "*": result = a * b; break;
        case "/": result = a / b; break;
        case "%": result = a % b; break;
      }

      document.getElementById("answer").textContent = `Result: ${result}`;
    });
  </script>
</body>
</html>
```

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

> The User Data script installs and starts Apache; the full website HTML is added here after you SSH into the instance.

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
