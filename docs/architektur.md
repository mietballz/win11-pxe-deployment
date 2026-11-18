# Architektur

## Dienste auf dem Deployment-Server

Ubuntu Server 24.04 LTS als VM auf Proxmox VE, statische Adresse im Deployment-VLAN (Netplan).

| Dienst | Software | Port | Aufgabe |
|---|---|---|---|
| ProxyDHCP | dnsmasq | 67/udp | Liefert nur Boot-Informationen, vergibt keine Adressen |
| TFTP | dnsmasq (`enable-tftp`) oder tftpd-hpa | 69/udp | Liefert `shimx64.efi` und `grubx64.efi` |
| HTTP-Boot | nginx | 80/tcp | Liefert `boot.ipxe`, `wimboot`, `bcd`, `boot.sdi`, `boot.wim` (~500 MB) |
| Installationsquelle | Samba (SMB3) | 445/tcp | Windows-Quellen, Treiber, Antwortdateien, Skripte – nur mit Anmeldung |

> **Hinweis dnsmasq:** `port=0` schaltet die DNS-Funktion ab. Sonst kollidiert dnsmasq mit
> `systemd-resolved` auf Port 53 und startet nicht.

Konfigurationsvorlagen: [`config/`](../config) und [`http/boot.ipxe`](../http/boot.ipxe)

## Secure-Boot-Bootkette

```
UEFI PXE
  └─ shimx64.efi            (Microsoft/Canonical-signiert, per TFTP)
       └─ grubx64.efi       (= offiziell signiertes iPXE-Shim unter dem von shim erwarteten Namen)
            └─ boot.ipxe    (per HTTP)
                 └─ wimboot + boot.wim
                      └─ Windows PE → startnet.cmd
```

Der Shim lädt standardmäßig eine Datei namens `grubx64.efi`. An dieser Stelle liegt das offiziell signierte
iPXE-Shim. Damit startet iPXE auf VM und Hardware, ohne dass auf jedem Client ein eigener MOK für eine
selbst signierte Datei registriert werden muss.

| Datei | Ablageort | Funktion |
|---|---|---|
| `shimx64.efi` | `/srv/tftp/` | Signierter Einstiegspunkt für Secure Boot |
| `grubx64.efi` | `/srv/tftp/` | Signiertes iPXE-Shim unter erwartetem Dateinamen |
| `mmx64.efi` | `/srv/tftp/` | MokManager – optional, nicht im Bootpfad |
| `boot.ipxe` | `/var/www/html/boot/` | iPXE-Skript für wimboot und boot.wim |
| `boot.wim` | `/var/www/html/boot/` | Angepasstes Windows PE |

## Warum HTTP und nicht HTTPS

Ein offiziell signiertes iPXE vertraut einem internen, selbst signierten Zertifikat nicht
(`Permission denied 0x0216eb`). Es lässt sich nicht mit eigenem Trust-Anchor neu bauen, ohne die Signatur
zu verlieren. Ein selbst gebautes iPXE hätte wiederum MOK-Enrollment auf jedem Client erfordert.

Entscheidung: **HTTP ausschließlich im internen, segmentierten VLAN.** Ausgeglichen wird das durch:

- Netzsegmentierung (Deployment-VLAN) und Zugriffsbeschränkung in nginx
- Samba nur mit Benutzeranmeldung, kein Gastzugriff
- kontrollierte Dateiberechtigungen
- Secure Boot validiert die Bootkette unabhängig vom Transportweg

## WinPE-Anpassung

`boot.wim` wird mit `wimlib-imagex` direkt auf dem Linux-Server angepasst:

- **Treiber-Injektion:** Storage- und Netzwerktreiber für VirtIO (`vioscsi.inf`, `netkvm.inf`) und die
  physische Hardware, damit Zielplatte und Netzwerkkarte in WinPE sofort erkannt werden.
- **Umgehung der 25H2-Setup-Engine:** `setup.exe` im WIM-Wurzelverzeichnis entfernt. Die neue Engine
  (ConX/SetupPrep) ignoriert Teile der Antwortdatei.
- **startnet.cmd:** Steuert den Ablauf in WinPE.

| Schritt | Mechanismus | Zweck |
|---|---|---|
| Treiber laden | `drvload` | Storage- und Netzwerktreiber verfügbar machen |
| Netzwerk starten | `wpeutil InitializeNetwork` / `WaitForNetwork` | Netzwerkstack starten, Warteschleife für Switch-Latenzen |
| Freigabe verbinden | `net use Z: \\<PXE_SERVER_IP>\install /user:deployment` | Installationsquelle per SMB3 |
| Setup starten | `Z:\sources\setup.exe /unattend:…` | Klassischer Setup-Pfad im UEFI-Modus |

> **Wichtig – WIM-Index:** `boot.wim` enthält zwei Indizes. wimboot bootet ohne Angabe **Index 1**.
> Treiber und `startnet.cmd` müssen in dem Index liegen, der tatsächlich gebootet wird – am sichersten in
> beiden identisch.

## Treiberstrategie

Der automatische Import aus `$WinPEDriver$` wurde deaktiviert. Windows Setup scannt diesen Ordner und
übernimmt viele Pakete – auch bootkritische, die nicht zum Zielsystem passen. Ergebnis waren
Setup-Abbrüche, obwohl DiskPart die Zielplatte korrekt sah.

Stattdessen gibt es eine kontrollierte Struktur:

| Pfad | Inhalt |
|---|---|
| `drivers/client/storage` | Storage-Treiber für physische Clients (z. B. Intel VMD/RST, nur bei Bedarf) |
| `drivers/client/network` | Netzwerktreiber passend zur Hardware |
| `drivers/vm/storage` | `vioscsi.inf` / `viostor.inf` |
| `drivers/vm/network` | `netkvm.inf` |

**Regel:** Ins WinPE nur `.inf`, `.sys`, `.cat` (und von der INF referenzierte Dateien) – keine Installer,
keine `.exe`-Pakete. Ballast im WinPE führt zu Setup-Abbrüchen.

## Verzeichnisstruktur

```
/srv/tftp/
├── shimx64.efi
├── grubx64.efi              # signiertes iPXE-Shim
└── mmx64.efi                # optional

/var/www/html/boot/
├── boot.ipxe
├── wimboot
├── bcd
├── boot.sdi
└── boot.wim

/srv/samba/install/
├── sources/                 # Windows-11-Quellen inkl. install.wim
├── drivers/
│   ├── client/{storage,network}
│   └── vm/{storage,network}
├── autounattend.xml
└── scripts/                 # Post-Install und Domain Join
```
