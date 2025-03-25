Je vois qu'il y a un problème de permissions avec le service BIND (named). Les messages d'erreur clés sont:

```
isc_stdio_open '/var/log/named-ddns.log' failed: permission denied
configuring logging: permission denied
loading configuration: permission denied
exiting (due to fatal error)
```

Le service named ne peut pas ouvrir/écrire dans le fichier journal que nous avons configuré. Voici comment résoudre ce problème:

1. Créez et configurez correctement le fichier journal:

```bash
# Créer le fichier journal
sudo touch /var/log/named-ddns.log

# Définir les bonnes permissions
sudo chown named:named /var/log/named-ddns.log
sudo chmod 644 /var/log/named-ddns.log

# Définir le contexte SELinux correct
sudo semanage fcontext -a -t named_log_t "/var/log/named-ddns.log"
sudo restorecon -v /var/log/named-ddns.log
```

2. Ensuite, modifiez votre fichier named.conf pour utiliser un chemin de journal plus standard ou assurez-vous que le chemin est accessible:

```bash
sudo nano /etc/named.conf
```

Modifiez la section logging comme ceci:
```
logging {
    channel ddns_log {
        file "/var/named/data/named-ddns.log";  // Utiliser un emplacement dans /var/named/data
        severity dynamic;
        print-time yes;
        print-severity yes;
        print-category yes;
    };
    category update { ddns_log; };
    category update-security { ddns_log; };
    category security { ddns_log; };
};
```

3. Assurez-vous que SELinux permet à named d'écrire dans les journaux:

```bash
sudo setsebool -P named_write_master_zones 1
sudo setsebool -P named_write_master_zones 1
```

4. Redémarrez le service:

```bash
sudo systemctl restart named
```

5. Vérifiez l'état:

```bash
sudo systemctl status named
```

Si vous continuez à avoir des problèmes avec la journalisation, vous pouvez également essayer de simplifier en supprimant temporairement la section de journalisation de named.conf pour voir si le reste de la configuration fonctionne.
