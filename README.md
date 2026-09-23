# Running Frappe on Windows

Frappe (and the `bench` CLI that manages it) is built for Linux. On Windows, the reliable way to run it is through **WSL2** (Windows Subsystem for Linux) — it gives you a real Linux environment inside Windows, with near-native performance. This guide walks through that path using the official **easy-install script**, which is the least fiddly option.

---

## Step 1: Enable WSL2

Open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

**What this does:** Installs the WSL2 subsystem, downloads a default Linux distribution (Ubuntu), and sets up the virtual machine platform Windows needs to run Linux natively. Restart your PC when prompted.

If WSL is already installed and you just need to make sure you're on version 2:

```powershell
wsl --set-default-version 2
```

**What this does:** Tells Windows to use the faster, more compatible WSL2 architecture (instead of the older WSL1) for any new Linux distro you install.

---

## Step 2: Launch Ubuntu and create your Linux user

After restarting, open **Ubuntu** from the Start menu. On first launch it will ask you to create a username and password for your Linux environment — this is separate from your Windows login.

---

## Step 3: Update Ubuntu's package list

Inside the Ubuntu terminal:

```bash
sudo apt update && sudo apt upgrade -y
```

**What this does:** `apt update` refreshes the list of available packages and their versions; `apt upgrade -y` installs any pending updates automatically (`-y` auto-confirms prompts). Keeps your base system current before installing anything else.

---

## Step 4: Install base prerequisites

```bash
sudo apt install python3-minimal build-essential python3-setuptools git -y
```

**What this does:**
- `python3-minimal` — the Python interpreter the install script and bench itself run on
- `build-essential` — compilers and build tools (gcc, make, etc.) needed to build some Python/Node packages from source
- `python3-setuptools` — packaging tools Python needs to install other Python packages
- `git` — needed to fetch Frappe's source code and apps

---

## Step 5: Download the easy-install script

```bash
wget https://raw.githubusercontent.com/frappe/bench/develop/easy-install.py
```

**What this does:** Downloads Frappe's official setup script, which automates installing Docker, pulling the right containers, configuring bench, and creating your first site — so you don't have to do each of those steps by hand.

---

## Step 6: Run the script

For a local/dev environment (recommended for trying it out on Windows):

```bash
python3 easy-install.py --dev
```

