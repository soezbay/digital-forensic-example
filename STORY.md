# STORY.md – Operation ByteHeist

## Zusammenfassung

Ein junger Finanzanalyst lädt auf seinem Firmenlaptop einen trojanisierten Retro-Spieleemulator
herunter. Die eingebettete Malware stiehlt Browser-Zugangsdaten via DPAPI, fängt 2FA-Codes der
Desktop-App WinAuth ab und ermöglicht anschließend laterale Bewegung ins Unternehmensnetzwerk.
Die Attacke wird der nordkoreanischen Lazarus Group (APT38) zugeordnet.

---

## Beteiligte Parteien

| Rolle | Details |
|---|---|
| **Opfer** | Finn Weber, 23 Jahre, Junior Analyst, Meridian Asset Management GmbH (fiktiv), Frankfurt a.M. |
| **Angreifer** | Lazarus Group / APT38, nordkoreanische staatlich gesponserte APT |
| **Betroffenes System** | Firmen-Laptop (Windows 11 Pro), HP EliteBook 840 G10 |
| **Betroffenes Netz** | Internes Firmennetz, Segment `192.168.10.0/24` |

---

## Malware

**AsyncRAT** – Commodity Remote Access Trojan, von Lazarus Group in dokumentierten Kampagnen eingesetzt

- **Typ:** Remote Access Trojan (RAT)
- **Sprache:** C# (.NET)
- **MalwareBazaar:** https://bazaar.abuse.ch/browse/tag/AsyncRAT/
- **Referenz-Hash (Beispiel):** SHA256 auf MalwareBazaar unter Tag `AsyncRAT` verfügbar
- **Lazarus-Kontext:** CISA Advisory AA20-239A (BLINDINGCAN/AIRDRY) dokumentiert vergleichbare
  Lazarus-TTPs; Lazarus setzt nachweislich auch Commodity-RATs wie AsyncRAT ein

---

## Angriffsverlauf (Attack Timeline)

### Phase 1 – Initial Access (T1204.002)

**Datum:** Montag, 14. April 2025, ca. 12:30 Uhr (Mittagspause)

Finn sucht auf seinem Firmen-Laptop nach dem SNES-ROM *„Final Fantasy VI"* zum Spielen nach
Feierabend. Er landet über eine Google-Suche auf `retrohaven[.]xyz` – einer von der Lazarus
Group betriebenen Fake-Download-Site mit täuschend echtem Design, die trojanisierte Emulatoren
und ROMs anbietet.

Er lädt `ZSNES_2.0_setup.exe` (~4,2 MB) herunter und führt das Installationsprogramm aus.

### Phase 2 – Execution & Dropper (T1204.002, T1059.001)

Der Installer installiert einen funktionierenden ZSNES-Emulator als Decoy und entpackt
gleichzeitig eine verschlüsselte Payload:

- `C:\Users\finn.weber\AppData\Roaming\Microsoft\Windows\SystemCache\svcupdate.exe`
  (AsyncRAT-Payload, 368 KB)

`svcupdate.exe` wird sofort im Hintergrund gestartet. Das Antivirusprogramm (Defender)
schlägt nicht an – die Payload ist mit einem legitimen gestohlenen Codesign-Zertifikat signiert.

### Phase 3 – Persistence (T1547.001)

Die Malware schreibt folgenden Registry-Eintrag:

`HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run Name: "WindowsSystemCache"`

`Value: "C:\Users\finn.weber\AppData\Roaming\Microsoft\Windows\SystemCache\svcupdate.exe"`


Zusätzlich wird ein geplanter Task `\Microsoft\Windows\UpdateCache` erstellt
(T1053.005 – Scheduled Task).

### Phase 4 – Credential Theft: Browser (T1555.003)

Die Malware liest die DPAPI-verschlüsselten Zugangsdaten aus Chrome:

- `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data` (SQLite)
- `%LOCALAPPDATA%\Google\Chrome\User Data\Local State` (AES-Key via DPAPI)

Gestohlene Credentials beinhalten u.a.:
- VPN-Portal: `vpn.meridian-am.de` – `finn.weber@meridian-am.de` / `Sommer2024!`
- Banking-Interface: `trade.meridian-am.de`

### Phase 5 – 2FA Interception: WinAuth (T1056, T1552)

Finn nutzt **WinAuth** (Desktop TOTP-Authenticator) für den VPN-Zugang und das Trading-Portal.

Die Malware greift auf zwei Wegen auf die TOTP-Secrets zu:

1. **Datei:** `%APPDATA%\WinAuth\winauth.xml` – enthält AES-verschlüsselte TOTP-Seeds
   (Schlüssel wird aus Windows-DPAPI abgeleitet → mit SYSTEM-Kontext entschlüsselbar)
