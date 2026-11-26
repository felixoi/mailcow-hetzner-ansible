# mailserver-ansible

Ansible playbook to set up a [mailcow](https://github.com/mailcow/mailcow-dockerized/) mail server on
[Hetzner Cloud](https://www.hetzner.com/cloud), from creating the server to a running mailcow, with optional encrypted
backups.

It
- creates the Hetzner Cloud server, its firewalls, SSH keys and reverse DNS entries,
- optionally creates and assigns [floating IPs](#floating-ips), so the server can be replaced without losing the IP
  reputation,
- configures network, DNS, hostname and timezone of the server and installs Docker,
- installs and configures mailcow using the [official mailcow ansible role](https://github.com/mailcow/mailcow-ansiblerole),
- optionally sets up [backups](#backups) with borgmatic to [BorgBase](https://www.borgbase.com/), a
  [Hetzner Storage Box](https://www.hetzner.com/storage/storage-box) or any other provider with Borg over SSH.

## Contents

- [Installation](#installation)
- [Configuration](#configuration)
  - [Encrypting secrets](#encrypting-secrets)
  - [SSH access to the server](#ssh-access-to-the-server)
  - [Floating IPs](#floating-ips)
- [Running the playbook](#running-the-playbook)
  - [Updating mailcow](#updating-mailcow)
- [Backups](#backups)
  - [BorgBase](#borgbase)
  - [Hetzner Storage Box](#hetzner-storage-box)
  - [Restoring](#restoring)

## Installation

You need Ansible (ansible-core 2.19 or newer) on your local machine and a Hetzner Cloud project with an API token
with read and write access.

This repository contains the official mailcow ansible role as submodule, so clone it with the recursive flag:
```
git clone --recursive https://github.com/felixoi/mailcow-hetzner-ansible.git
```

Install the required Ansible roles and collections:
```
ansible-galaxy install -r ansible/requirements.yml
```

Install the Python packages `netaddr` (IP address handling), `requests` and `python-dateutil` (Hetzner Cloud modules)
into the same Python environment as Ansible, e.g. `pip install netaddr requests python-dateutil`, or with your
distribution's package manager (e.g. `python3-netaddr python3-requests python3-dateutil`).

To update your local copy later, also update the submodule and the roles and collections:
```
git pull
git submodule update --recursive
ansible-galaxy install -r ansible/requirements.yml --force
```

## Configuration

Copy `config.yml.example` to `config.yml` and adjust the values. Every option is explained in the file. The secrets
in it are stored encrypted, see [Encrypting secrets](#encrypting-secrets).

Besides the options in the example, all options of the
[mailcow ansible role](https://github.com/mailcow/mailcow-ansiblerole) (`mailcow__*`) can be set in `config.yml` as well.

### Encrypting secrets

The secrets in `config.yml` (`hcloud_token`, `backup_passphrase`, `backup_private_key`) are encrypted with
[Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html). Ansible decrypts them in memory while
running the playbook, so they are never stored in plain text on your machine. All values are encrypted with the same
vault password; store it in your password manager, as the values can't be recovered without it.

Encrypt each secret and replace the example value in `config.yml` with the printed `!vault` block:
```
ansible-vault encrypt_string --ask-vault-pass --stdin-name hcloud_token
ansible-vault encrypt_string --ask-vault-pass --stdin-name backup_passphrase
ansible-vault encrypt_string --ask-vault-pass --stdin-name backup_private_key
```
Enter the vault password, then the value, press Enter and then Ctrl+D. To check a value later, run
`ansible localhost -m debug -a var=<name> --extra-vars "@config.yml" --ask-vault-pass`.

The secrets still end up in plain text on the mail server (readable by root only), as the backups need them to run
unattended.

### SSH access to the server

Ansible connects to the server as `root` using the keys in `server_ssh_keys`. The Hetzner firewall only allows the
mail ports by default, so choose how Ansible reaches SSH:

- **Public SSH:** set `server_firewall_ssh_enabled: true` to open port 22 to the whole internet.
- **VPN (e.g. WireGuard):** keep SSH closed and set `server_ssh_address` to the server's address inside your VPN.
  Ansible then connects to that address instead of the public IPv4. Setting up the VPN itself is not part of this
  playbook; if it needs an open port on the server, attach a firewall allowing it via `server_extra_firewalls`.
  For the very first run on a new server, the VPN doesn't exist yet, so temporarily enable public SSH.

### Floating IPs

A mail server's IP addresses build up a sending reputation. With
[floating IPs](https://docs.hetzner.com/cloud/floating-ips/overview/) the addresses can be kept when the server is
replaced. Set names for a floating IPv4 and/or IPv6:
```
server_floating_ipv4: mail-ipv4
server_floating_ipv6: mail-ipv6
```
The playbook then
- creates floating IPs with these names in the server's location if they don't exist yet (floating IPs are billed
  separately), or uses existing ones with these names,
- enables their delete protection, so they can't be deleted by accident,
- assigns them to the server, moving them away from any other server they are assigned to,
- adds them to the server's network configuration,
- sets their reverse DNS entry to `hostname`,
- makes mailcow use them as source address for all outgoing connections (`SNAT_TO_SOURCE` / `SNAT6_TO_SOURCE`).

Point the DNS records of `hostname` (A/AAAA, and therefore MX and SPF) to the floating IPs. For the IPv6 floating IP,
which is a /64 network, the first address (`::1`) is used. Other traffic of the server, e.g. package updates, still
uses the server's own IPs.

To move to a new server, first set it up without the floating IPs: change `server_name`, leave the two options empty
and run the playbook, then [restore the backup](#restoring) on the new server. Afterwards set the floating IP options
again and run the playbook once more to move the IPs over, so incoming mail only reaches the new server once it has
all mailboxes. Then remove the old server.

## Running the playbook

```
ansible-playbook ansible/site.yml --extra-vars "@config.yml" --ask-vault-pass
```

The playbook can be run as often as needed: it creates what is missing and applies changes of `config.yml`.

### Updating mailcow

A normal playbook run only applies the configuration and does not update mailcow. To update mailcow, run:
```
ansible-playbook ansible/site.yml --extra-vars "@config.yml" --extra-vars "mailcow_install_updates=true" --ask-vault-pass
```
If backups are enabled, a backup is created right before updating. Set `mailcow_install_updates: true` in `config.yml`
to update on every run instead.

Updates follow the branch set in `mailcow_branch` (`master` by default). The branch is only used for the initial install;
switching an existing installation to another branch has to be done with mailcow's `update.sh` on the server.

## Backups

Backups are optional and made with [borgmatic](https://torsion.org/borgmatic/) running as an additional container next
to mailcow. They are encrypted and sent over SSH to any provider supporting Borg. A backup runs every hour and contains
the mailboxes and data of `vmail`, `crypt`, `redis`, `rspamd` and `postfix` as well as the mailcow database.
The backup repository is checked for consistency weekly, and the backup data monthly.

The guides below set up an SSH key that can only add backups, but never delete them, so a compromised mail server
can't destroy existing backups. They end with these options in `config.yml`:

| Option | Description |
|---|---|
| `backup_enabled` | `true` to enable backups |
| `backup_passphrase` | Passphrase encrypting the backups. Choose a long random one and store it in your password manager: without it, the backups can't be restored. |
| `backup_private_key` | The private SSH key, base64 encoded |
| `backup_user`, `backup_host`, `backup_port`, `backup_path` | Where the backups are stored, see the provider guides |
| `backup_host_keys` | SSH host keys of the backup server, see below |
| `backup_healthchecks_enabled`, `backup_healthchecks_webhook` | Optional monitoring of the backups with [healthchecks](https://github.com/healthchecks/healthchecks) (hosted or self-hosted), e.g. `https://hc-ping.com/<ping_uuid>` |
| `backup_restorable` | Services whose data may be written by borgmatic, see [Restoring](#restoring) |

`backup_host_keys` makes sure backups are only sent to the real backup server. Each entry is a full public key in the
form `<type> <base64 key>` (e.g. `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...`), not a `SHA256:...` fingerprint. Get them
using
```
ssh-keyscan -p <backup_port> <backup_host> 2>/dev/null | cut -d' ' -f2-
```
and check that their fingerprints (`ssh-keyscan -p <backup_port> <backup_host> 2>/dev/null | ssh-keygen -lf -`) match
the ones published by your provider. If left empty, the host key is trusted on first connection and verified on every
connection afterwards.

### BorgBase

1. Create an account at [BorgBase](https://www.borgbase.com/) and add a new repository.
2. Create a new SSH keypair without a passphrase on your local machine:
   ```
   ssh-keygen -t ed25519 -N "" -f <pathofyourchoice>/backup_key
   ```
3. Add the public key (`<pathofyourchoice>/backup_key.pub`) under **Account > SSH Keys** and assign it to the repository
   with **Append-only access** ([details](https://docs.borgbase.com/repositories/append-only.html)).
4. Base64 encode the private key using `base64 -w 0 <pathofyourchoice>/backup_key`.
5. Configure the backup options. The repository ID is shown in the repository URL
   `ssh://<repo_id>@<repo_id>.repo.borgbase.com/./repo`:
   ```
   backup_enabled: true
   backup_passphrase: <choosesecretpassphrase>   # encrypted with ansible-vault
   backup_private_key: <private_key_base64>      # encrypted with ansible-vault
   backup_user: <repo_id>
   backup_host: <repo_id>.repo.borgbase.com
   backup_port: 22
   backup_path: ./repo
   backup_host_keys: [<host_key>, ...]
   ```
   BorgBase shows the expected host key fingerprint when hovering the fingerprint icon next to the repository.
6. Optionally set a [monitoring alert](https://docs.borgbase.com/repositories/settings.html) for the repository, so
   BorgBase notifies you when no new backup arrived for a number of days.

As the key has append-only access, pruning old backups doesn't free any space. Compact the repository from time to time
using **More > Compact repo** in the BorgBase control panel.

### Hetzner Storage Box

1. [Buy](https://www.hetzner.com/storage/storage-box) a storage box of your choice.
2. Reset the password for the root account to obtain the password.
3. Connect to your storage box and run `mkdir -p mailcow`.
4. Create a new subaccount with
   - Samba: disabled
   - WebDAV: disabled
   - SSH: enabled
   - External reachability: enabled
   - Read-only: disabled
   - Base directory: `mailcow`
5. Reset the password for the subaccount to obtain the password.
6. Create a new SSH keypair without a passphrase on your local machine:
   ```
   ssh-keygen -t ed25519 -N "" -f <pathofyourchoice>/backup_key
   ```
7. Edit the created `<pathofyourchoice>/backup_key.pub` file and prepend the following in the same line as the public key:
   ```
   from="<MAILCOW_SERVER_IP>",command="borg serve --append-only --restrict-to-path /home/backups/",no-pty,no-agent-forwarding,no-port-forwarding,no-X11-forwarding,no-user-rc 
   ```
   Whenever a connection is established using this key, `borg serve --append-only --restrict-to-path ...` is executed,
   even if a different command was requested, so the key can't do anything but add backups. If you want to allow
   destructive actions, create another keypair with the same restrictions but without `--append-only`.
8. Create the `.ssh` directory for the subaccount:
   ```
   ssh -p 23 <username_subaccount>@<username_subaccount>.your-storagebox.de mkdir .ssh
   ```
9. Copy the edited public key to the storage box:
   ```
   scp -P 23 <pathofyourchoice>/backup_key.pub <username_subaccount>@<username_subaccount>.your-storagebox.de:.ssh/authorized_keys
   ```
10. Disable **External reachability** for the subaccount.
11. Base64 encode the private key using `base64 -w 0 <pathofyourchoice>/backup_key`.
12. Configure the backup options:
    ```
    backup_enabled: true
    backup_passphrase: <choosesecretpassphrase>   # encrypted with ansible-vault
    backup_private_key: <private_key_base64>      # encrypted with ansible-vault
    backup_user: <username_subaccount>
    backup_host: <username_subaccount>.your-storagebox.de
    backup_port: 23
    backup_path: ./backups   # if changed, adjust --restrict-to-path from step 7
    backup_host_keys: [<host_key>, ...]
    ```
    Hetzner publishes the host key fingerprints of the storage boxes in its
    [documentation](https://docs.hetzner.com/storage/storage-box/access/access-ssh-rsync-borg/).

### Restoring

Restoring is not part of the playbook, as it should only be done under supervision and with great care. The data of
the services (`vmail`, `crypt`, `redis`, `rspamd`, `postfix`) and the database are backed up differently, so they are
also restored separately.

Run all following commands on the server via SSH in `/opt/mailcow-dockerized`.

#### List available backups

```
docker compose exec borgmatic-mailcow borgmatic list
```
Example output:
```
ssh://u111111-sub1@u111111-sub1.your-storagebox.de:23/./backups: Listing archives
mailcow-2022-12-19T21:22:42.775896   Mon, 2022-12-19 21:22:44 [d8293fe52f02a6e7c13e02e1ae82a56838949d4005c7a037947f6874d631d50c]
mailcow-2022-12-19T23:14:06.114196   Mon, 2022-12-19 23:14:07 [99bf7efe349aa58fcad707cfd6b0192c80a6e3457a6fa69542871c11bbdd72aa]
mailcow-2022-12-20T23:14:06.064385   Tue, 2022-12-20 23:14:07 [bf6d07f600087c0ec934061267f39972f5a6ad6c19dc9f2e09652019ccad9088]
```
The first column is the name of each backup, which the following commands use to select a backup. `latest` refers to
the most recent one.

#### Restore service data

By default, borgmatic can only read the service data. Add all services you want to restore to `backup_restorable`
(e.g. `backup_restorable: [vmail, crypt]`) and run the playbook to make their data writable for borgmatic.

Due to a [limitation of borg](https://borgbackup.readthedocs.io/en/stable/faq.html#are-there-other-known-limitations),
the folders borgmatic extracts to need to be empty for a point-in-time restore. Otherwise you may end up with
inconsistent mailboxes.

:warning: **The following commands are destructive, you may want to
[trigger a manual backup](https://docs.mailcow.email/third_party/borgmatic/third_party-borgmatic/#manual-archiving-run-with-debugging-output) of the current state beforehand!**

Empty the folder of a service:
```
docker compose exec borgmatic-mailcow sh -c 'rm -rf /mnt/source/<service>/*'
```
or of all services:
```
docker compose exec borgmatic-mailcow sh -c 'rm -rf /mnt/source/**/*'
```

Then restore the data:
```
docker compose exec borgmatic-mailcow borgmatic extract --archive latest --path mnt/source
```
To restore only some services, replace `--path mnt/source` with `--path mnt/source/<service>` for each of them.

Restoring particular files, e.g. a single mailbox, is possible as well, see the
[borgmatic docs](https://torsion.org/borgmatic/docs/how-to/extract-a-backup/#extract-particular-files). The limitation
above applies here too, so you may want to delete the existing files first.

Remove the services from `backup_restorable` and run the playbook again afterwards.

#### Restore the database

```
docker compose exec borgmatic-mailcow borgmatic restore --archive <name>
```

#### After restoring

Restart mailcow:
```
docker compose down && docker compose up -d
```
