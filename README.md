# BookStore (Static) — Containerized + CI/CD (GitHub Actions + GHCR + Remote Docker)

This document contains a ready-to-use project layout, files, and GitHub Actions workflows to: **containerize** your static HTML+CSS Book Store, **build** and **publish** a Docker image to GitHub Container Registry (GHCR), and **deploy** to a remote server via SSH + Docker Compose. It also includes an optional GitHub Pages workflow as a static fallback.

"C:\Users\sasik\OneDrive\Pictures\Screenshots\Screenshot 2025-11-18 120611.png"
## ✅ What you get here

* `Dockerfile` (nginx static server)
* `docker-compose.yml` (for local testing or remote deployment)
* `nginx.conf` (simple static site config)
* Minimal `index.html` + `css/styles.css` (sample)
* `.github/workflows/ci.yml` — Build image, run checks, push to GHCR
* `.github/workflows/deploy.yml` — SSH → pull image on remote host and restart container via docker-compose
* `.github/workflows/pages.yml` — optional GitHub Pages deploy of static site
* `README.md` with setup & secrets instructions

---

## Repo structure (recommended)

```
bookstore-static/
├─ index.html
├─ books.html
├─ css/
│  └─ styles.css
├─ images/
│  └─ ...
├─ Dockerfile
├─ docker-compose.yml
├─ nginx.conf
├─ README.md
└─ .github/
   └─ workflows/
      ├─ ci.yml
      ├─ deploy.yml
      └─ pages.yml
```

---

## 1) Dockerfile (nginx)

```dockerfile
# Dockerfile: serve static HTML/CSS with nginx
FROM nginx:stable-alpine

# Remove default nginx content
RUN rm -rf /usr/share/nginx/html/*

# Copy site content
COPY ./ /usr/share/nginx/html/

# Copy custom nginx config (optional)
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

---

## 2) docker-compose.yml (local & remote)

```yaml
version: '3.8'
services:
  bookstore:
    build: .
    image: ghcr.io/${{ github.repository_owner }}/bookstore-static:latest
    ports:
      - "80:80"
    restart: unless-stopped
```

Notes:

* For **local testing** you can `docker-compose up --build` and open `http://localhost`.
* On a **remote server**, the workflow pulls the image and uses a simple `docker-compose.yml` (same file or separate on remote host).

---

## 3) nginx.conf (simple)

```nginx
server {
  listen 80;
  server_name _;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }

  # optional: configure gzip, caching headers, etc.
}
```

---

## 4) Minimal index.html + css

`index.html`

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>BookStore — Static</title>
  <link rel="stylesheet" href="css/styles.css">
</head>
<body>
  <header class="site-header">
    <h1>BookStore</h1>
    <nav>
      <a href="index.html">Home</a>
      <a href="books.html">Books</a>
      <a href="about.html">About</a>
      <a href="contact.html">Contact</a>
    </nav>
  </header>

  <main>
    <section class="hero">
      <h2>Welcome to the BookStore</h2>
      <p>Static HTML + CSS demo — containerized and deployable via CI/CD.</p>
    </section>

    <section class="books-grid">
      <article class="book-card">
        <img src="images/book-placeholder.png" alt="Book">
        <h3>Sample Book</h3>
        <p>Author • ₹299</p>
      </article>
      <!-- add more cards -->
    </section>
  </main>

  <footer>
    <p>© BookStore</p>
  </footer>
