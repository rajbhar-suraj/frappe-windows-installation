# Running Frappe on Linux

Linux is Frappe's home turf, so you have two good options:

- **Option A: Docker (easy-install script)**. Fastest, least fiddly. Everything runs in containers.
- **Option B: Manual bench install**. Runs natively on your machine. Better for development and for learning how Frappe is put together.

This guide uses **Ubuntu/Debian** commands. On other distros, package names and the package manager will differ.

---

# Option A: Docker (easy-install script)

## Step 1: Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

**What this does:** `apt update` refreshes the list of available packages; `apt upgrade -y` installs pending updates (`-y` auto-confirms prompts).

## Step 2: Install prerequisites

```bash
sudo apt install python3 git wget -y
```

**What this does:**
- `python3` runs the install script
- `git` fetches Frappe's source and apps
- `wget` downloads the script

## Step 3: Download the easy-install script

```bash
wget https://raw.githubusercontent.com/frappe/bench/develop/easy-install.py
```

**What this does:** Downloads Frappe's official setup script, which installs Docker (if missing), pulls the required containers, and creates your first site.

## Step 4: Run the script

For a local/dev setup:

```bash
python3 easy-install.py --dev
```

For a production-style setup with SSL (needs a real domain pointing to your server):

```bash
python3 easy-install.py --prod --email your@email.tld
```

**What this does:** `--dev` sets up a simple development instance. `--prod` sets up a production-ready instance with Nginx and SSL certificates tied to your email. In both cases the script installs Docker, pulls the Frappe images, and creates a default site.

## Step 5: Find your passwords

```bash
cat $HOME/passwords.txt
```

**What this does:** Prints the auto-generated MySQL root password and Administrator password saved by the script.

## Step 6: Open your site

