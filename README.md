# Ansible Role: Nomad

This role installs and configures a Nomad cluster on Linux systems that use systemd.
There are a few assumptions made by this role.

The first is that you are using Nomad with alongside a Consul cluster as this is the easiest way to configure Nomad (and gives you that sweet, sweet service discovery).

Another assumption is that you will have separate Nomad servers and clients - this is the recommended way to deploy a Nomad in a production environment.

Finally, it assumes that if you wanted to enable and leverage HashiCorp Vault integration, you are likely following the official guides to integrating Vault into Nomad (particularly around mTLS and Vault token retrieval and renewal).

The role has a few optional features locked behind boolean variables that act as 'feature-flags', and they are set to false by default.
To enable them, simply set the relevant variable to 'true' and the 'feature-flag' will be enabled.
An example of this is the HashiCorp Vault integration, which is hidden behind the `nomad_vault_enabled` variable.

All role variables are documented in `defaults/main.yml` with comments explaining (and examples showing) their usage.

## Requirements

For the Ansible controller:
* The `netaddr` python package
* The `unzip` system package

For the target hosts/environment:
* Linux OS
* systemd
* A Consul cluster
* (Optional) A Vault cluster with the PKI secrets engine enabled

## Dependencies

If you do not already have a Consul cluster installed and configured, you can use my [Ansible role for consul](https://github.com/brickstool/ansible-role-consul) which can do this for you.

If you want to use consul-template, then you will also require that too  - my [Ansible role for consul-template](https://github.com/brickstool/ansible-role-consul-template) will do this for you.

You can install both roles via `ansible-galaxy` like so:

```sh
ansible-galaxy install brickstool.consul
ansible-galaxy install brickstool.consul_template
```

The consul-template role installs a instantiated systemd service template for consul-template.
The unit file will look a little like the following:

```ini
[Unit]
Description=consul-template for %I
Requires=network-online.target consul.service
After=network-online.target consul.service vault.service nomad.service

[Service]
ExecStart=/usr/local/bin/consul-template -config=/etc/consul-template.d/%I.hcl
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -INT $MAINPID
PIDFile=/run/consul-template-%I.pid
Restart=always
KillMode=process
KillSignal=SIGINT
RestartSec=60s
LimitNOFILE=4096
TimeoutSec=5

[Install]
WantedBy=multi-user.target
```

Notice the use of `%I` in the example?
That allows for dynamically specifying a configuration file located in `/etc/consul-template.d` when starting the service.
Using the example configuration file in this role (`templates/ctpl.nomad.hcl.j2`) along with the `systemd` unit file block above, you would execute the following command to enable and start the service:

```sh
$ sudo systemctl enable --now consul-template@nomad.service
```

## Example Playbook

The following examples are the minimum configuration you would need to successfully run this role.
It is assumed you already have a working Consul cluster and that each node has a local Consul agent configured.
Generate an actual encryption key for `nomad_encrypt_string` using `nomad operator keygen`, replacing "encryptme123" with the generated key in the examples below.

*For a group of 3 Nomad server nodes*:

```yaml
- hosts: nomad-servers
  become: yes
  roles:
    - role: brickstool.nomad
      vars:
        nomad_server: true #Setting this to true automatically sets `nomad_client` to false (unless otherwise specified)
        nomad_bootstrap_expect: 3
        nomad_encrypt_string: 'encryptme123='
```

*For a group of Nomad client nodes*:

```yaml
- hosts: nomad-clients
  become: yes
  roles:
    - role: brickstool.nomad
      vars:
        nomad_client: true #Assumed true by default, included here for clarity
        nomad_encrypt_string: 'encryptme123='
```

## License

MIT / BSD

## Author Information

Created by Samuel Noordhuis in 2020. Inspired heavily by the Ansible roles and writings of [Jeff Geerling](https://github.com/geerlingguy).

If you see any errors or think this role could be improved in some way, you are welcome to open an issue/feature request or create a pull request :)

## TO-DO

1. Add molecule tests
