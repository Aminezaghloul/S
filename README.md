Je vois que votre requête DNS pour ns1.est.intra fonctionne mais qu'il y a un problème avec la mise à jour dynamique du DNS (DDNS) puisque les clients n'apparaissent pas automatiquement dans la zone DNS.

Voici comment résoudre ce problème:

1. D'abord, vérifiez que le service DHCP fonctionne correctement et distribue des adresses IP.

2. Configurez correctement les permissions pour permettre les mises à jour dynamiques:

```bash
sudo chown -R named:named /var/named
sudo chmod 664 /var/named/est.intra.zone
sudo chmod 664 /var/named/192.168.1.rev
sudo restorecon -R /var/named
```

3. Modifiez la configuration de DHCP pour s'assurer que les mises à jour DNS sont correctement configurées:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Assurez-vous d'avoir cette configuration:

```
# Configuration DHCP avec DDNS pour est.intra
authoritative;

# Options globales
ddns-updates on;
ddns-update-style interim;
ddns-domainname "est.intra.";
ddns-rev-domainname "in-addr.arpa.";
update-static-leases on;
allow client-updates;  # Changez "ignore client-updates" à "allow client-updates"

# Définition de la clé DDNS
key "ddns-key" {
    algorithm hmac-md5;
    secret "8TBzJIkCJhHk7cgmISJRoA==";
}

# Associations des zones avec la clé
zone est.intra. {
    primary 192.168.1.1;  # Utiliser l'IP du serveur plutôt que 127.0.0.1
    key "ddns-key";
}

zone 1.168.192.in-addr.arpa. {
    primary 192.168.1.1;  # Utiliser l'IP du serveur plutôt que 127.0.0.1
    key "ddns-key";
}

# Configuration du réseau
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;
    option domain-name "est.intra";
    default-lease-time 3600;
    max-lease-time 7200;
    
    # Activer DDNS pour cette plage
    ddns-updates on;
}
```

4. Ajustez également la configuration de BIND pour s'assurer que les mises à jour dynamiques fonctionnent:

```bash
sudo nano /etc/named.conf
```

Modifiez la section des zones comme ceci:

```
// Configuration de la zone est.intra
zone "est.intra" IN {
    type master;
    file "est.intra.zone";
    allow-update { key "ddns-key"; };
    notify yes;
    also-notify { 192.168.1.1; };
};

// Configuration de la zone inverse
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.1.rev";
    allow-update { key "ddns-key"; };
    notify yes;
    also-notify { 192.168.1.1; };
};
```

5. Si vous utilisez SELinux, assurez-vous qu'il est configuré pour permettre les mises à jour dynamiques:

```bash
sudo setsebool -P named_write_master_zones 1
sudo setsebool -P dhcpd_write_leases 1
```

6. Redémarrez les services:

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

7. Vérifiez que DHCP et DNS fonctionnent correctement:

```bash
# Vérifiez les logs pour DHCP
sudo journalctl -u dhcpd -n 50

# Vérifiez les logs pour BIND
sudo journalctl -u named -n 50
```

8. Sur un client, renouvelez le bail DHCP:

```bash
# Si le client utilise NetworkManager
sudo nmcli connection down "nom_de_connexion"
sudo nmcli connection up "nom_de_connexion"

# Ou avec ip/dhclient
sudo dhclient -r
sudo dhclient
```

Ces modifications devraient permettre aux clients d'être automatiquement ajoutés à la zone DNS lorsqu'ils obtiennent une adresse IP via DHCP.
