# Crossroads - HackMyVM (Easy)

![Crossroads.png]

## Übersicht

*   **VM:** Crossroads
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Crossroads)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 10. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Crossroads_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, User- und Root-Zugriff auf der "Crossroads"-Maschine zu erlangen. Der Lösungsweg begann mit der Entdeckung eines Webservers und eines SMB-Dienstes. Über anonyme SMB-Enumeration wurde der Benutzer `albert` identifiziert. Dessen SMB-Passwort konnte mittels Brute-Force erraten werden. Dies ermöglichte den Zugriff auf `albert`s Home-Share, wo eine verwundbare `smb.conf` und ein SUID-Binary (`beroot`) gefunden wurden. Eine `preexec`-Anweisung in der `smb.conf` wurde ausgenutzt, um durch Hochladen eines bösartigen Skripts eine Reverse Shell als Benutzer `albert` zu erhalten. Die anschließende Privilegieneskalation zu Root erfolgte durch Ausführen des SUID-Binaries `beroot`, das nach Eingabe eines spezifischen Passworts die Root-Zugangsdaten preisgab.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `curl`
*   `wget`
*   `enum4linux`
*   `smbmap`
*   `medusa`
*   `smbclient`
*   `vi` (oder anderer Texteditor)
*   `nc` (netcat)
*   `python` (zur Shell-Stabilisierung)
*   `file`
*   `stegoveritas` (versucht, aber nicht erfolgreich/notwendig)
*   Standard Linux-Befehle (`ls`, `cat`, `cd`, `id`, `export`, `su`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Crossroads" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.128`).
    *   Umfassender `nmap`-Scan identifizierte offene Ports:
        *   Port 80 (HTTP): Apache httpd 2.4.38, Webseite "12 Step Treatment Center | Crossroads Centre Antigua", `robots.txt` verweist auf `/crossroads.png`.
        *   Port 139/445 (SMB): Samba 4.9.5-Debian, anonymer Gastzugriff erlaubt, Message Signing deaktiviert, Hostname `CROSSROADS`.

2.  **Web & SMB Enumeration (Information Gathering):**
    *   `gobuster` auf dem Webserver fand die Datei `/note.txt` mit dem Hinweis "three kings of blues" und "crossroads".
    *   `enum4linux` wurde gegen den SMB-Dienst ausgeführt und identifizierte via RID-Cycling den Benutzer `albert`.
    *   `medusa` wurde verwendet, um das SMB-Passwort für den Benutzer `albert` zu knacken (`bradley1`).
    *   Mittels `smbclient` und den Credentials `albert:bradley1` wurde auf die SMB-Share `/albert` zugegriffen. Dort wurden die `user.txt`, das SUID-Binary `beroot` und ein Verzeichnis `smbshare` mit der Datei `smb.conf` gefunden.

3.  **Initial Access (albert via SMB preexec RCE):**
    *   Die heruntergeladene `smb.conf` zeigte für die `[smbshare]` eine `preexec`-Direktive, die das Skript `/home/albert/smbshare/smbscript.sh` bei Verbindungsaufbau ausführt.
    *   Ein Bash-Skript (`smbscript.sh`) mit einer Reverse-Shell-Payload wurde erstellt.
    *   Ein `nc`-Listener wurde auf der Angreifer-Maschine gestartet.
    *   Das `smbscript.sh` wurde via `smbclient` in das Verzeichnis `/home/albert/smbshare` auf dem Ziel hochgeladen.
    *   Das erneute Herstellen einer Verbindung (oder die automatische Ausführung durch den SMB-Dienst) zur `smbshare` löste das `preexec`-Skript aus und etablierte eine Reverse Shell als Benutzer `albert`.

4.  **Post-Exploitation / Privilege Escalation (von albert zur Vorbereitung für Root):**
    *   In der Shell als `albert` wurde das zuvor identifizierte SUID-Binary `beroot` im Home-Verzeichnis `/home/albert/` näher untersucht.
    *   Das Ausführen von `./beroot` startete ein Programm, das nach einem Passwort fragte.
    *   Durch Eingabe des Passworts `lemuel` (möglicherweise abgeleitet aus dem "three kings"-Hinweis) gab das Programm die Anweisung, `ls` auszuführen und eine Datei namens `rootcreds` zu suchen.
    *   Die Datei `rootcreds` wurde im Verzeichnis `/home/albert/` gefunden.

5.  **Privilege Escalation (von `albert` (mit Kenntnis der Root-Credentials) zu root):**
    *   Der Inhalt der Datei `rootcreds` wurde mit `cat rootcreds` ausgelesen und enthielt die Root-Zugangsdaten: `root:___drifting___`.
    *   Mit dem Befehl `su root` und dem Passwort `___drifting___` wurde erfolgreich Root-Zugriff auf der Maschine erlangt.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer SMB-Zugriff und RID-Cycling:** Erlaubte die Enumeration des Benutzernamens `albert` ohne Authentifizierung.
*   **Schwache SMB-Passwörter / Brute-Force:** Das Passwort des Benutzers `albert` (`bradley1`) konnte durch einen Wörterbuchangriff auf den SMB-Dienst erraten werden.
*   **Unsichere SMB `preexec`-Konfiguration (RCE):** Die `preexec`-Direktive in der `smb.conf` verwies auf ein Skript in einem für den Benutzer `albert` schreibbaren Verzeichnis. Dies ermöglichte das Hochladen und Ausführen einer Reverse Shell und somit Remote Code Execution.
*   **SUID-Binary mit unsicherer Passwortprüfung / Hardcoded Credentials Leak:** Das SUID-Binary `beroot` verwendete ein festes (oder leicht ableitbares) Passwort (`lemuel`), um anschließend die Root-Zugangsdaten in einer Datei (`rootcreds`) preiszugeben.
*   **Information Disclosure (Web & SMB):** Hinweise in `/note.txt` und die Exposition der `smb.conf` lieferten wichtige Informationen für den Angriff.

## Flags

*   **User Flag (`/home/albert/user.txt`):** `912D12370BBCEA67BF28B03BCB9AA13F`
*   **Root Flag (`/root/root.txt`):** `876F96716C3606B09A89F0FA3C1D52EB`

## Tags

`HackMyVM`, `Crossroads`, `Easy`, `SMB`, `preexec`, `RCE`, `SUID`, `Brute-Force`, `Information Disclosure`, `Linux`, `Web`, `Privilege Escalation`
