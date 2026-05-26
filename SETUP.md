# Dev Environment Setup (ssc_frappe_docker)

This guide walks you through getting a working dev environment on your machine for the
`ssc_custom` Frappe app, on top of Frappe + ERPNext **version-16**, using VS Code Dev
Containers. Everything below assumes you already have access to:

- `github.com/thesidagarwal/ssc_frappe_docker` (this repo)
- `github.com/thesidagarwal/ssc_custom` (private)

If you don't have access yet, ask Sid to add you as a collaborator on both.

---

## 0. Host machine prerequisites

Install these on your laptop/PC (Windows, macOS, or Linux):

| Tool | Notes |
|------|-------|
| **Docker Desktop** | https://www.docker.com/products/docker-desktop/ — give it **≥ 4 GB RAM** (Settings → Resources). On Linux, install Docker Engine + the `docker compose` plugin and add your user to the `docker` group. |
| **VS Code** | https://code.visualstudio.com/ |
| **Dev Containers extension** | In VS Code: `Ctrl+Shift+X` → search `ms-vscode-remote.remote-containers` → Install. |
| **Git** | https://git-scm.com/downloads |

On **Windows**, install Git from the link above. WSL is not required for this setup —
Docker Desktop with the WSL2 backend is enough. The dev container provides Linux
internally; nothing on your host needs to be Linux.

---

## 1. SSH key for GitHub (required — `ssc_custom` is private)

The dev container mounts your `~/.ssh` folder so it can clone the private `ssc_custom`
repo using your SSH key. You need a working SSH key on GitHub before going further.

### Generate a key (skip if you already have one)

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
# Press Enter to accept default path (~/.ssh/id_ed25519)
# Optional: set a passphrase
```

On Windows PowerShell the command is the same; the key lands in
`C:\Users\<you>\.ssh\id_ed25519`.

### Add the public key to GitHub

```bash
# Print the public key
cat ~/.ssh/id_ed25519.pub
```

Copy the output → https://github.com/settings/keys → **New SSH key** → paste → Save.

### Verify it works

```bash
ssh -T git@github.com
# Expected: "Hi <username>! You've successfully authenticated, ..."
```

If you see "Permission denied", the key isn't registered on GitHub yet — re-do the
previous step.

---

## 2. Clone this repo

Use the SSH URL so push/pull work without prompts later:

```bash
git clone git@github.com:thesidagarwal/ssc_frappe_docker.git
cd ssc_frappe_docker
```

---

## 3. Copy the dev container preset and VS Code config

These two files/folders are gitignored on purpose so each developer can tweak them
locally without polluting the shared repo. Copy the team presets into place:

```bash
# From the repo root
cp -R devcontainer-ssc .devcontainer
cp -R development/vscode-example development/.vscode
```

On **Windows PowerShell**:

```powershell
Copy-Item -Recurse devcontainer-ssc .devcontainer
Copy-Item -Recurse development/vscode-example development/.vscode
```

---

## 4. Open in dev container

```bash
code .
```

In VS Code:

1. `Ctrl+Shift+P` → **Dev Containers: Reopen in Container**
2. Wait while VS Code builds the container, pulls images, and starts `mariadb`,
   `redis-cache`, `redis-queue`, and the `frappe` shell container. First run takes a
   few minutes.

When it's done, the VS Code window title shows `[Dev Container: Frappe Bench]` and the
integrated terminal opens inside the container as user `frappe`, in
`/workspace/development`.

---

## 5. Bootstrap bench + site + apps (one command)

Inside the dev container terminal:

```bash
python installer.py -j apps-ssc.json -t version-16
```

What this does:

- Runs `bench init` with `version-16` of frappe and the apps listed in
  `apps-ssc.json` (erpnext + ssc_custom on branch `develop`).
- Wires bench to the in-container `mariadb` and `redis-*` services.
- Creates site `development.localhost` (admin password: `admin`, MariaDB root
  password: `123`).
- Installs `erpnext` and `ssc_custom` on that site.

The `-t version-16` flag is important — the installer's built-in default is
`version-15`.

If this step fails with a clone error on `ssc_custom`, your SSH key isn't reachable
from inside the container. Check:

- Your key exists at `~/.ssh/id_ed25519` on your host.
- The Dev Containers extension successfully mounted `~/.ssh` (look for it in
  `/home/frappe/.ssh` inside the container).
- The key is added to your GitHub account (step 1).

---

## 6. Start the dev server

```bash
cd frappe-bench
bench start
```

Open http://development.localhost:8000 in your browser. Log in:

- Username: `Administrator`
- Password: `admin`

You should see ERPNext with `ssc_custom` features available.

---

## Daily workflow

- Apps live in `development/frappe-bench/apps/<app-name>/`. Each app is its own git
  repo with its own remote and branch — `cd` into it and use `git` as usual.
- After pulling new code in any app: `bench migrate` then restart `bench start`.
- Background workers / scheduler / socketio are launched by `bench start` (via
  Honcho). The `web` process is the gunicorn server on port 8000.
- VS Code debugger configs live in `development/.vscode/launch.json` (came from
  `vscode-example/`). Use them to attach the Python debugger to bench processes.
- Container names: this preset uses the default names (`devcontainer-frappe-1` etc.).
  If you also run other bench setups on the same machine, edit
  `.devcontainer/docker-compose.yml` to give them suffixed names and different host
  ports before rebuilding.

---

## Troubleshooting

| Symptom | Likely fix |
|---------|------------|
| `Permission denied (publickey)` when installer tries to clone `ssc_custom` | SSH key missing or not on GitHub. Redo step 1. |
| Port `8000` / `3307` / `9000` already in use on host | Edit `.devcontainer/docker-compose.yml` and shift the `ports:` entries (e.g. `8010-8015:8000-8005`), then **Dev Containers: Rebuild Container**. |
| `bench start` errors after a recent pull / `bench update` | Try `bench setup requirements` then `bench start` again. If a migration is the culprit: `bench --site development.localhost migrate`. |
| `~/.ssh` not visible inside container | Confirm the `mounts` section in `.devcontainer/devcontainer.json` still points to your host home, then **Rebuild Container**. |
| Want a fresh start | From host: `docker compose -f .devcontainer/docker-compose.yml down -v` (the `-v` wipes the mariadb volume — you'll lose the site DB), then reopen in container and re-run the installer. |
| MariaDB connection from host (DBeaver, etc.) | Connect to `localhost:3307`, user `root`, password `123`. |

---

## What's where in this repo

- `devcontainer-ssc/` — committed team preset; copied to `.devcontainer/` on setup.
- `development/apps-ssc.json` — apps list consumed by `installer.py`.
- `development/installer.py` — automates `bench init` + site creation.
- `development/vscode-example/` — VS Code launch/settings/tasks templates.
- `.devcontainer/`, `development/.vscode/`, `development/frappe-bench/` — local-only,
  gitignored. Never committed.

If you need to change something everyone uses (e.g. add a new app, bump versions),
edit `devcontainer-ssc/` or `development/apps-ssc.json` and open a PR.
