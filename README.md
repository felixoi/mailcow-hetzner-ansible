# mailserver-ansible

Ansible playbook to automatically set up a [mailcow](https://github.com/mailcow/mailcow-dockerized/) on [Hetzner Cloud](https://www.hetzner.com/cloud),
including provisioning of the server itself, with optional backups to [BorgBase](https://www.borgbase.com/),
a [Hetzner Storage Box](https://www.hetzner.com/storage/storage-box) or any other provider with Borg over SSH.

## Prerequisites

### Clone and update

This repository contains the [official mailcow ansible role](https://github.com/mailcow/mailcow-ansiblerole) as submodule.
In order for this playbook to work correctly, you need to clone the repository using the recursive flag:
```
git clone --recursive https://github.com/felixoi/mailcow-hetzner-ansible.git
```
Therefore, when updating the local repository, the submodule must also be set to the correct commit:
```
git pull
git submodule update --recursive
```

### Install dependencies

The playbook needs a few Ansible roles and collections and some Python packages on your local machine:
```
ansible-galaxy install -r ansible/requirements.yml
```
Install the Python packages `netaddr` (IP address handling), `requests` and `python-dateutil` (Hetzner Cloud modules)
into the same Python environment as Ansible, e.g. `pip install netaddr requests python-dateutil`, or with your
distribution's package manager (e.g. `python3-netaddr python3-requests python3-dateutil`).
Run the `ansible-galaxy` command again with `--force` after pulling changes of this repository, and from time to time
to keep the collections up to date.

## Running the playbook

Copy `config.yml.example` to `config.yml` and change the variables according to your wishes.
The secrets in it (`hcloud_token`, `backup_passphrase`, `backup_private_key`) are stored encrypted, see
[Encrypting secrets](#encrypting-secrets).

Run the playbook:
```
ansible-playbook ansible/site.yml --extra-vars "@config.yml" --ask-vault-pass
```
Instead of copying the config and defining the variables there you can also
[use environment variables to define them one by one](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#defining-variables-at-runtime).

### SSH access to the server

Ansible connects to the server as `root` using the keys in `server_ssh_keys`. The Hetzner firewall only allows the
mail ports by default, so choose how Ansible reaches SSH:

- **Public SSH:** set `server_firewall_ssh_enabled: true` to open port 22 to the whole internet.
- **VPN (e.g. WireGuard):** keep SSH closed and set `server_ip_override` to the server's address inside your VPN.
  Ansible then connects to that address instead of the public IPv4. Setting up the VPN itself is not part of this
  playbook; if it needs an open port on the server, attach a firewall allowing it via `server_extra_firewalls`.
  For the very first run on a new server, the VPN doesn't exist yet, so temporarily enable public SSH.

### Encrypting secrets

Secrets in `config.yml` are encrypted with [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).
Ansible decrypts them in memory while running the playbook, so they are never stored in plain text on your machine.
All values are encrypted with the same vault password; store it in your password manager, as the values can't be
recovered without it.

1. Encrypt each secret and replace the example value in `config.yml` with the printed `!vault` block:
   ```
   ansible-vault encrypt_string --ask-vault-pass --stdin-name hcloud_token
   ansible-vault encrypt_string --ask-vault-pass --stdin-name backup_passphrase
   ansible-vault encrypt_string --ask-vault-pass --stdin-name backup_private_key
   ```
   Enter the vault password, then the value, press Enter and then Ctrl+D.
2. To check a value later, run `ansible localhost -m debug -a var=<name> --extra-vars "@config.yml" --ask-vault-pass`.

The secrets still end up in plain text on the mail server (readable by root only), as the backups need them to run
unattended.

## Updating mailcow

A normal playbook run only applies the configuration and does not update mailcow. To update mailcow, run:
```
ansible-playbook ansible/site.yml --extra-vars "@config.yml" --extra-vars "mailcow_install_updates=true" --ask-vault-pass
```
Updates follow the branch set in `mailcow_branch` (`master` by default). The branch is only used for the initial install;
switching an existing installation to another branch has to be done with mailcow's `update.sh` on the server.
If backups are enabled, a backup is created right before updating.
Set `mailcow_install_updates: true` in `config.yml` to update on every run instead.

## Backups

Backups are optional and made with [borgmatic](https://torsion.org/borgmatic/) running as an additional container
next to mailcow. They are encrypted and sent over SSH to any provider supporting Borg.

### Setting up

#### BorgBase

1. Create an account at [BorgBase](https://www.borgbase.com/) and add a new repository.
2. Create a new ssh keypair without a passphrase on your local machine:
   ```
   ssh-keygen -t ed25519 -N "" -f <pathofyourchoice>/backup_key
   ```
3. Add the public key (`<pathofyourchoice>/backup_key.pub`) under **Account > SSH Keys** and assign it to the repository
   with **Append-only access**. The mailcow server can then add new backups, but never remove existing ones
   ([details](https://docs.borgbase.com/repositories/append-only.html)).
4. Transform the private key to base64 using `base64 -w 0 <pathofyourchoice>/backup_key`
5. Enable and configure backups using vars in this ansible script. The repository ID is shown in the repository URL
   `ssh://<repo_id>@<repo_id>.repo.borgbase.com/./repo`:
    1. `backup_enabled: true`
    2. `backup_passphrase: <choosesecretpassphrase>`
    3. `backup_private_key: <private_key_base64>`
    4. `backup_user: <repo_id>`
    5. `backup_host: <repo_id>.repo.borgbase.com`
    6. `backup_port: 22`
    7. `backup_path: ./repo`
    8. `backup_host_keys: [<host_key>, ...]`: full public keys as described in step 12.8 of the Storage Box setup below,
       using `ssh-keyscan <repo_id>.repo.borgbase.com`. BorgBase shows the expected fingerprint when hovering the
       fingerprint icon next to the repository.
6. Optionally set a [monitoring alert](https://docs.borgbase.com/repositories/settings.html) for the repository, so
   BorgBase notifies you when no new backup arrived for a number of days.

As the key has append-only access, pruning old backups doesn't free any space. Compact the repository from time to time
using **More > Compact repo** in the BorgBase control panel.

#### Hetzner Storage Box

1. [Buy](https://www.hetzner.com/storage/storage-box) a storage box of your choice
2. Reset the password for the root account to obtain the password.
3. Connect to your storage box and run `mkdir -p mailcow`
4. Create a new subaccount
   1. Samba: disabled
   2. WebDAV: disabled
   3. SSH: enabled
   4. External reachability: enabled
   5. Read-only: disabled
   6. Base directory: `mailcow`
5. Reset the password for the subaccount account to obtain the password.
6. Create a new ssh keypair without a passphrase on your local machine:
   ```
   ssh-keygen -t ed25519 -N "" -f <pathofyourchoice>/backup_key
   ```
7. Edit the created `<pathofyourchoice>/backup_key.pub` file: In the same line as the existing public key prepend
   ```
   from="<MAILCOW_SERVER_IP>",command="borg serve --append-only --restrict-to-path /home/backups/",no-pty,no-agent-forwarding,no-port-forwarding,no-X11-forwarding,no-user-rc 
   ```
   Whenever a connection is established using the generated keypair the command `borg serve --append-only --restrict-to-path ...`
   will be executed even if a completely different one was specified. Thus, it's not possible to perform actions outside the append-only mode by the specified key.   
   If you want to allow destructive actions you can remove the `--append-only` option or even better create another keypair with the same restrictions but without the `--append-only` option.
8. Create the .ssh directory for the subaccount:
   ```
   ssh -p 23 <username_subaccount>@<username_subaccount>.your-storagebox.de mkdir .ssh
   ```
9. Copy the edited public key to the storage box to allow connection using the key:
   ```
   scp -P 23 <pathofyourchoice>/backup_key.pub <username_subaccount>@<username_subaccount>.your-storagebox.de:.ssh/authorized_keys
   ```
10. Disable `External reachability` for the created subaccount
11. Transform the private key to base64 using `base64 -w 0 <pathofyourchoice>/backup_key`
12. Enable and configure backups using vars in this ansible script:
    1. `backup_enabled: true`
    2. `backup_passphrase: <choosesecretpassphrase>`
    3. `backup_private_key: <private_key_base64>`
    4. `backup_user: <username_subaccount>`
    5. `backup_host: <username_subaccount>.your-storagebox.de`
    6. `backup_port: 23`
    7. `backup_path: ./backups` (if changed, the `restrict-to-path` arg in step 7 must be edited accordingly)
    8. `backup_host_keys: [<host_key>, ...]`: the SSH host keys of the storage box, so backups are only sent to the real server.
       Each entry is a full public key in the form `<type> <base64 key>` (e.g. `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...`),
       not a `SHA256:...` fingerprint. Get them using
       ```
       ssh-keyscan -p 23 <username_subaccount>.your-storagebox.de 2>/dev/null | cut -d' ' -f2-
       ```
       and check that their fingerprints (`ssh-keyscan -p 23 <username_subaccount>.your-storagebox.de 2>/dev/null | ssh-keygen -lf -`)
       match the ones [published by Hetzner](https://docs.hetzner.com/storage/storage-box/access/access-ssh-rsync-borg/).
       If left empty, the host key is trusted on first connection and verified on every connection afterwards.
13. In case you want to monitor your backup cron job using [healthchecks](https://github.com/healthchecks/healthchecks) (hosted or self-hosted):
    1. `backup_healthchecks_enabled: true`
    2. `backup_healthchecks_webhook: https://hc-ping.com/<ping_uuid>`

### Restoring

Restoring is not part of the playbook, as it should be done only under supervision and with extreme caution.
The backup contains the data directories of the following services: `vmail`, `crypt`, `redis`, `rspamd`, `postfix`.
Additionally, the MySQL database is backed up using the MySQL Borgmatic integration. As the backup mechanism is different 
from that of the other services, it is also necessary to distinguish between them during recovery.

#### Prerequisites

You need an shh connection to the cloud server and run all following commands in the `/opt/mailcow-dockerized`.

#### List all available backups

```
docker compose exec borgmatic-mailcow borgmatic list
```
Example output:
```
ssh://u111111-sub1@u111111-sub1.your-storagebox.de:23/./backups: Listing archives
mailcow-2022-12-19T21:22:42.775896   Mon, 2022-12-19 21:22:44 [d8293fe52f02a6e7c13e02e1ae82a56838949d4005c7a037947f6874d631d50c]
mailcow-2022-12-19T23:14:06.114196   Mon, 2022-12-19 23:14:07 [99bf7efe349aa58fcad707cfd6b0192c80a6e3457a6fa69542871c11bbdd72aa]
mailcow-2022-12-20T23:14:06.064385   Tue, 2022-12-20 23:14:07 [bf6d07f600087c0ec934061267f39972f5a6ad6c19dc9f2e09652019ccad9088]
mailcow-2022-12-21T14:14:07.412074   Wed, 2022-12-21 14:14:08 [048bb19ae8577815c4b415c832ee6d5b221e65d1e93a046e596572dd224dca95]
mailcow-2022-12-21T15:14:07.988515   Wed, 2022-12-21 15:14:09 [cbb2f877c13ee5f73e009ddd32e32d859ad8bb29689faed148378462f7fc16a2]
mailcow-2022-12-21T16:14:08.082914   Wed, 2022-12-21 16:14:09 [43545410400baf66ba95fe581a5ae861f13a05172a92ab5f0c066e3176910e90]
```
Every backup has an unique name which is shown in the first column. The name is used in the following commands to
specify which backup you want to restore. It's also possible to use `latest` instead to reference the latest
available backup.

#### Restore services but the MySQL database

If you want to restore a service beside the MySQL database you need to set the `backup_restorable` config variable
correctly (add all services you want to restore to the list) and run the playbook for the changes to take effect.

```
docker compose exec borgmatic-mailcow borgmatic extract [PATHS] --archive latest
```
`[PATHS]` needs to be replaced by you as following:
- If you want to restore all possible services use `--path mnt/source`
- If you want to restore specific services only add `--path mnt/source/<service_name>` following for **every** service, where
  `<service_name>` is one of the services listed before.

**Be aware** that due to a [limitation of borg backup](https://borgbackup.readthedocs.io/en/stable/faq.html#are-there-other-known-limitations)
the folders borgmatic extracts to need to be empty for a point-in-time restore. If they are not empty, you may end up in weird mailbox states.

:warning: **The following commands are destructive, you may want to
[trigger a manual backup](https://docs.mailcow.email/third_party/borgmatic/third_party-borgmatic/#manual-archiving-run-with-debugging-output) of the current state beforehand!**

To empty the folder for a service you can use
```
docker compose exec borgmatic-mailcow sh -c 'rm -rf /mnt/source/<service>/*'
```
or for all services but the MySQL database
```
docker compose exec borgmatic-mailcow sh -c 'rm -rf /mnt/source/**/*'
```

Restoring is also possible for particular files of a backup (e.g. restoring only one mailbox to a previous state).
Check out the [borgmatic docs](https://torsion.org/borgmatic/docs/how-to/extract-a-backup/#extract-particular-files) for instructions.
The mentioned limitation also applies for restoring particular files which means that you may want to delete the existing files/folders beforehand.

#### Restore MySQL database

If you want to restore the MySQL database run the following command:
```
docker compose exec borgmatic-mailcow borgmatic restore --archive <name>
```

#### After restoring

Restart mailcow:
```
docker compose down && docker compose up -d
```
