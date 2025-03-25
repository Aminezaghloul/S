Je constate deux problèmes principaux dans vos captures d'écran:

1. Le service DHCP ne démarre pas à cause d'une erreur dans la configuration ou l'override
2. BIND signale des problèmes avec les zones DNS

## Résolution des problèmes

### 1. Corriger le service DHCP

Le problème principal est que votre fichier override pour dhcpd n'est pas correctement configuré.

```bash
# Supprimer le fichier override actuel qui cause des problèmes
sudo rm /etc/systemd/system/dhcpd.service.d/override.conf

# Créer un nouveau fichier override correct
sudo mkdir -p /etc/systemd/system/dhcpd.service.d/
sudo nano /etc/systemd/system/dhcpd.service.d/override.conf
```

Dans ce fichier, écrivez exactement:
```
[Service]
ExecStart=
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd
```

### 2. Corriger la configuration DHCP

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Remplacez tout le contenu par:
```
# Configuration DHCP simple
authoritative;

# Configuration DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
ignore client-updates;

# Configuration des noms d'hôte automatiques
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
}
```

### 3. Corriger les zones DNS

```bash
# Vérifier que les fichiers de zone existent
ls -la /var/named/est.intra.zone
ls -la /var/named/192.168.1.rev

# Si nécessaire, recréer ces fichiers
sudo nano /var/named/est.intra.zone
```

```
$TTL 86400
@       IN      SOA     ns1.est.intra. admin.est.intra. (
                         2024032502 ; Serial
                         3600       ; Refresh
                         1800       ; Retry
                         604800     ; Expire
                         86400 )    ; Minimum TTL
        IN      NS      ns1.est.intra.
ns1     IN      A       192.168.1.1
```

```bash
sudo nano /var/named/192.168.1.rev
```

```
$TTL 86400
@       IN      SOA     ns1.est.intra. admin.est.intra. (
                        2024032502 ; Serial
                        3600       ; Refresh
                        1800       ; Retry
                        604800     ; Expire
                        86400 )    ; Minimum TTL
        IN      NS      ns1.est.intra.
1       IN      PTR     ns1.est.intra.
```

### 4. Corriger les permissions

```bash
# Corriger les permissions
sudo chown -R named:named /var/named
sudo chmod 664 /var/named/est.intra.zone
sudo chmod 664 /var/named/192.168.1.rev
sudo restorecon -R /var/named

# Configurer SELinux pour BIND
sudo setsebool -P named_write_master_zones 1

# La commande dhcpd_write_leases n'existe pas dans toutes les versions de SELinux
# Ne l'utilisez que si elle existe
```

### 5. Redémarrer les services

```bash
# Recharger systemd
sudo systemctl daemon-reload

# Redémarrer BIND
sudo systemctl restart named

# Redémarrer DHCP
sudo systemctl restart dhcpd
```

### 6. Vérifier l'état des services

```bash
sudo systemctl status named
sudo systemctl status dhcpd
```

Ces corrections devraient résoudre les problèmes avec vos services DHCP et DNS. Si vous continuez à rencontrer des problèmes, veuillez partager les messages d'erreur spécifiques pour que je puisse vous aider davantage.