Go to `http://localhost` (or your server's IP/domain) in a browser and log in:

- **Username:** `Administrator`
- **Password:** from `passwords.txt`

## Daily Docker commands

```bash
docker ps
```
**What this does:** Lists running containers, to confirm Frappe's services are up.

```bash
docker exec -it <container-name> bash
```
**What this does:** Opens a shell inside the backend container, where bench lives (usually `/home/frappe/frappe-bench`).

```bash
docker compose -p <project-name> down
docker compose -p <project-name> up -d
```
**What this does:** `down` stops and removes the containers (data volumes are kept); `up -d` starts them again in the background.

> **Tip:** After Docker is freshly installed, run `newgrp docker` or log out and back in, so your user can run `docker` without `sudo`.

---

# Option B: Manual bench install (native)

## Step 1: Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

**What this does:** Refreshes package lists and applies updates before installing anything.

## Step 2: Install system dependencies

```bash
sudo apt install -y git python-is-python3 python3-dev python3-setuptools python3-pip \
  python3-venv software-properties-common mariadb-server mariadb-client \
  redis-server xvfb libfontconfig wkhtmltopdf libmariadb-dev curl pipx
```

**What this does:**
- `git`: source control, used by bench to fetch apps
- `python-is-python3`, `python3-dev`, `python3-setuptools`, `python3-pip`, `python3-venv`: the Python runtime plus tools for building and isolating packages
- `mariadb-server` / `mariadb-client`: the database Frappe stores data in
- `redis-server`: used for caching and background job queues
- `xvfb`, `libfontconfig`, `wkhtmltopdf`: needed to render PDFs (invoices, reports)
- `libmariadb-dev`: headers needed to build the Python MariaDB connector
- `curl`: used to download the Node installer
- `pipx`: installs Python command-line tools in their own isolated environments

## Step 3: Configure MariaDB

Run the security setup:

```bash
sudo mysql_secure_installation
```

**What this does:** Walks you through setting the MariaDB root password, removing anonymous users, and disabling remote root login. Answer "Y" to the hardening prompts.

Then edit the config:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Under the `[mysqld]` section add:

```ini
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

And add this new section at the bottom:

```ini
[mysql]
default-character-set = utf8mb4
```

**What this does:** Forces MariaDB to use the `utf8mb4` character set, which Frappe requires so all languages and emojis store correctly. Without it, site creation fails.

Restart MariaDB to apply the changes:

```bash
sudo systemctl restart mariadb
```

**What this does:** Reloads MariaDB with the new configuration.

## Step 4: Install Node.js (via nvm) and Yarn

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
source ~/.bashrc
nvm install 18
npm install -g yarn
```

**What this does:**
- The `curl ... | bash` line installs **nvm** (Node Version Manager), which lets you install and switch Node versions per user without `sudo`
- `source ~/.bashrc` reloads your shell so the `nvm` command becomes available
- `nvm install 18` installs Node 18
- `npm install -g yarn` installs Yarn, the package manager Frappe uses for its JavaScript assets

> **Version note:** The required Node and Python versions depend on the Frappe version you install. Check the official Frappe installation docs for the right combination. For example, Frappe v15 needs Python 3.10+ and Node 18.

## Step 5: Install bench

```bash
pipx install frappe-bench
pipx ensurepath
source ~/.bashrc
```

**What this does:** Installs the `bench` CLI (the tool that creates and manages Frappe environments) in an isolated environment. `pipx ensurepath` adds it to your PATH, and `source ~/.bashrc` reloads your shell. If your distro allows it, `pip3 install frappe-bench` also works.

Confirm it installed:

```bash
bench --version
```

## Step 6: Create a bench (the Frappe workspace)

```bash
bench init --frappe-branch version-15 frappe-bench
cd frappe-bench
```

**What this does:** `bench init` creates a folder called `frappe-bench`, sets up a Python virtual environment, downloads the Frappe framework at the chosen version, and builds its JavaScript assets. This takes several minutes. `cd` moves you into the new folder.

## Step 7: Create a site

```bash
bench new-site site1.local
```

**What this does:** Creates a new site with its own database. It asks for the MariaDB root password (from Step 3) and lets you set the site's Administrator password.

Add the site name to your hosts file so your browser can resolve it:

```bash
bench --site site1.local add-to-hosts
```

**What this does:** Adds `site1.local` to `/etc/hosts`, pointing it at your own machine.

## Step 8: Start Frappe

```bash
bench start
```

**What this does:** Starts the web server, background workers, scheduler, Redis, and the socket.io server all together, running in the foreground. Press `Ctrl+C` to stop.

Open `http://site1.local:8000` (or `http://localhost:8000`) and log in with:

- **Username:** `Administrator`
- **Password:** the one you set in Step 7

## Step 9: Install an app (optional)

```bash
bench get-app erpnext --branch version-15
bench --site site1.local install-app erpnext
```

**What this does:** `get-app` downloads the app's code into your bench; `install-app` installs it on your site and creates its database tables.

---

## Using VS Code

**Native install (Option B):** Just open the bench folder:

```bash
code ~/frappe-bench
```

**What this does:** Opens the whole bench workspace in VS Code so you can edit `apps/` and `sites/` directly. If you're on a remote Linux server, use the **Remote - SSH** extension to connect and open the folder.

**Docker install (Option A):** Install the **Dev Containers** extension, press `Ctrl+Shift+P`, run `Dev Containers: Attach to Running Container...`, choose your backend container, then open `/home/frappe/frappe-bench`.

---

## Useful bench commands

| Command | What it does |
|---|---|
| `bench start` | Runs all services in development mode |
| `bench --site site1.local console` | Opens a Python console connected to the site |
| `bench --site site1.local migrate` | Applies database schema updates after code changes |
| `bench --site site1.local clear-cache` | Clears cached data when the UI behaves oddly |
| `bench build` | Rebuilds JavaScript/CSS assets |
| `bench new-app myapp` | Scaffolds a new custom app |
| `bench update` | Pulls updates for all apps and runs migrations |
| `bench --site site1.local backup` | Creates a database and files backup |

---

## Troubleshooting

- **"Access denied" or character set errors on `new-site`:** Recheck the MariaDB config in Step 3 and restart MariaDB.
- **`bench: command not found`:** Run `pipx ensurepath` and reopen your terminal.
- **Site not loading in browser:** Confirm `bench start` is still running and the site name is in `/etc/hosts`.
- **Port 8000 already in use:** Stop the other process, or run `bench serve --port 8001`.
- **Docker permission denied:** Run `newgrp docker` or log out and back in after installing Docker.
