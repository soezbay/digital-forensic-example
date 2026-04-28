# V1.1 STORY.md – Operation ByteHeist

## 🟠 Zusammenfassung

Ein junger Finanzanalyst lädt auf seinem Firmenlaptop einen trojanisierten Retro-Spieleemulator
herunter. Die eingebettete Malware stiehlt Browser-Zugangsdaten via DPAPI, fängt 2FA-Codes der
Desktop-App WinAuth ab und ermöglicht anschließend laterale Bewegung ins Unternehmensnetzwerk.

Die Angriffskette **imitiert dokumentierte TTPs der Lazarus Group (APT38)** zu Übungszwecken –
es handelt sich um ein **konstruiertes Szenario** (simulierte Attribution, kein realer Incident).
Als Implant wird aus Reproduzierbarkeits-Gründen der Open-Source-Commodity-RAT **AsyncRAT** als
Proxy für einen Lazarus-typischen custom RAT (z.B. BLINDINGCAN, LightlessCan) verwendet.

🔻 **Gewählte forensische Bereiche:** Disk Forensics ✓ · Network Forensics ✓
*(Memory Forensics: optional)*

---

## 🟠 Beteiligte Parteien

| Rolle | Details |
|---|---|
| **Opfer** | Finn Weber, 23 Jahre, Junior Analyst, Meridian Asset Management GmbH (fiktiv), Frankfurt a.M. |
| **Angreifer** | Simuliertes Lazarus-Szenario (APT38); für die Lab-Reproduktion wird AsyncRAT als Commodity-Proxy für einen custom RAT eingesetzt |
| **Betroffenes System** | Firmen-Laptop (Windows 11 Pro), HP EliteBook 840 G10, Hostname `FW-LAPTOP-042`, IP `192.168.10.15` |
| **Betroffenes Netz** | Internes Firmennetz, Segment `192.168.10.0/24` (Client-VLAN) |

---

## 🟠 Malware

🔻 **AsyncRAT** – Open-Source Commodity RAT, eingesetzt als **Proxy-Implant** zur Reproduktion
Lazarus-typischer Capabilities (Keylogging, Credential-Dumping, C2-Beaconing, Remote-Shell).

> **Attribution – Disclaimer:** AsyncRAT ist **nicht** als originäres Lazarus-Tool dokumentiert.
> Lazarus setzt primär eigene Implants ein (BLINDINGCAN, MagicRAT, LightlessCan, ScoringMathTea).
> In diesem konstruierten Fall dient AsyncRAT als frei verfügbarer Ersatz, um die dokumentierten
> TTPs (Fake-Download-Site, signierter Dropper, DPAPI-Credential-Theft, HTTPS-C2, Lateral Movement)
> in einer Lab-Umgebung nachzustellen.

- **Typ:** Remote Access Trojan (RAT)
- **Sprache:** C# (.NET)
- **Samples:** https://bazaar.abuse.ch/browse/tag/AsyncRAT/
- **Referenz-Hash:** SHA256 eines AsyncRAT-Samples aus MalwareBazaar wird in `REPRODUCE.md` fixiert
- **TTP-Vorlage:** Lazarus-Operationen wie *Operation Dream Job* (MITRE C0022) und *TraderTraitor*
  (CISA AA22-108A) nutzen trojanisierte Installer bzw. Fake-Sites als Initial-Access-Vektor – dieses
  Muster wird hier mit AsyncRAT + Fake-Emulator-Site nachgebildet.

---

## 🟠 Angriffsverlauf (Attack Timeline)

### 🔷 Phase 1 – Initial Access & Delivery (T1608.006 – SEO Poisoning)

🔻 **Datum:** Montag, 14. April 2025, ca. 12:30 Uhr (Mittagspause)

Finn sucht auf seinem Firmen-Laptop nach dem SNES-ROM *„Final Fantasy VI"* zum Spielen nach
Feierabend. Er landet über eine SEO-manipulierte Google-Suche auf `retrohaven[.]xyz` – einer
vom Angreifer vorbereiteten Fake-Download-Site mit täuschend echtem Design, die trojanisierte
Emulatoren und ROMs anbietet.

Er lädt `RetroArch_1.19.1_setup.exe` (~75 MB) herunter und führt das Installationsprogramm aus.

### 🔷 Phase 2 – Execution & Dropper (T1204.002)

