I see that you're making progress with connectivity - you can ping both 192.168.1.1 and 192.168.1.101, but the DDNS (Dynamic DNS) portion isn't working properly. The client isn't automatically getting registered in the DNS zone files as we can see from the error when you tried `nslookup 192.168.1.101`.

Let's fix the DDNS configuration to ensure client hostnames are automatically added to the DNS zone files when they get IP addresses. Here's what we need to do:

1. First, make sure your DHCP configuration is properly set up for DDNS:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Use this improved configuration:

```
# DHCP with DDNS Configuration
authoritative;

# Define the DDNS key
key "ddns-key" {
  algorithm hmac-sha256;
  secret "YourStaticSecretKey123456789";
}

# DDNS configuration - critical settings for automatic updates
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
ignore client-updates;

# This is critical for automatic hostname generation
ddns-hostname = concat("client-", binary-to-ascii(10, 8, "-", leased-address));
ddns-domainname "est.intra";

# Domain settings
option domain-name "est.intra";
option domain-name-servers 192.168.1.101;

# DDNS zones - these must match your actual DNS configuration
zone est.intra. {
  primary 192.168.1.101;
  key ddns-key;
}

zone 1.168.192.in-addr.arpa. {
  primary 192.168.1.101;
  key ddns-key;
}

# Network configuration - the subnet where DHCP server is located
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.101;
  default-lease-time 3600;
  max-lease-time 7200;
}
```

2. Now, make sure your DNS configuration allows updates:

```bash
sudo nano /etc/named.conf
```

Check the key section and zones:

```
key "ddns-key" {
    algorithm hmac-sha256;
    secret "YourStaticSecretKey123456789";  # Use the same secret as in dhcpd.conf
};

zone "est.intra" IN {
    type master;
    file "est.intra.zone";
    allow-update { key "ddns-key"; };  # This is critical
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.in-addr.arpa.zone";
    allow-update { key "ddns-key"; };  # This is critical
};
```

3. Make sure SELinux permissions allow DNS updates:

```bash
sudo setsebool -P named_write_master_zones 1
```

4. Check the ownership and permissions of your zone files:

```bash
sudo chown named:named /var/named/est.intra.zone
sudo chown named:named /var/named/1.168.192.in-addr.arpa.zone
sudo chmod 644 /var/named/est.intra.zone
sudo chmod 644 /var/named/1.168.192.in-addr.arpa.zone
```

5. Restart both services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

6. Test that both services are running correctly:

```bash
sudo systemctl status named
sudo systemctl status dhcpd
```

7. If everything looks good, try on the client machine:

```bash
# Release current IP
sudo dhclient -r
# Request new IP
sudo dhclient
```

8. Check if the record was added:

```bash
# Check forward lookup
dig @192.168.1.101 client-192-168-1-101.est.intra

# Check reverse lookup
dig @192.168.1.101 -x 192.168.1.101
```

9. If you're still having issues, check the logs to see what's happening:

```bash
sudo journalctl -u named | tail -50
sudo journalctl -u dhcpd | tail -50
```

10. You can also try manually adding a DNS record to see if your permissions are correct:

```bash
# Install nsupdate if needed
sudo dnf install bind-utils

# Test updating DNS with nsupdate
nsupdate -k /path/to/keyfile
> server 192.168.1.101
> update add test.est.intra 86400 A 192.168.1.123
> send
> quit
```

The key to getting automatic DNS updates is ensuring:
1. The DHCP and DNS configurations use the same key
2. The DNS server allows updates with that key
3. The zone files have the right permissions
4. SELinux allows the updates
5. The DHCP server is configured to send updates

If all these are set correctly, when a client gets an IP address, it should automatically be added to the DNS zone files.
