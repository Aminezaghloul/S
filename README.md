I can see from your error logs that you're having a configuration file error in your DHCP server. The specific error is:

```
Configuration file errors encountered -- exiting
```

Let's fix the DHCP configuration file by creating a simpler, correct configuration. Here's what we'll do:

1. First, let's check your DHCP configuration for syntax errors:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

2. Let's create a new, clean DHCP configuration file:

```bash
sudo mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak
sudo nano /etc/dhcp/dhcpd.conf
```

3. Add this simplified configuration to the file:

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
option domain-name-servers 192.168.1.1;

# DDNS zones
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
  default-lease-time 3600;
  max-lease-time 7200;
}
```

4. Check if there's an issue with your network interface. Make sure the DHCP server knows which interface to use:

```bash
sudo nano /etc/sysconfig/dhcpd
```

Add your network interface (like eth0 or enp0s3):

```
DHCPDARGS="eth0"  # Replace with your actual interface
```

5. Check if your IP is correctly set on the interface:

```bash
ip addr
```

If your server doesn't have the 192.168.1.1 IP address, set it:

```bash
sudo ip addr add 192.168.1.1/24 dev eth0  # Replace eth0 with your interface
```

6. Check for syntax errors in the new configuration:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

7. Start the DHCP service:

```bash
sudo systemctl restart dhcpd
```

If you're still experiencing issues, there might be a problem with the interaction between DHCP and DNS. Make sure that the key in both configurations matches exactly.

Check your named.conf file to ensure the key section matches:

```bash
sudo nano /etc/named.conf
```

The key section in named.conf should look like:

```
key "ddns-key" {
    algorithm hmac-sha256;
    secret "YourStaticSecretKey123456789";
};
```

Make sure the secret is exactly the same in both the DHCP and DNS configuration files.

After making these changes, restart both services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

If you're still encountering issues, please let me know the specific error message from:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

This will help identify the exact syntax error in your configuration.
