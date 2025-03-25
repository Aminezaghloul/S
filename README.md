L'erreur "Unit dhcpd.service has a bad unit file setting" persiste après votre modification. Cela est probablement dû à un problème dans le fichier override que nous avons créé précédemment.

Voici comment résoudre ce problème:

1. Vérifiez et corrigez le fichier d'override:
```bash
sudo nano /etc/systemd/system/dhcpd.service.d/override.conf
```

Assurez-vous que le contenu est exactement comme ceci (pas d'espaces ou de caractères supplémentaires):
```
[Service]
ExecStart=
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd ens160
```

Notez la ligne `ExecStart=` vide qui est nécessaire pour écraser la ligne ExecStart originale.

2. Rechargez la configuration systemd:
```bash
sudo systemctl daemon-reload
```

3. Redémarrez le service:
```bash
sudo systemctl restart dhcpd
```

4. Vérifiez que le service fonctionne correctement:
```bash
sudo systemctl status dhcpd
```

Cette correction devrait résoudre le problème de "bad unit file setting" tout en maintenant votre configuration DHCP pour qu'elle fonctionne sur l'interface réseau souhaitée.
