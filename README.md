Je vais vous fournir une configuration complète, étape par étape, avec la clé DDNS que je vais générer moi-même. Cette configuration permettra de mettre en place un serveur DHCP et DNS avec DDNS (Dynamic DNS) pour le domaine "est.intra".

# Configuration complète DHCP et DDNS pour le domaine est.intra

## 1. Installation des paquets nécessaires

```bash
sudo dnf install bind bind-utils dhcp-server -y
```

## 2. Génération de la clé DDNS

```bash
# Générer la clé
sudo dnssec-keygen -a HMAC-MD5 -b 128 -n HOST ddns-key

# Récupérer la valeur de la clé
sudo cat Kddns-key*.private | grep Key
```

Pour cet exemple, nous utiliserons une clé générée : `Ef5gH6jK7lM8nO9pQ0rS1t==`

## 3. Configuration du serveur DNS (BIND)

Éditer le fichier de configuration principal:

```bash
sudo nano /etc/named.conf
```

```
// Fichier de configuration BIND pour le domaine est.intra
options {
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };
    listen-on-v6 port 53 { ::1; };
    directory "/var/named";
    dump-file "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    recursion yes;
    allow-query { localhost; 192.168.1.0/24; };
    allow-transfer { none; };
    
    dnssec-enable yes;
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

## 4. Création des fichiers de zone DNS

Créer le fichier de zone directe:

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

Créer le fichier de zone inverse:

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

## 5. Configurer les permissions pour les fichiers de zone

```bash
sudo chown -R named:named /var/named
sudo chmod 640 /var/named/est.intra.zone
sudo chmod 640 /var/named/192.168.1.rev
```

## 6. Configuration du serveur DHCP avec support DDNS

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

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

## 7. Configuration du pare-feu et SELinux

```bash
# Ouvrir les ports nécessaires dans le pare-feu
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=dhcp
sudo firewall-cmd --reload

# Configurer SELinux pour autoriser les mises à jour DDNS
sudo setsebool -P named_write_master_zones 1
```

## 8. Démarrage et activation des services

```bash
# Vérifier la configuration DNS
sudo named-checkconf
sudo named-checkzone est.intra /var/named/est.intra.zone
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/192.168.1.rev

# Démarrer et activer le service DNS
sudo systemctl enable named
sudo systemctl start named
sudo systemctl status named

# Démarrer et activer le service DHCP
sudo systemctl enable dhcpd
sudo systemctl start dhcpd
sudo systemctl status dhcpd
```

## 9. Configuration du client

Sur la machine cliente, configurez la connexion réseau pour utiliser DHCP:

```bash
# Si vous utilisez NetworkManager
sudo nmcli connection modify "System eth0" ipv4.method auto
sudo nmcli connection down "System eth0"
sudo nmcli connection up "System eth0"

# OU si vous préférez éditer directement le fichier
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0
```

Avec le contenu:
```
DEVICE=eth0
BOOTPROTO=dhcp
ONBOOT=yes
```

Puis redémarrez le réseau:
```bash
sudo systemctl restart NetworkManager
```

## 10. Vérification de la configuration

Sur le serveur:

```bash
# Vérifier que le service DNS fonctionne
sudo dig @localhost est.intra

# Vérifier que le service DHCP fonctionne
sudo systemctl status dhcpd

# Vérifier les baux DHCP
sudo cat /var/lib/dhcpd/dhcpd.leases

# Surveiller les logs pour le débogage
sudo tail -f /var/log/messages
```

Sur le client:

```bash
# Vérifier l'attribution d'adresse IP
ip addr show

# Tester la résolution DNS
ping ns1.est.intra

# Vérifier que le client a bien été ajouté automatiquement à la zone DNS
dig @192.168.1.1 -x $(hostname -I | awk '{print $1}')
```

## Résolution des problèmes courants

1. **Les mises à jour DNS ne fonctionnent pas**:
   - Vérifiez les logs: `sudo journalctl -u named -u dhcpd`
   - Assurez-vous que la clé est correctement configurée dans les deux fichiers
   - Vérifiez que SELinux autorise les mises à jour: `sudo getsebool named_write_master_zones`

2. **Problèmes de permissions**:
   ```bash
   sudo chown -R named:named /var/named
   sudo chmod 640 /var/named/est.intra.zone /var/named/192.168.1.rev
   sudo chmod 750 /var/named/data
   ```

3. **Problèmes avec SELinux**:
   ```bash
   sudo setenforce 0  # Désactiver temporairement SELinux pour tester
   # Si cela résout le problème, configurer correctement SELinux plutôt que de le désactiver
   sudo setenforce 1  # Réactiver SELinux
   ```

Cette configuration complète vous permettra d'avoir un serveur DHCP avec DDNS qui ajoutera automatiquement les clients dans la zone DNS lorsqu'ils obtiendront une adresse IP.
