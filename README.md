I see the specific error now: "Not configured to listen on any interfaces!" This means your DHCP server doesn't know which network interface to use. Let's fix this:

1. First, create or edit the DHCP service configuration file:

```bash
sudo nano /etc/sysconfig/dhcpd
```

2. Add the following line, specifying your network interface (based on your previous screenshots, it looks like you're using ens160):

```
DHCPDARGS="ens160"
```

3. Save and exit the file.

4. Restart the DHCP service:

```bash
sudo systemctl restart dhcpd
```

If you're still having issues, let's try an alternative approach. Instead of using the /etc/sysconfig/dhcpd file, we can modify the systemd service file directly:

```bash
sudo systemctl edit dhcpd.service
```

Add these lines:

```
[Service]
ExecStart=
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid ens160
```

Save and exit, then run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart dhcpd
```

This should explicitly tell the DHCP server to listen on the ens160 interface. 

Also, make sure that your current DHCP configuration in /etc/dhcp/dhcpd.conf includes all the necessary subnet declarations, particularly the subnet where your server is located, even if you don't want to assign IPs from it.

If you continue to encounter issues, please let me know, and we can try other troubleshooting steps.
