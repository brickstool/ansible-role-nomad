<p align="center">
  <a href="https://github.com/snoord/ansible-role-nomad/actions">
    <img alt="GitHub Actions" src="https://github.com/snoord/ansible-role-nomad/workflows/build/badge.svg?branch=master">
  </a>
  <a href="https://github.com/semantic-release/semantic-release">
    <img alt="semantic-release" src="https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg">
  </a>
</p>

<h1 align="center" style="border-bottom: none;">Ansible Role: Nomad</h1>


This role installs and configures a Nomad cluster on Linux systems that use `systemd`.
A few assumptions made by this role are that:

* Nomad will be installed inside of a Consul cluster
* Nomad servers will be separate from Nomad clients (this is the recommended way to deploy a Nomad in a production environment)
* (Optional) The Vault instance/cluster present is configured as per the official Nomad guides for [Vault](https://www.nomadproject.io/docs/integrations/vault-integration/) [integration](https://learn.hashicorp.com/nomad/vault-integration/vault-pki-nomad)

The role has a few optional features locked behind boolean variables that act as 'feature-flags'.
They are set to false by default.
To enable them, simply set the relevant variable to 'true' and the 'feature-flag' will be enabled.

An example of this is the HashiCorp Vault integration, which is hidden behind the `nomad_vault_enabled` variable.

All role variables are documented in `defaults/main.yml` with comments explaining (and examples showing) their usage.

## Requirements

For the Ansible controller:
* The `netaddr` python package
* The `unzip` system package

For the target hosts/environment:
* Linux
* `systemd`
* Consul cluster
* (Optional) A Vault cluster with the PKI secrets engine enabled

## Dependencies

If you do not already have a Consul cluster installed and configured, you can use my [Ansible role for Consul](https://github.com/snoord/ansible-role-consul) to create one.

If you want to use `consul-template`, then you will also require that too  - my [Ansible role for `consul-template`](https://github.com/snoord/ansible-role-consul-template) will do this for you.

You can install both roles via `ansible-galaxy` like so:

```sh
ansible-galaxy install snoord.consul
ansible-galaxy install snoord.consul_template
```

The `consul-template` role installs a instantiated `systemd` service template for `consul-template`.
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
It is assumed you have a working Consul cluster, and each node has a Consul agent running locally.
Generate an actual encryption key for `nomad_encrypt_string` using `nomad operator keygen`, replacing "encryptme123" with the generated key in the examples below.

*For a group of 3 Nomad server nodes*:

```yaml
- hosts: nomad-servers
  become: yes
  roles:
    - role: snoord.nomad
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
    - role: snoord.nomad
      vars:
        nomad_client: true #Assumed true by default, included here for clarity
        nomad_encrypt_string: 'encryptme123='
```

## License

MIT / BSD

## Author Information

Created by Samuel Noordhuis in 2020. Inspired heavily by the Ansible roles and writings of [Jeff Geerling](https://github.com/geerlingguy).

If you see any errors or think this role could be improved in some way, you are welcome to open an issue/feature request or create a pull request :)
