# kernel-modules
Kernel-Module (Treiber) für Linux erstellen
## Punkt 2
Die IDs lassen sich im Terminal mit diesen drei Befehlszeilen auslesen:
```
sudo lshw -numeric -html > lshw.html
sudo lspci -nn > lspci.txt
sudo lsusb -v > lsusb.txt
```
Im Gerät steckt also ein Realtek-Chip, was auch am ersten Teil der ID „0bda“ erkennbar ist, die zu Realtek gehört (siehe: https://usb-ids.gowdy.us/read/UD). Der zweite Teil der ID kennzeichnet das Gerät.

Meist genügt eine Google-Suche nach der ID, die dann beispielsweise zu https://linux-hardware.org/index.php?id=usb:0bda-b812 führt. 

Tipp: Auf https://github.com/morrownr/USB-WiFi finden Sie eine Übersicht mit empfehlenswerten USB-WLAN-Adaptern, die Linux standardmäßig unterstützt. https://wiki.ubuntuusers.de/WLAN/Karten bietet Informationen zur Inbetriebnahme von WLAN-Karten und WLAN-USB-Sticks.

## Punkt3

Welchen Kernel das System aktuell nutzt finden Sie im Terminal mit
```
uname -a
```
heraus. 
Wer einen noch neueren Kernel benötigt, muss ihn selbst erstellen. Eine Anleitung für Ubuntu und Linux Mint finden Sie über https://m6u.de/BMUK.

## Punkt 4

Auf https://linux-hardware.org/index.php?id=usb:0bda-b812 (siehe Punkt 2) sind mehrere Links zu den Treiber-Quellen für den Chipsatz RTL88x2bu zu finden. Wir haben den Quelltext von https://github.com/morrownr/88x2bu-20210702 heruntergeladen und verwenden ihn als Beispiel.Auf https://linux-hardware.org/index.php?id=usb:0bda-b812 (siehe Punkt 2) sind mehrere Links zu den Treiber-Quellen für den Chipsatz RTL88x2bu zu finden. Wir haben den Quelltext von https://github.com/morrownr/88x2bu-20210702 heruntergeladen und verwenden ihn als Beispiel.

**Schritt 1:** Zur Vorbereitung installieren Sie im Terminal einige zusätzliche Pakete
```
sudo apt install git build-essential dkms linux-headers-$(uname -r) 
```
**Schritt 2:** Erstellen Sie ein Arbeitsverzeichnis und laden Sie den Quelltext des Treibers herunter (vier Zeilen):
```
mkdir -p ~/src
cd ~/src
git clone https://github.com/morrownr/88x2bu-20210702.git
cd ~/src/88x2bu-20210702
```
**Schritt 3:** Es empfiehlt sich, den Treiber zuerst manuell zu erstellen und seine Funktion auszuprobieren. Dazu sind diese drei Befehlszeilen nötig:
```
make clean
make
sudo insmod 88x2bu.ko
```
Damit wird der Treiber direkt aus dem aktuellen Ordner „~/src/88x2bu-20210702“ geladen und Sie können sich mit
```
dmesg
```
die letzten Kernel-Meldungen ausgeben lassen, die über den erfolgreich geladenen Treiber informieren oder Fehlermeldungen anzeigen. Mit der Zeile
```
sudo rmmod 88x2bu
```
lässt sich der Treiber wieder entladen.

Bei den meisten Treiber ist abschließen die Zeile
```
sudo make install
```
erforderlich, womit der Treiber in einen Ordner unterhalb von „/lib/modules/[Kernel-Version]“ kopiert und beim nächsten Linux-Start automatisch geladen wird. Bei diesem Treiber ist das nicht erforderlich, weil ein Script die Aufgabe übernimmt (siehe Schritt 4).

Wenn Secure Boot aktiviert ist, erzeugt insmod eine Fehlermeldung und der Treiber wird nicht geladen, weil er nicht digital signiert ist (siehe Punkt 5). Andernfalls funktioniert die WLAN-Hardware jetzt und Verbindungen lassen sich wie gewohnt über den Netzwerk-Manager herstellen.

**Schritt 4:** Standardmäßig wird dieser Treiber besonders komfortabel per Script eingerichtet:
```
sudo ./install-driver.sh
```
Sie werden gefragt, ob Sie die Konfigurationsdatei „/etc/modprobe.d/88x2bu.conf“ mit den Treiber-Optionen bearbeiten möchten. In der Regel ist das nicht nötig, Erklärungen zu den Optionen sind in der Konfigurationsdatei enthalten. Wenn Sie Linux jetzt neu starten, wird der neue Treiber automatisch geladen und die WLAN-Hardware lässt sich nutzen.

## Punkt 5
Ob ein Modul signiert ist, finden Sie mit
```
modinfo [Modulname]
```
heraus. 

Wenn sich ein Modul nicht über DKMS installieren lässt, kann man die Signatur auch manuell vornehmen. Unter Ubuntu 20.04 oder Linux Mint 20 verwenden Sie dazu diese beiden Befehlszeilen:
```
export KERNEL_BUILD=/lib/modules/$(uname -r)/build
sudo -E $KERNEL_BUILD/scripts/sign-file sha256 /var/lib/shim-signed/mok/MOK.priv /var/lib/shim-signed/mok/MOK.der [Modul].ko
```
Die Datei „[Modul].ko“ muss in dem Verzeichnis liegen, in dem Sie die Zeilen ausführen.

## Treiber für DKMS anpassen
Auf dieser Seite finden Sie als Beispiel den Ordner „sample-module-1.1.1“, den Sie nach „/usr/src“ kopieren. Danach führt man die drei Zeilen aus:
```
sudo dkms add sample-module/1.1.1
sudo dkms build sample-module/1.1.1
sudo dkms install sample-module/1.1.1
```
Der Demo-Treiber verwendet ein Gerät unter „/dev“, dessen Rückmeldung „Hello world from kernel mode!“ sich mit
```
cat /dev/sample-driver
```
ausgeben lässt. Über die Befehlszeile
```
sudo dkms remove sample-module/1.1.1 --all
```
lässt sich der Treiber wieder entfernen.

„Makefile“ und „dkms.conf“ des Demo-Treibers enthält einige Komfortfunktionen.

Das Makefile erstellt nach Aufruf von "make" alleine nur das Modul. Mit "make sign" lässt es sich digigal signieren, wenn Secure Boot aktiviert ist.

Die Abfrage 
```
SIG_MOK	:=	$(shell mokutil --sb-state 2>/dev/null || echo "(unknown)")
```
ermittelt den Secure-Boot-Status.

"make modules_install" installiert das Modul und signiert es, wenn Secure Boot aktiviert ist.

"dkms.conf" erstellt das Modul mit 
```
MAKE[0]="'make' all TARGET=${kernelver}"
```
für den aktuellen Kernel. Das dient nur der Demonstration, weil "Makefile" die Version des laufenden Kernels ohnehin ermittelt, wenn "TARGET" nicht angegeben ist.

Die Option 
```
AUTOINSTALL="yes"
```
sorgt dafür, das dass Modul automatisch für neuere Kernel erstellt wird.






