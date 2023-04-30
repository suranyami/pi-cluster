# Raspberry Pi Cluster

[![CI](https://github.com/geerlingguy/pi-cluster/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/pi-cluster/actions?query=workflow%3ACI)

<p align="center"><a href="https://www.youtube.com/watch?v=kgVz4-SEhbE"><img src="images/turing-pi-2-hero.jpg?raw=true" width="500" height="auto" alt="Turing Pi 2 - Raspberry Pi Compute Module Cluster" /></a></p>

This repository contains examples and automation used in various Raspberry Pi clustering scenarios, as seen on [Jeff Geerling's YouTube channel](https://www.youtube.com/c/JeffGeerling).

<p align="center"><a href="https://www.youtube.com/watch?v=ecdm3oA-QdQ"><img src="images/deskpi-super6c-running.jpg?raw=true" width="500" height="auto" alt="DeskPi Super6c Mini ITX Raspberry Pi Compute Module Cluster" /></a></p>

The inspiration for this project was my first Pi cluster, the [Raspberry Pi Dramble](https://www.pidramble.com), which is still running in my basement to this day!

## Usage

  1. Make sure you have [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed.
  2. Copy the `example.hosts.ini` inventory file to `hosts.ini`. Make sure it has the `manager` and `node`s configured correctly (for my examples I named my nodes `node[1-4].local`).
  3. Copy the `example.config.yml` file to `config.yml`, and modify the variables to your liking.

### Raspberry Pi Setup

I am running Raspberry Pi OS on various Pi clusters. You can run this on any Pi cluster, but I tend to use Compute Modules without eMMC ('Lite' versions) and I often run them using [32 GB SanDisk Extreme microSD cards](https://amzn.to/3G35QbY) to boot each node. For some setups (like when I run the [Compute Blade](https://computeblade.com) or [DeskPi Super6c](https://deskpi.com/collections/deskpi-super6c), I boot off NVMe SSDs instead.

In every case, I flashed Raspberry Pi OS (64-bit, lite) to the storage devices using Raspberry Pi Imager.

To make network discovery and integration easier, I edit the advanced configuration in Imager, and set the following options:

  - Set hostname: `node1.local` (set to `2` for node 2, `3` for node 3, etc.)
  - Enable SSH: 'Allow public-key', and paste in my public SSH key(s)
  - Configure wifi: (ONLY on node 1, if desired) enter SSID and password for local WiFi network

After setting all those options, making sure only node 1 has WiFi configured, and the hostname is unique to each node (and matches what is in `hosts.ini`), I inserted the microSD cards into the respective Pis, or installed the NVMe SSDs into the correct slots, and booted the cluster.

### SSH connection test

To test the SSH connection from my Ansible controller (my main workstation, where I'm running all the playbooks), I connected to each server individually, and accepted the hostkey:

```
ssh pi@node1.local
```

This ensures Ansible will also be able to connect via SSH in the following steps. You can test Ansible's connection with:

```
ansible all -m ping
```

It should respond with a 'SUCCESS' message for each node.

### Storage Configuration

> **Warning**: This playbook is configured to set up a ZFS mirror volume on node 3, with two drives connected to the built-in SATA ports on the Turing Pi 2.
>
> It is not yet genericized for other use cases (e.g. boards that are _not_ the Turing Pi 2).

This playbook will create a storage location on node 3 by default. You can use one of the storage configurations by switching the `storage_type` variable from `filesystem` to `zfs` in your `config.yml` file.

#### Filesystem Storage

If using filesystem (`storage_type: filesystem`), make sure to use the appropriate `storage_nfs_dir` variable in `config.yml`.

### Upgrading the cluster

Run the upgrade playbook:

```
ansible-playbook upgrade.yml
```

### Monitoring the cluster

Prometheus and Grafana are used for monitoring. Grafana can be accessed via port forwarding (or you could choose to expose it another way).

To access Grafana:

  1. Make sure you set up a valid `~/.kube/config` file (see 'K3s installation' above).
  1. Run `kubectl port-forward service/cluster-monitoring-grafana :80`
  1. Grab the port that's output, and browse to `localhost:[port]`, and bingo! Grafana.

The default login is `admin` / `prom-operator`, but you can also get the secret with `kubectl get secret cluster-monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -D`.

You can then browse to all the Kubernetes and Pi-related dashboards by browsing the Dashboards in the 'General' folder.

### Benchmarking the cluster

See the README file within the `benchmarks` folder.

### Shutting down the cluster

The safest way to shut down the cluster is to run the following command:

```
ansible all -m community.general.shutdown -b
```

> Note: If using the SSH tunnel, you might want to run the command _first_ on nodes 2-4, _then_ on node 1. So first run `ansible 'all:!manager' [...]`, then run it again just for `manager`.

Then after you confirm the nodes are shut down (with K3s running, it can take a few minutes), press the cluster's power button (or yank the Ethernet cables if using PoE) to power down all Pis physically. Then you can switch off or disconnect your power supply.

### Static network configuration (optional, but recommended)

I using my cluster both on-premise and remote (using a 4G LTE modem connected to the first Pi), I set it up on its own subnet (10.1.1.x). You can change the subnet that's used via the `ipv4_subnet_prefix` variable in `config.yml`.

To configure the local network for the Pi cluster (this is optional—you can still use the rest of the configurations without a custom local network), run the playbook:

```
ansible-playbook networking.yml
```

After running the playbook, until a reboot, the Pis will still be accessible over their former DHCP-assigned IP address. After the nodes are rebooted, you will need to make sure your workstation is connected to an interface using the same subnet as the cluster (e.g. 10.1.1.x).

> Note: After the networking changes are made, since this playbook uses DNS names (e.g. `node1.local`) instead of IP addresses, your computer will still be able to connect to the nodes directly—assuming your network has IPv6 support. Pinging the nodes on their new IP addresses will _not_ work, however. For better network compatibility, it's recommended you set up a separate network interface on the Ansible controller that's on the same subnet as the Pis in the cluster:
>
> On my Mac, I connected a second network interface and manually configured its IP address as `10.1.1.10`, with subnet mask `255.255.255.0`, and that way I could still access all the nodes via IP address or their hostnames (e.g. `node2.local`).

Because the cluster subnet needs its own router, node 1 is configured as a router, using `wlan0` as the primary interface for Internet traffic by default. The other nodes get their Internet access through node 1.

#### Switch between 4G LTE and WiFi (optional)

The network configuration defaults to an `active_internet_interface` of `wlan0`, meaning node 1 will route all Internet traffic for the cluster through it's WiFi interface.

Assuming you have a [working 4G card in slot 1](https://www.jeffgeerling.com/blog/2022/using-4g-lte-wireless-modems-on-raspberry-pi), you can switch node 1 to route through an alternate interface (e.g. `usb0`):

  1. Set `active_internet_interface: "usb0"` in your `config.yml`
  2. Run the networking playbook again: `ansible-playbook networking.yml`

You can switch back and forth between interfaces using the steps above.

#### Reverse SSH and HTTP tunnel configuration (optional)

For my own experimentation, I ran my Pi cluster 'off-grid', using a 4G LTE modem, as mentioned above.

Because my mobile network provider uses CG-NAT, there is no way to remotely access the cluster, or serve web traffic to the public internet from it, at least not out of the box.

I am using a reverse SSH tunnel to enable direct remote SSH and HTTP access. To set that up, I configured a VPS I run to use TCP Forwarding (see [this blog post for details](https://www.jeffgeerling.com/blog/2022/ssh-and-http-raspberry-pi-behind-cg-nat)), and I configured an SSH key so node 1 could connect to my VPS (e.g. `ssh my-vps-username@my-vps-hostname-or-ip`).

Then I set the `reverse_tunnel_enable` variable to `true` in my `config.yml`, and configured the VPS username and hostname options.

Doing that and running the `main.yml` playbook configures `autossh` on node 1, and will try to get a connection through to the VPS on ports 2222 (to node 1's port 22) and 8080 (to node 1's port 80).

After that's done, you should be able to log into the cluster _through_ your VPS with a command like:

```
$ ssh -p 2222 pi@[my-vps-hostname]
```

> Note: If autossh isn't working, it could be that it didn't exit cleanly, and a tunnel is still reserving the port on the remote VPS. That's often the case if you run `sudo systemctl status autossh` and see messages like `Warning: remote port forwarding failed for listen port 2222`.
>
> In that case, log into the remote VPS and run `pgrep ssh | xargs kill` to kill off all active SSH sessions, then `autossh` should pick back up again.

> **Warning**: Use this feature at your own risk. Security is your own responsibility, and for better protection, you should probably avoid directly exposing your cluster (e.g. by disabling the `GatewayPorts` option) so you can only access the cluster while already logged into your VPS).

## Caveats

These playbooks are used in both production and test clusters, but security is _always_ your responsibility. If you want to use any of this configuration in production, take ownership of it and understand how it works so you don't wake up to a hacked Pi cluster one day!

## Author

The repository was created in 2023 by [Jeff Geerling](https://www.jeffgeerling.com), author of [Ansible for DevOps](https://www.ansiblefordevops.com), [Ansible for Kubernetes](https://www.ansibleforkubernetes.com), and [Kubernetes 101](https://www.kubernetes101book.com).
