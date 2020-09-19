# Wireguard Site-to-Site VPN

The purpose is where you have two sites, each with an EdgeOS router at the edge,
and you want to establish a VPN between them so they appear to be on the same
LAN.

The following steps need to be take on the router for each site. This assumes
you're logged in to the route via SSH and are comfortable using the `configure`
CLI.

## Install Wireguard

1. The [WireGuard for Ubiquiti](WireGuard/wireguard-vyatta-ubnt) project has
`.deb` releases available on their
[Releases](https://github.com/WireGuard/wireguard-vyatta-ubnt/releases) page.
Download the proper `.deb` file for the model router you have.
   `$ curl -s -L -o wireguard.deb
   https://github.com/WireGuard/wireguard-vyatta-ubnt/releases/download/1.0.20200908-1/e50-v2-v1.0.20200908-v1.0.20200827.deb`
1. Install the package.
   `$ sudo dpkg -i wireguard.deb`
1. Test installation by making sure the `wg` CLI was installed.
   `$ which wg`

## Generate Keys

1. Generate your private key.
   `$ wg genkey > /config/auth/wg.key`
1. Generate your public key.
   `$ cat /config/auth/wg.key | wg pubkey > /config/user-data/wg.public`
1. Set permissions on the private key.
   `$ chmod 0600 /config/auth/wg.key`
   `$ chown root /config/auth/wg.key`

## Allow WireGuard through the firewall

This assumes you're using the standard port `51820` for WireGuard; if not,
replace any instance in this document you see that value with your desired port
number.

1. `configure`
1. `set firewall name WAN_LOCAL rule 20 action accept`
1. `set firewall name WAN_LOCAL rule 20 protocol udp`
1. `set firewall name WAN_LOCAL rule 20 description 'WireGuard'`
1. `set firewall name WAN_LOCAL rule 20 destination port 51820`
1. `commit; save; exit`

## Decide on an address scheme

Each router will have/need 3 IP addresses, 2 of which (Public and Private)
should already be established:
1. Public - this is the Internet-facing public address. NOTE: if you don't have
a static IP assigned to your Public address, you may need to use some form of
dynamic DNS so you can route between sites using hostnames.
1. Private - this is your internal LAN address, for example `192.168.1.1`
1. VPN - this is the address for the local endpoint of the VPN. It should also
be a local address (in the `192.168.x.x`, `172.[16-31].x.x` or `10.x.x.x`
ranges). I typically take an un-allocated LAN segment that's toward the other
end of the range: `192.168.33.x`, for example. Each router will need a different
VPN address on the same network range.

NOTE: Each of your sites should be on different LAN segments. If Site A is
`192.168.1.x`, then Site B should be `192.168.2.x`.

## Configure the interface

You will need:
- `${LOCAL_ADDRESS}` - this is the `/24` CIDR of your router's VPN address. So
  if your router's VPN address is `192.168.33.1`, this is `192.168.33.1/24`.

1. `configure`
1. `set interfaces wireguard wg0 address ${LOCAL_ADDRESS}`
1. `set interfaces wireguard wg0 listen-port 51820`
1. `set interfaces wireguard wg0 route-allowed-ips true`
1. `set interfaces wireguard wg0 private-key /config/auth/wg.key`
1. `commit; save; exit`

## Configure the VPN connection between each site

You will need the following for each site you want to connect the local router
to:
- `${PEER_KEY}` - this is the WireGuard public key for the *other site*.
- `${PEER_HOSTNAME}` - this is the Public address for the *other site*. This can
  also be a hostname, so if you're using dynamic DNS you would use that
  hostname for the remote/peer site.
- `${PEER_ADDRESS}` - this is the `/32` CIDR of the VPN address for the *other
  site*. If the remote router's VPN address is `192.168.33.2`, this is
  `192.168.33.2/32`.
- `${REMOTE_NETWORK}` - this is the CIDR for the LAN segment of the *other
  site*. AKA the network the remote/peer's Private address is on. If the remote
  site's network is `192.168.2.x`, then this would be `192.168.2.0/24`.

1. `configure`
1. `set interfaces wireguard wg0 peer ${PEER_KEY} allowed-ips ${PEER_ADDRESS}`
1. `set interfaces wireguard wg0 peer ${PEER_KEY} allowed-ips ${REMOTE_NETWORK}`
1. `set interfaces wireguard wg0 peer ${PEER_KEY} endpoint
${PEER_HOSTNAME}:51820`
1. `set interfaces wireguard wg0 peer ${PEER_KEY} persistent-keepalive 25`
1. `commit; save; exit`

## Check the VPN status

1. `sudo wg`
   This should show the remote/peer connections, with non-zero values going both
   ways under the `transfer:` line.
1. Try using `ping` on local Private and VPN addresses, and remote VPN and
Private addresses. You may need to adjust your routing table by adding the VPN
address for each remote/peer connection (`${PEER_ADDRESS}` here is *not* a CIDR,
it's just an IP address):
   `ip route add ${PEER_ADDRESS} dev wg0 scope link`
1. Try using `ping` on an address that's on the remote/peer network (ie, not the
remote/peer router itself, but a machine that's behind it).
