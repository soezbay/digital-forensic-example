# DFIR Lehrmaterialien

In diesem Projekt finden Sie die Lehrmaterialien für die Vorlesung Digital Forensik und Incident Response.
* [Intro](DFIR00-Introduction.pdf)
* [Slides](DFIR-slides.pdf)


### Prüfungsformat

* Kombinationsprüfung: 40% Artefaktentwicklung im Laufe des Semesters, 60% mdl. Prüfung am Ende des Semesters, 
  * beide Teile müssen individuell bestanden sein (pro Prüfungsteil >= 50% erreicht haben)
* Prüflinge müssen die Deliverables und Artefakte zu einem selbst konstruierten Fall erzeugen
* MUST HAVE: Auswahl von 2 aus den 3 Bereichen
  * Disk Forensics
  * Network Forensics
  * Memory Forensics
* MUST HAVE: Story und Artefakte müssen im Laufe des Semester erstellt worden sein (Dokumentation per individuellem Gitlab-Repo)
* DELIVERABLE: Markdown-File STORY.md (Deadline **30.04.26**): Story des Angriffs
* DELIVERABLE: Markdown-File REPRODUCE.md (Deadline **30.05.26**): Anleitung zur Reproduktion geeignet
* DELIVERABLE: Markdown-File ANALYSIS.md (Deadline **30.06.26**): eigene Analyse des Angriffs unter Einbeziehung forensischer Werkzeuge und Methoden
    1. Zusammenfassung/Executive Summary
    2. Akquisition/Acquisition
    3. Analyse/Analysis
    4. Ergebnisse/Findings/Results
    5. Artefakte/Artifacts metadata
* ARTEFAKTE: TODO


## Einmalig: Einrichten der Git-Umgebung

### SSH Public Key hinterlegen
Sie müssen einmalig ein SSH-Schlüsselpaar erzeugen und den öffentlichen Schlüssen (public key) im Gitlab hinterlegen. 
Gehen Sie dazu wie folgt vor. Rufen Sie auf der Kommandozeile das Programm `ssh-keygen` auf. Sofern dies Ihr erstes Schlüsselpaar ist, können Sie jegliche Fragen einfach per ENTER (ohne Eingaben) abfrühstücken.
```
# SSH-Schlüsselpaar erzeugen
chris@ubuntu:~$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/chris/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/chris/.ssh/id_rsa.
Your public key has been saved in /home/chris/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:54JFs5zxhMM7cVo8V0yk9vChCVHzdSKolQ+dXkgS8rw chris@cubuntu
chris@cubuntu:~$ 

# Dies erstellt 2 Dateien im Verzeichnis `.ssh` unterhalb Ihres Home Directory
chris@cubuntu:~$ ls -al .ssh
-rw-------  1 chris chris  1675 Apr 30 11:08 id_rsa
-rw-r--r--  1 chris chris   395 Apr 30 11:08 id_rsa.pub
```

Die Datei `id_rsa.pub` ist der öffentliche Schlüssel. Den Inhalt dieser Datei müssen Sie in die Zwischenablage kopieren.

```
chris@cubuntu:~$ cat .ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCr2Xs1grwGkXVnQURYX6UsK31Z5BheqhnftrpiuljXTRnkbciOfe1H7JwDmMfPlrIfbJRBnWQ4TGqs3xDLUspoLcxsaL1L5MmjmTrJt6hRUzYZaLbcu15tixSpQbGbLmEf11bfR7xZm/v1feIsOwOro8orvD1EmmyunF3/PUsnf3LOUNP227ZUdm0NSEJ91cApN+4Ob/RrCTBxVGlcdYme8L9Bew4JzIZPMhaWaJ10RFbT76gaQDv5W0UYSFV8jBgD7FguQAc7JQeST0RISUPbDqoUr+CI4E60U9C12mltZKPa9QVmv/RGp+a/9TybZ4f7qs3GUSdmHwBRJJv79xsN chris@cubuntu
```

Besuchen Sie dann Ihre Profileinstellungen in Gitlab unter https://gitlab.internet-sicherheit.de/-/profile

Dort klicken Sie im Menü links auf den Punkt `SSH Keys`.
![](git-settings-ssh.png)

Fügen Sie nun den öffentlichen Schlüssel ein, sprich den gesamten Inhalt der Datei `.ssh/id_rsa.pub`. Klicken Sie abschließend auf `Add key`.

![](git-addkey.png)



### Auschecken der Repositories
Gehen Sie wie folgt vor, um sich die Git-Umgebung für das dfir-Praktikum einzurichten. 
Sie benötigen 2 Repositories:
* `dfir/tasks`
* `dfir/\<IHR_BENUTZERNAME>`

Erstellen Sie dazu am besten ein übergeordnetes Verzeichnis `dfir`. Wechseln Sie dann in das Verzeichnis `dfir` und klonen Sie die beiden Repositories wie folgt:
```
# Übergeordnetes Verzeichnis erstellen und betreten
mkdir dfir
cd dfir

# Repository für Aufgabenblätter klonen
git clone git@gitlab.internet-sicherheit.de:dfir26/tasks.git

# Repository für Ihre Lösungen (<IHR_BENUTZERNAME> ist Ihr Gitlab-Benutzername, z.B. 'rmueller') klonen
git clone git@gitlab.internet-sicherheit.de:dfir26/<IHR_BENUTZERNAME>.git
```

Die resultierende Verzeichnis- und Dateistruktur sollte wie folgt aussehen:
```
atlas:dfir chris$ tree

dfir
├── tasks                           # hierunter sind alle Aufgabenblätter
│   ├── README.md                   
│   ├── ...                         # Unterlagen 
│   │   └── ...                     # Unterlagen 
│   └── ...
│       └── ...
│
└── <IHR_BENUTZERNAME>              # Ihre Arbeitsergebnisse
    ├── README.md                   
    ├── STORY.md                    # siehe Pruefungsformat
    ├── REPRODUCE.md                # siehe Pruefungsformat
    ├── ANALYSIS.md                 # siehe Pruefungsformat
    └── ARTIFACTS.md                # Wo sind die Artefakte?
```


## Unterlagen aktualisieren

Zum Abfragen einer neuen Aufgabe müssen Sie das Projekt `tasks` aktualisieren (`git pull`).

```
atlas:dfir chris$ cd tasks
atlas:tasks chris$ git pull
```



## Lösung einreichen

Sie sollen unterhalb Ihres Repositories eine bestimmte Verzeichnisstruktur (s.o.) anlegen und mittels Git auf den Server "pushen".

```
atlas:dfir chris$ cd <IHR_BENUTZERNAME>
atlas:<IHR_BENUTZERNAME> chris$ git add STORY.md
atlas:<IHR_BENUTZERNAME> chris$ git commit -m 'Fallbeschreibung erstellt'
atlas:<IHR_BENUTZERNAME> chris$ git push
```
