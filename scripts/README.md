# Skripte

| Skript | Phase | Läuft in | Aufgabe |
|---|---|---|---|
| `startnet.cmd` | WinPE | `boot.wim` | Treiber laden, Netzwerk starten, Freigabe verbinden, Setup starten |
| `autounattend.xml` | Setup | Installationsfreigabe | Regionaleinstellungen, OOBE überspringen, lokales Admin-Konto, FirstLogonCommands |
| `SetupComplete.cmd` | nach Setup | installiertes Windows (SYSTEM) | Modus erkennen, Folgeaufgaben als geplante Tasks verankern |
| `remove-bloatware.ps1` | Post-Install | installiertes Windows | Vorinstallierte Apps entfernen |
| `install-apps.ps1` | Post-Install | installiertes Windows | Standardsoftware per winget, Updates per PSWindowsUpdate |
| `set-computername.ps1` | Pre-Join | installiertes Windows | Gerätetyp erkennen, freien Namen ermitteln, Rename |
| `domain-join.ps1` | Join | installiertes Windows | `Add-Computer`, AutoLogon entfernen, Neustart |
| `write-device-user-mapping.ps1` | Post-Join | Client | JSON mit Gerät und Benutzer schreiben |
| `update-ad-device-user-mapping.ps1` | Post-Join | Domänencontroller (Task) | `Description` / `ManagedBy` setzen |
| `register-serialnumber-ad.ps1` | Post-Join | Client (SYSTEM) | BIOS-Seriennummer in `serialNumber` schreiben |

Kennwörter werden in keinem Skript im Repository gespeichert. Alle Stellen, an denen Anmeldedaten nötig
sind, sind mit `<PLATZHALTER>` markiert.