Der Installer installiert einen funktionierenden RetroArch-Emulator als Decoy und entpackt
gleichzeitig eine verschlüsselte Payload:

- `C:\Users\finn.weber\AppData\Roaming\Microsoft\Windows\SystemCache\svcupdate.exe`
  (AsyncRAT-Payload, 368 KB)

`svcupdate.exe` wird sofort im Hintergrund gestartet. Das Antivirusprogramm (Defender)
schlägt nicht an – die Payload ist mit einem **gestohlenen legitimen Codesign-Zertifikat** signiert
(Lazarus nutzte diese Technik real u.a. bei der 3CX-Supply-Chain-Kompromittierung).

> **Lab-Reproduktion:** In der VM wird statt eines gestohlenen Zertifikats ein **self-signed
> Code-Signing-Cert** verwendet; zusätzlich wird Defender per Exclusion-Path für den Payload-Ordner
> neutralisiert. Genaue Schritte folgen in `REPRODUCE.md`.

### 🔷 Phase 3 – Persistence (T1547.001)

Die Malware schreibt folgenden Registry-Value in den klassischen Run-Key:

- **Schlüssel:** `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`
- **Value-Name:** `WindowsSystemCache`
- **Value-Typ:** `REG_SZ`
- **Value-Data:** `C:\Users\finn.weber\AppData\Roaming\Microsoft\Windows\SystemCache\svcupdate.exe`

Zusätzlich wird ein geplanter Task `\Microsoft\Windows\UpdateCache` erstellt
(T1053.005 – Scheduled Task, Trigger: `AtLogon`, RunLevel: `LeastPrivilege`).

### 🔷 Phase 4 – Credential Theft: Browser (T1555.003)

Die Malware liest die DPAPI-verschlüsselten Zugangsdaten aus Chrome (Version 126):

- `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data` (SQLite)
- `%LOCALAPPDATA%\Google\Chrome\User Data\Local State` (AES-Key via DPAPI)

> **Hinweis zu Chrome 126 im April 2025:** Chrome wird bei Meridian AM zentral per Group Policy
> verwaltet (`HKLM\SOFTWARE\Policies\Google\Update\UpdateDefault=0`), d.h. Auto-Updates sind
> deaktiviert und Versionen werden nur quartalsweise nach IT-Freigabe ausgerollt. Zum Tatzeitpunkt
> ist noch die letzte freigegebene Version Chrome 126 installiert.
>
> **Relevanz:** Ab Chrome 127 (Juli 2024) schützt **Application-Bound Encryption (ABE)** den
> AES-Key im `Local State`; der hier beschriebene Pfad wäre bei aktualisiertem Chrome nicht mehr
> ohne Elevation möglich.

Gestohlene Credentials beinhalten u.a.:
- VPN-Portal: `vpn.meridian-am.de` – `finn.weber@meridian-am.de` / `Sommer2024!`
- Banking-Interface: `trade.meridian-am.de`

### 🔷 Phase 5 – 2FA Interception: WinAuth (T1555)

Finn nutzt **WinAuth** (Desktop TOTP-Authenticator) für den VPN-Zugang und das Trading-Portal.
In WinAuth ist die Option **„Encrypt to only be useable on this computer"** aktiviert, d.h. die
TOTP-Seeds in `winauth.xml` werden zusätzlich zum Master-Passwort via **DPAPI** im Benutzerkontext
verschlüsselt. *(Ohne diese Option wäre der Inhalt nur mit PBKDF2-abgeleitetem Master-Passwort-Hash
geschützt.)*

Die Malware greift auf zwei Wegen auf die TOTP-Secrets zu:

1. **Datei (T1555):** `%APPDATA%\WinAuth\winauth.xml` – DPAPI-verschlüsselte TOTP-Seeds;
   entschlüsselbar im Benutzerkontext von `finn.weber` via `CryptUnprotectData()`
2. **Prozessspeicher (T1555, optional):** Die Malware liest den Heap von `WinAuth.exe` via
   `ReadProcessMemory()` und extrahiert entschlüsselte TOTP-Secrets, sobald WinAuth geöffnet ist

### 🔷 Phase 6 – C2 Communication & Exfiltration (T1071.001, T1041)

Gestohlene Daten werden via HTTPS an den C2-Server exfiltriert:

- **C2-Domain:** `api.cdn-updates[.]com`
- **C2-IP:** `185.234.XX.XX` *(Platzhalter – in `REPRODUCE.md` durch IP der Lab-C2-VM ersetzt)*
- **Protokoll:** HTTPS auf Port 443 (Beacon-Intervall: ~60 Sekunden)
- **Exfiltration:** Credentials + TOTP-Seeds als AES-verschlüsseltes JSON-Blob

### 🔷 Phase 7 – Lateral Movement (T1021.002, T1078)

Mit den gestohlenen VPN-Credentials und den live generierten 2FA-Codes verbindet sich
ein Lazarus-Operator remote in das Firmennetz.

Von Finns Laptop (`192.168.10.15`) aus:
- Netzwerk-Scan via ICMP/ARP: Entdeckung von `FS-MERIDIAN-01` (`192.168.10.5`)
- SMB-Verbindung auf `\\FS-MERIDIAN-01\Finance$` mit gestohlenen Domain-Credentials
- Exfiltration von Finanzdaten (Kundenlisten, Fondsmodelle, M&A-Unterlagen)

---

## 🟠 Forensische Artefakte (Übersicht)

### 🔷 Disk Forensics ✓ (Primär)

🔻 **Dateisystem / Dropped Files**

| Pfad | Beschreibung |
|---|---|
| `C:\Users\finn.weber\Downloads\RetroArch_1.19.1_setup.exe` | Initialer Dropper (trojanisiert) |
| `%APPDATA%\Microsoft\Windows\SystemCache\svcupdate.exe` | AsyncRAT-Payload |
| `C:\Windows\System32\Tasks\Microsoft\Windows\UpdateCache` | Scheduled Task XML (Persistenz) |
| `%APPDATA%\WinAuth\winauth.xml` | TOTP-Seeds (DPAPI-verschlüsselt) |

🔻 **Browser-Artefakte (Chrome 126)**

| Pfad | Beschreibung |
|---|---|
| `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data` | SQLite mit verschlüsselten Passwörtern |
| `%LOCALAPPDATA%\Google\Chrome\User Data\Local State` | DPAPI-AES-Key für Login Data |
| `%LOCALAPPDATA%\Google\Chrome\User Data\Default\History` | Download-History (retrohaven[.]xyz) |
| `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cookies` | Session-Cookies |

🔻 **Registry**

| Schlüssel / Value | Beschreibung |
|---|---|
| `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` → Value `WindowsSystemCache` | Autostart-Eintrag (REG_SZ auf `svcupdate.exe`) |
| `HKLM\SYSTEM\CurrentControlSet\Services\Schedule\TaskCache\Tasks\{GUID}` | Scheduled-Task-Cache (Persistenz-Replik) |
| `NTUSER.DAT` → `SOFTWARE\WinAuth` (falls vorhanden) | WinAuth-Konfiguration / Backup der Authenticator-Einträge |

🔻 **Windows-Artefakte**

| Pfad | Beschreibung |
|---|---|
| `C:\Windows\Prefetch\RETROARCH_1.19.1_SETUP.EXE-*.pf` | Prefetch: Ausführungsnachweis mit Timestamp |
| `C:\Windows\Prefetch\SVCUPDATE.EXE-*.pf` | Prefetch: Payload-Ausführung |
| `C:\Windows\System32\winevt\Logs\Security.evtx` | Login-Events (4624/4625), Prozess-Audit (4688) |
| `C:\Windows\System32\winevt\Logs\System.evtx` | Scheduled Task-Erstellung |
| `C:\Users\finn.weber\AppData\Roaming\Microsoft\Windows\Recent\*.lnk` | LNK-Dateien (zuletzt geöffnet) |
| `C:\Users\finn.weber\NTUSER.DAT` | Registry-Hive: komplette Nutzer-Aktivität |
| `C:\$MFT` | Master File Table: Zeitstempel aller Dateioperationen |

---

### 🔷 Network Forensics ✓ (Primär)

🔻 **DNS-Queries (PCAP / DNS-Log)**

| Domain | Beschreibung |
|---|---|
| `retrohaven[.]xyz` | Aufruf der Fake-Download-Site |
| `api.cdn-updates[.]com` | C2-Kommunikation (Beacon) |

🔻 **HTTP/HTTPS-Traffic**

