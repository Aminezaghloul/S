I see the issue now. Your network interface is "enp3s0" and it has an IP address of 192.168.1.24, not 192.168.1.1 as we assumed in the configuration. Let's fix your DHCP configuration to match your actual network setup.

Follow these steps:

1. First, let's check if DHCP is configured to use the correct interface:

```bash
sudo nano /etc/sysconfig/dhcpd
```

Change it to:
```
DHCPDARGS="enp3s0"
```

2. Now let's update your DHCP configuration to match your actual network settings:

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
option domain-name-servers 192.168.1.24;  # Using your actual IP

# DDNS zones
zone est.intra. {
  primary 192.168.1.24;  # Using your actual IP
  key ddns-key;
}

zone 1.168.192.in-addr.arpa. {
  primary 192.168.1.24;  # Using your actual IP
  key ddns-key;
}

# Network configuration - using your actual network
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.24;  # Using your actual IP
  default-lease-time 3600;
  max-lease-time 7200;
}
```

3. Also, make sure your DNS configuration (named.conf) is updated to listen on your actual IP:

```bash
sudo nano /etc/named.conf
```

Change the listen-on line to:
```
listen-on port 53 { 127.0.0.1; 192.168.1.24; };
```

And update any other references to 192.168.1.1 to 192.168.1.24 in your named.conf file.

4. Also check your zone files to make sure they reference the correct IP:

```bash
sudo nano /var/named/est.intra.zone
```

Update the A record for the DNS server to:
```
dns     IN      A       192.168.1.24
```

```bash
sudo nano /var/named/1.168.192.in-addr.arpa.zone
```

Update the PTR record to reference the correct IP:
```
24      IN      PTR     dns.est.intra.  # Your server IP's last octet
```

5. After making these changes, check the configuration for errors:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo named-checkconf
```

6. Restart the services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

If the DHCP service still fails to start, run:

```bash
sudo dhcpd -d -f -cf /etc/dhcp/dhcpd.conf
```

This will run DHCP in the foreground with debug output, which should show exactly what's causing the error.

Another common issue could be that you have another DHCP server running on the same network, or there's a port conflict. Make sure no other DHCP services are running:

```bash
sudo ss -tulpn | grep 67
```

If anything shows up, you'll need to stop that service before starting dhcpd.
