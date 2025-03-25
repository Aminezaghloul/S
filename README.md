Je comprends qu'il faut résoudre le problème côté serveur et non côté client. En effet, le problème est probablement lié à votre configuration DHCP.

Pour résoudre ce problème définitivement depuis le serveur, je propose la solution suivante:

## Configurer votre serveur DHCP (192.168.1.1) pour déclarer les deux réseaux, mais n'attribuer des adresses que sur 192.168.1.0/24

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Utilisez cette configuration:

```
# Configuration DHCP complète
authoritative;

# Configuration DDNS
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
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

# Réseau prioritaire 192.168.1.0/24 - AVEC plage d'adresses
subnet 192.168.1.0 netmask 255.255.255.0 {
    # Paramètres pour réponse ultra-rapide
    min-secs 0;
    max-response-delay 0;
    min-response-delay 0;
    
    # Attribution d'adresses
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;
    option domain-name "est.intra";
    ddns-updates on;
    
    # Priorité élevée
    default-lease-time 86400;  # 24 heures
    max-lease-time 172800;     # 48 heures
}

# Réseau secondaire 192.168.5.0/24 - SANS plage d'adresses
# Cela permet au serveur de reconnaître ce réseau mais de ne pas attribuer d'adresses
subnet 192.168.5.0 netmask 255.255.255.0 {
    # Pas de directive "range"
    option routers 192.168.5.254;
    option domain-name-servers 192.168.1.1;
    option domain-name "est.intra";
    
    # Priorité basse pour les demandes sur ce réseau
    max-lease-time 60;        # 1 minute seulement
    default-lease-time 30;    # 30 secondes
}
```

Cette configuration:
1. Reconnaît les deux réseaux (nécessaire pour que DHCP fonctionne sur les deux interfaces)
2. Attribue des adresses uniquement sur le réseau 192.168.1.0/24
3. Donne priorité aux réponses sur le réseau 192.168.1.0/24
4. Configure des baux très courts sur le réseau 192.168.5.0/24 (si jamais une adresse est attribuée)

Après avoir modifié la configuration, redémarrez le service:

```bash
sudo systemctl restart dhcpd
```

Puis, sur le client, renouvelez le bail:

```bash
sudo dhclient -r
sudo dhclient
```

Avec cette configuration, votre serveur DHCP devrait répondre aux requêtes venant des deux réseaux, mais n'attribuera des adresses que sur le réseau 192.168.1.0/24.
