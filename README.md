# Bridge Music Server: self-hosting

Run your own Bridge Music Server with Docker. Purchases from the Bridge
Music marketplace land in your music folder automatically, and you can
play them in the server's web UI, the Bridge Music iOS app or any
Subsonic-compatible app.

This repository has only what you need to install the server: a
`docker-compose.yml`, an `.env.example`, and the open-source notices.
The image is `ghcr.io/k-pet/bridge-server`.

When you finish this guide you'll have:

- a server reachable at your own HTTPS address
- the server linked to your Bridge Music account
- purchases delivered to `/data/music` and playable right away

---

## 1. Prerequisites

| Item | Why |
|------|-----|
| Docker 24 or later with Compose v2 (or Podman) | The server is one container |
| A Linux or macOS host with at least 2 GB RAM and enough disk for your music | Your library lives on this disk |
| A Bridge Music account | You sign in with it to link the server |
| A way to reach port 8888 at a public HTTPS address (recommended) | So purchases are pushed to you the moment they clear |

**Can't expose the server publicly?** It still works. The server checks
with the marketplace every 15 minutes for anything that wasn't pushed
to it, so purchases arrive a little later. There is no setting to
change.

---

## 2. Configure `.env`

Clone or download this repository onto the host, then:

```bash
cp .env.example .env
$EDITOR .env
```

Set these lines:

| Variable | What it is |
|----------|------------|
| `MUSIC_DIR` | Where your music lives on disk. Defaults to `./data/music`. |
| `BRIDGE_LABEL` | A friendly name for the server, such as "Living Room". |
| `BRIDGE_EXTERNAL_URL` | The public HTTPS address from step 3. |

**Bringing an existing music library?** Also set `PUID` and `PGID`. The
container runs unprivileged (uid 1000 by default) and on boot takes
ownership of anything under `/data` it doesn't already own. On a normal
disk that rewrites your music files' ownership to 1000. On a share it
can't chown (read-only, NFS or SMB with root squash), playback works
but imports, deletes and tag edits fail. Setting `PUID`/`PGID` to the
folder's current owner avoids both:

```bash
stat -c '%u:%g' /path/to/your/music
```

Optional: `BRIDGE_STORAGE_LIMIT_BYTES` caps how large the library may
grow. Leave it unset to let the disk be the limit. `BRIDGE_CPUS`,
`BRIDGE_MEMORY` and `BRIDGE_PIDS` set the container's resource limits;
lower them if you run several containers on one host.

Everything else sets itself up. The server's identity and delivery
secret are generated on first boot and saved in
`/data/bridge/credentials.json`.

> Never commit `.env` to version control, and keep it at mode 0600.

---

## 3. Put the server on a public HTTPS address

Pick one.

### Your own domain with a reverse proxy

1. Point a DNS record at the host (`music.example.com → 1.2.3.4`).
2. Terminate TLS in front of the container. Caddy is the simplest:

   ```caddyfile
   music.example.com {
     reverse_proxy 127.0.0.1:8888
   }
   ```

   nginx with Let's Encrypt works too.
3. Set `BRIDGE_EXTERNAL_URL=https://music.example.com` in `.env`.

### Cloudflare Tunnel

No DNS records, port forwarding or firewall changes.

```bash
cloudflared tunnel login
cloudflared tunnel create bridge
cloudflared tunnel run --url http://localhost:8888 bridge
```

Put the tunnel's HTTPS address in `BRIDGE_EXTERNAL_URL`. Use a named
tunnel: a quick `trycloudflare.com` address changes every time
`cloudflared` restarts, and you would have to re-link each time.

Never expose port 8888 over plain HTTP.

---

## 4. Start the server

```bash
docker compose up -d
docker compose logs -f bridge-music
```

Wait for `bridge server starting port=8888`, then check it:

```bash
curl -s http://localhost:8888/api/health
# {"status":"ok"}

curl -s "https://music.example.com/api/health"
# {"status":"ok"}   (if this fails, step 3 isn't finished)
```

---

## 5. Sign in and link the server

**Link the server before you share its address.** Until someone links
it, the server serves nothing except the link screen, and the first
account to press **Link** becomes its owner.

1. Open your server's address in a browser.
2. **Sign in** with your Bridge Music account, or **Sign up** if you
   don't have one yet.
3. Pick a username if asked.
4. Check the server name, address and account shown, then press
   **Link this server**. If you're signed in as the wrong account, use
   **Sign in as someone else** on the same screen.
