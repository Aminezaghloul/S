Maintenant je comprends mieux le problème. Vous avez deux interfaces réseau sur votre serveur:
- Une interface sur le réseau 192.168.1.0/24 (avec l'adresse 192.168.1.1)
- Une autre interface sur le réseau 192.168.5.0/24 (avec l'adresse 192.168.5.130)

Et le client reçoit toujours une adresse IP du réseau 192.168.5.0/24 au lieu du réseau 192.168.1.0/24.

Voici comment résoudre ce problème spécifique:

## 1. Configuration DHCP pour spécifier l'interface exacte

Modifiez le fichier /etc/sysconfig/dhcpd pour spécifier l'interface qui est connectée au réseau 192.168.1.0/24:

```bash
sudo nano /etc/sysconfig/dhcpd
```

```
# Spécifiez UNIQUEMENT l'interface connectée au réseau 192.168.1.0/24
# Remplacez "interface_name" par le nom réel de votre interface (ex: ens160, eth0, etc.)
DHCPDARGS="interface_name"
```

## 2. Vérifiez les interfaces réseau et leurs adresses

```bash
ip addr show
```

Notez le nom exact de l'interface connectée au réseau 192.168.1.0/24 et utilisez-le dans DHCPDARGS.

## 3. Configurez DHCP pour servir uniquement le réseau 192.168.1.0/24

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

```
# Configuration DHCP avec DDNS pour est.intra
authoritative;

# Configuration DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
allow client-updates;
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

# Configuration UNIQUEMENT pour le réseau 192.168.1.0/24
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

# Déclarer le réseau 192.168.5.0/24 sans plage d'adresses
# Cette déclaration est nécessaire si l'interface est sur ce réseau,
# mais nous ne voulons pas distribuer d'adresses
subnet 192.168.5.0 netmask 255.255.255.0 {
    # Ne pas inclure de directive "range" ici
    # Cela déclare juste que le serveur connaît ce réseau
    # mais ne distribuera pas d'adresses
}
```

## 4. Assurez-vous que l'interface du client est correctement connectée

Vérifiez que le client est physiquement connecté au réseau 192.168.1.0/24 et non au réseau 192.168.5.0/24.

## 5. Si nécessaire, forcez le client à demander une adresse spécifique

Sur le client, vous pouvez configurer dhclient pour demander explicitement une adresse du réseau 192.168.1.0/24:

```bash
sudo nano /etc/dhcp/dhclient.conf
```

Ajoutez:
```
# Demander une adresse dans le réseau 192.168.1.0/24
send dhcp-requested-address 192.168.1.150;
```

## 6. Redémarrer les services et tester

```bash
sudo systemctl restart dhcpd
```

Sur le client:
```bash
sudo dhclient -r
sudo dhclient -v
```

Ces modifications devraient forcer votre serveur DHCP à distribuer uniquement des adresses du réseau 192.168.1.0/24 et à ignorer les demandes sur le réseau 192.168.5.0/24.
