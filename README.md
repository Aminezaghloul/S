Je vais corriger la configuration complète en utilisant uniquement le réseau 192.168.1.0, conformément au PDF. Voici les configurations mises à jour :

## 1. Configuration de BIND (/etc/named.conf)

```
// Configuration complète de BIND pour est.intra avec DDNS

options {
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };  // IP du serveur sur le réseau 192.168.1.0
    listen-on-v6 port 53 { ::1; };
    directory "/var/named";
    dump-file "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    recursion yes;
    allow-query { localhost; 192.168.1.0/24; };
    allow-transfer { none; };
    
    dnssec-validation yes;
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

// Définition de la clé DDNS
key "ddns-key" {
    algorithm hmac-md5;
    secret "Ef5gH6jK7lM8nO9pQ0rS1t==";
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

## 2. Fichier de zone directe (/var/named/est.intra.zone)

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

## 3. Fichier de zone inverse (/var/named/192.168.1.rev)

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

## 4. Configuration DHCP (/etc/dhcp/dhcpd.conf)

```
# Configuration DHCP avec DDNS pour est.intra
authoritative;

# Options globales
ddns-update-style interim;
ddns-domainname "est.intra.";
ddns-rev-domainname "in-addr.arpa.";
update-static-leases on;
ignore client-updates;

# Définition de la clé DDNS (même clé que dans named.conf)
key "ddns-key" {
    algorithm hmac-md5;
    secret "Ef5gH6jK7lM8nO9pQ0rS1t==";
}

# Associations des zones avec la clé
zone est.intra. {
    primary 127.0.0.1;
    key "ddns-key";
}

zone 1.168.192.in-addr.arpa. {
    primary 127.0.0.1;
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

## 5. Commandes pour appliquer la configuration

```bash
# Corriger les permissions
sudo chown -R named:named /var/named
sudo chmod 640 /var/named/est.intra.zone
sudo chmod 640 /var/named/192.168.1.rev

# Vérifier la configuration
sudo named-checkconf
sudo named-checkzone est.intra /var/named/est.intra.zone
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/192.168.1.rev

# Redémarrer les services
sudo systemctl restart named
sudo systemctl restart dhcpd

# Vérifier l'état des services
sudo systemctl status named
sudo systemctl status dhcpd
```

## 6. Configuration d'interface réseau (si nécessaire)

Si vous devez configurer l'interface réseau du serveur manuellement :

```bash
# Configurer l'adresse IP statique pour le serveur
sudo nmcli connection modify "System eth0" ipv4.method manual ipv4.addresses 192.168.1.1/24
sudo nmcli connection down "System eth0"
sudo nmcli connection up "System eth0"
```

## 7. Vérification après configuration

Sur le serveur :
```bash
# Vérifier que named fonctionne
dig @localhost est.intra

# Vérifier les logs pour débogage
sudo journalctl -u named --since "15 minutes ago"
sudo journalctl -u dhcpd --since "15 minutes ago"
```

Cette configuration utilise exclusivement le réseau 192.168.1.0/24, avec le serveur DNS et DHCP à l'adresse 192.168.1.1, conformément au schéma présenté dans le PDF.
