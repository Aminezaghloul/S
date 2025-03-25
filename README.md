Je vois maintenant le problème clé. Le fichier `/etc/sysconfig/dhcpd` contient explicitement un message indiquant qu'il n'est plus utilisé dans les versions récentes de Fedora/RHEL:

```
# WARNING: This file is NOT used anymore.
```

Pour résoudre ce problème, vous devez créer un fichier d'override pour systemd:

## 1. Créer un fichier d'override pour le service DHCP

```bash
# Créer le répertoire systemd override
sudo mkdir -p /etc/systemd/system/dhcpd.service.d/

# Créer le fichier de configuration override
sudo nano /etc/systemd/system/dhcpd.service.d/override.conf
```

Dans ce fichier, ajoutez:
```
[Service]
# Remplacez ens160 par l'interface connectée au réseau 192.168.1.0/24
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid ens160
```

## 2. Recharger la configuration de systemd et redémarrer le service dhcpd

```bash
sudo systemctl daemon-reload
sudo systemctl restart dhcpd
```

## 3. Vérifiez que le service fonctionne correctement

```bash
sudo systemctl status dhcpd
```

## 4. Assurez-vous que votre configuration dhcpd.conf est correcte

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Assurez-vous que votre configuration contient uniquement une section pour le réseau 192.168.1.0/24:

```
# Configuration DHCP avec DDNS pour est.intra
authoritative;

# Configuration DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
allow client-updates;

# Options pour nommer les clients
use-host-decl-names on;
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

# Uniquement le réseau 192.168.1.0/24
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

# Ne déclarez pas de subnet pour 192.168.5.0/24 si vous ne voulez pas y distribuer d'adresses
```

Cette configuration devrait forcer le serveur DHCP à n'écouter que sur l'interface spécifiée (connectée au réseau 192.168.1.0/24) et à ne distribuer que des adresses IP de ce réseau.
