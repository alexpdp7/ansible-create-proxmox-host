A playbook that creates an LXC container and joins it to a FreeIPA domain.

Typical usage:

```
---
- hosts: <proxmox_host>
  tasks:
    - include_role:
        name: create-proxmox-host
      vars:
        hostname: <hostname>
        vmid: "{{ hostvars['<hostname>']['proxmox_vmid'] }}"
        ipa_domain: "{{ ipa_domain_name }}"
        ipa_username: admin
        ipa_password: "{{ ipa_admin_password }}"
        root_password: "{{ hostvars['<hostname>']['root_password'] }}"
        flavor: ubuntu_20_04
```

I keep IPA data on group_vars/all/(vars|vault) and proxmox_vmid on host_vars/<hostname>/(vars|vault).

I only need a single line in the inventory for each host's hostname, and I put:

```
ansible_become: True
ansible_user: <my ipa username>
```

in the host's variables to connect.

Parameters:

* flavor: only ubuntu_20_04 is supported now
* vmid
* hostname
* memory: default 512, in megabytes
* swap: default 512, in megabytes
* disk: default 4, in gigabytes
* root_password
* extra_opts: to pass on to `pct create`
* ipa_idrange_start, ipa_idrange_start, ipa_idrange_size
* ipa_domain, ipa_password, ipa_username

# Docker setup

Use the following to create a zvol to hold `/var/lib/docker` and get Docker working when using ZFS:

```
---
- hosts: <proxmox_host>
  tasks:
    - name: Create Docker zvol
      zfs:
        name: rpool/user/<hostname>-docker
        state: present
        extra_zfs_properties:
          volsize: 32G
    - name: Format Docker zvol
      shell: "test -f /etc/ansible/mkfs-<hostname>-docker || mkfs.ext4 /dev/zvol/rpool/user/<hostname>-docker && touch /etc/ansible/mkfs-<hostname>-docker"
    - name: Mount Docker zvol
      mount:
        path: /mnt/<hostname>-docker
        src: /dev/zvol/rpool/user/<hostname>-docker
        fstype: ext4
        state: mounted
    - name: fix perms Docker zvol
      file:
        path: /mnt/<hostname>-docker
        mode: 0711
        owner: "100000"
        group: "100000"
    - include_role:
        name: create-proxmox-host
      vars:
...
        extra_opts: -features nesting=1,keyctl=1 -mp0 /mnt/dokku-docker,mp=/var/lib/docker
```

Note: as this does not follow the disk naming conventions that Proxmox uses, Proxmox features like snapshots and migration might cease to work.
See #1 for hints about following the disk naming conventions to solve this problem.
