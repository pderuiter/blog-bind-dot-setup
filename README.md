# blog-bind-dot-setup

Companion code for the blog post
[Setting Up DNS Over TLS (DoT) on BIND](https://blog.bsdserver.nl/setting-up-dns-over-tls-dot-on-bind/).

The post was written for BIND on OpenBSD. This repository automates the Linux variant on Ubuntu 24.04:

- BIND 9.20 from the ISC packages (`ppa:isc/bind`); forwarding over TLS needs 9.20 and Ubuntu ships 9.18
- the `tls wbyc` block in its own file (`/etc/bind/tls_certs`), included from `named.conf.options`
- plain DNS on port 53 and DNS over TLS on port 853 for the same networks (`listen-on` and
  `listen-on tls wbyc`, IPv4 and IPv6)
- one `forwarders` statement with a `tls` name per server: OpenDNS, Cloudflare and Quad9 over port 853,
  verified against the system CA bundle and each provider's certificate name
- a self-signed certificate for `example.com` (or your own certificate and key)

### Differences from the OpenBSD post

| OpenBSD (post) | Linux (this repo) |
| --- | --- |
| `/var/named/etc/tls_certs`, included as `/etc/tls_certs` (chroot) | `/etc/bind/tls_certs` |
| `/etc/ssl/example.com.fullchain.pem`, `/etc/ssl/private/example.com.key` | `/etc/bind/tls/example.com.fullchain.pem`, `/etc/bind/tls/example.com.key` |
| CA bundle `/etc/ssl/cert.pem` | `/etc/ssl/certs/ca-certificates.crt` |
| user `_bind`, `rcctl restart isc_named` | user `bind`, `systemctl restart named` |

The certificate and key live in `/etc/bind/tls` because Ubuntu's AppArmor profile for `named` only
allows reads below `/etc/bind`, and `/etc/ssl/private` is not accessible to the `bind` user.

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04
- Outbound TCP 853 to the DoT providers

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configure BIND
cd ../ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Run the post's checks (plain DNS, dig +tls, forwarded queries over DoT, no TLS verification errors)
ansible-playbook ../tests/verify.yml
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `bind_dot_setup_listen_v4` | 127.0.0.1, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | Addresses/networks for the port 53 and port 853 listeners |
| `bind_dot_setup_listen_v6` | `any` | IPv6 listeners |
| `bind_dot_setup_tls_name` | `wbyc` | Name of the server `tls` block |
| `bind_dot_setup_cert_source` | `selfsigned` | `selfsigned` or `files` |
| `bind_dot_setup_cert_file` / `bind_dot_setup_key_file` | `""` | Certificate (full chain) and key on the controller, for `files` |
| `bind_dot_setup_server_name` | `example.com` | Name in the self-signed certificate |
| `bind_dot_setup_upstreams` | OpenDNS, Cloudflare, Quad9 | DoT forwarders: `tls` name, `remote-hostname` and addresses |
| `bind_dot_setup_ca_file` | `/etc/ssl/certs/ca-certificates.crt` | CA bundle for the upstream certificates |
| `bind_dot_setup_forward` | `""` | Empty keeps BIND's default (`forward first`); `only` never falls back to recursion |
| `bind_install_source` | `isc` | ISC packages, needed for BIND 9.20 |

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND 9.20 from the ISC packages
- `ansible/roles/bind_dot_setup` writes the certificate, the `tls_certs` file and the DoT configuration
- `tests/verify.yml` runs the functional checks from the post

## License

MIT
