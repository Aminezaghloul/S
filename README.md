The error suggests there might be an issue with your DHCP configuration. Let's check what's wrong with the DHCP service by examining the error details:

```bash
sudo systemctl status dhcpd
sudo journalctl -xeu dhcpd.service
```

One common issue is that the DHCP service needs to know which network interface to listen on. Let's check and update the configuration:

1. First, confirm your network interfaces:

```bash
ip a
```

2. Edit the DHCP service configuration:

```bash
sudo nano /etc/sysconfig/dhcpd
```

Add your network interface (likely ens160 or similar):
```
DHCPDARGS="ens160"  # Replace with your actual interface
```

3. Another potential issue might be with your current network configuration. Your DHCP server is trying to serve IPs on the 192.168.1.0/24 subnet, but your server appears to be on a different subnet. 

Let's run a syntax check on your DHCP configuration:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

This will tell us exactly what's wrong with the configuration file.

4. You need to make sure that the subnet where your DHCP server is located is also declared in the configuration, even if you don't want to serve IPs from that subnet. For example:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Add a section for the subnet where your DHCP server's IP (192.168.250.x) is located:

```
# Also declare the network where the DHCP server is located
subnet 192.168.250.0 netmask 255.255.255.0 {
  # This subnet declaration is required but we're not serving IPs here
}
```

5. Make sure there are no syntax errors in your configuration file, such as missing semicolons or braces.

6. After making these changes, restart the DHCP service:

```bash
sudo systemctl restart dhcpd
```

If you continue to have issues, please share the output of the status and error messages so I can help you troubleshoot further.