| Indikator | Beschreibung |
|---|---|
| `GET https://retrohaven[.]xyz/downloads/RetroArch_1.19.1_setup.exe` | Download des Droppers |
| TLS SNI: `api.cdn-updates[.]com` auf Port 443 | C2-Beacon (Intervall ~60 s, regelmäßiges Muster) |
| JA3 / JA4 TLS-Client-Hello-Fingerprint | Stabiler Hash des AsyncRAT-Clients – IoC bei wiederholten Beacons |
| HTTP POST-Requests mit verschlüsseltem JSON-Body | Datenexfiltration (Credentials, TOTP-Seeds) |

🔻 **Internes Netzwerk (Lateral Movement)**

| Indikator | Beschreibung |
|---|---|
| ICMP-Sweep `192.168.10.0/24` | Host-Discovery vom Opfer-Laptop |
| SMB2-Verbindung `192.168.10.15 → 192.168.10.5:445` | Zugriff auf `\\FS-MERIDIAN-01\Finance$` |
| VPN-Authentifizierung mit gültigen Credentials | Remote-Zugang des Angreifers ins Netz |

🔻 **Capture-Datei**

| Datei | Beschreibung |
|---|---|
| `capture.pcapng` | Vollständiger Netzwerk-Mitschnitt (Wireshark/tcpdump in VM) |

---

### 🔷 Memory Forensics *(Optional / Bonus)*

Artefakte sind mit Volatility 3 analysierbar.

| Artefakt | Beschreibung |
|---|---|
| Eigenständiger Prozess `svcupdate.exe` | AsyncRAT als eigener .NET-Prozess (`pstree`, `cmdline`) |
| Aktive TCP-Sockets zu Port 443 | C2-Verbindung (`netstat` via Volatility) |
| Entschlüsselte Credentials im Prozess-Heap | Sichtbar nach DPAPI-Entschlüsselung im RAM |

---

## 🟠 MITRE ATT&CK Mapping

| Taktik | Technik | ID |
|---|---|---|
| Resource Development | Stage Capabilities: SEO Poisoning | T1608.006 |
| Execution | User Execution: Malicious File | T1204.002 |
| Persistence | Registry Run Keys / Startup Folder | T1547.001 |
| Persistence | Scheduled Task | T1053.005 |
| Credential Access | Credentials from Web Browsers | T1555.003 |
| Credential Access | Credentials from Password Stores (WinAuth: `winauth.xml`) | T1555 |
| Credential Access | Credentials from Password Stores (WinAuth: Process Memory) *(optional)* | T1555 |
| Command & Control | Application Layer Protocol: Web Protocols | T1071.001 |
| Exfiltration | Exfiltration Over C2 Channel | T1041 |
| Lateral Movement | SMB / Windows Admin Shares | T1021.002 |
| Lateral Movement | Valid Accounts | T1078 |

---

## 🟠 Referenzen

🔻 **Threat Intelligence (Lazarus / APT38 – Kontext für TTP-Vorlage, keine reale Attribution)**

- MITRE ATT&CK – Lazarus Group (G0032):
  https://attack.mitre.org/groups/G0032/
- MITRE ATT&CK – Operation Dream Job (C0022):
  https://attack.mitre.org/campaigns/C0022/
- CISA AA20-239A – *„FASTCash 2.0: North Korea's BeagleBoyz Robbing Banks"*:
  https://www.cisa.gov/news-events/cybersecurity-advisories/aa20-239a
- CISA AA22-108A – *„TraderTraitor: North Korean State-Sponsored APT Targets Blockchain Companies"*:
  https://www.cisa.gov/news-events/cybersecurity-advisories/aa22-108a
- JPCERT/CC – *BLINDINGCAN – Malware Used by Lazarus*:
  https://blogs.jpcert.or.jp/en/2020/09/BLINDINGCAN.html

🔻 **Malware (Commodity-RAT für Reproduktion)**

- MalwareBazaar – AsyncRAT Samples:
  https://bazaar.abuse.ch/browse/tag/AsyncRAT/

🔻 **Technische Referenzen**

- Red Canary – *Stealers evolve to bypass Google Chrome's new app-bound encryption* (Chrome 127 ABE):
  https://redcanary.com/blog/threat-intelligence/google-chrome-app-bound-encryption/
- The Hacker News – *Google Chrome Adds App-Bound Encryption*:
  https://thehackernews.com/2024/08/google-chrome-adds-app-bound-encryption.html
- WinAuth (Desktop TOTP Authenticator):
  https://winauth.github.io/winauth/
- RetroArch (legitime Version):
  https://www.retroarch.com/