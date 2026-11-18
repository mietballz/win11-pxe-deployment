# Betrieb (Kurzform)

| Aufgabe | Vorgehen | Hinweis |
|---|---|---|
| Windows-Image aktualisieren | Neue Quelle unter `/srv/samba/install/sources` ablegen, `boot.wim` prüfen | Vorher Backup |
| Treiber ergänzen | Zielsystembezogen unter `drivers/client` oder `drivers/vm` ablegen | Kein breiter `$WinPEDriver$`-Import |
| `autounattend.xml` ändern | Datei bearbeiten, mit `xmllint --noout` prüfen | Keine Kennwörter im Klartext dokumentieren |
| `startnet.cmd` ändern | `unix2dos`, per `wimmountrw` in **beide** Indizes schreiben, `wimunmount --commit`, nach `/var/www/html/boot/` kopieren | Test-VM vor dem nächsten PXE-Boot ausschalten |
| Dienste prüfen | `systemctl status dnsmasq nginx smbd` | Nach Änderungen neu starten |
| Domain-Join-Log prüfen | `Get-Content C:\Logs\DomainJoin\domain-join.log -Tail 80` | Drei Erfolgsmeldungen müssen vorhanden sein |
| AD-Mapping prüfen (DC) | `Get-ADComputer -Filter 'Name -like "ORGWS*"' -Properties Description,ManagedBy,serialNumber` | Mapping-Task ggf. manuell starten |
| Test | PXE-Boot mit VM, bei Bedarf physischer Client | Testmatrix TF-01 bis TF-12 aktualisieren |

## Nützliche Befehle

Dienste und Logs:

```bash
systemctl status dnsmasq nginx smbd
journalctl -u dnsmasq -f
tail -f /var/log/nginx/pxe-access.log
```

WIM untersuchen:

```bash
wiminfo /var/www/html/boot/boot.wim
wimdir /var/www/html/boot/boot.wim 1 | grep -i startnet
```

Backup vor einer Änderung:

```bash
cp -a /var/www/html/boot/boot.wim /var/www/html/boot/boot.wim.bak-$(date +%Y%m%d-%H%M)
```

`startnet.cmd` sicher in Index 1 schreiben (für Index 2 wiederholen):

```bash
mkdir -p /mnt/wim && wimmountrw /var/www/html/boot/boot.wim 1 /mnt/wim
```

```bash
cp startnet.cmd /mnt/wim/Windows/System32/startnet.cmd
```

```bash
wimunmount --commit /mnt/wim
```

Antwortdatei prüfen:

```bash
xmllint --noout /srv/samba/install/autounattend.xml
```
