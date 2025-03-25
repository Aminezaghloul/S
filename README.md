Je vais modifier la configuration pour que le serveur DNS et DHCP soit à l'adresse 192.168.1.1 au lieu de 192.168.5.130. Voici la configuration complète mise à jour:

## 1. Configuration complète du serveur DNS (BIND)

### Fichier /etc/named.conf
```
// Configuration complète de BIND pour est.intra avec DDNS

options {
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };  // Adresse IP serveur 192.168.1.1
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

// Définition de la clé DDNS avec une clé base64 valide
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

### Fichier /var/named/est.intra.zone
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

### Fichier /var/named/192.168.1.rev
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

## 2. Configuration du serveur DHCP avec DDNS

### Fichier /etc/dhcp/dhcpd.conf
```
# Configuration DHCP avec DDNS pour est.intra
authoritative;

# Options globales
ddns-update-style interim;
ddns-domainname "est.intra.";
ddns-rev-domainname "in-addr.arpa.";
update-static-leases on;
ignore client-updates;

# Définition de la clé DDNS - exactement la même que dans named.conf
key "ddns-key" {
    algorithm hmac-md5;
    secret "8TBzJIkCJhHk7cgmISJRoA==";
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
    option domain-name-servers 192.168.1.1;  # Adresse IP du serveur DNS (192.168.1.1)
    option domain-name "est.intra";
    default-lease-time 3600;
    max-lease-time 7200;
    
    # Activer DDNS pour cette plage
    ddns-updates on;
}
```

## 3. Configuration de l'interface réseau

Vous devez configurer l'interface réseau du serveur pour qu'elle utilise l'adresse IP 192.168.1.1. Si vous utilisez NetworkManager:

```bash
# Identifier le nom de votre interface réseau (eth0, enp0s3, etc.)
ip addr show

# Configurer l'interface avec l'adresse IP 192.168.1.1
sudo nmcli connection modify "nom_de_votre_connexion" ipv4.method manual ipv4.addresses 192.168.1.1/24
sudo nmcli connection down "nom_de_votre_connexion"
sudo nmcli connection up "nom_de_votre_connexion"
```

## 4. Commandes d'application et vérification

```bash
# Corriger les permissions des fichiers de zone
sudo chown -R named:named /var/named
sudo chmod 640 /var/named/est.intra.zone /var/named/192.168.1.rev

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

Cette configuration place le serveur DNS et DHCP à l'adresse 192.168.1.1, conformément à votre demande. Assurez-vous que votre interface réseau est correctement configurée avec cette adresse IP avant de démarrer les services.
