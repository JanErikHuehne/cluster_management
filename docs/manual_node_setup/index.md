## Network Connection

Nodes should get their IP via a DHCP service directly.

### Enforcing an IP directly
```bash
ip route | grep default # this gives the current gateway
systyemctl is_active NetworkManager # should say active
nmcli con show # find the connection name e.g. "Wired connection 1"
```
Setting up the connection

```bash
sudo nmcli con mod "<connection_name>" ipv4.method manual ipv4.addresses <address>/24 ipv4.gateway <gateway> ipv4.dns "1.1.1.1"
```

Bringint the connection up

```bash
sudo nmcli con up "<connection name>"
```