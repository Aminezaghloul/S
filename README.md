I can see that your DHCP is working (the client got the IP 192.168.1.101), but the DNS updates aren't being applied automatically. Let's fix that so each client's domain name gets automatically added to the zone files.

Here's what we need to configure:

1. First, let's update your dhcpd.conf file with the correct DDNS settings:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Add this complete configuration:

```
# DHCP with DDNS Configuration
authoritative;

# Define the DDNS key for secure updates
key "ddns-key" {
  algorithm hmac-sha256;
  secret "YourStaticSecretKey123456789";
}

# Critical DDNS configuration settings
ddns-update-style interim;
ddns-updates on;
update-static-leases on;
ignore client-updates;

# Hostname and domain settings
ddns-hostname = concat("client-", binary-to-ascii(10, 8, "-", leased-address));
ddns-domainname "est.intra";
use-host-decl-names on;

# Domain settings
option domain-name "est.intra";
option domain-name-servers 192.168.1.1;

# DDNS zones configuration - critical for automatic updates
zone est.intra. {
  primary 192.168.1.1;
  key ddns-key;
}

zone 1.168.192.in-addr.arpa. {
  primary 192.168.1.1;
  key ddns-key;
}

# Network configuration
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  
  # These settings ensure client hostnames are registered
  ddns-rev-domainname "in-addr.arpa.";
  option host-name = concat("client-", binary-to-ascii(10, 8, "-", leased-address));
  
  default-lease-time 3600;
  max-lease-time 7200;
}
```

2. Make sure your named.conf has proper configuration for DDNS:

```bash
sudo nano /etc/named.conf
```

Make sure it includes:

```
// Define the DDNS key
key "ddns-key" {
    algorithm hmac-sha256;
    secret "YourStaticSecretKey123456789";
};

options {
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };
    // other options...
    allow-query { localhost; 192.168.1.0/24; };
    // Permit zone transfers and dynamic updates
    allow-transfer { localhost; };
    
    // This should be set explicitly for DDNS to work
    allow-update { key ddns-key; };
    
    // These settings might help with troubleshooting
    notify yes;
    recursion yes;
    dnssec-enable yes;
    dnssec-validation yes;
    
    // Logging is enabled
    directory "/var/named";
};

// Forward zone
zone "est.intra" IN {
    type master;
    file "est.intra.zone";
    allow-update { key "ddns-key"; };
    journal "est.intra.zone.jnl";  // This is important for dynamic updates
};

// Reverse zone
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.in-addr.arpa.zone";
    allow-update { key "ddns-key"; };
    journal "1.168.192.in-addr.arpa.zone.jnl";  // This is important for dynamic updates
};
```

3. Enable logging for both DHCP and DNS to troubleshoot issues:

```bash
sudo nano /etc/named.conf
```

Add this logging section:

```
logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
    
    channel ddns_log {
        file "/var/log/named_ddns.log";
        severity debug;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    
    category update { ddns_log; };
    category update-security { ddns_log; };
};
```

4. Check that your zone files are properly configured:

```bash
sudo nano /var/named/est.intra.zone
```

Ensure it contains:

```
$TTL 86400
@       IN      SOA     ns1.est.intra. root.est.intra. (
                        2023032901      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL
;
@       IN      NS      ns1.est.intra.
ns1     IN      A       192.168.1.1
```

5. Make sure your reverse zone is properly configured:

```bash
sudo nano /var/named/1.168.192.in-addr.arpa.zone
```

Ensure it contains:

```
$TTL 86400
@       IN      SOA     ns1.est.intra. root.est.intra. (
                        2023032901      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL
;
@       IN      NS      ns1.est.intra.
1       IN      PTR     ns1.est.intra.
```

6. Make sure SELinux permits DDNS updates:

```bash
sudo setsebool -P named_write_master_zones 1
```

7. Ensure proper permissions on zone files:

```bash
sudo chown named:named /var/named/est.intra.zone
sudo chown named:named /var/named/1.168.192.in-addr.arpa.zone
sudo chmod 644 /var/named/est.intra.zone
sudo chmod 644 /var/named/1.168.192.in-addr.arpa.zone
```

8. Make sure your DHCP server is allowed to connect to the DNS server (port 53):

```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

9. Restart both services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

10. On the client machine, release and renew the IP to trigger a DDNS update:

```bash
sudo dhclient -r
sudo dhclient
```

11. After these changes, check if the client was added to DNS:

```bash
dig @192.168.1.1 client-192-168-1-101.est.intra
dig @192.168.1.1 -x 192.168.1.101
```

12. If you're still having issues, check the logs:

```bash
sudo tail -50 /var/log/named_ddns.log
sudo journalctl -u named | grep update
sudo journalctl -u dhcpd | grep update
```

This comprehensive configuration should ensure that when clients get IP addresses via DHCP, their hostnames are automatically added to the DNS zone files. The key aspects are making sure the DHCP and DNS servers are using the same key, the zone files have the proper permissions, and SELinux allows the updates.
