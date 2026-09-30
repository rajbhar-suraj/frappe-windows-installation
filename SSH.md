# SSH Access Guide — End to End
### What every command does and why it's needed

---

## What is SSH?

SSH (Secure Shell) is a way to **remotely control another computer via terminal** — securely, over a network. Instead of sitting in front of the server, you type commands on your own machine and they run on the remote one.

You need two things to connect:
- A **key pair** (like a lock and key) — generated once on your machine
- The server must **know your public key** — so it can verify you're allowed in

---

## PART 1 — On the person's machine (who needs access)

### Step 1: Generate a key pair

```bash
ssh-keygen -t ed25519 -C "yourname@example.com"
```

| Part | What it means |
|------|--------------|
| `ssh-keygen` | Tool that generates SSH keys |
| `-t ed25519` | The encryption algorithm — ed25519 is modern, fast, and secure |
| `-C "yourname@example.com"` | A comment/label — just so you know whose key it is later |

**Why necessary:** Without a key pair, you have no way to authenticate. SSH on this server has password login disabled, so a key is the only way in.

This creates two files:
- `~/.ssh/id_ed25519` — your **private key** (never share this, ever)
- `~/.ssh/id_ed25519.pub` — your **public key** (this is what you share)

---

### Step 2: View and copy your public key

```bash
cat ~/.ssh/id_ed25519.pub
```

| Part | What it means |
|------|--------------|
| `cat` | Prints a file's content to the screen |
| `~/.ssh/id_ed25519.pub` | The public key file in your `.ssh` folder |

**Why necessary:** You need to send this exact line to the server owner so they can add it. The output looks like:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... yourname@example.com
```

Copy the whole line — it must be one single line with no line breaks.

---

## PART 2 — On the server (the machine being accessed)

### Step 3: Create the `.ssh` directory

```bash
sudo mkdir -p /home/jbs_user/.ssh
```

| Part | What it means |
|------|--------------|
| `sudo` | Run as admin (superuser) — needed because you're touching another user's folder |
| `mkdir` | Make a new directory (folder) |
| `-p` | Don't throw an error if the folder already exists |
| `/home/jbs_user/.ssh` | The hidden `.ssh` folder inside jbs_user's home directory |

**Why necessary:** SSH looks for authorized keys inside this specific folder. If it doesn't exist, SSH has nowhere to check and will reject the connection.

---

### Step 4: Set correct permissions on the `.ssh` folder

```bash
sudo chmod 700 /home/jbs_user/.ssh
```

| Part | What it means |
|------|--------------|
| `chmod` | Change permissions on a file or folder |
| `700` | Only the owner can read, write, and enter — nobody else can touch it |
| `/home/jbs_user/.ssh` | The folder we're locking down |

**Why necessary:** SSH is extremely strict about permissions. If the `.ssh` folder is readable by others, SSH will **silently refuse** to use the keys inside it — you won't even get an error message.

Permission breakdown:
```
7 = owner (jbs_user) can read + write + execute
0 = group cannot do anything
0 = others cannot do anything
```

---

### Step 5: Add their public key to authorized_keys

```bash
echo "ssh-ed25519 AAAA...their-key... theirname@example.com" \
  | sudo tee -a /home/jbs_user/.ssh/authorized_keys
