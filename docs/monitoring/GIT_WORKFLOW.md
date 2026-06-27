
# Commit and Push

## Pre-Commit Checks

Navigate to the DevOps repository:

```bash
cd humor-memory-game-kubernetes
```

Confirm you are on the correct branch:

```bash
git status

git branch
```

Confirm `.env` is still ignored — run this before every commit:

```bash
git check-ignore -v .env
# Expected: .gitignore:1:.env    .env
```


---

## Copy Documentation Files Into the Repository

If documentation files were drafted on your local machine, transfer them using `scp` from your local terminal:

```bash
# Copy each file
scp ~/"Notes/1 Projects/Devops Projects/Humor Memory Game Project/5.0 Observability/"*.md sre@192.168.30.12:~/humor-memory-game-kubernetes/docs/monitoring
```

Confirm all documentation files are in place on the VM:

```bash
ls ~/humor-memory-game-kubernetes/docs/monitoring
# Expected:
GIT_WORKFLOW.md
MONITORING_METRICS.md
MONITORING_SETUP.md
MONITORING_CUSTOM_DASHBOARDS.md
MONITORING_PROMETHEUS.md
MONITORING_TROUBLESHOOTING.md
MONITORING_DASHBOARDS.md
MONITORING-README.md
README-KUBERENTES.md
```

---

## Verify Full Repository Structure

```bash
find ~/humor-memory-game-kubernetes -type f | grep -v ".git" | sort
```

Expected output:

```
/home/sre/humor-memory-game-kubernetes/docs/kubernetes/BUG_002_BACKEND_PROBE_DEATH_SPIRAL.md
/home/sre/humor-memory-game-kubernetes/docs/kubernetes/DEPLOYING_THE_APPLICATION.md
/home/sre/humor-memory-game-kubernetes/docs/kubernetes/INSTALLING_K3S.md
/home/sre/humor-memory-game-kubernetes/docs/kubernetes/INTRODUCTION_TO_KUBERNETES.md
/home/sre/humor-memory-game-kubernetes/docs/kubernetes/PREPARING_THE_RASPBERRY_PIS.md
/home/sre/humor-memory-game-kubernetes/docs/kubernetes/SRE_SETUP_GUIDE.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/GIT_WORKFLOW.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING_CUSTOM_DASHBOARDS.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING_DASHBOARDS.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING_METRICS.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING_PROMETHEUS.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING-README.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING_SETUP.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/MONITORING_TROUBLESHOOTING.md
/home/sre/humor-memory-game-kubernetes/docs/monitoring/README-KUBERENTES.md
/home/sre/humor-memory-game-kubernetes/env-patch.yaml
/home/sre/humor-memory-game-kubernetes/game_check.sh
/home/sre/humor-memory-game-kubernetes/manifests/01-namespace.yaml
/home/sre/humor-memory-game-kubernetes/manifests/03-configmap.yaml
/home/sre/humor-memory-game-kubernetes/manifests/04-pvc-postgres.yaml
/home/sre/humor-memory-game-kubernetes/manifests/05-pvc-redis.yaml
/home/sre/humor-memory-game-kubernetes/manifests/06-postgres-deployment.yaml
/home/sre/humor-memory-game-kubernetes/manifests/07-postgres-service.yaml
/home/sre/humor-memory-game-kubernetes/manifests/08-redis-deployment.yaml
/home/sre/humor-memory-game-kubernetes/manifests/09-redis-service.yaml
/home/sre/humor-memory-game-kubernetes/manifests/10-backend-deployment.yaml
/home/sre/humor-memory-game-kubernetes/manifests/11-backend-service.yaml
/home/sre/humor-memory-game-kubernetes/manifests/12-frontend-deployment.yaml
/home/sre/humor-memory-game-kubernetes/manifests/13-frontend-service.yaml
/home/sre/humor-memory-game-kubernetes/manifests/14-ingress.yaml
/home/sre/humor-memory-game-kubernetes/manifests/postgres-init-configmap.yaml
/home/sre/humor-memory-game-kubernetes/monitoring/dashboards/advanced-custom-dashboard.json
/home/sre/humor-memory-game-kubernetes/monitoring/dashboards/comprehensive-dashboard.json
/home/sre/humor-memory-game-kubernetes/monitoring/dashboards/custom-dashboard.json
/home/sre/humor-memory-game-kubernetes/monitoring/humor-service-monitor.yaml
/home/sre/humor-memory-game-kubernetes/monitoring/monitoring-values.yaml
/home/sre/humor-memory-game-kubernetes/monitoring/populate-game-metrics.sh
/home/sre/humor-memory-game-kubernetes/monitoring/prometheus-rbac.yaml
/home/sre/humor-memory-game-kubernetes/README.md
```

> `.env` must **not** appear in this list.

---

## Check Git Status

```bash
git status
```

Expected:

