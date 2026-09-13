# Changes made to the VPS Deployer bot

1. **`!manage` → sshx button**: already existed and worked; relabeled to
   `🌐 sshx` exactly as requested (was `🌐 SSHX`). It installs sshx on the
   VPS if missing and DMs the sshx URL to the user.

2. **Removed self-deploy (`!deploy`)**: the free/role-based instant deploy
   command and its role/slot config are gone.

3. **Credits + Coupons**:
   - `!addcoupon <code> <amount>` (admin only) — creates a single-use
     coupon worth `<amount>` credits.
   - `!redeem <code>` — any user redeems a coupon for credits.
   - `!balance` — check your credit balance.
   - New `credits` and `coupons` tables added to the database automatically.

4. **`!plans`**: lists 4GB / 8GB / 12GB / 16GB plans (RAM/CPU/Disk/price)
   plus a "Custom — contact an admin" option. Edit the `PLANS` dict near
   the top of `bot.py` to change specs or prices — the starting prices
   are just a placeholder (100 credits per GB) since none were given.

5. **`!buy <plan>`**: spends credits, lets the user pick an OS, and
   deploys the VPS automatically (reuses the old deploy logic, generalized
   to the chosen plan). Credits are only charged after the VPS is
   successfully created — if deployment fails, nothing is charged.

6. **DMs disabled for all commands**: any command used in a DM now
   replies "Commands can't be used in DMs — please head back to the
   server and run this command there instead!" (the requested sentence,
   lightly polished).

7. **`!setpwd <container> <password>`**: lets a user change the root
   password of a VPS *they own* (not the host machine). Requires the
   container name since a user can own more than one VPS.

8. **`!autofix <container>`**: runs a battery of common fixes (broken
   packages, networking, DNS, SSH restart, disk/log cleanup) on a VPS the
   user owns. If any step can't be confirmed, it DMs the main admin
   automatically as an escalation and tells the user to open a ticket if
   the issue persists.

9. **`requirements.txt`** added with all dependencies actually imported
   by `bot.py` (discord.py, python-dotenv, requests, psutil, PyNaCl).

## Things you may want to adjust
- `PLANS` dict (top of `bot.py`) — specs/prices are placeholders.
- Coupon redemption is currently single-use per code, no expiry.
- `!autofix` runs best-effort shell fixes — it can't detect every
  possible error, so it always escalates to the admin's DMs if a step
  fails, per your instructions.

## Railway hosting
Railway can't run the actual LXC/VPS host — it blocks privileged
containers, kernel modules, and nested containerization entirely, which
is exactly what `lxc` commands need. If ALL your nodes are "Dynamic"
(remote, driven over the node HTTP API rather than local `lxc` calls),
Railway *can* host the Discord bot process itself, since at that point
it's just outbound HTTP/Python. If you use "Local" node mode (bot + LXD
on the same box), Railway won't work for that node at all.

### "Invalid token" on Railway
This is a token/config issue, not a Railway-support issue — the bot code
doesn't behave differently on Railway vs. anywhere else. What changed
in this update:
- `DISCORD_TOKEN` (and `PTERODACTYL_APP_API_KEY`) are now automatically
  stripped of whitespace and surrounding quotes on load — the #1 cause
  of "invalid token" on Railway is pasting the value into the Variables
  tab with an extra space, newline, or quote marks still attached.
- Startup now logs the token's length and warns (without ever printing
  the token itself) if it looks malformed — whitespace, quotes, a
  `"Bot "` prefix, or an unusually short length.
- If Discord still rejects it, the error message now lists the actual
  common causes: the token was regenerated in the Developer Portal
  after you copied the old one (this immediately invalidates the old
  one), the Railway variable name isn't exactly `DISCORD_TOKEN`
  (case-sensitive), a leftover local `.env` masking that Railway's own
  variable was never actually set, or the Client Secret/Application ID
  got pasted instead of the Bot Token.
- Added a `Procfile` (`worker: python bot.py`) so Railway runs this as
  a background worker instead of assuming it's a web service waiting
  on a `$PORT` — a bot with no HTTP server can otherwise get flagged
  unhealthy/restarted by Railway's default web healthcheck.

**One more thing to know about Railway specifically:** its filesystem
is ephemeral between deploys unless you attach a persistent Volume.
This bot stores everything (VPS records, credits, coupons, port
forwards) in a local SQLite file (`vps.db`). On Railway, mount a Volume
and point it at the bot's working directory — otherwise a redeploy
will wipe your database.


## New: `!pterodactyl <container>` and `!wings <container>`
Both install into the SAME container the user already owns (per your
choice). Flow:
1. `!pterodactyl <container>` — button opens a modal (domain, admin
   username/email/password). Installs MariaDB, PHP, nginx, the Panel,
   runs migrations, creates the admin account, and issues a real Let's
   Encrypt certificate via certbot for the domain. Forwards a
   Cloudflare-compatible HTTPS port (443/2053/2083/2087/2096/8443 — first
   free one on that node) plus port 80 for the ACME/redirect.
2. `!wings <container>` — requires step 1 done first. Buttons let the
   user pick a public port bucket (`8080` HTTP, or `443`/`8443` HTTPS).
   A modal asks for the Wings domain. The bot then:
   - Installs Docker + the Wings binary + a systemd service.
   - Creates a Location and a Node **directly via the Panel's own
     Application API** (the same officially-documented mechanism the
     Panel's "Create Node" admin page uses) — no manual copy-pasting of
     deploy commands.
   - Runs the official `wings configure --panel-url ... --token ... --node ...`
     command, which is Pterodactyl's own supported way to generate
     `config.yml` — nothing hand-rolled or guessed here.
   - Forwards the chosen host port to the container's Wings port, plus
     an SFTP port.
3. After Wings finishes, buttons offer to install the **Blueprint**
   framework (Panel add-on manager) via its official installer script.

**Requires `PTERODACTYL_APP_API_KEY` in `.env`** for `!wings` to work —
this has to be created once, manually, from inside a Panel (Account
Settings → API Credentials → Create New, permissions Nodes: Read/Write
+ Locations: Read/Write). `!wings` will tell the user clearly if it's
missing rather than failing silently. This is the one piece I couldn't
safely automate end-to-end: Panel has no CLI command to mint this key,
and hand-crafting the token/hash myself would be guessing at internals
that change between Panel versions — using the real documented API
instead means this won't quietly break on a Panel update.

**Known limitation on SSL for Wings:** Wings serves plain HTTP inside
the container in this setup (to avoid a port conflict with the Panel's
own nginx on port 443 in the same container). If you expose it on a
443/8443-style port, set that specific domain's Cloudflare SSL mode to
**Flexible** (HTTPS at the edge, HTTP to origin) — otherwise Cloudflare
will show SSL handshake errors. The Panel's own domain is unaffected —
it gets a real certbot certificate and can use Full/Strict.

**Both commands install a fresh Location per container** (named after
the container) rather than asking the user to pick from existing
locations — keeps things self-contained but means you'll see one
Location per customer in the Panel's admin view.

