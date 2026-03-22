# Deployment guide

## Part 1 — Manual steps (what each piece does)

### Local Docker build and run

| Command | Purpose |
|--------|---------|
| `docker build -t hello:local .` | Build the image from the `Dockerfile` in the current directory. Tag it `hello:local` (no space in the tag). |
| `docker run -d -p 8080:8080 hello:local` | Run a container in the background (`-d`), map host port **8080** to container **8080** (`-p`). |

### Push to Docker Hub

| Step | Purpose |
|------|---------|
| `docker login` | Authenticate to Docker Hub (username + access token or password). |
| `docker tag hello:local YOUR_DOCKERHUB_USER/hello:latest` | Give the local image the name Docker Hub expects: `username/repository:tag`. |
| `docker push YOUR_DOCKERHUB_USER/hello:latest` | Upload the image so EC2 (or anyone with access) can `docker pull` it. |

### EC2 — first-time setup (Amazon Linux style)

1. **SSH key pair** — You download a `.pem` private key; AWS keeps the public half on the instance. Only someone with the `.pem` can SSH as `ec2-user` (for Amazon Linux).
2. **Connect** — From the folder that contains the key:
   ```bash
   ssh -i /path/to/demo-keypair.pem ec2-user@YOUR_EC2_PUBLIC_IP
   ```
   (`ssh -i` selects the identity file; there must be a space between `-i` and the path.)
3. **`sudo yum update -y`** — `sudo` runs as root; `yum` is the package manager on RHEL/Amazon Linux; `-y` assumes “yes” to prompts.
4. **Install Docker** — e.g. `sudo yum install docker -y` (exact package name can vary by AMI; some AMIs use `dnf`).
5. **`docker --version`** — Confirms the CLI is installed.
6. **`sudo systemctl start docker`** — Starts the Docker daemon now.
7. **`sudo systemctl enable docker`** — Enables Docker to start automatically after a reboot.
8. **`sudo usermod -aG docker ec2-user`** — Adds `ec2-user` to the `docker` group so you can run `docker` without `sudo` after you log out and back in.
9. **`exit`** then SSH again — Group membership applies to new sessions.
10. **Run your app** — `docker pull YOUR_DOCKERHUB_USER/hello:latest`, then `docker run -d -p 8080:8080 YOUR_DOCKERHUB_USER/hello:latest`.
11. **Health check** — Open `http://YOUR_EC2_PUBLIC_IP:8080/health/` in a browser (path depends on your app).

### Dockerfile (multi-stage) — recap

- **Build stage:** Maven compiles the app inside the image (no need to copy `target/` from your laptop).
- **Run stage:** Only the JRE and the fat JAR run in production.

---

## Part 2 — Fully automated (GitHub Actions → Docker Hub → EC2)

The workflow runs **when you push to `main`** and can also be started **by hand**:

- GitHub repo → **Actions** → workflow **Docker build-> push-> deploy** → **Run workflow** → branch **main** → **Run workflow**.

That uses the **latest commit** on the branch you select (usually `main`) — same steps as an automatic deploy.

On each run, `.github/workflows/deploy.yml`:

1. Checks out the repo.
2. Logs in to Docker Hub.
3. Builds the image with `docker build` and tags it as `YOUR_USER/hello:latest`.
4. Pushes that image to Docker Hub.
5. SSHs to EC2 and runs: stop/remove old container, `docker pull`, `docker run` with port **8080**.

### GitHub repository secrets (Settings → Secrets and variables → Actions)

| Secret | Meaning |
|--------|---------|
| `DOCKERHUB_USERNAME` | Docker Hub username (same as in image name). |
| `DOCKERHUB_TOKEN` | Docker Hub **access token** (recommended), not your account password. |
| `EC2_HOST` | EC2 public IP or DNS, e.g. `13.51.121.216`. |
| `EC2_USERNAME` | SSH user, usually `ec2-user` on Amazon Linux. |
| `EC2_KEY` | **Full contents** of the private `.pem` file (paste from `cat demo-keypair.pem`), including `BEGIN`/`END` lines. |

After secrets are set, deploy by **pushing to `main`** or using **Run workflow** — no local `docker build` / `push` / SSH for routine deploys.

### EC2 prerequisites for automation

- Docker installed and running (same as manual setup).
- Security group allows **inbound TCP 8080** (and **22** from GitHub Actions IPs is tricky; often people use a fixed runner IP or allow SSH broadly temporarily — tighten for production).
- The instance must accept SSH using the same key stored in `EC2_KEY`.

---

## Security notes

- **Never commit** `.pem` files, AWS passwords, or Docker tokens to git.
- If any password or key was shared in plain text, **rotate** it in AWS / Docker Hub and generate a new key pair if needed.
- Prefer **Docker Hub access tokens** over account passwords for `DOCKERHUB_TOKEN`.
