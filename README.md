I see the issue now. The /etc/sysconfig/dhcpd file is deprecated in newer versions of Fedora. As the warning message indicates, "This file is NOT used anymore."

Let's follow the instructions provided in the file to properly configure the DHCP server:

1. First, we need to create a custom systemd service file:

```bash
sudo cp /usr/lib/systemd/system/dhcpd.service /etc/systemd/system/
```

2. Now, edit the service file:

```bash
sudo nano /etc/systemd/system/dhcpd.service
```

3. Find the ExecStart line and modify it to include your interface name (ens160):

```
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid ens160
```

4. Save and exit the file.

5. Reload the systemd configuration and restart the DHCP service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart dhcpd
```

Additionally, as mentioned in the file, DHCP listens only on interfaces for which it finds subnet declarations in dhcpd.conf. Let's make sure your dhcpd.conf has the correct subnet declarations:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Make sure it includes all the subnets that match your interfaces. Based on your earlier screenshots, you should have:

```
# Basic DHCP with DDNS configuration
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
option domain-name-servers 192.168.250.179;  # Or whatever your DNS server IP is

# DDNS zones
zone est.intra. {
  primary 192.168.250.179;  # Your DNS server IP
  key ddns-key;
}

zone 1.168.192.in-addr.arpa. {
  primary 192.168.250.179;  # Your DNS server IP
  key ddns-key;
}

# Subnet matching your server's interface
subnet 192.168.250.0 netmask 255.255.255.0 {
  # This is required for DHCP to listen on this interface
  # No range needed if you don't want to assign IPs here
}

# Subnet for clients
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  default-lease-time 3600;
  max-lease-time 7200;
}
```

After making these changes, restart the DHCP service again:

```bash
sudo systemctl restart dhcpd
```

Check the status to see if it's working:

```bash
sudo systemctl status dhcpd
```

This should fix the "Not configured to listen on any interfaces!" error.