5. You'll see "Server linked". Continue to your library.

The iOS app picks up the linked server automatically the next time you
sign in there with the same account.

### Adding other people

As the owner, go to **Settings → People** and add someone by their
exact Bridge Music username or email address.

- **Admin**: can import, edit and delete music and manage people.
- **Listener**: can browse and stream. This is the default.

Everyone gets their own playlists, favourites and play counts. Removing
someone deletes those. Purchases made by a listener go to the server
*they* own, not to yours.

### Navidrome and Subsonic apps

The server includes Navidrome. Its web UI and Subsonic API are at
`https://music.example.com/navidrome`. Use that address (ending in
`/navidrome`) in Subsonic apps. The admin username and password are in
**Settings → Navidrome admin → Show credentials**.

---

## 6. Updating

**Back up Navidrome's data first.** A newer Navidrome upgrades its
database on first start, and an older image can't open it afterwards.

```bash
mkdir -p backups
docker compose stop
docker run --rm -v bridge-navidrome:/src:ro -v "$PWD/backups:/dst" alpine \
  tar czf "/dst/navidrome-$(date +%Y%m%d).tgz" -C /src .
```

Then update:

```bash
docker compose pull
docker compose up -d
```

To roll back, stop the stack, restore that backup into the
`bridge-navidrome` volume, then start the older image.

To pin a version, change `image:` in `docker-compose.yml` from `:latest`
to a release tag such as `ghcr.io/k-pet/bridge-server:1.2.3`.

---

## 7. Backups

Back up three things:

| What | Where | Why |
|------|-------|-----|
| Your music | `MUSIC_DIR` on the host | Your library |
| `bridge-data` volume | `/data/bridge` | The server's identity, delivery secret, owner and Navidrome admin login |
| `bridge-navidrome` volume | `/data/navidrome` | Playlists, favourites, play counts |

Snapshot the named volumes with:

```bash
mkdir -p backups
docker compose stop
for v in bridge-data bridge-navidrome; do
  docker run --rm -v "$v:/src:ro" -v "$PWD/backups:/dst" alpine \
    tar czf "/dst/$v-$(date +%Y%m%d).tgz" -C /src .
done
docker compose start
```

To restore one, stop the stack and unpack it into the volume:

```bash
docker run --rm -v bridge-data:/dst -v "$PWD/backups:/src" alpine \
  sh -c 'rm -rf /dst/* && tar xzf /src/bridge-data-YYYYMMDD.tgz -C /dst'
```

Losing `bridge-data` means the server gets a new identity and you have
to link it again. You can also download your whole library as a zip
from **Settings → Your library**.

---

## 8. Troubleshooting

| Symptom | What to check |
|---------|---------------|
| The container keeps restarting | `docker compose logs bridge-music`. `WARNING could not chown` means set `PUID`/`PGID` (step 2). |
| The public health check fails | Your reverse proxy or tunnel isn't forwarding to port 8888, or `BRIDGE_EXTERNAL_URL` is wrong. |
| No link step appears after signing in | `BRIDGE_EXTERNAL_URL` isn't reaching the container: `docker compose exec bridge-music env \| grep BRIDGE_EXTERNAL_URL` |
| "This server is already linked to another account" | Someone else linked it first. The owner must unlink it (iOS app: Self Hosted → Unlink) before you can link it. |
| Everything suddenly shows the link screen again | The server was unlinked. Sign in as the owner and press **Link** again. |
| Purchases arrive late | The server isn't reachable from the internet, so it relies on the 15-minute check. Fix step 3 for instant delivery. |
| Deliveries rejected as `webhook too old` | The host clock is off. Enable time sync (`timedatectl set-ntp true`). |
| Imports, deletes or tag edits fail but playback works | The music folder isn't writable by the container. Set `PUID`/`PGID` to its owner. |
| A Subsonic app says the password is wrong | The server address must end in `/navidrome`. |
| The library is empty after a delivery | Restart the container. If it persists, open an issue with the logs. |

---

## Licences

The Bridge server itself is proprietary. The image also includes
open-source software, including Navidrome (GPL-3.0), s6-overlay (ISC) and
FFmpeg. See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) and
[licenses/](licenses/). A running server shows the same notices under
**Settings → About → Open-source licences**.

Each release of this repository has the source archive of the Navidrome
version in that image attached. To request source for anything else in
the image, [open an issue](https://github.com/k-pet/bridge-server-selfhost/issues).
