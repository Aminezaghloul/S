Pour résoudre le problème de nom d'hôte manquant, voici la solution complète:

## 1. Modifier la configuration DHCP pour générer automatiquement des noms d'hôte

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Modifiez votre fichier pour inclure cette configuration:

```
# Configuration DHCP avec DDNS pour est.intra
authoritative;

# Configuration DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
allow client-updates;

# Configuration critique pour l'attribution automatique de noms d'hôte
use-host-decl-names on;
ignore client-updates;
# Cette ligne génère automatiquement un nom pour chaque client basé sur son IP
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
}
```

## 2. Redémarrer le service DHCP

```bash
sudo systemctl restart dhcpd
```

## 3. Vérifier que le service fonctionne correctement

```bash
sudo systemctl status dhcpd
```

## 4. Sur le client, renouveler le bail DHCP

```bash
sudo dhclient -r
sudo dhclient -v
```

## 5. Vérifier que le nom d'hôte a été correctement ajouté à la zone DNS

```bash
# Sur le serveur
sudo cat /var/named/est.intra.zone
sudo cat /var/named/192.168.1.rev

# Vérifier si le nom est correctement résolu
dig @localhost client-192-168-1-100.est.intra
```

Cette configuration générera automatiquement des noms d'hôte au format "client-192-168-1-100" pour les clients qui obtiennent l'adresse IP 192.168.1.100, et les enregistrera dans les zones DNS directe et inverse.

Si vous préférez un format de nom différent, vous pouvez ajuster la ligne `ddns-hostname` selon vos besoins.
