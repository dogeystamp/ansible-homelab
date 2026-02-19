# Homeserver Ansible Playbook

This Ansible playbook allows me to set up and configure all my home lab servers completely automatically, with minimal manual intervention.
It is for personal use; do not rely on this for anything important.
Special thanks to [Wolfgang](https://github.com/notthebee/) for the idea of automating the installation process.
This project was largely inspired by his own [infra](https://github.com/notthebee/infra) repo.

The playbook sets up the following services:

- [Forgejo](https://forgejo.org/) git forge
- [Continuwuity](https://continuwuity.org/) Matrix chat server
- [Navidrome](https://www.navidrome.org) music server
- [Radicale](https://radicale.org/v3.html) calendar/tasks/contacts sync server
- [Paperless-ngx](https://docs.paperless-ngx.com/) document server
- [Caddy](https://caddyserver.com/) reverse proxy
- [WireGuard](https://www.wireguard.com/) VPN server
- [NFTables](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page) firewall.
- [BorgBackup](https://www.borgbackup.org/) backup server (also used for server backups).

The other important feature of this playbook
is the automatic provisioning and deployment of QEMU virtual machines
as defined in the Ansible hosts file.
This playbook is designed so that most services run containerized in Podman
within QEMU VMs on a single host.
Note this playbook is designed for the ARM64 architecture specifically;
libvirt operates differently on different host archs.

## Handbook

These are rough notes on how to use this playbook.
The primary audience of this section is future me.
I do not guarantee that instructions here will work or are up to date.

### First-time configuration

These steps are for configuring the servers from scratch.

First, the hosts are the Ansible Controller (a Raspberry Pi 3B)
and the Hypervisor (a Raspberry Pi 4 8GB).

Disk partitioning:

- Controller has no disks.
- Hypervisor has an enclave disk, and the paths in `group_vars/all/10-defaults.yml`.
    - Notably, `ext_image` is a `qcow2` image containing a single file partition on it (I think?)

- Flash the Controller with OpenSUSE Leap (as of writing, this was Leap 15.6. I later upgraded to Leap 16).
- Connect to the Controller over SSH as `root`
- Install `git`, `systemd-network`
- Uninstall `yast2-firewall` and `yast2-users`
- Write something like this to `/etc/systemd/network/20-wired.network`:

    ```
        [Match]
        Name=eth0

        [Network]
        Address=192.168.0.4/24
        Gateway=192.168.0.1
        DNS=1.1.1.1
    ```

- `systemctl disable wicked`
- `systemctl enable --now systemd-networkd`
- Remove `/etc/resolv.conf`, then create it again as a regular file with `nameserver 1.1.1.1`
- Reconnect to the Controller at the static local IP you assigned it
- `systemctl mask wicked`
- `useradd -m erus` (This is the local admin user)
- Replace `/etc/sudoers` with `erus ALL=(ALL:ALL) NOPASSWD: ALL` (Ansible will rewrite this later)
- Add your pubkey to `~erus/.ssh/authorized_keys`
- Reconnect to Controller as `erus` instead of `root`
- Copy the following into `~/.config/git/allowed_signers`:
    ```
    dogeystamp@disroot.org namespaces="git" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICsmYC1XDX747Bd9QVQRgQbp7HUcGhWIMBBVnlknPubV
    ```
- Copy the following into `~/.config/git/config`
    ```
    [gpg.ssh]
    allowedSignersFile = "~/.config/git/allowed_signers"

    [merge]
    verifySignatures = true
    ```

- `git clone https://github.com/dogeystamp/ansible-homelab.git`
- Install Ansible and `wireguard-tools`
- `cd ansible-homelab; ansible-playbook local1.yml`

We've now done basic configuration on the Controller host.
We now configure the Hypervisor host.

- Provision the Hypervisor host with OpenSUSE Tumbleweed, a local admin `erus` user and passwordless sudo, similar to the above steps
- Provision the Hypervisor with `systemd-networkd`, a static IP and something reasonable in `/etc/resolv.conf`
- The following steps are run as `imperator@controller` (the remote administrator). Run `su -l imperator`.
- Copy the SSH pubkey for `imperator` onto `erus@hypervisor`.
- Clone this repo as `imperator`.
- `cd ansible-homelab`
- `dd bs=512 count=4 if=/dev/random | ansible-vault encrypt --output files/lukskey.secret`
- Copy `group_vars/vault-template.yml` to `group_vars/all/99-vault.yml` with `ansible-vault` and edit accordingly. Don't leave the vault as plain-text.
- Keep a backup of the secret and vault files.
- `ansible-playbook -i inventory.yml remote1.yml`

The playbook has now configured the virtual machines on the Hypervisor,
along with the VPN that is used for the VMs to communicate.

- Return to the `erus@controller` user.
- `ansible-playbook local2.yml`

The Controller should now be connected to the VPN.

- Sign in as `imperator@controller`.
- `ansible-playbook -J -i inventory.yml remote2.yml`

This is the general procedure for first time usage of this playbook.

### Adding a new backup path

Edit `inventory.yml` with the new path.
As `imperator`, run `ansible-playbook -J -i inventory.yml remote2.yml --tags "borg_client,borg_server"`.

### Adding a new backup host

Do the same steps as for adding a new backup path.
Also, edit `~/.local/bin/rs_borg_dl.sh` locally.
After the first backup, go check the backup directory on the server
and ensure that it allows group `rX` permissions.

### Updates

For each podman service, do `podman pull container/container:latest`
and then `systemctl --user restart container`.

On SUSE Tumbleweed, do `sudo zypper dup --no-recommends`.
On Leap, do `sudo zypper up --no-recommends`.

### Destroy LibVirt VM

Sign in to the VM's user.

```
virsh destroy $USER; virsh undefine --nvram $USER; rm -r ~/machine/; rm -r ~/.config; loginctl disable-linger
```

After, run `userdel -r [vm-user]` as root.
The `-r` removes the home directory.