</body>
</html>
```

`css/styles.css`

```css
*{box-sizing:border-box}
body{font-family:Arial,Helvetica,sans-serif;margin:0;color:#222}
.site-header{display:flex;justify-content:space-between;align-items:center;padding:12px 20px;background:#f7f7f7}
.site-header nav a{margin-left:12px;text-decoration:none;color:#333}
.hero{padding:40px 20px;background:linear-gradient(90deg,#f0f4ff,#ffffff)}
.books-grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(180px,1fr));gap:16px;padding:20px}
.book-card{padding:12px;border:1px solid #eaeaea;border-radius:8px;text-align:center}
footer{padding:12px;text-align:center;background:#fafafa}
```

---

## 5) GitHub Actions — CI: `ci.yml` (build image, run simple checks, push to GHCR)

Create `.github/workflows/ci.yml`:

```yaml
name: CI / Build & Push Image

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU (for multi-arch, optional)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/bookstore-static:latest
            ghcr.io/${{ github.repository_owner }}/bookstore-static:${{ github.sha }}

      - name: Verify site files exist
        run: |
          if [ ! -f index.html ]; then echo "index.html missing" && exit 1; fi

      - name: Optional: Run HTML proofer (basic check)
        run: |
          echo "Skipping advanced checks — add linters if you want."
```

Notes:

* Workflow builds and pushes the Docker image to GHCR using the repository's `GITHUB_TOKEN` (which has package:write permission). If you prefer using a personal token, create `GHCR_TOKEN` secret.
* Tags: `latest` and commit SHA.

---

## 6) GitHub Actions — Deploy to remote host via SSH: `deploy.yml`

This workflow runs on push to `main` and SSHs into your remote server (VM) to pull the newly pushed image and restart the container using `docker-compose`.

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Remote Server

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install ssh client
        run: sudo apt-get update && sudo apt-get install -y openssh-client

      - name: Setup SSH key
        uses: webfactory/ssh-agent@v0.8.1
        with:
          ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}

      - name: Copy docker-compose to remote (optional)
        run: |
          scp -o StrictHostKeyChecking=no docker-compose.yml ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:${{ secrets.REMOTE_DIR }}/docker-compose.yml

      - name: Pull image & restart (remote)
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} "cd ${{ secrets.REMOTE_DIR }} && docker-compose pull && docker-compose up -d --remove-orphans"
```

### Required repository secrets (Settings → Secrets → Actions)

* `DEPLOY_SSH_KEY` — Private SSH key (PEM format) for the remote server user that has permission to run docker/docker-compose
* `SSH_HOST` — e.g. `203.0.113.45` or `example.com`
* `SSH_USER` — remote username (e.g. `ubuntu`)
* `REMOTE_DIR` — path on remote host where `docker-compose.yml` is (e.g. `/home/ubuntu/bookstore`)

### Remote host prerequisites

On the remote host:

* Docker & Docker Compose installed
* `docker-compose.yml` present at `${{ REMOTE_DIR }}` (workflow copies it)
* Optionally login to GHCR on the host or use `docker login ghcr.io` with PAT so `docker-compose pull` can access the image. You can store GHCR credentials in the remote host's `~/.docker/config.json`, or pass the token as an environment variable in the compose file and use `docker login` command before pull (scriptable).

---

## 7) Optional: GitHub Pages workflow (static fallback): `pages.yml`

If you want the static files deployed to GitHub Pages as well, create `.github/workflows/pages.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]

jobs:
  deploy-pages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./
```

This publishes your repo root files to the `gh-pages` branch so GitHub Pages can serve them.

---

## 8) README.md (short instructions)

```markdown
# BookStore (Static) — Containerized + CI/CD

This repository contains a static BookStore site (HTML + CSS) and CI/CD pipelines to build a Docker image and deploy it.

## How it works
1. Push to `main` → GitHub Actions `ci.yml` builds the Docker image and pushes to GHCR (ghcr.io/${{ github.repository_owner }}/bookstore-static:latest).
2. `deploy.yml` SSHs into your remote host, copies `docker-compose.yml`, pulls the image from GHCR, and restarts the container.
3. `pages.yml` (optional) publishes static files to GitHub Pages.

## Setup (secrets)
Add the following GitHub repository secrets:
- `DEPLOY_SSH_KEY` — Private key for SSH (no passphrase recommended for CI)
- `SSH_HOST` — remote server host
- `SSH_USER` — remote user
- `REMOTE_DIR` — target dir on remote host

Optional: use GHCR PAT if you need more permissions than `GITHUB_TOKEN`.

## Local dev
```

# build and run locally

docker-compose up --build

# open [http://localhost](http://localhost)

```

## Remote host prerequisites
- Docker & docker-compose
- Allow SSH key login
- (Optional) docker login to ghcr.io with PAT on the remote host
```

---

## 9) Security & good practices

* Use a deploy-only SSH key with limited permissions.
* Protect the `main` branch and require PR reviews before merging.
* Rotate credentials (keys & tokens) periodically.
* If you prefer not to use SSH remote deploy, consider using Render, DigitalOcean App Platform, AWS ECS/Fargate, or GitHub Actions self-hosted runners — they each have their own integration steps.

---

## 10) Want me to generate the actual files for you?

I can create ready-to-copy file contents for any of the files above (Dockerfile, workflows, index.html, etc.) as separate code blocks or a zip. Tell me which files you want first and I’ll paste them ready to copy.

Happy to help — Bujji! 🚀
