# Pi-hole over Tailscale (Docker)

Run **Pi-hole** privately over your **Tailscale** tailnet. 
No LAN ports exposed. All access (DNS + Web UI) happens over your 100.x tailnet IP. 
I personally like composing on Dockge, but do as you wish. 

## What you get

* Pi-hole reachable at `http://<tailnet-ip>/` (e.g., `http://100.115.70.33/`)
* DNS for your tailnet via Pi-hole (ad-blocking, logs, custom lists)
* Persistent state across restarts (both Pi-hole and Tailscale)

---

## Requirements

* Docker & Docker Compose
* A Tailscale account + an **auth key** with reusable/ephemeral OK
* Optional: Dockge/Portainer/etc. (works the same)

---

## Quick start

### 1) Create folders

```bash
mkdir -p ./tailscale ./etc-pihole ./etc-dnsmasq.d
```

### 2) `.env` file (recommended)

```bash
# .env
TS_AUTH_KEY=tskey-xxxxxxxxxxxxxxxxxxxx
WEBPASSWORD=change-me    # you can change from the UI later
TZ=UTC
```

### 3) `docker-compose.yml`

> Default below uses **TUN mode** (fast, standard). See “Userspace mode” further down if you prefer no kernel caps.

```yaml
services:
  pihole-tailscale:
    image: tailscale/tailscale:latest
    hostname: pihole
    cap_add:
      - NET_ADMIN
      - NET_RAW
    devices:
      - /dev/net/tun
    environment:
      - TS_AUTH_KEY=${TS_AUTH_KEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_HOSTNAME=pihole
      - TS_ACCEPT_DNS=false        # don't rewrite resolv.conf inside container
    volumes:
      - ./tailscale:/var/lib/tailscale

  pihole:
    image: pihole/pihole:latest
    # share the exact same network namespace as tailscale
    network_mode: service:pihole-tailscale
    depends_on:
      - pihole-tailscale
    environment:
      - TZ=${TZ}
      - WEBPASSWORD=${WEBPASSWORD}
      - DNSMASQ_LISTENING=all      # listen broadly (works in both tun & userspace)
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped

networks: {}
```

Bring it up:

```bash
docker compose up -d
```

### 4) Find the tailnet IP

```bash
# show the 100.x address Tailscale gave this stack
docker compose exec pihole-tailscale tailscale ip -4
# example output: 100.115.70.33
```

Open the web UI at `http://<that-ip>/` and log in with your `WEBPASSWORD`.

---

## Configure Pi-hole + Tailscale DNS

1. **Pi-hole**: Settings → **DNS** → *Interface listening behavior* →
   **Listen on all interfaces, permit all origins** → Save.

2. **Tailscale Admin**: DNS → **Add nameserver** → your Pi-hole tailnet IP (e.g. `100.115.70.33`).
   Optionally enable **Override local DNS** globally, or turn on **Use tailnet DNS** per-device in the Tailscale app.

3. **Test from a tailnet device**

   ```bash
   # mac/linux
   dig @100.115.70.33 example.com +short
   # windows
   nslookup example.com 100.115.70.33
   ```

   You should also see queries appear in Pi-hole → **Query Log**.

---

## Userspace mode (no /dev/net/tun, fewer caps)

If you can’t/won’t use `NET_ADMIN`/`/dev/net/tun`, switch the Tailscale service to userspace:

```yaml
  pihole-tailscale:
    image: tailscale/tailscale:latest
    hostname: pihole
    environment:
      - TS_AUTH_KEY=${TS_AUTH_KEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_HOSTNAME=pihole
      - TS_ACCEPT_DNS=false
      - TS_TUN_MODE=userspace-networking   # 👈 add this
    volumes:
      - ./tailscale:/var/lib/tailscale
    # remove cap_add & devices in this mode
```

Keep `DNSMASQ_LISTENING=all` on Pi-hole. Everything else stays the same.

---

## Common commands

* **Reset the web password**

  ```bash
  docker compose exec pihole pihole setpassword 'new-strong-password'
  ```

  (Run without an argument to be prompted interactively.)

* **Update Pi-hole blocklists (“gravity”)**

  ```bash
  docker compose exec pihole pihole -g
  ```

* **See the tailnet IP again**

  ```bash
  docker compose exec pihole-tailscale tailscale ip -4
  ```

* **View logs**

  ```bash
  docker compose logs -f pihole
  docker compose logs -f pihole-tailscale
  ```

---

## Troubleshooting

* **Can’t reach the web UI?**
  Make sure you’re using the 100.x tailnet IP from `tailscale ip -4`. No host ports are published.

* **Login fails**
  Run `pihole setpassword` inside the `pihole` service (see above). Then refresh the login page.

* **No queries showing up**
  Confirm your device’s Tailscale app has **Use tailnet DNS** enabled, and that the tailnet DNS server in admin console is your Pi-hole IP.

* **Pi-hole says “only permit origins” is required**
  Keep “Listen on all interfaces, permit all origins” enabled. Because access is only via Tailscale, it stays private to your tailnet.

---

## Notes

* Do **not** publish ports on the host; Tailscale is your private network edge.
* State is persisted in `./tailscale`, `./etc-pihole`, `./etc-dnsmasq.d`.
* If you later want the node to accept advertised routes or be an exit node, add the appropriate `TS_EXTRA_ARGS` to the Tailscale service (outside the scope of this minimal setup).

---

Happy blocking ✨ If you want a one-liner badge or a diagram for the repo, say the word and I’ll add it.
