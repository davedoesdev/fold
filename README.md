# Fold

Run virtual machines on a virtual network created by [Weave](https://github.com/zettio/weave).

- Requires Weave (of course).
- Requires [my fork of nfdhcpd](https://github.com/davedoesdev/nfdhcpd) in order to provide each VM with address and configuration data.
- Like Weave each subnet is fully isolated &mdash; Fold sets up the necessary firewall rules.
- Supports custom configuration data per VM using DNS TXT records. Useful for service discovery in multi-VM applications.
- Mix docker containers with VMs - they can talk to each other over the same Weave network.
- Full IPv6 support.
- Supports [KVM](http://www.linux-kvm.org/page/Main_Page) only at the moment.

The motivation for Fold is to support applications you don't trust enough to run in containers. For example, multi-tenant environments or when you've pulled in many dependencies and haven't audited all the third party modules your application uses.

# Usage

You need to create a Weave network (`weave launch`) before running Fold. Please see the [Weave documentation](https://github.com/zettio/weave) for details.

Once you've done this, use the `fold` command to run virtual machines on the network:

```shell
fold [-4 <ipaddr>/<subnet>]
     [-6 <ip6prefix>]
     [-m <multicast-addr>]...
     [-k <rdiscd-keyfile>]
     [-d <rdiscd-img-dir>]
     <nfdhcp-binding-file>
     <macaddr>
     kvm [<kvm args> ...]
```

You should supply at least one of `-4` or `-6`.

*`-4`* gives the VM an IPv4 address and subnet on the Weave network. For example, `-4 10.0.1.100/24`. The VM will be able to communicate with other VMs on the same subnet.

*`-6`* gives the VM an IPv6 address and prefix on the Weave network. For example, `-6 fde5:824d:d315:3bb1::/64`. The VM will be able to communicate with other VMs with the same prefix.

*`-m`* allows the VM to receive IPv4 or IPv6 multicast packets for a given address. You can repeat `-m` more than once, for example `-m 239.1.2.3 -m ff3e::4321:1234`.

*`-k`* specifies that the VM should use an IPv6 [RFC 7217](https://tools.ietf.org/html/rfc7217) stable privacy address. Fold uses [rdiscd](https://github.com/AGWA/rdiscd) to generate the address. `rdiskd-mkaddress` should be available in your `PATH`.

The argument to `-k` is the file containing the secret key used to generate the stable privacy address. Fold puts the stable privacy address into a file called `interface-id` on a temporary ext2fs disk image. The path to the image file will be substituted for `[rdisk_img_file]` in the rest of the arguments. For example:

```shell
fold ... -k keyfile ... kvm -fda [rdisk_img_file]
```

Note that Fold doesn't support IPv6 [RFC 4941](http://tools.ietf.org/html/rfc4941) temporary addresses so you should disable these in your VM's guest operating system.

If you don't specify `-k` then your VM's guest OS is assumed to be using [Modified EUI-64](http://tools.ietf.org/html/rfc4291#section-2.5.1).

*`-d`* specifies the directory in which to write the temporary disk image file (only if `-k` is specified). Defaults to the current directory.

`<nfdhcp-binding-file>` is the (mandatory) binding file containing configuration data for `nfdhcpd`. It will be used by `nfdhcpd` to supply information such as a hostname and DNS address and TXT records to the VM's guest OS. Please see the [nfdhcpd documentation](https://github.com/davedoesdev/nfdhcpd) for more information. Here's an example:

```
HOSTNAME=test
NAMESERVERS=10.0.1.50
NAMESERVERS6=fde5:824d:d315:3bb1::1
ADDRESS:database.=10.0.1.1
ADDRESS6:database.=fde5:824d:d315:3bb1::9
```

Fold will add `IPADDR`, `SUBNET`, `PREFIX` and `IP6ADDR` to the settings in your binding file.

`<macaddr>` is a (mandatory) MAC address to give the VM. Note you can use the `utils/lamac` script to generate a random [locally administered address](http://en.wikipedia.org/wiki/MAC_address#Address_details).

`<kvm args>` should contain arguments necessary to load your VM, for example `-hda disk.qcow2`.

# Examples

Run a Ubuntu VM on the Weave network, with IPv4 and IPv6 addresses (IPv6 using EUI-64):

```shell
fold -4 10.0.1.100/24 -6 fde5:824d:d315:3bb1::/64 test-binding 52:54:00:12:34:56 kvm -hda ubuntu.qcow2
```

Run a Ubuntu VM on the Weave network, with IPv4 and IPv6 unicast and multicast addresses (IPv6 unicast address using stable privacy):

```shell
fold -4 10.0.1.100/24 -6 fde5:824d:d315:3bb1::/64 -k keyfile -m 239.1.2.3 test-binding 52:54:00:12:34:56 kvm -hda ubuntu.qcow2 -fda [rdisk_img_file]
```
