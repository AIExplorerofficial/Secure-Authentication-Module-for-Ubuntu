# Secure 2FA Authentication Module for Ubuntu

Welcome to the Secure 2FA (Two-Factor Authentication) Module! This project allows you to add an extra layer of security to your Ubuntu system. By installing this module, you can require a 6-digit code from an authenticator app (like Google Authenticator or Authy) in addition to your regular password when logging into your computer or server (e.g., via SSH or local login).

If you are new to Linux, don't worry! This guide is written specifically for you and will walk you through the entire process step-by-step.

## Prerequisites

Before installing the module, your Ubuntu system needs some basic tools to build the software from its source code.

Open your terminal and run the following commands to install the required packages. To display the QR codes in your terminal during setup, you will also need the Python `qrcode` library:

```bash
sudo apt update
sudo apt install build-essential libpam0g-dev libssl-dev python3 python3-qrcode
```

---

## Step 1: Build the Module

First, you need to compile the code into a module that Ubuntu can understand.

1. Open your terminal and navigate to this project's folder (where this README is located).
2. Run the `make` command:

```bash
make
```

If successful, you will see a message saying `Built pam_auth.so`.

---

## Step 2: Install the Module

Now, install the compiled module into your system.

Run the following command (you will need to enter your Ubuntu password):

```bash
sudo make install
```

**WARNING - CRITICAL UBUNTU SPECIFIC STEP:**
The default `make install` command copies the module to `/lib/security/`. However, on modern 64-bit Ubuntu systems (like 20.04 and 22.04), PAM modules actually live in a different directory. If you skip this fix, **every login may fail**!

Verify the correct path on your system by checking where the standard `pam_unix.so` module resides:

```bash
ls /lib/x86_64-linux-gnu/security/pam_unix.so
```

If the file exists there, you **must** copy the installed 2FA module into this correct directory:

```bash
sudo cp /lib/security/pam_auth.so /lib/x86_64-linux-gnu/security/
```

---

## Step 3: Enroll Users (Generate QR Codes)

**CAUTION - ENROLL ALL USERS FIRST!**
Before going to Step 4, ensure that **every user account** that requires system access is enrolled. The module blocks any user who isn't enrolled. If you enable the module globally without setting up 2FA for other admin or active users, they **will be locked out**.

To set up 2FA for a user account, use the provided setup tool:

```bash
sudo python3 tools/setup_user.py <your_username>
```

*(Replace `<your_username>` with your actual Ubuntu username).*

**What happens next?**

1. A large QR Code will appear on your terminal screen.
2. Open your authenticator app (like Google Authenticator or Authy) on your phone.
3. Add a new account by scanning the QR code on your screen.
4. The script will ask you to type in the 6-digit code currently showing on your phone to verify everything is working correctly.

Note: The secret key is securely saved in `/etc/auth_module/<your_username>.secret`.

---

## Step 4: Enable 2FA on Your System

Now that you have enrolled your account, you can enable 2FA for different services.

### Option A: Enable 2FA for SSH (Remote Login)

There are two files you must edit to enable 2FA for remote logins over SSH. Never close your active root session while making these changes. Keep a redundant SSH session open!

**1. Update the SSH Daemon Configuration:**

You must tell the SSH daemon to ask the user interactive questions. Open the SSH config file:

```bash
sudo nano /etc/ssh/sshd_config
```

Find these settings and ensure they are uncommented and set to `yes`:

```text
UsePAM yes
KbdInteractiveAuthentication yes
```

*(Note: On Ubuntu 20.04 and older, look for `ChallengeResponseAuthentication yes` instead).*

**2. Update the PAM Configuration for SSH:**

Now tell PAM to trigger the 2FA module.

```bash
sudo nano /etc/pam.d/sshd
```

Scroll through the file and look for the specific line `@include common-auth` (this line checks your standard password). **Directly below that line**, add the following text:

```text
auth required pam_auth.so
```

Save the file and exit (`Ctrl+O`, `Enter`, then `Ctrl+X` in nano).

**3. Restart SSH:**

Apply the changes by restarting the SSH service:

```bash
sudo systemctl restart ssh
```

### Option B: Enable 2FA for the `sudo` Command

If you want to require 2FA every time you use the `sudo` command:

1. Open the sudo authentication settings file:

```bash
sudo nano /etc/pam.d/sudo
```

2. Find the line `@include common-auth`.
3. Add `auth required pam_auth.so` right below it.
4. Save and close the file. The next time you run a `sudo` command, it will ask for your password and your 6-digit authenticator code.

---

## Security Features

### Rate Limiting and Lockouts

The module has built-in protection against brute-force attacks. **If you enter an incorrect TOTP code 5 times, your account will be locked out for 15 minutes.** Wait out the duration; you cannot bypass this manually.

### Tamper-Evident Audit Log

This tool includes a comprehensive security audit log located at `/var/log/auth_module.log`. Every authentication attempt (successful or failed) is recorded here.

You can proactively check that nobody has secretly modified or erased traces in this log by running:

```bash
make verify-log
```

If the log is intact, it will confirm successfully.

---

## How to Disable 2FA

If you ever want to turn off the 2FA requirement, simply go back to the files you edited (like `/etc/pam.d/sshd` or `/etc/pam.d/sudo`) and remove the line:

`auth required pam_auth.so`

Save the file, and your system will go back to its normal password-only login.

---

## Troubleshooting

- **No TOTP prompt over SSH!**
  Ensure you explicitly updated `/etc/ssh/sshd_config` with `KbdInteractiveAuthentication yes` (or `ChallengeResponseAuthentication yes` for older versions) and restarted the SSH service. The PAM config alone isn't enough.

- **Login fails immediately after typing password! (PAM Module Missing)**
  The system can't find `pam_auth.so`. Validate that the module is correctly inside `/lib/x86_64-linux-gnu/security/` and not just in `/lib/security/`.

- **User gets "Permission Denied" without a TOTP prompt!**
  If a user has not been enrolled using `setup_user.py`, the system strictly denies them access by default to prevent bypasses. Run the setup script for that user.

- **My valid codes are constantly being rejected!**
  Authenticator apps rely heavily on the exact time. Check your phone's clock sync as well as the Ubuntu server's clock (`date`). Consider using `chrony` or `systemd-timesyncd` to keep server time perfectly accurate.

- **I was locked out suddenly!**
  You reached the 5-fail limit. The account gets a mandatory 15-minute rate limit freeze. Wait it out and try again.

If you are a developer looking to test internally without changing system configs, use:

```bash
make test
./auth_test
```