```
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	deleted:    BUG_002_BACKEND_PROBE_DEATH_SPIRAL.md
	deleted:    DEPLOYING_THE_APPLICATION.md
	deleted:    INSTALLING_K3S.md
	deleted:    INTRODUCTION_TO_KUBERNETES.md
	deleted:    PREPARING_THE_RASPBERRY_PIS.md
	deleted:    SRE_SETUP_GUIDE.md
	modified:   ../manifests/08-redis-deployment.yaml
	modified:   ../manifests/11-backend-service.yaml

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	kubernetes/
	monitoring/
	../env-patch.yaml
	../game_check.sh
	../manifests/postgres-init-configmap.yaml
	../monitoring/

no changes added to commit (use "git add" and/or "git commit -a")
```

---

## Stage All Files

```bash
git add .

```

```bash
git status
```

---

## Commit

```bash
git commit -m "add all monitoring docs and yaml files. Create new direcroty for kunbernetes docs"
```

---

## Push to GitHub

```bash
git push origin main
```

Expected output:

```
Enumerating objects: XX, done.
Counting objects: 100% (XX/XX), done.
Compressing objects: 100% (XX/XX), done.
Writing objects: 100% (XX/XX), X.XX KiB | X.XX MiB/s, done.
To git@github.com:<github-username>/humor-memory-game-kubernetes.git
   xxxxxxx..xxxxxxx  main -> main
```

---

## Verify on GitHub

Open the repository in a browser:

```
https://github.com/<github-username>/humor-memory-game-devops
```

Confirm:

- [ ] `README.md` renders automatically on the landing page
- [ ] `docker/` directory visible with `backend/` and `frontend/` subdirectories
- [ ] `docs/` directory visible with all nine documents
- [ ] `game_check.sh` visible in root
- [ ] `docker-compose.yml` visible in root
- [ ] `.env.example` visible in root
- [ ] `.env` is **NOT** visible — git-ignored and absent from the repository

---

## 14. Troubleshooting

### Push rejected — not up to date

```bash
git pull origin main --rebase
git push origin main
```

### Accidentally staged .env

```bash
# Unstage immediately
git reset HEAD .env

# Confirm it is no longer staged
git status
# .env must not appear in the staged files list

# Verify it is in .gitignore
cat .gitignore | grep ".env"
```

### Wrong files committed

```bash
# Undo the last commit — keeps files, unstages them
git reset HEAD~1

# Review what is now unstaged
git status

# Re-add only the correct files and recommit
git add README.md docker/ docs/ ...
git commit -m "correct commit message"
```

### SSH authentication failed on push

```bash
# Test the SSH connection
ssh -T git@github.com

# If it fails, re-add the key to the SSH agent
ssh-add ~/.ssh/id_ed25519

# Retry the push
git push origin main
```

### Unable to register on GitHub
#### Why the "Generate a new SSH key" step is missing

In your previous successful setups, the `gh` tool detected that you had **no SSH keys** on your machine and therefore triggered the **"Generate a new SSH key? Yes/No"** prompt.

**Now, your environment is different:** You have files in your `~/.ssh/` directory (likely the `pi_key` or others you created while troubleshooting earlier). Because the CLI finds _something_ in that folder, it skips the "Generate a new SSH key" prompt to avoid overwriting your existing files, but then fails to link them because they aren't the standard expected keys.

### The Fix: Force the "Clean Slate"

To force the CLI to behave exactly as you remember, you need to hide your existing keys temporarily so the CLI thinks the environment is fresh.

1.**Move your current keys aside:**

```
mkdir ~/.ssh/backup
mv ~/.ssh/id_* ~/.ssh/backup/ 2>/dev/null
mv ~/.ssh/pi_key* ~/.ssh/backup/ 2>/dev/null
```
    
2. **Clear your auth state:**
 
```
gh auth logout
```

3. **Run the process again:**
  
```
gh auth login
```

Because the `~/.ssh/` folder is now empty (or at least doesn't contain standard keys), **the GitHub CLI will finally present you with the "Generate a new SSH key? Yes" prompt** that you are looking for.

### Why this works

By moving the files, you are effectively "tricking" the CLI into thinking this is a brand-new, fresh installation of Linux. Once it generates the new key, it will proceed to upload it to your account, and everything will align with your established documentation.

When completed do this
### 1. Move the keys back

Run this command to move everything from your backup folder back into the main directory:

```
mv ~/.ssh/backup/* ~/.ssh/
```

### 2. Clean up the backup folder

Once you have moved them, you can remove the now-empty backup directory:

```
rmdir ~/.ssh/backup
```

### 3. Restore Permissions

SSH is extremely strict about file permissions. If the permissions were changed during the move, SSH will reject the keys with a "Permissions are too open" error. Reset them to ensure they work correctly:

```
# Set the directory permissions
chmod 700 ~/.ssh

# Set permissions for your standard keys
chmod 600 ~/.ssh/id_*

# Set permission for your specific pi_key
chmod 600 ~/.ssh/pi_key

# Set permissions for the public keys (these can be readable)
chmod 644 ~/.ssh/*.pub
```