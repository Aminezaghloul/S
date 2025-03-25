Je vais vous fournir une configuration complète pour résoudre votre problème et permettre l'utilisation de deux pools d'adresses IP. Pour cela, nous allons utiliser le fichier DHCPDARGS pour spécifier les interfaces.

## 1. Configuration des interfaces réseau (DHCPDARGS)

Créez ou modifiez le fichier DHCPDARGS:

```bash
sudo nano /etc/sysconfig/dhcpd
```

Contenu du fichier (spécifiez les interfaces où DHCP doit écouter):
```
# Spécifiez les interfaces pour le service DHCP
DHCPDARGS="ens160"
```

## 2. Configuration BIND complète

### Fichier /etc/named.conf
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
    allow-query { localhost; 192.168.1.0/24; 192.168.5.0/24; };
    allow-transfer { none; };
    
    dnssec-validation yes;
};

// Journalisation pour DDNS - très utile pour résoudre les problèmes
logging {
    channel ddns_log {
        file "/var/log/named-ddns.log";
        severity dynamic;
        print-time yes;
        print-severity yes;
        print-category yes;
    };
    category update { ddns_log; };
    category update-security { ddns_log; };
    category security { ddns_log; };
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
    notify yes;
};

// Configuration de la zone inverse pour 192.168.1.0/24
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.1.rev";
    allow-update { key "ddns-key"; };
    notify yes;
};
```

### Fichier /var/named/est.intra.zone
```
$TTL 86400
@       IN      SOA     ns1.est.intra. admin.est.intra. (
                         2024032502 ; Serial - augmentez après chaque modification
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
                        2024032502 ; Serial - augmentez après chaque modification
                        3600       ; Refresh
                        1800       ; Retry
                        604800     ; Expire
                        86400 )    ; Minimum TTL
        IN      NS      ns1.est.intra.
1       IN      PTR     ns1.est.intra.
```

## 3. Configuration DHCP complète

### Fichier /etc/dhcp/dhcpd.conf
```
# Configuration DHCP avec DDNS pour est.intra

# Configuration globale
authoritative;
log-facility local7;

# Configuration DDNS - essentielle pour l'enregistrement automatique
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
allow client-updates;
update-conflict-detection false;
ddns-domainname "est.intra.";
ddns-rev-domainname "in-addr.arpa.";
use-host-decl-names on;

# Définition de la clé DDNS - exactement la même que dans named.conf
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
    
    # Vous pouvez définir des hôtes fixes comme ceci:
    # host client1 {
    #   hardware ethernet xx:xx:xx:xx:xx:xx; # Remplacez par l'adresse MAC du client
    #   fixed-address 192.168.1.101;
    #   option host-name "client1";
    # }
}

# Si vous avez besoin de configurer le réseau 192.168.5.0/24 également
# subnet 192.168.5.0 netmask 255.255.255.0 {
#     range 192.168.5.100 192.168.5.200;
#     option routers 192.168.5.1;
#     option domain-name-servers 192.168.1.1;
#     option domain-name "est.intra";
#     default-lease-time 3600;
#     max-lease-time 7200;
#     
#     ddns-updates on;
# }
```

## 4. Configuration pour la journalisation DHCP

```bash
sudo nano /etc/rsyslog.conf
```

Ajoutez:
```
# Journalisation DHCP
local7.*                        /var/log/dhcpd.log
```

## 5. Permissions et SELinux

```bash
# Ajuster les permissions des fichiers de zone
sudo chown -R named:named /var/named
sudo chmod 664 /var/named/est.intra.zone
sudo chmod 664 /var/named/192.168.1.rev

# Restaurer les contextes SELinux
sudo restorecon -R /var/named

# Configurer SELinux pour autoriser les mises à jour DDNS
sudo setsebool -P named_write_master_zones 1
sudo setsebool -P dhcpd_write_leases 1
```

## 6. Redémarrage des services

```bash
# Créer les fichiers de log s'ils n'existent pas
sudo touch /var/log/named-ddns.log
sudo touch /var/log/dhcpd.log
sudo chmod 644 /var/log/named-ddns.log
sudo chmod 644 /var/log/dhcpd.log
sudo chown named:named /var/log/named-ddns.log

# Redémarrer les services
sudo systemctl restart rsyslog
sudo systemctl restart named
sudo systemctl restart dhcpd

# Vérifier l'état des services
sudo systemctl status named
sudo systemctl status dhcpd
```

## 7. Vérification et dépannage

```bash
# Vérifier les logs pour DDNS
sudo tail -f /var/log/named-ddns.log

# Vérifier les logs pour DHCP
sudo tail -f /var/log/dhcpd.log

# Vérifier que les zones sont correctement configurées
sudo named-checkzone est.intra /var/named/est.intra.zone
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/192.168.1.rev

# Vérifier les baux DHCP
sudo cat /var/lib/dhcpd/dhcpd.leases
```

## 8. Configuration du client

Sur le client, renouvelez le bail DHCP:
```bash
sudo dhclient -r
sudo dhclient -v
```

Avec ces configurations, le problème devrait être résolu et les clients devraient être automatiquement ajoutés aux zones DNS lorsqu'ils obtiennent une adresse IP. Le fichier DHCPDARGS vous permet de spécifier l'interface réseau sur laquelle le serveur DHCP doit écouter, ce qui résout le problème de conflit entre serveurs DHCP.
