# Projektdokumentation

**Planung und Implementierung eines PXE-Servers auf Linux-Basis zur automatisierten Bereitstellung von
Windows 11 Pro mit UEFI Secure Boot, automatisierter Nachinstallation und Active Directory Domain Join**

Betriebliches Abschlussprojekt · Fachinformatiker für Systemintegration · geplante Projektzeit 40 Stunden

> Diese Fassung ist für die Veröffentlichung gekürzt und anonymisiert. Unternehmen, Personen, Domäne,
> Rechnernamen und Netzadressen sind durch neutrale Beispiele ersetzt.

---

## 1 Projektumfeld und Ausgangssituation

### 1.1 IT-Umfeld

Die Infrastruktur umfasst eine virtualisierte Serverumgebung auf Basis von **Proxmox VE** sowie physische
Clients (HP-Hardware mit UEFI Secure Boot). Für das Projekt wurde ein **Ubuntu Server 24.04 LTS** als VM
bereitgestellt. Sie dient als zentrale PXE-, TFTP-, HTTP- und Samba-Instanz im internen Management-VLAN.

Die IP-Vergabe erfolgt weiterhin über den produktiven DHCP-Dienst. Der Deployment-Server darf diesen
nicht ersetzen oder verändern. Deshalb wird **dnsmasq im ProxyDHCP-Modus** eingesetzt: Der produktive
DHCP-Server vergibt die Adresse, der PXE-Server liefert nur die Boot-Informationen.

### 1.2 IST-Analyse

Windows-11-Clients wurden bisher manuell über lokale Installationsmedien eingerichtet.

| Problemfeld | Beschreibung |
|---|---|
| Zeitaufwand | 60–90 Minuten aktive Administrationszeit pro Client |
| Skalierung | Jede Installation bindet einen Administrator vollständig |
| Aktualität | USB-Medien sind eine Momentaufnahme; danach umfangreiche Updates und Treiber-Downloads nötig |
| Fehleranfälligkeit | Manuelle Benutzeranlage und Softwareinstallation ohne standardisierten Ablauf |

### 1.3 Problemstellung

Windows 11 Pro soll automatisiert über das Netzwerk bereitgestellt werden – **ohne Secure Boot zu
deaktivieren** und **ohne den produktiven DHCP-Dienst zu verändern**. Zu berücksichtigen sind virtuelle
Maschinen in Proxmox und physische HP-Hardware. Als Erweiterung des Projektumfangs kam der automatische
Beitritt zur AD-Domäne hinzu.

---

## 2 Zielsetzung und Anforderungen

### 2.1 SOLL-Konzept

Clients starten per UEFI-Netzwerkboot, gelangen über eine offiziell signierte Secure-Boot-Kette zu iPXE,
laden Windows PE und führen das Setup automatisiert aus. Installationsdateien kommen über Samba, Boot-Dateien
über TFTP und HTTP. Nach der Installation treten die Clients automatisch der Domäne bei, erhalten einen
Namen nach Namensschema (z. B. `ORGWS001`), werden in `OU=Clients` abgelegt und per AD-Attribut einem
Benutzer zugeordnet.

### 2.2 Anforderungen

| ID | Art | Anforderung | Priorität |
|---|---|---|---|
| REQ-01 | funktional | Windows 11 Pro wird per PXE über das lokale Netzwerk bereitgestellt | Muss |
| REQ-02 | funktional | Installation läuft unbeaufsichtigt über `autounattend.xml`; OOBE wird übersprungen, ein lokales Admin-Konto angelegt | Muss |
| REQ-03 | funktional | Standardsoftware wird automatisiert per winget installiert | Soll |
| REQ-04 | funktional | UEFI Secure Boot bleibt auf allen Clients aktiv | Muss |
| REQ-05 | nicht-funktional | Der produktive DHCP-Server wird nicht verändert (ProxyDHCP) | Muss |
| REQ-06 | nicht-funktional | Die aktive Administratorzeit pro Client sinkt deutlich | Soll |
| REQ-07 | nicht-funktional | Treiber werden kontrolliert und zielsystembezogen eingebunden | Muss |
| REQ-08 | funktional | Automatischer Domänenbeitritt (OU=Clients) mit Namensschema und Gerät/Benutzer-Zuordnung | Muss |

