# mailserver-ansible

Ansible playbook to automatically set up a [mailcow](https://github.com/mailcow/mailcow-dockerized/) on [Hetzner Cloud](https://www.hetzner.com/cloud),
including provisioning of the server itself, with optional backups to a
[Hetzner Storage Box](https://www.hetzner.com/storage/storage-box) (or other providers with available ssh access).

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
