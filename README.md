Pour que les clients soient automatiquement ajoutés aux zones DNS lorsqu'ils obtiennent une adresse IP et font un ping, vous devez vous assurer que la configuration DDNS est correctement paramétrée. Voici la configuration pour résoudre ce dernier problème :

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Modifiez la configuration pour inclure les paramètres suivants :

```
# Configuration DHCP avec DDNS
authoritative;

# Configuration cruciale pour DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
ignore client-updates;
update-optimization off;
update-conflict-detection false;

# Paramètres de nommage automatique
ddns-hostname = concat("client-", binary-to-ascii(10, 8, "-", leased-address));
ddns-domainname "est.intra.";
ddns-rev-domainname "in-addr.arpa.";

# Définition de la clé DDNS
key "ddns-key" {
    algorithm hmac-md5;
    secret "8TBzJIkCJhHk7cgmISJRoA==";
}

# Associations des zones avec la clé
zone est.intra. {
    primary 192.168.1.1;
    key "ddns-key";
}

zone 1.168.192.in-addr.arpa. {
    primary 192.168.1.1;
    key "ddns-key";
}

# Configuration du réseau 192.168.1.0/24
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;
    option domain-name "est.intra";
    default-lease-time 3600;
    max-lease-time 7200;
    
    # Activation des mises à jour DDNS pour cette plage
    ddns-updates on;
}
```

Ensuite, redémarrez le service DHCP :

```bash
sudo systemctl restart dhcpd
```

Sur le client, renouvelez le bail DHCP et effectuez un ping :

```bash
sudo dhclient -r
sudo dhclient
ping ns1.est.intra
```

Vérifiez que l'enregistrement a bien été ajouté à la zone DNS :

```bash
sudo cat /var/named/est.intra.zone
```

Si l'enregistrement n'apparaît pas automatiquement, redémarrez aussi le service BIND :

```bash
sudo systemctl restart named
```

Cette configuration permet au serveur DHCP de mettre à jour automatiquement les zones DNS lorsqu'un client obtient une adresse IP, et l'enregistrement devrait apparaître même après un simple ping.
