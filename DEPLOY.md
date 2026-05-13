# Deploying p2pool.org

This site is a small static page. Two hosting options below — pick one. The site files don't change between them; only DNS and the serving mechanism differ.

- **Option A: GitHub Pages** (recommended) — free, no server to maintain, source and hosting in one place.
- **Option B: Self-hosted DigitalOcean droplet** — sovereignty over hosting at the cost of ~$4/mo and a server to keep patched.

---

# Option A: GitHub Pages

## 1. Push the repo

If you haven't created the GitHub repo yet, from this project directory:

```bash
git init
git add .
git commit -m "Initial steward page for p2pool.org"
gh repo create treib-holdings/p2pool.org --public --source=. --push
```

(Or via the GitHub web UI: create a new public repo named `p2pool.org` under `treib-holdings`, then add it as the `origin` remote and push.)

## 2. Enable Pages

In the repo on github.com: **Settings → Pages**:

- **Source:** Deploy from a branch
- **Branch:** `main` / `/ (root)`
- **Save**

Within a minute or two, the site will be live at `https://treib-holdings.github.io/p2pool.org`.

## 3. Set the custom domain

Still in **Settings → Pages**:

- **Custom domain:** `p2pool.org`
- **Save**

(The `CNAME` file in the repo already contains `p2pool.org`, so this just confirms it.)

## 4. Configure DNS

At your registrar for `p2pool.org`, set the following records. (Apex domains on GitHub Pages need four `A` records and four `AAAA` records pointing at GitHub's anycast IPs.)

| Type   | Host  | Value                          | TTL |
|--------|-------|--------------------------------|-----|
| A      | `@`   | `185.199.108.153`              | 300 |
| A      | `@`   | `185.199.109.153`              | 300 |
| A      | `@`   | `185.199.110.153`              | 300 |
| A      | `@`   | `185.199.111.153`              | 300 |
| AAAA   | `@`   | `2606:50c0:8000::153`          | 300 |
| AAAA   | `@`   | `2606:50c0:8001::153`          | 300 |
| AAAA   | `@`   | `2606:50c0:8002::153`          | 300 |
| AAAA   | `@`   | `2606:50c0:8003::153`          | 300 |
| CNAME  | `www` | `treib-holdings.github.io.`    | 300 |

Verify these IPs are still current against GitHub's docs before applying — they've been stable for years, but worth checking: <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site>

Verify propagation from your laptop:

```bash
dig p2pool.org +short
dig www.p2pool.org +short
```

## 5. Enable HTTPS

Back in **Settings → Pages**, wait for the green **"DNS check successful"** message (10–60 minutes after DNS propagates), then tick **Enforce HTTPS**. GitHub provisions a Let's Encrypt cert. First-issuance can take another 10–60 minutes.

## 6. Verify

```bash
curl -I https://p2pool.org
curl -I https://www.p2pool.org   # should redirect to p2pool.org
```

Then open <https://p2pool.org> in a browser. Done.

## Updating the site

Edit `index.html` or `style.css` locally, then:

```bash
git add -p
git commit -m "..."
git push
```

GitHub redeploys within ~30 seconds.

---

# Option B: Self-hosted DigitalOcean droplet

If you prefer to host the site yourself — e.g., to avoid relying on GitHub for delivery — the droplet path is preserved here. Uses Caddy for auto-HTTPS via Let's Encrypt.

## What you'll provision

- 1 × DigitalOcean droplet, smallest plan (~$4/mo, Ubuntu 24.04 LTS)
- DNS `A` and `AAAA` records for `p2pool.org` and `www.p2pool.org`
- Caddy 2.x web server

Total cost: ~$48/year plus domain renewal. The droplet idles at well under 1% CPU and ~80 MB RAM serving this site.

## 1. Create the droplet

In the DigitalOcean console:

- **Region:** closest to your expected audience (e.g. NYC1, SFO3, AMS3)
- **Image:** Ubuntu 24.04 (LTS) x64
- **Size:** Basic / Regular SSD / **$4/mo** (512 MB / 10 GB / 1 vCPU)
- **Authentication:** SSH key (paste in the public key from your laptop; no password auth)
- **Hostname:** `p2pool-www`
- **Enable IPv6:** yes

Note the droplet's IPv4 and IPv6 addresses once it boots.

## 2. Point DNS at the droplet

At your registrar for `p2pool.org`, create these records (replacing any GitHub Pages records first):

| Type   | Host  | Value           | TTL |
|--------|-------|-----------------|-----|
| A      | `@`   | _droplet IPv4_  | 300 |
| A      | `www` | _droplet IPv4_  | 300 |
| AAAA   | `@`   | _droplet IPv6_  | 300 |
| AAAA   | `www` | _droplet IPv6_  | 300 |

Verify propagation:

```bash
dig p2pool.org +short
dig www.p2pool.org +short
```

> **Important:** complete DNS before step 6. Caddy requests a Let's Encrypt cert on first start, and that ACME challenge needs DNS resolving to the droplet.

## 3. Initial server setup

SSH in as root:

```bash
ssh root@<droplet-IPv4>
```

Update packages and create a non-root user:

```bash
apt update && apt upgrade -y
adduser deploy
usermod -aG sudo deploy
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```

Set up a basic firewall:

```bash
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

Disable root SSH and password auth (optional but recommended):

```bash
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh
```

From now on, log in as the deploy user:

```bash
exit
ssh deploy@<droplet-IPv4>
```

## 4. Install Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

Caddy installs as a systemd service and starts on boot.

## 5. Upload the site files

From your laptop, in this project directory:

```bash
scp index.html style.css Caddyfile deploy@<droplet-IPv4>:/tmp/
```

Then on the droplet:

```bash
ssh deploy@<droplet-IPv4>
sudo mkdir -p /var/www/p2pool
sudo cp /tmp/index.html /tmp/style.css /var/www/p2pool/
sudo chown -R caddy:caddy /var/www/p2pool
sudo cp /tmp/Caddyfile /etc/caddy/Caddyfile
```

## 6. Validate and start Caddy

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Watch the logs while it provisions Let's Encrypt certs:

```bash
sudo journalctl -u caddy -f
```

You should see lines like `certificate obtained successfully` for both `p2pool.org` and `www.p2pool.org`. First cert issuance takes 10–60 seconds.

## 7. Verify

From your laptop:

```bash
curl -I https://p2pool.org
curl -I https://www.p2pool.org   # should 308 → p2pool.org
```

Open <https://p2pool.org> in a browser. You should see a valid cert padlock and the page render.

## Updating the droplet site

Whenever you edit `index.html` or `style.css` locally:

```bash
scp index.html style.css deploy@<droplet-IPv4>:/tmp/
ssh deploy@<droplet-IPv4> 'sudo install -o caddy -g caddy -m 644 /tmp/index.html /tmp/style.css -t /var/www/p2pool/'
```

No Caddy reload needed — static files are read on each request.

If you change `Caddyfile`:

```bash
scp Caddyfile deploy@<droplet-IPv4>:/tmp/
ssh deploy@<droplet-IPv4> 'sudo cp /tmp/Caddyfile /etc/caddy/Caddyfile && sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy'
```

## Droplet maintenance

Set up automatic security updates so this is hands-off:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

Otherwise, once a month or so:

```bash
ssh deploy@<droplet-IPv4>
sudo apt update && sudo apt upgrade -y
sudo reboot   # only if kernel was updated
```

Caddy will renew Let's Encrypt certs automatically (every ~60 days). No action needed from you.
