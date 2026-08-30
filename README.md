# Reverse Proxy VM Provisioning

Simple Ansible playbook that provisions a reverse proxy VM:

1. Creates a local user with UID 1000
2. Installs additional Python packages via `pip3`
3. Installs and configures NGINX as a reverse proxy for 4 domains
4. Installs and configures `dnf-automatic` for daily unattended updates
5. Installs and runs Uptime Kuma under PM2 (with boot persistence)
6. Limits how many old kernel versions are kept installed
7. Enables compressed zram swap on `/dev/zram0`

Target OS: **CentOS Stream 10** (uses the `yum`/`dnf` package manager, SELinux,
and `firewalld` — all handled below).

## Layout

```
Ansible/
├── ansible.cfg
├── .gitignore
├── inventory/hosts.ini        # generic host alias; real values are vaulted
├── group_vars/
│   ├── all.yml                # variables you'll want to edit
│   └── reverse_proxy/
│       ├── vars.yml           # references vaulted variables
│       └── vault.yml          # encrypted: real host/user/domains
├── site.yml                   # provisioning playbook entrypoint
├── update.yml                 # yum update playbook entrypoint
└── roles/
    ├── base_user/              # creates the UID 1000 user
    ├── pip_packages/           # installs pip3 + extra packages
    ├── uptime_kuma/            # installs + runs Uptime Kuma under PM2
    ├── nginx_proxy/            # installs NGINX + per-domain reverse proxy configs
    ├── zram/                   # enables zram-backed swap on /dev/zram0
    ├── system_update/          # applies yum package updates, optional reboot
    ├── dnf_automatic/          # installs + configures daily unattended updates
    └── dnf_kernel_limit/       # limits how many old kernels are kept installed
```

## Before you run it

Edit these:

- `group_vars/reverse_proxy/vault.yml` (encrypted) — set the real hostname,
  SSH user, SSH key path, and domain names (see
  [Secrets / vault setup](#secrets--vault-setup) below).
- `group_vars/all.yml` — set:
  - `proxy_user.name` — the username you want for UID 1000
  - `pip_packages` — the list of packages to install via pip3
  - `nginx_sites` — the backend each domain proxies to (e.g.
    `http://127.0.0.1:8001` for an app running locally on the VM, or
    `http://10.0.0.5:80` for an app on another host); domain names
    themselves come from the vault file above.

`inventory/hosts.ini` itself only needs a generic alias and shouldn't
normally need editing.

This project targets CentOS/RHEL-family hosts:

- Packages are installed with `ansible.builtin.yum` (which auto-detects and
  uses the `dnf` backend on modern CentOS/RHEL, so this works fine on
  CentOS Stream 10 even though `dnf` is the "real" package manager there).
- Site configs are written directly to `/etc/nginx/conf.d/*.conf`, matching
  the RHEL/CentOS `nginx` package layout (no `sites-available`/`sites-enabled`
  convention like on Debian/Ubuntu).
- SELinux's `httpd_can_network_connect` boolean is enabled automatically so
  NGINX is allowed to proxy_pass to backends — without this you'll get
  `502 Bad Gateway` errors on a stock CentOS box with SELinux enforcing.
- `firewalld` is opened for the `http`/`https` services, since it's enabled
  by default on CentOS and otherwise blocks inbound port 80/443.

This requires the `ansible.posix` collection for the `seboolean` and
`firewalld` modules:

```bash
ansible-galaxy collection install ansible.posix
```

## Running it

```bash
# Check connectivity first
ansible reverse_proxy -m ping

# Dry run to see what would change
ansible-playbook site.yml --check --diff

# Apply for real
ansible-playbook site.yml
```

You'll likely need `--ask-become-pass` if your SSH user needs a sudo password,
and `-l reverse_proxy` isn't necessary since the playbook already targets
that group.

## Keeping the server patched (`update.yml`)

A separate playbook applies `yum update` (via `ansible.builtin.yum`, which
uses the `dnf` backend under the hood) to the same hosts:

```bash
# Dry run
ansible-playbook update.yml --check --diff

# Apply for real
ansible-playbook update.yml
```

It also installs `yum-utils` (for `needs-restarting`) and reports whether a
reboot is required afterward (e.g. after a kernel update). By default it
**won't reboot automatically** — patching a live reverse proxy shouldn't
cause a surprise outage. To opt in, either for a single run:

```bash
ansible-playbook update.yml -e system_update_auto_reboot=true
```

or set `system_update_auto_reboot: true` permanently in `group_vars/all.yml`.

This is just a playbook you run on demand (or wire up via cron/CI on your
control machine) — it complements (rather than replaces) the always-on
`dnf-automatic` setup from `site.yml`, e.g. for triggering an update outside
its daily schedule or when you want the reboot-required check surfaced
explicitly.

## Unattended daily updates (`dnf_automatic` role, part of `site.yml`)

`site.yml` installs and enables `dnf-automatic`, which downloads *and
installs* updates once a day via its own `dnf-automatic.timer` systemd timer
— no cron or external scheduler needed. Defaults (in
`roles/dnf_automatic/defaults/main.yml`):

- `dnf_automatic_apply_updates: true` — actually installs updates (not just
  downloads/checks them). Set to `false` to only be notified.
- `dnf_automatic_upgrade_type: default` — installs all available updates.
  Set to `security` to only install security updates.

The timer's schedule (`OnCalendar=*-*-* 6:00`, i.e. daily around 6am with up
to a 60-minute random delay) comes from the package's own
`/usr/lib/systemd/system/dnf-automatic.timer` unit and isn't overridden here.
Check status any time with:

```bash
ansible reverse_proxy -m command -a "systemctl list-timers dnf-automatic.timer" -b
```

Note this does **not** reboot the host automatically, even if an update
(e.g. a new kernel) needs one — run `update.yml` (see above) if you want
reboot detection/handling as well.

## Kernel version retention (`dnf_kernel_limit` role, part of `site.yml`)

Sets `installonly_limit` in `/etc/dnf/dnf.conf` (which `/etc/yum.conf` is a
symlink to, so this covers both `yum` and `dnf`). Kernel packages
(`kernel-core`, `kernel-modules`, etc.) are "installonly" — each update adds
a new version alongside the old one rather than replacing it, since you may
need to boot back into a previous kernel. This setting caps how many are
kept: once a new kernel update would exceed the limit, the oldest installed
kernel is automatically removed.

- `kernel_installonly_limit: 2` (default, in
  `roles/dnf_kernel_limit/defaults/main.yml`) — keeps the current kernel plus
  one previous one. dnf/yum enforce a hard minimum of 2.

This only takes effect the next time kernel packages are updated (e.g. via
`update.yml` or the daily `dnf-automatic` run) — it doesn't retroactively
remove any kernels already installed beyond the limit. Check what's
currently installed with:

```bash
ansible reverse_proxy -m command -a "rpm -q kernel-core"
```

## Notes / next steps

- This currently serves plain HTTP (port 80) for each domain. Once DNS is
  pointed at the VM, running `certbot --nginx` (already installed via pip3
  above, or use your distro's certbot package) is the easiest way to add
  HTTPS/Let's Encrypt certs per domain.

## Secrets / vault setup

This repo hasn't been published yet, so real infrastructure details are kept
out of plaintext using `ansible-vault`:

- `inventory/hosts.ini` only contains a generic alias (`reverse_proxy_vm`) —
  no real hostname, SSH user, or key path.
- `group_vars/reverse_proxy/vault.yml` (encrypted) holds the real
  `ansible_host`, `ansible_user`, SSH key path, and the real domain names.
- `group_vars/reverse_proxy/vars.yml` (plaintext) and `group_vars/all.yml`
  reference those as `{{ vault_* }}` variables, so the *structure* stays
  readable/diffable while only the *values* are encrypted.
- `ansible.cfg` points at `vault_password_file = .vault_pass.txt`, so
  `ansible-playbook`/`ansible` commands decrypt automatically without any
  extra flags.

`.vault_pass.txt` is a randomly generated password, gitignored, and lives
only on this machine. **Back it up somewhere safe (e.g. a password
manager)** — if it's lost, `group_vars/reverse_proxy/vault.yml` can't be
decrypted and you'd need to reconstruct it from scratch (real host, user,
ssh key path, domains).

To view or edit the vaulted values:

```bash
ansible-vault view group_vars/reverse_proxy/vault.yml
ansible-vault edit group_vars/reverse_proxy/vault.yml
```

To add more secrets later, either add more `vault_*` keys to that same file,
or create additional `group_vars/<group>/vault.yml` files following the same
pattern for other host groups.

`.gitignore` also excludes `.entire/` (local state for the Entire CLI
checkpoint/session tool) and `*.retry` files — neither should ever be
committed.
