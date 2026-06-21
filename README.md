# Linux Security Project

Hardening an **Ubuntu Server** to a professional standard — locking down a homelab server to significantly reduce the possibility of a breach, while understanding the *why* behind every control.

## Contents
- [SSH Key Authentication](#ssh-key-authentication)
- [Creating SSH Key Authentication](#creating-ssh-key-authentication)
- [Firewall (UFW) Setup](#firewall-ufw-setup)
- [Automatic Security Updates](#automatic-security-updates)
- [Fail2ban](#fail2ban)
- [Project Summary](#project-summary)

---

## SSH Key Authentication

To access your server, you typically have a login that requires a username and password. When you use "something you know," anyone who learns the password by guessing, brute force, or stealing it has access to the server, and every login sends that secret across the network.

Key authentication uses "something you have," which is a secret that never has to leave your Mac and never travels across the network. The server can automatically verify you hold it without ever seeing it.

### The Key Pair

A key pair has two components (two halves of a lock):

| | Private Key | Public Key |
|---|---|---|
| **Role** | The only key that opens the padlock | Think of it like a padlock — hand out copies and bolt one onto every server |
| **File** | `id_ed25519` | `id_ed25519.pub` |
| **Lives** | Only on your Mac | Copied to servers |
| **Shareable** | **Never** | Freely |

Giving out the padlock (public key) tells an attacker nothing about the key. `ssh-keygen` creates the mathematically linked pair.

### What Happens When You Connect

1. You run `ssh username@server`
2. The server looks into the user's `~/.ssh/authorized_keys` file
3. It picks your key and uses it to create a **challenge** (a randomized puzzle that only the matching key could solve)
4. Your Mac solves the challenge with your private key **locally** — the private key never leaves your Mac or goes over the internet
5. The server checks the answer against the public key
6. Math checks out — you're in, and no password is ever sent

`ssh-copy-id` is just the one-time step of putting your padlock on the server's "allowed list" (`authorized_keys`).

### Why Do This?

- SSH key authentication protects the **client-authentication** step of the client-server handshake. It's how the client proves its identity to the server, replacing a weak shared-secret password with a cryptographic proof.
- There's no way for the password to be brute forced or guessed. The key is astronomically large.
- Nothing secret crosses the wire. Even on a hostile network, there's nothing to steal.
- It scales. One private key on your Mac unlocks 100 servers. Revoke access by deleting one line from a server's `authorized_keys` — no password resets needed.
- Once keys are set up, passwords are just a weaker door left unlocked.

### Where the Passphrase Fits (Common Confusion)

- **Passphrase:** unlocks your private key, on your Mac only. Protects you if your laptop is stolen.
- **Server Password:** what we're replacing.

> Key auth makes remote attacks against the server pointless. The passphrase defends against offline attacks on a stolen private key file, and its strength determines how long that offline attack takes.

The two layers cover two different threats: **the key** stops the network/server attack, **the passphrase** stops the stolen-file attack. Defense in depth.

---

## Creating SSH Key Authentication

### Step 1: Generate the Key

```bash
ssh-keygen -t ed25519 -C "tacevedo@ubuntuhomelab-from-mac"
```

- `ssh-keygen`: the key-generator program; its whole job is to create the key pair
- `-t ed25519`: `-t` means **type**. You're saying make the key using the ed25519 algorithm (the modern, strong kind)
- `-C "tacevedo@ubuntuhomelab-from-mac"`: `-C` is the comment feature to label the key so it's distinguishable in the `~/.ssh/authorized_keys` file among other keys

Generate the key, save it to the default file (`~/.ssh/id_ed25519`), create your passphrase, then confirm both keys are saved:

```bash
ls -l ~/.ssh/id_ed25519*
```

The `*` character checks for all files starting with `id_ed25519`, whether or not there's anything after it. This is because there are two keys in the pair, and it saves you from running the command twice.

### Step 2: Copy the Public Key to the VM

```bash
ssh-copy-id username@<your-vm-ip>
```

Log in via password one last time in order to drop the public key into the server's `~/.ssh/authorized_keys` directory.

### Step 3: Log in via SSH Key

Notice this time it asks for the **passphrase** instead of the actual server password. You now have cryptographic, passwordless login to the server.

### Step 4: Lock the Door Behind You

Now that you have key authentication enabled, disable passwords so the server can't be compromised via password brute-force.

> **Best practice:** Never disable password login until you have confirmed key authentication is live.

To do this, change the SSH server's config file `/etc/ssh/sshd_config` and set three things:

- `PasswordAuthentication no` — server stops accepting passwords
- `PermitRootLogin no` — nobody can SSH in directly as root
- `PubkeyAuthentication yes` — keys stay enabled (usually already on)

`/etc/ssh/sshd_config` is the config file for the SSH daemon. When you change the rules and restart the service, the rules change for everyone who tries to connect.

#### Why Turn Off `PermitRootLogin`?

**Root is the universal target.** The reason we block root login is because root is the universal name of the all-powerful account found on basically every Linux server. With that in mind, half the puzzle is already solved when an attacker targets your system — the only thing left is to figure out the password. When you disable root login, the attacker is forced to also figure out the account name, making it exponentially harder.

**Accountability and blast radius.** Professionals rarely log in as the root user. They use `sudo` to do admin tasks for two critical reasons:

- **Audit trail:** logs show "user ran this command" rather than an anonymous "root did something" entry. On a team you can identify who did what.
- **Smaller blast radius:** a normal user session cannot nuke the entire system — you must use `sudo`. When logged in as root, it's easier to do catastrophic damage with a wrong command. It's the equivalent of carrying a loaded gun everywhere versus taking it out only when you mean to.

#### Why Enable `PubkeyAuthentication`?

This confirms the server accepts key-based authentication. It will be the **only** method of authentication after turning off `PasswordAuthentication`, so it's more important than ever to test and confirm your key works.

### Editing the Config File

**Confirm your safety net:** keep the current SSH session open throughout. This is your lifeline — if anything goes wrong, this already-authenticated session lets you undo it. Do not close it until you've tested a fresh login.

**Back up the config first** — before making changes to any config file, back it up in case you need to restore it:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

**Open the file in nano** (without `sudo`, the file will be unwritable):

```bash
sudo nano /etc/ssh/sshd_config
```

Edit the config file by removing the `#` and changing the settings, then write out.

> **Important:** Modern Ubuntu contains extra config files in a drop-in folder (`/etc/ssh/sshd_config.d/`). SSH pulls the **first value it sees**, and more often than not it reads those extra config files first, which can override your changes. Make sure to check and update those override files too.

**Validate the changes** — this checks for errors in the config files. If nothing returns, there's no issue:

```bash
sudo sshd -t
```

**Restart the SSH daemon** to apply changes:

```bash
sudo systemctl restart ssh
```

**Last step:** rerun the forced password login command in a new terminal to confirm password access has been disabled.

---

## Firewall (UFW) Setup

Right now the server accepts connection attempts on any port. A firewall flips that idea by not allowing any connection attempts on any port except the ones you explicitly allow. It's the "who can even knock on my door" layer.

### What Does the Firewall Actually Do?

The server has thousands of "doors" (ports 1–65535) and different services listen on different ones:

- SSH on port **22**
- Web on **80 / 443**

Right now, every port is potentially answerable, meaning anything running on the server is potentially reachable from the network. The firewall is essentially a **bouncer** that stands at every port and enforces a guest list.

The professional default is **"default deny"**: deny everyone, then allow only the specific doors you name. This is the gold standard. We use **UFW** (Uncomplicated Firewall), Ubuntu's friendly front-end to the underlying firewall.

### The Golden Rule

> **Allow SSH before you enable the firewall. Always. No exceptions.**

You're currently using SSH to connect to the server remotely over port 22. If you implement default deny without putting port 22 on the guest list, you will lock yourself out of the server with no way to get back in.

### Implementation

```bash
# 1. Check the current state of the firewall
sudo ufw status

# 2. Set the default policies
sudo ufw default deny incoming    # Block all inbound connections by default
sudo ufw default allow outgoing   # Let your server reach out (updates, DNS, etc.)

# 3. Allow SSH (always do this before enabling)
sudo ufw allow OpenSSH

# 4. Enable the firewall
sudo ufw enable

# 5. Verify
sudo ufw status verbose
```

You should see the rules allowing port 22, denying incoming, and allowing outgoing.

---

## Automatic Security Updates

This enables automatic downloading and installation of software updates. Updates provide important patches to plug security holes and fix bugs. By auto-installing security patches, the server always stays up to date without you having to babysit it. The solution is a package called **unattended-upgrades**.

```bash
# 1. Install unattended-upgrades (-y answers yes to all follow-up questions)
sudo apt update
sudo apt install unattended-upgrades -y

# 2. Enable it with the official setup helper
sudo dpkg-reconfigure --priority=low unattended-upgrades

# 3. Verify it's active
sudo systemctl status unattended-upgrades

# 4. Confirm what it will patch
cat /etc/apt/apt.conf.d/20auto-upgrades
```

You want to see both lines set to `"1"`:

```
APT::Periodic::Update-Package-Lists "1";   # Refresh the update list daily
APT::Periodic::Unattended-Upgrade "1";     # Auto-install security updates
```

To find the log activity:

```bash
ls /var/log/unattended-upgrades/
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log
```

> Knowing where a service logs is a core sysadmin instinct. For most services, that answer is somewhere under `/var/log/`.

---

## Fail2ban

SSH keys only protect against brute-force login attempts. However, bots will still pound your server with thousands of failed attempts, wasting resources and filling logs.

Fail2ban watches your logs, and when an IP racks up repeated failures, it auto-bans that IP at the firewall for a while. It makes attackers pay for even trying.

**How it works:** reads `/var/log/` → manipulates the firewall → runs as a background daemon (always on).

```bash
# 1. Install fail2ban
sudo apt install fail2ban -y

# 2. Create your own config
# Don't edit the original directly — package updates overwrite it.
# Make a .local copy that takes precedence:
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# 3. Start it and enable on boot
sudo systemctl enable --now fail2ban

# 4. Verify it's running and watching SSH
sudo systemctl status fail2ban
sudo fail2ban-client status sshd
```

---

## Project Summary

In this project, the goal was to create a secure Linux server that would significantly reduce the possibility of a breach.

I started by switching to **SSH key authentication** for secure login. After that, I wanted to ensure the server couldn't be breached via brute-force password login, so I **disabled password login over SSH and disabled root login access**. I also set up a **passphrase** on my Mac to ensure the key on my local machine is secure from an attacker stealing the private key file.

Next, I enabled a **UFW firewall** using the default-deny gold standard rule. The firewall does not allow any connection attempts to reach the server over the network unless the port is explicitly allowed. The only ports I allowed were SSH on port 22 and web traffic on 80/443. All other ports are blocked, significantly reducing the number of vulnerabilities or "open doors" available to an attacker.

I also implemented **automatic security patch downloads** using the unattended-upgrades package. This means I don't have to constantly babysit the server and manually apply patches — it handles security updates automatically, ensuring the server is always up to date.

Lastly, I addressed **bot protection** with Fail2ban. Even with SSH key authentication only, bots will still hammer the server with requests, exhausting resources and filling up the logs. Fail2ban solves this by watching the logs and identifying repeat-offender IP addresses. When there are enough failed attempts, fail2ban bans that IP from the server, significantly reducing the number of requests bots can make.

Overall, this project helped me understand the foundational and important aspects of locking down a server. I learned about authentication best practices, firewalls, security automation, and IP banning. Together, all of these services play a big role in hardening a server's security.
