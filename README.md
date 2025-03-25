Je vais vous fournir une configuration complète pour résoudre le problème des deux serveurs DHCP. Cette configuration permettra à votre serveur de répondre plus rapidement et d'être considéré comme prioritaire par le client.

## Configuration complète du serveur DHCP

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

```
# Configuration DHCP pour est.intra avec priorité élevée
# Paramètres pour réponse prioritaire
min-secs 0;
response-timeout 1;

# Déclaration d'autorité
authoritative;

# Configuration DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
allow client-updates;

# Configuration des noms d'hôte automatiques
use-host-decl-names on;
ignore client-updates;
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
    
    # Activer DDNS pour cette plage
    ddns-updates on;
    
    # Configuration pour une réponse prioritaire
    max-response-delay 1;
    min-response-delay 0;
}
```

## Configuration de l'override systemd pour DHCP

```bash
sudo mkdir -p /etc/systemd/system/dhcpd.service.d/
sudo nano /etc/systemd/system/dhcpd.service.d/override.conf
```

```
[Service]
ExecStart=
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd ens160
```

## Configuration du serveur DNS (BIND)

```bash
sudo nano /etc/named.conf
```

```
// Configuration complète de BIND pour est.intra avec DDNS

options {
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };
    listen-on-v6 port 53 { ::1; };
    directory "/var/named";
    dump-file "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    recursion yes;
    allow-query { any; };
    dnssec-validation yes;
};

// Définition de la clé DDNS
key "ddns-key" {
    algorithm hmac-md5;
    secret "8TBzJIkCJhHk7cgmISJRoA==";
};

// Configuration de la zone est.intra
zone "est.intra" IN {
    type master;
    file "est.intra.zone";
    allow-update { key "ddns-key"; };
};

// Configuration de la zone inverse
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.1.rev";
    allow-update { key "ddns-key"; };
};
```

## Fichiers de zone DNS

```bash
sudo nano /var/named/est.intra.zone
```

```
$TTL 86400
@       IN      SOA     ns1.est.intra. admin.est.intra. (
                         2024032501 ; Serial
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
                        2024032501 ; Serial
                        3600       ; Refresh
                        1800       ; Retry
                        604800     ; Expire
                        86400 )    ; Minimum TTL
        IN      NS      ns1.est.intra.
1       IN      PTR     ns1.est.intra.
```

## Permissions et redémarrage des services

```bash
# Ajuster les permissions
sudo chown -R named:named /var/named
sudo chmod 664 /var/named/est.intra.zone
sudo chmod 664 /var/named/192.168.1.rev
sudo restorecon -R /var/named

# Configurer SELinux
sudo setsebool -P named_write_master_zones 1
sudo setsebool -P dhcpd_write_leases 1

# Recharger et redémarrer les services
sudo systemctl daemon-reload
sudo systemctl restart named
sudo systemctl restart dhcpd

# Vérifier l'état des services
sudo systemctl status named
sudo systemctl status dhcpd
```

Cette configuration complète devrait résoudre votre problème en faisant en sorte que votre serveur DHCP réponde plus rapidement que l'autre serveur, et que les clients soient automatiquement enregistrés dans la zone DNS avec des noms générés basés sur leur adresse IP.
