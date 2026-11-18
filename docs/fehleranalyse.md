# Fehleranalyse und Problemlösung

Die Ursachen wurden nicht nur behoben, sondern systematisch analysiert und die Entscheidungen begründet.

## Während des Projekts

| Fehlerbild | Ursache | Lösung | Nachweis |
|---|---|---|---|
| UEFI PXE: *Access Denied* / *Selected boot image did not authenticate* | Nicht vertrauenswürdige Bootkette unter Secure Boot; eigener MOK-Ansatz auf Hardware nicht zuverlässig | Umstellung auf offiziell signierten Shim/iPXE-Shim (`shimx64.efi → grubx64.efi`), kein MOK-Enrollment mehr | iPXE startet auf VM und HP-Client |
| iPXE: *Permission denied (0x0216eb)* bei HTTPS | Signiertes iPXE vertraut dem internen, selbst signierten Zertifikat nicht | Bewusste Entscheidung für HTTP im internen VLAN, abgesichert über Segmentierung und Samba-Anmeldung | `boot.ipxe` und `boot.wim` laden stabil |
| Setup meldet Starttreiberproblem | Zu breiter Import aus `$WinPEDriver$` mit unpassenden bootkritischen Treibern | `$WinPEDriver$` deaktiviert, kontrollierte `drivers/`-Struktur | Setup läuft ohne Treiberabbruch |
| Windows 11 25H2 ignoriert Teile der `autounattend.xml`, OOBE erscheint | Neue Setup-Engine (ConX/SetupPrep) überspringt Antwortdateien im `oobeSystem`-Pass | `setup.exe` aus dem WIM-Wurzelverzeichnis entfernt; `startnet.cmd` ruft `sources\setup.exe` direkt auf | Desktop ohne OOBE, Konto angelegt |
| *„< kann syntaktisch an dieser Stelle nicht verarbeitet werden“* | Spitze Klammern werden in Batch als Umleitung interpretiert; Linux-Zeilenenden (LF) | Spitze Klammern entfernt; nach jeder Bearbeitung `unix2dos` vor der WIM-Integration | `startnet.cmd` läuft durch |
| Änderungen an `startnet.cmd` ohne Wirkung | Datei nicht erneut in die `boot.wim` geschrieben; WinPE lud die alte Version | Nach jeder Änderung `wimlib-imagex update`, danach VM vollständig ausschalten vor dem nächsten PXE-Boot | Änderungen werden übernommen |
| Kein Netzwerk nach der Installation | Per `drvload` in WinPE geladene Treiber werden nicht ins installierte Windows übernommen | Treiber zusätzlich über einen gezielten Pfad im Setup einbinden | Netzwerk nach Neustart verfügbar |
| Domain Join schlägt fehl / falsche OU | Join lief noch mit temporärem Setup-Namen (vor dem Rename) | Reihenfolge geändert: Rename + Neustart vor `domain-join.ps1` | Computerobjekt mit richtigem Namen in `OU=Clients` |

## Erkenntnisse aus dem Produktivbetrieb

| Fehlerbild | Ursache | Lösung |
|---|---|---|
| Treiber liegt nachweislich in der `boot.wim`, `drvload` findet ihn trotzdem nicht | `boot.wim` hat zwei Indizes; gepflegt wurde Index 2, wimboot bootet ohne Angabe aber Index 1 | Index 1 auf den Stand von Index 2 gebracht; künftig beide identisch pflegen |
| `drvload` meldet `0x80070002` für den VMD-Treiber | Die INF referenziert eine weitere Datei, die im schlanken Treiberordner fehlte | Alle in der INF referenzierten Dateien mitliefern |
| Neuer Gerätename kollidiert beim Join (*„Das Konto existiert bereits“*) | Freie-Nummer-Suche lief nur in einer OU; gleichnamige Geräte lagen in anderen OUs | Suche domänenweit plus Freiheitsprüfung mit Hochzählen |
| `schtasks` meldet *„Erforderliche Option TR fehlt“* | Ein `!` im Kennwort wird bei `setlocal EnableDelayedExpansion` als Variablenbegrenzer gelesen und verschluckt Parameter | Task-Anlage in eine Subroutine mit `setlocal DisableDelayedExpansion` ausgelagert |
| `boot.wim` nach dem Aktualisieren 0 Byte groß | Abgebrochener Schreibvorgang beim Update über einen Symlink | Aus Backup wiederhergestellt; Änderungen nur noch per `wimmountrw` / `wimunmount --commit`, vorher immer Backup |
| Interne Netzwerkkarte bekommt in WinPE keine DHCP-Adresse (APIPA), USB-Adapter am selben Port schon | Verhalten der internen Karte im WinPE-Treiberstack; Port/VLAN durch Kabeltausch ausgeschlossen | Deployment dieser Modelle über USB-Ethernet-Adapter |

## Arbeitsregeln, die sich daraus ergeben haben

- Vor jeder Änderung an `boot.wim` ein datiertes Backup anlegen.
- Batch-Dateien immer mit CRLF speichern (`unix2dos`).
- Treiber- und Antwortdatei-Experimente nie direkt auf dem produktiven Stand – im Zweifel auf die letzte
  stabile Version zurückrollen.
- Nach einer `boot.wim`-Änderung Test-VMs vollständig ausschalten, nicht nur neu starten.
