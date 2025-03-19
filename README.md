I see your network configuration has changed. You now have two network interfaces:

1. `enp3s0` - This doesn't show an IP address in the output
2. `ens224` - This has IP address 192.168.5.129/24

Let's configure DHCP to work with the `ens224` interface since it has an IP address. Here's what we'll do:

1. First, set the correct interface in the DHCP configuration:

```bash
sudo nano /etc/sysconfig/dhcpd
```

Change it to:
```
DHCPDARGS="ens224"
```

2. Update your DHCP configuration to match your current network settings:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Replace the content with:

```
# Basic DHCP configuration with DDNS
authoritative;

# Define the DDNS key
key "ddns-key" {
  algorithm hmac-sha256;
  secret "YourStaticSecretKey123456789";
}

# DDNS configuration
ddns-update-style interim;
ddns-updates on;
ignore client-updates;

# Domain settings
option domain-name "est.intra";
option domain-name-servers 192.168.5.129;  # Using your ens224 IP

# DDNS zones
zone est.intra. {
  primary 192.168.5.129;  # Using your ens224 IP
  key ddns-key;
}

zone 5.168.192.in-addr.arpa. {  # Notice the change to match 192.168.5.x network
  primary 192.168.5.129;  # Using your ens224 IP
  key ddns-key;
}

# Network configuration - using your ens224 network
subnet 192.168.5.0 netmask 255.255.255.0 {
  range 192.168.5.130 192.168.5.200;  # Range starts after your server IP
  option routers 192.168.5.129;  # Using your ens224 IP
  default-lease-time 3600;
  max-lease-time 7200;
}
```

3. Update your DNS server configuration:

```bash
sudo nano /etc/named.conf
```

Change the listen-on line and any IP references:
```
listen-on port 53 { 127.0.0.1; 192.168.5.129; };
allow-query { localhost; 192.168.5.0/24; };
```

4. Update your zone files:

```bash
sudo nano /var/named/est.intra.zone
```

Change to:
```
$TTL 86400
@       IN      SOA     dns.est.intra. admin.est.intra. (
                        2023032901      ; Serial
                        3600            ; Refresh (1 hour)
                        1800            ; Retry (30 minutes)
                        604800          ; Expire (1 week)
                        86400 )         ; Minimum TTL (1 day)
;
@       IN      NS      dns.est.intra.
dns     IN      A       192.168.5.129
```

5. Create or update the reverse zone file for the 192.168.5.x network:

```bash
sudo nano /var/named/5.168.192.in-addr.arpa.zone
```

Add:
```
$TTL 86400
@       IN      SOA     dns.est.intra. admin.est.intra. (
                        2023032901      ; Serial
                        3600            ; Refresh (1 hour)
                        1800            ; Retry (30 minutes)
                        604800          ; Expire (1 week)
                        86400 )         ; Minimum TTL (1 day)
;
@       IN      NS      dns.est.intra.
129     IN      PTR     dns.est.intra.  ; Your server IP's last octet
```

6. Update the zone reference in named.conf to point to the new reverse zone file:

```bash
sudo nano /etc/named.conf
```

Find the reverse zone section and change it to:
```
zone "5.168.192.in-addr.arpa" IN {
    type master;
    file "5.168.192.in-addr.arpa.zone";
    allow-update { key "ddns-key"; };
};
```

7. Set the correct permissions:

```bash
sudo chown named:named /var/named/est.intra.zone
sudo chown named:named /var/named/5.168.192.in-addr.arpa.zone
```

8. Check your configurations for errors:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo named-checkconf
sudo named-checkzone est.intra /var/named/est.intra.zone
sudo named-checkzone 5.168.192.in-addr.arpa /var/named/5.168.192.in-addr.arpa.zone
```

9. Restart the services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

If you still have issues, run DHCP in debug mode to see the exact error:

```bash
sudo dhcpd -d -f -cf /etc/dhcp/dhcpd.conf
```

This will show you detailed error messages that can help identify the problem.