---

## 3 Projektplanung

### 3.1 Phasen- und Zeitplanung

| Phase | Tätigkeiten | SOLL | IST |
|---|---|---|---|
| Analyse | IST-Aufnahme, Anforderungen, Alternativen, Risiken | 8 h | 8 h |
| Entwurf | Architektur, Secure-Boot-Ansatz, Dienste- und AD-Konzept | 4 h | 4 h |
| Implementierung | Ubuntu-Dienste, TFTP/HTTP/Samba, WinPE, Antwortdatei, Skripte, Domain-Join-Kette | 15 h | 17 h |
| Test / QS | VM- und Hardwaretests, Fehleranalyse, Domain-Join-Test (3 Clients) | 5 h | 5 h |
| Dokumentation | Projekt- und Betriebsdokumentation | 8 h | 6 h |
| **Gesamt** | | **40 h** | **40 h** |

Die Implementierung lag 2 Stunden über Plan – vor allem wegen der Analyse der 25H2-Setup-Engine, der
Treiberstrategie und der Domain-Join-Kette. Ausgeglichen wurde das in der Dokumentationsphase.

### 3.2 Ressourcen

| Ressource | Einsatz |
|---|---|
| Ubuntu Server 24.04 LTS | PXE-, TFTP-, HTTP- und Samba-Server (VM auf Proxmox VE) |
| Proxmox VE | Virtualisierungs- und Testumgebung (UEFI/OVMF, q35, VirtIO) |
| HP-Client | Physischer Praxistest mit Secure Boot und NVMe-SSD |
| Windows 11 Pro 25H2 | Installationsmedium (`install.wim`) aus der vorhandenen Lizenz |
| VirtIO- / HP-Treiber | Kontrolliert unter `drivers/client` und `drivers/vm` |
| dnsmasq, nginx, Samba | Basisdienste – ausschließlich Open Source |
| Windows Server (AD DS) | Bestehender Domänencontroller, keine Zusatzlizenz |

### 3.3 Wirtschaftlichkeitsbetrachtung

Beispielrechnung mit angenommenen internen Sätzen:

| Kennzahl | Manuell (IST) | PXE (SOLL) |
|---|---|---|
| Aktive Arbeitszeit pro Client | 75 min | 15 min |
| Interner Stundensatz | 45,00 € | 45,00 € |
| Kosten pro Installation | 56,25 € | 11,25 € |
| Rollouts pro Monat | 15 | 15 |
| Personalkosten pro Monat | 843,75 € | 168,75 € |
| **Einsparung pro Monat** | | **675,00 €** |

Projektkosten 1.200,00 € (40 h × 30,00 €), Serverbetrieb 25,00 €/Monat:
**1.200,00 € / (675,00 € − 25,00 €) ≈ 1,85 Monate Amortisation.**

### 3.4 Bewertung von Lösungsalternativen

| Kriterium | Gewicht | WDS | FOG Project | iPXE + dnsmasq + nginx |
|---|---|---|---|---|
| Lizenzkosten | 20 % | 2 | 4 | 5 |
| Secure-Boot-Fähigkeit | 20 % | 3 | 2 | 4 |
| Integration ohne DHCP-Änderung | 20 % | 2 | 3 | 5 |
| Flexibilität / Anpassbarkeit | 20 % | 3 | 4 | 5 |
| Eignung für heterogene Hardware | 20 % | 3 | 3 | 4 |
| **Gesamt** | 100 % | 2,6 | 3,2 | **4,6** |

