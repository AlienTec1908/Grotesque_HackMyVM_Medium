# Grotesque - HackMyVM (Medium) - Writeup

![Grotesque Icon](Grotesque.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Grotesque" (Schwierigkeitsgrad: Medium).

## Metadaten

*   **Maschine:** Grotesque (HackMyVM - Medium)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Grotesque](https://hackmyvm.eu/machines/machine.php?vm=Grotesque)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** 16. November 2022
*   **Original Writeup:** [https://alientec1908.github.io/Grotesque_HackMyVM_Medium/](https://alientec1908.github.io/Grotesque_HackMyVM_Medium/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mit `nmap` identifizierte zwei Webserver: Einen WEBrick-Server auf Port 66 und einen Apache-Server auf Port 80 (der nur 404-Fehler lieferte). Die Untersuchung des WEBrick-Servers (Port 66) ergab das Vorhandensein einer Bilddatei `/sshpasswd.png` und eines Verzeichnisses `/lyricsblog/`.

Die Analyse von `/sshpasswd.png` offenbarte versteckten Brainfuck-Code. Die Entschlüsselung dieses Codes (mittels dcode.fr) lieferte den Benutzernamen `erdalkomurcu` und einen MD5-Hash (`BC78...`). Das Verzeichnis `/lyricsblog/` wurde als WordPress-Installation identifiziert.

Der Login in den WordPress-Adminbereich (`/lyricsblog/wp-login.php`) gelang mit dem Benutzernamen `erdalkomurcu` und dem MD5-Hash als Passwort (oder einem daraus geknackten Passwort). Über den WordPress Theme-Editor wurde die `404.php`-Datei des aktiven Themes bearbeitet und eine einfache PHP-Webshell (`system($_GET['cmd']);`) eingeschleust. Durch Aufruf der modifizierten `404.php` mit einer Bash-Reverse-Shell-Payload wurde initialer Zugriff als Benutzer `www-data` erlangt.

Als `www-data` wurde die `wp-config.php`-Datei gelesen, welche die MySQL-Zugangsdaten enthielt: Benutzer `raphael`, Passwort `_double_trouble_`. Da dieses Passwort für den Systembenutzer `raphael` wiederverwendet wurde, gelang der Wechsel mittels `su raphael`.

Im Home-Verzeichnis von `raphael` wurde die User-Flag und eine KeePass-Datenbankdatei (`.chadroot.kdbx`) gefunden. Die KeePass-Datei wurde auf das Angreifer-System übertragen. Mit `keepass2john` und `john` wurde das Master-Passwort (`chatter`) geknackt. Das Öffnen der KeePass-Datenbank offenbarte das Root-Passwort (`.:.subjective.:.`).

Der finale Schritt war der Wechsel zum Root-Benutzer mittels `su root` und dem gefundenen Passwort, gefolgt vom Auslesen der Root-Flag.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   dcode.fr (Webservice)
*   `curl` (impliziert)
*   WordPress Theme Editor
*   `nc` (netcat)
*   `python3` (`pty`, `http.server` modules)
*   `stty`
*   `cat`
*   `su`
*   `ls`
*   `wget`
*   `mv`
*   `keepass2john`
*   `john`
*   KeePass Client (impliziert)
*   `id`
*   `pwd`
*   `cd`
*   `export`

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`), Portscan (`nmap`) -> Port 66 (WEBrick), Port 80 (Apache/404).
2.  **Enumeration Port 66:** `/sshpasswd.png` und `/lyricsblog/` gefunden.
3.  **Information Disclosure:** Brainfuck-Code aus `sshpasswd.png` extrahieren. Mit dcode.fr entschlüsseln -> Username `erdalkomurcu` & MD5-Hash `BC78...`. `/lyricsblog/` als WordPress identifizieren.
4.  **Initial Access (WordPress RCE):** Login bei `/lyricsblog/wp-login.php` mit `erdalkomurcu:BC78...`. Theme Editor -> `404.php` bearbeiten -> PHP Webshell (`system(...)`) einfügen. `nc`-Listener starten. Webshell via URL (`.../404.php?cmd=...bash_reverse_shell...`) triggern -> Shell als `www-data`. Stabilisieren (`python pty`, `stty`).
5.  **Privilege Escalation (`www-data` -> `raphael`):** `cat wp-config.php` -> DB-Credentials `raphael:_double_trouble_` finden. Mit `su raphael` und Passwort wechseln (Passwort-Wiederverwendung).
6.  **Privilege Escalation Vector (`raphael` -> `root`):** Home-Verzeichnis von `raphael` untersuchen -> User-Flag und KeePass-Datei `.chadroot.kdbx` finden.
7.  **KeePass Crack:** `.chadroot.kdbx` übertragen (`python3 -m http.server`, `wget`). Hash extrahieren (`keepass2john`). Master-Passwort knacken (`john`) -> `chatter`.
8.  **Root Password Discovery:** KeePass-Datei mit `chatter` öffnen -> Root-Passwort `.:.subjective.:.` finden.
9.  **Root Access:** Mit `su root` und Passwort `.:.subjective.:.` wechseln.
10. **Flags:** Root-Flag (`cat /root/root.txt`) lesen.

## Flags

*   **User Flag (`/home/raphael/user.txt`):** `F6ACB21652E095630BB1BEBD1E587FE7`
*   **Root Flag (`/root/root.txt`):** `AF7DD472654CBBCF87D3D7F509CB9862`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---
