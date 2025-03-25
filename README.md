Pour que le serveur DNS ajoute automatiquement le nom d'hôte et l'adresse IP des clients à la zone DNS lorsqu'ils font un ping, nous devons modifier la configuration DHCP pour inclure la mise à jour des enregistrements PTR (reverse) et A (forward) dans la zone DNS.

Voici les modifications nécessaires pour que cela fonctionne correctement:

## 1. Modification du fichier dhcpd.conf

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Modifiez le fichier comme suit:

```
# Configuration DHCP avec DDNS pour est.intra

# Configuration globale
authoritative;
log-facility local7;

# Configuration DDNS - cruciale pour l'enregistrement automatique
ddns-updates on;
ddns-update-style interim;
update-static-leases on;
allow client-updates;
update-conflict-detection false;

# Important - utiliser le nom d'hôte du client
use-host-decl-names on;
get-lease-hostnames true;

# Ces lignes sont essentielles pour que le nom d'hôte soit ajouté automatiquement
ddns-hostname = pick-first-value(option host-name, host-decl-name, concat("client-", binary-to-ascii(10, 8, "-", leased-address)));
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

## 2. Assurez-vous que la configuration du client fournit un nom d'hôte

Sur chaque client, assurez-vous que le nom d'hôte est correctement configuré:

```bash
# Vérifier le nom d'hôte actuel
hostname

# Si nécessaire, définir un nom d'hôte permanent
sudo hostnamectl set-hostname client1.est.intra
```

## 3. Configuration pour forcer l'envoi du nom d'hôte pendant les requêtes DHCP

Sur le client, modifiez le fichier de configuration dhclient:

```bash
sudo nano /etc/dhcp/dhclient.conf
```

Ajoutez ou modifiez les lignes suivantes:

```
send host-name = gethostname();
supersede host-name = gethostname();
```

## 4. Redémarrer les services sur le serveur

```bash
sudo systemctl restart named
sudo systemctl restart dhcpd
```

## 5. Renouveler le bail DHCP sur le client

```bash
sudo dhclient -r
sudo dhclient -v
```

## 6. Vérification

Pour vérifier que tout fonctionne correctement:

1. Sur le client:
   ```bash
   ping ns1.est.intra
   ```

2. Sur le serveur, vérifiez les zones DNS:
   ```bash
   sudo cat /var/named/est.intra.zone
   sudo cat /var/named/192.168.1.rev
   ```

3. Vérifiez les journaux DDNS:
   ```bash
   sudo tail -f /var/log/named-ddns.log
   ```

4. Vérifiez que vous pouvez résoudre le nom du client depuis le serveur:
   ```bash
   dig @localhost client1.est.intra
   ```

5. Vérifiez la résolution inverse:
   ```bash
   dig @localhost -x 192.168.1.101  # Remplacez par l'IP du client
   ```

Si tout est correctement configuré, lorsqu'un client obtient une adresse IP via DHCP, son nom d'hôte et son adresse IP seront automatiquement ajoutés aux zones DNS directe et inverse, même s'il fait simplement un ping. Le serveur DHCP envoie les mises à jour DNS au nom du client.
