# Fold

**Note: I'm no longer maintaining this project. Let me know if you wish to contribute.**

Run virtual machines on a virtual network created by [Weave](https://github.com/zettio/weave).

- Requires Weave (of course).
- Requires [my fork of nfdhcpd](https://github.com/davedoesdev/nfdhcpd) in order to provide each VM with address and configuration data.
- Like Weave each subnet is fully isolated &mdash; Fold sets up the necessary firewall rules.
- Supports custom configuration data per VM using DNS TXT records. Useful for service discovery in multi-VM applications.
- Mix docker containers with VMs - they can talk to each other over the same Weave network.
- Interoperates with weaveDNS: VMs can lookup Weave records and containers can lookup Fold records.
- Full IPv6 support.
- Supports [KVM](http://www.linux-kvm.org/page/Main_Page) and [Rumprun](https://github.com/rumpkernel/rumprun) for lightweight VMs.

The motivation for Fold is to support applications you don't trust enough to run in containers. For example, multi-tenant environments or when you've pulled in many dependencies and haven't audited all the third party modules your application uses.

# Usage

## Setting up Weave network

You need to create a Weave network (`weave launch`) before running Fold. Please see the [Weave documentation](https://github.com/zettio/weave) for details.

Fold supports Weave [automatic IP address management (IPAM)](http://docs.weave.works/weave/latest_release/ipam.html) and [mixed automatic and manual allocation](http://docs.weave.works/weave/latest_release/ipam.html#manual).

When using IPAM, allocate an IP address using:

```shell
fold -i <iface-name> -4 auto [net:<cidr> | net:default]
```

You must supply an interface name &mdash; Weave will assign the address to this name. Supply a CIDR if you don't want to use Weave's default subnet. The allocated IP address will be written to standard output.

To release the IP address later, after running your VM, you can use:

```shell
fold -i <iface-name> -4 release
```

When using mixed allocation, give your VMs IP addresses from the manual allocation range.

If you want your Weave containers to be able to make DNS queries against Fold, use the Docker `--dns` option when launching your containers. For example, if you're going to run a VM with IP address `10.0.1.100` and want to make DNS queries against it then use `docker run --dns=10.0.1.100 ...`

## Running VMs using Fold

Once you've setup your Weave network, use the `fold` command to run virtual machines on the network:

```shell
fold [-i <iface-name>]
     [-4 <ipaddr>/<subnet>]
     [-6 <ip6prefix>]
     [-s]
     [-m <multicast-addr>]...
     [-k <rdiscd-keyfile>]
     [-d <rdiscd-img-dir>]
     [-b <nfdhcpd-binding-file>]
     [-a <iptables-gw-action>]
     [-A <ip6tables-gw-action>]
     <macaddr>
     kvm|qemu-system-* [<hypervisor arg>...]
```

You should supply at least one of `-4` or `-6`.

**`-i`** names the interface created for the VM and attached to the Weave network bridge. If you don't supply one then Fold makes a name up for you. If you're using IPAM then use the same name you used to allocate the IP address.

**`-4`** gives the VM an IPv4 address and subnet on the Weave network. For example, `-4 10.0.1.100/24`. The VM will be able to communicate with other VMs on the same subnet. If you're using IPAM, use the IP allocated to the interface name.

**`-6`** gives the VM an IPv6 address and prefix on the Weave network. For example, `-6 fde5:824d:d315:3bb1::/64`. The VM will be able to communicate with other VMs with the same prefix. Your VM's guest OS is assumed to be using [Modified EUI-64](http://tools.ietf.org/html/rfc4291#section-2.5.1), unless you specify the `-s` or `-k` options.

**`-s`** sets the VM's IPv6 address to the full 128-bit value you specified with `-6` (i.e. with the prefix specifier removed).

**`-m`** allows the VM to receive IPv4 or IPv6 multicast packets for a given address. You can repeat `-m` more than once, for example `-m 239.1.2.3 -m ff3e::4321:1234`.

**`-k`** specifies that the VM should use an IPv6 [RFC 7217](https://tools.ietf.org/html/rfc7217) stable privacy address. Fold uses [rdiscd](https://github.com/AGWA/rdiscd) to generate the address. `rdiscd-mkaddress` should be available in your `PATH`.

The argument to `-k` is the file containing the secret key used to generate the stable privacy address. Fold puts the stable privacy address into a file called `interface-id` on a temporary ext2fs disk image. The path to the image file will be substituted for `_rdiscd_img_file_` in the rest of the arguments. For example:

```shell
fold ... -k keyfile ... kvm -fda _rdiscd_img_file_
```

Note that Fold doesn't support IPv6 [RFC 4941](http://tools.ietf.org/html/rfc4941) temporary addresses so you should disable these in your VM's guest operating system.

**`-d`** specifies the directory in which to write the file containing the stable privacy address (only if `-k` is specified). Defaults to the current directory.

**`-b`** specifies a binding file containing configuration data for `nfdhcpd`. It will be used by `nfdhcpd` to supply information such as a hostname and DNS address and TXT records to the VM's guest OS. Please see the [nfdhcpd documentation](https://github.com/davedoesdev/nfdhcpd) for more information. Here's an example:

```ini
HOSTNAME=test
GATEWAY=10.0.1.90
GATEWAY_MAC=7a:f4:96:e0:1c:a0
GATEWAY6=fde5:824d:d315:3bb1::91
GATEWAY6_MAC=7e:b7:70:3b:81:77
NAMESERVERS=10.0.1.50
NAMESERVERS6=fde5:824d:d315:3bb1::1
ADDRESS:database.=10.0.1.1
ADDRESS6:database.=fde5:824d:d315:3bb1::9
TXT:table.=Users
DNS_NAMESERVERS=172.17.42.1
```

Fold will add `IPADDR`, `SUBNET`, `PREFIX`, `IP6ADDR` and `BRIDGE` to the settings in your binding file.

Note that if you define an IPv4 gateway, Fold requires a setting which `nfdhcpd` doesn't use: `GATEWAY_MAC`. Fold needs the MAC of your gateway in order to make sure traffic coming in and out of the subnet (or prefix for IPv6) goes via the gateway only.

One way to get a gateway on your Weave network is to use [Weave host network integration](http://docs.weave.works/weave/latest_release/features.html#host-network-integration). Use `weave expose` to get the gateway IP address and `weave ps weave:expose` to get the gateway MAC.

If you want your VM to be able to make queries against weaveDNS, you need to set `DNS_NAMESERVERS` to the IP address of the _Docker_ bridge device (usually `docker0`). Use `ip addr show dev docker0` or `ifconfig docker0` and parse the output to find the IP address. For example:

```shell
ifconfig docker0 | grep 'inet addr:' | cut -d: -f2  | awk '{print $1}'
```

**`-a`** specifies the `iptables` action to take for packets outside the subnet coming from or going to the VM's IP address (via the gateway, if you defined one). The default is `-a "-j RETURN"` (resume with non-Fold processing) but you can use `-a "-g ..."` or `-a "-j ..."` to continue processing using a chain you've defined.

**`-A`** does the same as `-a` but for `ip6tables` and IPv6 packets.

**`<macaddr>`** is a (mandatory) MAC address to give the VM. Note you can use the `utils/lamac` script to generate a random [locally administered address](http://en.wikipedia.org/wiki/MAC_address#Address_details).

**`<hypervisor arg>...`** should contain arguments necessary to load your VM, for example:
  - `-hda disk.qcow2` (KVM)
  - `-kernel myrumpkernel.bin -append '{"net": {"if": "vioif0",, "type": "inet",, "method":"dhcp"},, "cmdline": "myapp"}'` (Rumprun).

## Environment variables

Fold uses `iptables`, `ip6tables` and `ebtables`. To change how Fold invokes these commands, set the `IPTABLES`, `IP6TABLES` and `EBTABLES` environment variables respectively. For example, to support concurrent use:

```shell
IPTABLES='iptables --wait' IP6TABLES='ip6tables --wait' EBTABLES='ebtables --concurrent' fold ...
```

By default, Fold expects to find the global `nfdhcpd` configuration file at `/etc/nfdhcpd/nfdhcpd.conf`. You can override this by setting `NFDHCPD_CFG`.

# Examples

Run a Ubuntu VM on the Weave network, with IPv4 and IPv6 addresses (IPv6 using EUI-64):

```shell
fold -4 10.0.1.100/24 -6 fde5:824d:d315:3bb1::/64 -b test-binding 52:54:00:12:34:56 kvm -hda ubuntu.qcow2
```

Run a Ubuntu VM on the Weave network, with IPv4 and IPv6 unicast and multicast addresses (IPv6 unicast address using stable privacy):

```shell
fold -4 10.0.1.100/24 -6 fde5:824d:d315:3bb1::/64 -k keyfile -m 239.1.2.3 -b test-binding 52:54:00:12:34:56 kvm -hda ubuntu.qcow2 -fda _rdiscd_img_file_
```

Run a Rumprun kernel on a Weave network (10.9.200.1/16):

```shell
fold -4 10.9.200.1/16 52:54:00:12:34:56 kvm -kernel rumprun-packages/nodejs/build-4.3.0/out/Release/node-hello-world.bin -append '{"net": {"if": "vioif0",, "type": "inet",, "method":"dhcp"},, "cmdline": "node"}' -m 256
```