Gewählt wurde die Linux-Lösung: lizenzkostenfrei, flexibel und per ProxyDHCP ohne Änderung des produktiven
DHCP integrierbar. Die höhere technische Komplexität wurde bewusst in Kauf genommen.

---

## 4 Durchführung

Die technische Umsetzung ist in eigenen Dokumenten beschrieben:

- [Architektur, Dienste, Bootkette, WinPE und Treiber](architektur.md)
- [Active Directory Domain Join](domain-join.md)

### 4.1 Unattended Installation und Post-Install

Die `autounattend.xml` enthält die regionalen Einstellungen für Deutschland (`de-DE`, GeoID 94,
`W. Europe Standard Time`) und überspringt die OOBE. Der Namespace `xmlns:wcm` wurde global im Kopf der
Datei deklariert – eine fehlerhafte Deklaration hatte zuvor zu Installationsabbrüchen geführt.

Nach dem ersten Login ruft `FirstLogonCommands` die Post-Install-Kette auf: Entfernen vorinstallierter Apps,
Installation der Standardsoftware per **winget**, Windows-Updates über **PSWindowsUpdate** und
anschließend die Domain-Join-Kette.

---

## 5 Fehleranalyse

Siehe [fehleranalyse.md](fehleranalyse.md) – mit Fehlerbild, Ursache, Lösung und Nachweis für jedes Problem.

---

## 6 Qualitätssicherung und IT-Sicherheit

### 6.1 Testprotokoll

| TF | Testfall | System | Erwartetes Ergebnis | Status |
|---|---|---|---|---|
| TF-01 | UEFI PXE Boot | VM | Client lädt `shimx64.efi` und startet iPXE | ✅ |
| TF-02 | UEFI PXE Boot | HP-Client | iPXE startet über den signierten Shim | ✅ |
| TF-03 | HTTP-Boot | VM + HP | `boot.ipxe` und `boot.wim` werden per HTTP geladen | ✅ |
| TF-04 | WinPE-Netzwerk | VM + HP | SMB3-Verbindung zur Freigabe erfolgreich | ✅ |
| TF-05 | Festplattenerkennung | VM + HP | DiskPart zeigt die Zielplatte | ✅ |
| TF-06 | Unattended Setup | VM + HP | Setup startet ohne Eingaben | ✅ |
| TF-07 | Benutzer / OOBE | VM + HP | Lokales Admin-Konto angelegt, OOBE übersprungen | ✅ |
| TF-08 | Post-Install | VM + HP | Software installiert, Log unter `C:\Logs` | ✅ |
| TF-09 | Rename | VM + HP | Client trägt nach Neustart den finalen Namen | ✅ |
| TF-10 | Domain Join | 3 Clients | Computerobjekt in `OU=Clients`, Log mit Erfolgsmeldungen | ✅ |
| TF-11 | AutoLogon-Cleanup | 3 Clients | Lokale AutoLogon-Einträge entfernt | ✅ |
| TF-12 | Gerät/Benutzer-Mapping | 3 Clients | `Description` und `ManagedBy` gesetzt | ✅ |

### 6.2 Technische und organisatorische Maßnahmen

| Schutzziel | Maßnahme | Bewertung |
|---|---|---|
| Integrität | Secure Boot mit offiziell signiertem Shim/iPXE-Shim | Start über unsignierte Bootloader wird verhindert |
| Verfügbarkeit | ProxyDHCP statt Änderung des produktiven DHCP | Ausfall des Deployment-Servers beeinträchtigt die IP-Vergabe nicht |
| Vertraulichkeit | Samba nur mit Anmeldung; HTTP nur im internen VLAN | Kein anonymer Zugriff auf die Installationsquelle |
| Nachvollziehbarkeit | Logs, Testmatrix, Abnahme | Protokolle unter `C:\Logs\SystemSetup` und `C:\Logs\DomainJoin` |
| Least Privilege | Dedizierte Dienstkonten mit minimalen `dsacls`-Rechten | Kein Schritt nutzt Domain-Admin-Anmeldedaten |