```

| Part | What it means |
|------|--------------|
| `echo "..."` | Prints the public key text |
| `\|` | Pipes (sends) that output into the next command |
| `sudo tee` | Writes the input to a file with admin privileges |
| `-a` | Append — adds to the end of the file without deleting existing content |
| `authorized_keys` | The file SSH reads to decide who is allowed in |

**Why necessary:** This is the core step. SSH checks this file on every login attempt. If a connecting person's private key matches any public key in this file, they get in.

> **Warning:** Using `tee` without `-a` overwrites the whole file and wipes everyone else's access.

To add a second person later, just repeat this same command with their key — `-a` keeps all existing keys.

---

### Step 6: Fix ownership of the `.ssh` folder

```bash
sudo chown -R jbs_user:jbs_user /home/jbs_user/.ssh
```

| Part | What it means |
|------|--------------|
| `chown` | Change the owner of a file or folder |
| `-R` | Recursive — applies to everything inside the folder too |
| `jbs_user:jbs_user` | Set owner to jbs_user, and group to jbs_user |
| `/home/jbs_user/.ssh` | The folder (and its contents) we're fixing |

**Why necessary:** If you created the folder or file as root (via sudo), it might be owned by root. SSH won't use keys from files it doesn't consider owned by the correct user — it will silently reject the connection.

---

### Step 7: Lock down the authorized_keys file

```bash
sudo chmod 600 /home/jbs_user/.ssh/authorized_keys
```

| Part | What it means |
|------|--------------|
| `chmod 600` | Only the owner can read and write — nobody else |
| `authorized_keys` | The file storing all allowed public keys |

**Why necessary:** Same strict rule as the folder — if this file is readable by group or others, SSH refuses to use it. 600 is the exact permission SSH requires.

```
6 = owner can read + write
0 = group cannot do anything
0 = others cannot do anything
```

---

## PART 3 — Verify everything is correct

### Step 8: Check the key is valid

```bash
sudo ssh-keygen -lf /home/jbs_user/.ssh/authorized_keys
```

| Part | What it means |
|------|--------------|
| `ssh-keygen -l` | Show the fingerprint of a key |
| `-f` | Specify the file to read from |

**Why necessary:** If the key was pasted incorrectly (extra space, line break, truncated), this command will error. If it prints a fingerprint like `SHA256:abc123...` the key format is valid.

---

### Step 9: Confirm permissions on all three paths

```bash
sudo ls -lad /home/jbs_user /home/jbs_user/.ssh /home/jbs_user/.ssh/authorized_keys
```

| Part | What it means |
|------|--------------|
| `ls` | List files |
| `-l` | Long format — shows permissions, owner, size |
| `-a` | Include hidden files (like `.ssh`) |
| `-d` | Show the directory itself, not what's inside it |

**Why necessary:** Lets you visually confirm all three paths have the right permissions and owner before testing. What you want to see:

| Path | Permission | Owner |
|------|-----------|-------|
| `/home/jbs_user` | `drwxr-xr-x` (755) | `jbs_user` |
| `/home/jbs_user/.ssh` | `drwx------` (700) | `jbs_user` |
| `/home/jbs_user/.ssh/authorized_keys` | `-rw-------` (600) | `jbs_user` |

---

## PART 4 — Connect to the server

### Step 10: SSH in (same network / LAN)

```bash
ssh jbs_user@192.168.0.115
```

| Part | What it means |
|------|--------------|
| `ssh` | The command to initiate a secure connection |
| `jbs_user` | The account to log into on the remote machine |
| `@` | Separator between username and address |
| `192.168.0.115` | The server's local IP address on the same network |

**Why necessary:** This is the actual connection command. On first connect you'll see a prompt asking you to confirm the server's fingerprint — type `yes`. After that, you're in.

---

### Step 11: Navigate to the bench folder

```bash
cd ~/frappe-bench-16
```

| Part | What it means |
|------|--------------|
| `cd` | Change directory — move into a folder |
| `~` | Shortcut for the home directory (`/home/jbs_user`) |
| `frappe-bench-16` | The bench folder name |

```bash
bench version
```

Confirms bench is working. If this prints app versions, everything is set up correctly.

---

## PART 5 — If connecting from the internet (outside your network)

The public IP `103.60.211.142` belongs to your **router**, not the server directly. Traffic hitting that IP only reaches the server if you tell the router to forward it.

### Set up port forwarding on your router

Log into your router admin panel and add a rule:

| Field | Value |
|-------|-------|
| External port | `22` |
| Internal IP | `192.168.0.115` |
| Internal port | `22` |
| Protocol | TCP |

Then they connect using:

```bash
ssh jbs_user@103.60.211.142
```

### Check your firewall allows SSH

```bash
sudo ufw status
sudo ufw allow 22/tcp
```

| Part | What it means |
|------|--------------|
| `ufw` | Ubuntu's built-in firewall tool |
| `status` | Show current firewall rules |
| `allow 22/tcp` | Open port 22 for TCP traffic (SSH's port) |

**Why necessary:** Even if port forwarding is set up on the router, the server's own firewall might be blocking port 22. Both need to allow it.

### Confirm SSH is actually running

```bash
sudo systemctl status ssh
sudo ss -tlnp | grep :22
```

| Part | What it means |
|------|--------------|
| `systemctl status ssh` | Shows if the SSH service is running |
| `ss -tlnp` | Lists all open/listening ports |
| `grep :22` | Filters to only show port 22 |

**Why necessary:** If SSH isn't running or isn't listening on port 22, nobody can connect regardless of keys or firewall rules.

---

## Quick Permission Fix (if something is wrong)

Run these all at once to reset everything to correct values:

```bash
sudo chmod 755 /home/jbs_user
sudo chmod 700 /home/jbs_user/.ssh
sudo chmod 600 /home/jbs_user/.ssh/authorized_keys
sudo chown -R jbs_user:jbs_user /home/jbs_user/.ssh
```

---

## Adding / Removing People

**Add someone new** — just append their key:
```bash
echo "ssh-ed25519 AAAA...newkey... newperson@example.com" \
  | sudo tee -a /home/jbs_user/.ssh/authorized_keys
```

**Remove someone** — open the file and delete their line:
```bash
sudo nano /home/jbs_user/.ssh/authorized_keys
# Delete their line, then Ctrl+O to save, Ctrl+X to exit
```

**Clear everything and start fresh:**
```bash
sudo truncate -s 0 /home/jbs_user/.ssh/authorized_keys
```

---

## Summary — The Whole Flow

```
THEIR MACHINE                          YOUR SERVER
─────────────────────────────────────────────────────
1. ssh-keygen  →  creates key pair
2. cat id_ed25519.pub  →  copy this line
                               ↓ paste it
                         3. mkdir .ssh
                         4. chmod 700 .ssh
                         5. echo "key" | tee -a authorized_keys
                         6. chown jbs_user:jbs_user .ssh
                         7. chmod 600 authorized_keys
                         8. verify with ssh-keygen -lf

CONNECT (same network):
ssh jbs_user@192.168.0.115

CONNECT (internet):
Router port forward 22 → 192.168.0.115
ssh jbs_user@103.60.211.142
```
