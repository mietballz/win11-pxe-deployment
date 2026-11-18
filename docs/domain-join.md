# Active Directory Domain Join

Nach der Post-Install-Automatisierung treten die Clients automatisch der Domäne `corp.example.com` bei.
Der Ablauf ist in drei getrennte Phasen geteilt. Ein noch nicht beigetretener Client ist dadurch zu keinem
Zeitpunkt von domänenabhängigen Freigaben oder Benutzerinformationen abhängig.

## Phasenmodell

```mermaid
flowchart LR
    A[Pre-Join<br/>set-computername.ps1] -->|Rename + Neustart| B[Domain Join<br/>domain-join.ps1]
    B -->|Neustart| C[Post-Join<br/>Mapping + serialNumber]
```

| Phase | Skript | Aufgabe |
|---|---|---|
| Pre-Join | `set-computername.ps1` | Hardware erkennen, freien Namen nach Schema ermitteln, Rename + Neustart |
| Domain Join | `domain-join.ps1` | `Add-Computer` in `OU=Clients`, AutoLogon entfernen, Neustart |
| Post-Join: Mapping | `write-device-user-mapping.ps1` + DC-Task `update-ad-device-user-mapping.ps1` | JSON erzeugen, `Description` / `ManagedBy` am Computerobjekt setzen |
| Post-Join: Seriennummer | `register-serialnumber-ad.ps1` | Eigene BIOS-Seriennummer per SELF-Recht in `serialNumber` schreiben |

> **Reihenfolge ist entscheidend:** Läuft der Join vor dem Rename, landet das Gerät mit dem zufälligen
> Setup-Namen in der Domäne. `set-computername.ps1` muss daher **vor** `domain-join.ps1` laufen – mit
> Neustart dazwischen.

## Namensvergabe (`set-computername.ps1`)

Namensschema: Präfix nach Gerätetyp + dreistellige Nummer, z. B. `ORGWS001` (Desktop) oder `ORGLAP001`
(Notebook).

- Gerätetyp über `Win32_SystemEnclosure` (Chassis-Typ) erkennen
- Nächste freie Nummer **domänenweit** ermitteln – nicht nur in einer OU, sonst kollidiert der Name mit
  Geräten in anderen OUs
- Vor dem Rename prüfen, dass der Name in der Domäne wirklich frei ist
- Name gegen ein festes Muster validieren (z. B. `^ORG(WS|LAP)[0-9]{3}$`)

## Domänenbeitritt (`domain-join.ps1`)

| Aufgabe | Beschreibung |
|---|---|
| Beitritt | `Add-Computer` gegen `corp.example.com`, Ziel `OU=Clients,DC=corp,DC=example,DC=com` |
| AutoLogon-Cleanup | Lokale AutoLogon-Einträge nach erfolgreichem Join entfernen |
| Log | `C:\Logs\DomainJoin\domain-join.log` |
| Neustart | Damit die erste Domänenanmeldung möglich ist |

**Erfolgskriterium:** Das Log enthält in dieser Reihenfolge:

```
Domain Join erfolgreich abgeschlossen.
AutoLogon wurde erfolgreich deaktiviert.
Starte System neu fuer Domaenenanmeldung.
```

## Gerät/Benutzer-Zuordnung

Nach der ersten Domänenanmeldung schreibt `write-device-user-mapping.ps1` eine JSON-Datei mit Gerätenamen und
Benutzer. Ein geplanter Task auf dem Domänencontroller verarbeitet sie mit
`update-ad-device-user-mapping.ps1` und setzt `Description` und `ManagedBy` am Computerobjekt.

| Client | Benutzer | Ergebnis |
|---|---|---|
| ORGWS001 | user01 | `Description` / `ManagedBy` gesetzt |
| ORGWS002 | user02 | `Description` / `ManagedBy` gesetzt |
| ORGWS003 | user03 | `Description` / `ManagedBy` gesetzt |

Das Skript erkennt verschiedene Feldnamen im JSON (`UserName`, `AssignedUser`, `User`, `SamAccountName`) und
überspringt leere Benutzerwerte.

## Seriennummer im AD

Der Client schreibt seine BIOS-Seriennummer selbst in das Attribut `serialNumber` des eigenen
Computerobjekts. Dafür wird dem Computerkonto (SELF) genau dieses eine Recht delegiert:

```cmd
dsacls "OU=Clients,DC=corp,DC=example,DC=com" /I:S /G "SELF:RPWP;serialNumber;computer"
```

Ein einmaliger geplanter Task (Ausführung als SYSTEM) startet das Skript und löscht sich nach Erfolg selbst.

## Dienstkonten (Least Privilege)

Kein Schritt der Kette nutzt Domain-Admin-Anmeldedaten.

| Konto | Verwendung | Berechtigung |
|---|---|---|
| `deployread` | Lesende AD-Abfragen (z. B. `Get-ADUser` im Mapping-Task) | Lesen auf Computer- und Benutzerobjekte in `OU=Clients` |
| `deploysvc` | Domänenbeitritt und Mapping-Task | Computerobjekte in `OU=Clients` anlegen; `Description` / `ManagedBy` schreiben |
| `deployldap` | LDAP-Abfragen bei der Namensvergabe | Lesen/Schreiben nur in der Staging-OU |
| SELF (Computerkonto) | Seriennummer | `RPWP` nur auf `serialNumber` des eigenen Objekts |

> **Praxis-Tipp `dsacls`:** Für alle Eigenschaften eines Objekttyps ist das Eigenschaftsfeld leer –
> also zwei Semikolons, z. B. `RPWP;;contact`. `RPWP;contact` wird abgelehnt, weil `contact` ein Objekttyp
> und keine Eigenschaft ist.

## Prüfen auf dem Domänencontroller

```powershell
Get-ADComputer -Filter 'Name -like "ORGWS*"' -Properties Description,ManagedBy,serialNumber |
  Select-Object Name,Description,ManagedBy,serialNumber
```
