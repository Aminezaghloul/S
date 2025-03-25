Je vois un résultat très intéressant ! Votre client a maintenant obtenu des adresses IP sur les DEUX réseaux :

1. `192.168.5.131/24` avec passerelle `192.168.5.255`
2. `192.168.1.101/24` avec passerelle `192.168.1.255` 

Et le plus impressionnant, c'est que le client a accepté l'offre de votre serveur DHCP (192.168.1.1) comme on peut le voir dans le message:
```
DHCPACK of 192.168.1.101 from 192.168.1.1 (xid=0x7383ce67)
bound to 192.168.1.101 -- renewal in 1349 seconds.
```

C'est exactement ce que nous voulions ! Le client a bien obtenu une adresse IP du réseau 192.168.1.0/24 tout en conservant sa connectivité au réseau 192.168.5.0/24.

Cette configuration permet au client d'être connecté aux deux réseaux simultanément, ce qui est une configuration d'interface réseau à multiples adresses IP (multihomed).

Pour vérifier maintenant que les mises à jour DNS fonctionnent correctement, essayez:

```bash
# Sur le serveur
sudo cat /var/named/est.intra.zone

# Vérifiez que vous pouvez résoudre le nom du client
dig @192.168.1.1 client-192-168-1-101.est.intra
```

Si l'enregistrement DNS n'est pas encore créé, depuis le client, essayez:

```bash
ping ns1.est.intra
```

Cette configuration semble fonctionner correctement maintenant!
