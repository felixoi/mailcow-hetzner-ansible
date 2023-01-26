# mailserver-ansible

Ansible playbook to automatically set up a [mailcow](https://github.com/mailcow/mailcow-dockerized/) on [Hetzner Cloud](https://www.hetzner.com/cloud),
including provisioning of the server itself, with optional backups to a
[Hetzner Storage Box](https://www.hetzner.com/storage/storage-box) (or other providers with available ssh access).

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

## Running the playbook

Copy `config.yml.example` to `config.yml` and change the variables according to your wishes.

Run the playbook:
```
ansible-playbook ansible/site.yml --extra-vars "@config.yml"
```
Instead of copying the config and defining the variables there you can also
[use environment variables to define them one by one](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#defining-variables-at-runtime).

## Setting up backups using a Hetzner Storage Box

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
6. Create a new ssh keypair on your local machine (`<pathofyourchoice>/id_rsa`, RSA, 4096, no password)
7. Edit the created `<pathofyourchoice>/id_rsa.pub` file: In the same line as the existing public key prepend
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
   scp -P 23 <pathofyourchoice>/id_rsa.pub <username_subaccount>@<username_subaccount>.your-storagebox.de:.ssh/authorized_keys
   ```
10. Disable `External reachability` for the created subaccount
11. Transform the private key to base64 using `echo $(cat <pathofyourchoice>/id_rsa | base64 -w 0);`
12. Enable and configure backups using vars in this ansible script:
    1. `backup_enabled: yes`
    2. `backup_passphrase: <choosesecretpassphrase>`
    3. `backup_private_key: <private_key_base64>`
    4. `backup_user: <username_subaccount>`
    5. `backup_host: <username_subaccount>.your-storagebox.de`
    6. `backup_port: 23`
    7. `backup_path: ./backups` (if changed, the `restrict-to-path` arg in step 6 must be edited accordingly)
13. In case you want to monitor your backup cron job using [healthchecks](https://github.com/healthchecks/healthchecks) (hosted or self-hosted):
    1. `backup_healthchecks_enabled: yes`
    2. `backup_healthchecks_webhook: https://hc-ping.com/<ping_uuid>`

# How to restore from a backup

Restoring is not part of the playbook, as it should be done only under supervision and with extreme caution.
The backup contains the data directories of the following services: `vmail`, `crypt`, `redis`, `rspamd`, `postfix`.
Additionally, the MySQL database is backed up using the MySQL Borgmatic integration. As the backup mechanism is different 
from that of the other services, it is also necessary to distinguish between them during recovery.

## Prerequisites

You need an shh connection to the cloud server and run all following commands in the `/opt/mailcow-dockerized`.

## List all available backups

```
docker-compose exec borgmatic-mailcow borgmatic list
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

## Restore services but the MySQL database

If you want to restore a service beside the MySQL database you need to set the `backup_restorable` config variable
correctly (add all services you want to restore to the list) and run the playbook for the changes to take effect.

```
docker-compose exec borgmatic-mailcow borgmatic extract [PATHS] --archive latest
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
docker-compose exec borgmatic-mailcow sh -c 'rm -rf /mnt/source/<service>/*'
```
or for all services but the MySQL database
```
docker-compose exec borgmatic-mailcow sh -c 'rm -rf /mnt/source/**/*'
```

Restoring is also possible for particular files of a backup (e.g. restoring only one mailbox to a previous state).
Check out the [borgmatic docs](https://torsion.org/borgmatic/docs/how-to/extract-a-backup/#extract-particular-files) for instructions.
The mentioned limitation also applies for restoring particular files which means that you may want to delete the existing files/folders beforehand.

## Restore MySQL database

If you want to restore the MySQL database run the following command:
```
docker-compose exec borgmatic-mailcow borgmatic restore --archive <name>
```

## After restoring

Restart mailcow:
```
docker-compose down && docker-compose up -d
```