---

## 7 Soll-Ist-Vergleich

| Req. | Anforderung | Umsetzung | Status |
|---|---|---|---|
| REQ-01 | Windows 11 per PXE | iPXE / wimboot / nginx | ✅ erfüllt |
| REQ-02 | Unbeaufsichtigte Installation | 25H2-Problem über klassischen Setup-Pfad umgangen | ✅ erfüllt |
| REQ-03 | Automatische Softwareinstallation | winget + PSWindowsUpdate | ✅ erfüllt |
| REQ-04 | Secure Boot aktiv | Signierter Shim/iPXE-Shim, auf VM und HP getestet | ✅ erfüllt |
| REQ-05 | Kein Eingriff in DHCP | dnsmasq im ProxyDHCP-Modus | ✅ erfüllt |
| REQ-06 | Adminzeit deutlich reduziert | gemessen ca. 5 Minuten aktive Zeit | ✅ erfüllt |
| REQ-07 | Kontrollierte Treiberstrategie | `drivers/client` und `drivers/vm` statt `$WinPEDriver$` | ✅ erfüllt |
| REQ-08 | Domänenbeitritt mit Namensschema und Mapping | Rename-, Join- und Mapping-Skripte, validiert auf 3 Clients | ✅ erfüllt |

---

## 8 Fazit und Ausblick

Eine vollständig automatisierte Windows-11-Bereitstellung mit Secure Boot, unbeaufsichtigter Installation,
Software-Nachinstallation und Domänenbeitritt ist **ohne Änderung des produktiven DHCP-Servers** möglich.
Die eigentlichen Herausforderungen lagen nicht in der Installation der Dienste, sondern in der sicheren
Bootkette, der Transportentscheidung für iPXE, der Treiberstrategie und der richtigen Reihenfolge der
Domain-Join-Kette.

**Mögliche Erweiterungen:**

- Multi-OS-Bootmenü (weitere Windows- oder Linux-Images)
- Zentrale Auswertung der Deployment-Logs (Webhook oder Log-Aggregation)
- Hardwareprofile mit Auswahl von Treiber- und Softwarepaketen je Gerätetyp
- Dynamische Ziel-OU nach Gerätetyp oder Standort

---

## Abkürzungen

| Abkürzung | Bedeutung |
|---|---|
| AD | Active Directory |
| DHCP | Dynamic Host Configuration Protocol |
| iPXE | Erweiterter Open-Source-Netzwerk-Bootloader |
| MOK | Machine Owner Key (Secure-Boot-Schlüssel) |
| NBP | Network Bootstrap Program |
| OOBE | Out-of-Box Experience (Windows-Ersteinrichtung) |
| OU | Organizational Unit |
| PXE | Preboot eXecution Environment |
| SBAT | Secure Boot Advanced Targeting |
| SMB | Server Message Block |
| TFTP | Trivial File Transfer Protocol |
| UEFI | Unified Extensible Firmware Interface |
| VLAN | Virtual Local Area Network |
| WIM | Windows Imaging Format |
| WinPE | Windows Preinstallation Environment |

## Quellen

1. Microsoft: Windows Assessment and Deployment Kit (ADK) – <https://learn.microsoft.com/de-de/windows-hardware/get-started/adk-install>
2. Microsoft: Unattended Windows Setup Reference – <https://learn.microsoft.com/de-de/windows-hardware/customize/desktop/unattend/>
3. iPXE: Dokumentation – <https://ipxe.org/docs>
4. Canonical: Ubuntu Server Dokumentation – <https://ubuntu.com/server/docs>
5. Proxmox: Proxmox VE Dokumentation – <https://pve.proxmox.com/pve-docs/>
6. Microsoft: Active Directory Domain Services – <https://learn.microsoft.com/de-de/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview>