2. **Prozessspeicher:** Die Malware führt Process Memory Scraping auf `WinAuth.exe` durch
   und extrahiert die TOTP-Secrets im Klartext, sobald WinAuth aktiv ist

### Phase 6 – C2 Communication & Exfiltration (T1071.001, T1041)

Gestohlene Daten werden via HTTPS an den C2-Server exfiltriert:

- **C2-Domain:** `api.cdn-updates[.]com`
- **C2-IP:** `185.234.XX.XX` (registriert über Bulletproof-Hoster)
- **Protokoll:** HTTPS auf Port 443 (Beacon-Intervall: ~60 Sekunden)
- **Exfiltration:** Credentials + TOTP-Seeds als AES-verschlüsseltes JSON-Blob

### Phase 7 – Lateral Movement (T1021.002, T1078)

Mit den gestohlenen VPN-Credentials und den live generierten 2FA-Codes verbindet sich
ein Lazarus-Operator remote in das Firmennetz.

Von Finns Laptop aus:
- Netzwerk-Scan via ICMP/ARP: Entdeckung von `FS-MERIDIAN-01` (`192.168.10.5`)
- SMB-Verbindung auf `\\FS-MERIDIAN-01\Finance$` mit gestohlenen Domain-Credentials
- Exfiltration von Finanzdaten (Kundenlisten, Fondsmodelle, M&A-Unterlagen)

---

## Forensische Artefakte (Übersicht)

### Disk Forensics

| Pfad | Beschreibung |
|---|---|
| `C:\Users\finn.weber\Downloads\ZSNES_2.0_setup.exe` | Initialer Dropper |
| `%APPDATA%\Microsoft\Windows\SystemCache\svcupdate.exe` | AsyncRAT-Payload |
| `HKCU\...\CurrentVersion\Run\WindowsSystemCache` | Persistenz-Eintrag |
| `%LOCALAPPDATA%\Google\Chrome\...\Login Data` | Gestohlene Browser-Credentials |
| `%APPDATA%\WinAuth\winauth.xml` | TOTP-Seeds (verschlüsselt) |
| `C:\Windows\Prefetch\ZSNES_2.0_SETUP.EXE-*.pf` | Prefetch-Eintrag |
| [C:\Windows\System32\winevt\Logs\Security.evtx](cci:7://file:///Windows/System32/winevt/Logs/Security.evtx:0:0-0:0) | Login-Events |

### Network Forensics

| Indikator | Typ |
|---|---|
| `retrohaven[.]xyz` | Malicious Download-Site |
| `api.cdn-updates[.]com` | C2-Server |
| `185.234.XX.XX:443` | C2-IP |
| SMB-Traffic zu `192.168.10.5` | Lateral Movement |

### Memory Forensics

| Artefakt | Beschreibung |
|---|---|
| Prozessinjektion in `explorer.exe` | AsyncRAT-Hollow Process |
| TOTP-Secrets in `WinAuth.exe` Heap | Klartext-Seeds |
| Entschlüsselte Credentials im RAM | DPAPI-Ergebnis |
| Netzwerk-Sockets von `svcupdate.exe` | Aktive C2-Verbindung |

---

## MITRE ATT&CK Mapping

| Taktik | Technik | ID |
|---|---|---|
| Initial Access | User Execution: Malicious File | T1204.002 |
| Execution | User Execution | T1204 |
| Persistence | Registry Run Keys/Startup Folder | T1547.001 |
| Persistence | Scheduled Task | T1053.005 |
| Credential Access | Credentials from Web Browsers | T1555.003 |
| Credential Access | Process Memory Scraping | T1056 |
| Credential Access | Unsecured Credentials in Files | T1552.001 |
| Command & Control | Application Layer Protocol: HTTPS | T1071.001 |
| Exfiltration | Exfiltration Over C2 Channel | T1041 |
| Lateral Movement | SMB/Windows Admin Shares | T1021.002 |
| Lateral Movement | Valid Accounts | T1078 |

---

## Referenzen

- CISA Advisory AA20-239A: BLINDINGCAN – Lazarus Group RAT
  https://www.cisa.gov/uscert/ncas/alerts/aa20-239a
- MalwareBazaar – AsyncRAT Samples:
  https://bazaar.abuse.ch/browse/tag/AsyncRAT/
- MITRE ATT&CK – Lazarus Group (G0032):
  https://attack.mitre.org/groups/G0032/
- WinAuth (Desktop TOTP Authenticator):
  https://winauth.github.io/winauth/