For a production-style setup with SSL (only if you're exposing this on a real domain):

```bash
python3 easy-install.py --prod --email your@email.tld
```

**What this does:** `python3` runs the script you just downloaded. The `--dev` flag sets up a development instance (simpler, no SSL, easy to poke around in). The `--prod` flag sets up a production-ready instance with Docker containers, Nginx, and SSL certificates tied to the email you provide. The script installs Docker automatically if it isn't already present, pulls the Frappe images, and creates a default site.

---

## Step 7: Find your generated passwords

```bash
cat $HOME/passwords.txt
```

**What this does:** `cat` prints a file's contents to the terminal. This file holds the auto-generated MySQL root password and the Administrator password for your new Frappe site, which the script created for you.

---

## Step 8: Access your site

Open a browser on the **Windows** side (not inside Ubuntu) and go to:

```
http://localhost
```

**Why this works:** WSL2 automatically forwards `localhost` ports to Windows, so a site running inside your Ubuntu environment is reachable from your normal Windows browser without extra configuration.

Log in with:
- **Username:** `Administrator`
- **Password:** the one from `passwords.txt`

---

## Everyday commands once it's running

```bash
docker ps
```
**What this does:** Lists all currently running Docker containers — useful to confirm Frappe's containers (web, database, redis, etc.) are up.

```bash
docker compose down
```
**What this does:** Stops and removes the running containers for your Frappe instance (run this from the project folder the script created, usually named after your `--project` value or `frappe-bench`).

```bash
docker compose up -d
```
**What this does:** Restarts your Frappe instance in the background (`-d` = detached mode, so it keeps running after you close the terminal).

---

## Accessing the bench folder inside the container

The easy-install script runs everything inside Docker containers, so the `frappe-bench` folder isn't just sitting in your WSL filesystem — it lives inside a container's own filesystem. Here's how to get to it.

### Find your running containers

```bash
docker ps
```

**What this does:** Lists all running containers with their names. Look for one named something like `<project>-backend-1` or `frappe-bench` — that's the container where bench actually lives and runs.

### Open a shell inside the backend container

```bash
docker exec -it <container-name> bash
```

**What this does:**
- `docker exec` runs a command inside an already-running container
- `-it` makes it interactive (a live terminal, not just one command's output)
- `bash` starts a shell inside the container, landing you inside its Linux filesystem — including the bench folder, usually at `/home/frappe/frappe-bench`

### Use bench commands from inside the container

```bash
cd frappe-bench
bench --site site1.local console
bench --site site1.local migrate
bench new-app myapp
```

**What this does:** Once inside the container's shell, you're using the container's own copy of bench, so every command runs directly against the live site — opening a Python console, applying database migrations, or scaffolding a new custom app.

### Exit back to Windows/WSL

```bash
exit
```

**What this does:** Closes the interactive shell and drops you back into your normal WSL terminal. The container itself keeps running in the background — this doesn't stop it.

### Copy files out to Windows if needed

```bash
docker cp <container-name>:/home/frappe/frappe-bench/apps/myapp ./myapp
```

**What this does:** `docker cp` copies files or folders between your host machine and a container, in either direction. Handy for grabbing a log file, a custom app you built, or a backup without opening a full shell session.

---

## Using VS Code with the container

Editing code entirely through `docker exec` works but is painful — no syntax highlighting, no file tree, no easy multi-file editing. VS Code's **Dev Containers** extension solves this by opening a full VS Code window pointed directly at the container's filesystem.

### Step 1: Install the extension

In VS Code (on Windows), open the Extensions panel (`Ctrl+Shift+X`) and install **Dev Containers** (published by Microsoft).

**What this does:** Adds the ability for VS Code to open a remote development session inside any Docker container or WSL distro, instead of just your local Windows filesystem.

### Step 2: Attach to the running container

Open the Command Palette (`Ctrl+Shift+P`) and run:

```
Dev Containers: Attach to Running Container...
```

Then pick your Frappe backend container from the list (the same one you found with `docker ps` earlier).

**What this does:** Opens a new VS Code window whose file explorer, terminal, and extensions all run *inside* the container. You're now browsing and editing the actual `frappe-bench` folder as it exists inside Docker, not a copy.

### Step 3: Open the bench folder

Once attached, use **File → Open Folder** and navigate to:

```
/home/frappe/frappe-bench
```

**What this does:** Points VS Code's workspace at the bench directory itself, so you get a normal file tree for `apps/`, `sites/`, and your custom app code.

### Step 4: Install useful extensions inside the container

VS Code will prompt you to install extensions again since they run inside the container's environment. At minimum, install **Python** and, if you're building a custom app, **ESLint** for the JS side.

**What this does:** Extensions installed on your Windows VS Code don't automatically apply inside the container session — the container has its own isolated extension host, so linting/autocomplete need to be set up there too.

### Alternative: connect via WSL instead of the container directly

If you'd rather edit files through WSL's own filesystem (e.g. if you mounted a local folder into the container as a volume), install the **WSL** extension instead, then `Ctrl+Shift+P` → `WSL: Connect to WSL`, and open the mounted folder from there. This only works if the compose setup maps a host folder into the container as a volume — check your project's `docker-compose.yml` for a `volumes:` entry pointing at `frappe-bench`.

---

## Notes

- Always work **inside the Ubuntu/WSL terminal**, not Windows Command Prompt or PowerShell — the script and bench commands are Linux-only.
- Docker Desktop for Windows should have **WSL2 integration** enabled (Docker Desktop → Settings → Resources → WSL Integration) if you install Docker Desktop separately instead of letting the script install it.
- If `wsl --install` doesn't work, your Windows version may need a manual update first — check `winver` and ensure you're on Windows 10 version 2004+ or any Windows 11.
