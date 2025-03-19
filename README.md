I understand you want the client to take an IP from the first pool that you defined rather than changing the pool. Let's troubleshoot why it's not taking an IP from your intended pool.

Based on your network configuration, I see that your interface ens160 has the IP addresses 192.168.250.179 and 192.168.250.180. This suggests your DHCP server is operating on the 192.168.250.0/24 network.

Let's do the following:

1. First, let's verify what pool you want to use. What is the IP range you want your clients to use?

2. Let's check your current DHCP configuration to see what pools are defined:

```bash
sudo cat /etc/dhcp/dhcpd.conf
```

3. Make sure the DHCP service is configured to listen on the correct interface:

```bash
sudo nano /etc/sysconfig/dhcpd
```

Set it to:
```
DHCPDARGS="ens160"  # Ensure this is the interface connected to the client network
```

4. To fix the client not getting an IP from your desired pool, modify your dhcpd.conf to only include the subnet for your intended pool. For example, if you want clients to use 192.168.1.0/24:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

```
# DHCP with DDNS Configuration
authoritative;

# Define the DDNS key
key "ddns-key" {
  algorithm hmac-sha256;
  secret "YourStaticSecretKey123456789";
}

# DDNS configuration
ddns-update-style interim;
ddns-updates on;
update-static-leases on;
ddns-hostname = concat("client-", binary-to-ascii(10, 8, "-", leased-address));
ddns-domainname "est.intra";
allow unknown-clients;
use-host-decl-names on;

# Domain settings
option domain-name "est.intra";
option domain-name-servers 192.168.250.179;

# DDNS zones
zone est.intra. {
  primary 192.168.250.179;
  key ddns-key;
}

zone 1.168.192.in-addr.arpa. {
  primary 192.168.250.179;
  key ddns-key;
}

# Network configuration
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  default-lease-time 3600;
  max-lease-time 7200;
}

# You must also declare the network where your DHCP server is located
subnet 192.168.250.0 netmask 255.255.255.0 {
  # This is just to tell DHCP server it's on this network
  # No range is defined so no IPs will be assigned from this network
}
```

5. For DDNS to work properly with the 192.168.1.0/24 pool, update your reverse zone in named.conf:

```bash
sudo nano /etc/named.conf
```

```
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.in-addr.arpa.zone";
    allow-update { key "ddns-key"; };
};
```

6. Create or update the reverse zone file:

```bash
sudo nano /var/named/1.168.192.in-addr.arpa.zone
```

```
$TTL 86400
@       IN      SOA     dns.est.intra. admin.est.intra. (
                        2023032902      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL
;
@       IN      NS      dns.est.intra.
```

7. Set correct permissions:

```bash
sudo chown named:named /var/named/1.168.192.in-addr.arpa.zone
```

8. Verify your network interfaces and routing to ensure the clients can reach the DHCP server and receive offers from the correct subnet.

9. Restart services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

If your client is still not getting an IP from the desired pool, you may need to check network connectivity between the client and server, or consider if there's another DHCP server on the network interfering with yours.
