# Installationsguide
Die folgenden Schritte zeigen wie man das Program aufsetze und die generale Umgebung auf dem Raspberry pi schafft um ihn als Gehin geeignet für den Weltzerstörungsknopf zu machen.

## Vor der installation
Vor der Installation versichere dich bitte, dass:
* auf dem Pi eine version von [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/) installiert ist. Es wird keine spezifische version vorausgesetzt und alles kann (und wird meist auch) ohne Desktop-Environment, als nur über eine Konsole installiert. Somit ist es perfekt möglich Raspberry Pi OS Lite zu nutzen sowie jedes andere auch.
* du physischen Zugriff auf den pi hast und nicht nur eine SSH-Verbindung, da später der Pi in einen Wifi-Access-Point verwandelt wird. Daher könnte bei einer möglichen Misskonfiguration der Netzwerkzugriff auf den Pi eingeschränkt werden.

*Kurze Notiz: Alle CLI-Befehle unterhalb werden über `sudo` ausgeführt, da sie alle Root-Berechtigungen brauchen.*

## Vorbereitungen für den Wifi-Access-Point
Die folgenden Schritte installieren die Software-Packete, welche später noch gebraucht werden, wenn der RPi kein Internet-Zugriff mehr hat.
Also versichere dich bitte, dass dert RPi jetzt erstmal eine Internet-Verbindung hat, entwerder über dein Desktop-Environment oder durch das hinzufügen von folgenden Zeilen in `sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`:

```bash
network={
        ssid="<WIFI-Name>" 
		psk="<PASS>"
        scan_ssid=1
}
```

Nun bitte einmal den Pi neustarten über: `sudo reboot 0`.

### Updates, updates, upgrades
Nun, da wir über ein Internet-Verbindung verfügen, sollten wir uns versicheren, dass der RPi über die neuste Software für alles verfügt, als führe bitte
```bash
sudo apt update && sudo apt upgrade -y
```
aus.

### Autoboot aktivieren
Zuerst muss nun der pi so vorbereitet werden, dass der mit der Welti-Hardware booten kann.
Dazu muss in der Datei `sudo nano /boot/config.txt` in die Zeile unter `# Additional overlays and parameters are documented /boot/overlays/README` folgendes hinzugefügt werden: `gpio=14=op,dl`, so dass es so aussieht:

```bash
# Additional overlays and parameters are documented /boot/overlays/README
gpio=14=op,dl
```

### Installtion der Software
Folgende Software wird gebraucht, bitte über die angegebenen Befehle installieren:

|Name|Befehl|Info|
|-|-|-|
|[Git](https://git-scm.com/)|`sudo apt install git`|Wird zum download der Welti-Software gebraucht|
|[Python3](https://www.python.org/)|`sudo apt-get install python3.9`|Wird zum ausführen der Welti-Software gebraucht|
|[Pip3](https://pypi.org/project/pip/)|`sudo python -m ensurepip --upgrade`|Genutzt zum installieren der Welti-Software-Packete|
|Hostapd|`sudo apt -y install hostapd`|Software zum Konfigurieren des RPi als Wifi-Access-Point|
|Dnsmasq|`sudo apt -y install dnsmasq`|Software um den RPi in einen DNS/DHCP Server zu verwandeln|

Außerdem müssen die letzten beiden Software-Packete (`hostapd` / `dnsmasq`) fürs erste deaktiviert werden.

```bash
sudo service hostapd stop
sudo service dnsmasq stop
```

### Die Welti-Software installieren
Nun können wir uns endlich darum kümmen die tatsächliche Welti-Software zu installieren.
Dazu bitte das Repository clonen:

```bash
git clone https://github.com/artandtechspace/Weltzerstoerungsknopf-Software.git
mv Weltzerstoerungsknopf-Software Welti
```

und versicheren, dass auch das Configurations-Webinterface mit geladen wird durch das initzialisieren der Git-Submodule:

```bash
cd Welti
git submodule update --init --recursive
```

Jetzt bitte selber über die CLI oder den Datei-Explorer versicheren, dass folgende Dateien existieren und auch an ihrem richtigen Platz liegen:
* `main.py`: `/home/pi/Welti/main.py`
* `index.html`: `/home/pi/Welti/rsc/webpage/Weltzerstoerungsknopf-Configinterface/index.html`

Vielleicht müsst du auch die Datei `sudo nano /home/pi/Welti/webserver/WebProgram.py` editieren um ganz unten den port von `port=5000` zu `port=80` zu ändern, wenn nicht bereits geschien.

### Installieren der Python-Packete
Zum installieren aller gebrauchten Librarys auf dem pi, bitte folgende Befehle ausführen:

```bash
# Neopixel-support from adafruit
sudo pip3 install rpi_ws281x adafruit-circuitpython-neopixel
sudo python3 -m pip install --force-reinstall adafruit-blinka

# GPIO-Support
sudo apt-get -y install python3-rpi.gpio

# Other librarys
sudo pip install -r requirements-on-pi.txt
```

## Erstellen der .service-datei
Um die Welti-Software direkt als Startup-Program ausführen zu lassen und general als Hintergrund-Service, wird über system-d eine service-Datei erstellt.
Fülle die Datei `sudo nano /etc/systemd/system/Welti.service` mit:

```ini
[Unit]
Description=Weltzerstörungsknopf-Service
After=network.target

[Service]
ExecStart=/usr/bin/sudo /usr/bin/python -m main
WorkingDirectory=/home/pi/Welti

Restart=on-failure
RestartSec=1s

[Install]
WantedBy=multi-user.target
RequiredBy=network.target
```

Nun kann der service gestartet werden:
```bash
sudo service Welti start
sudo service Welti status
```

und als Boot-Program eingetragen werden:
```bash
sudo systemctl enable Welti.service
```

## Den RPi als Wifi-Access-Point registieren
Von jetzt an wird nicht länger eine Internet-Verbindung gebraucht, da wir den RPi nun als eigenen Wifi-Access-Point konfigurieren für das Web-Config-Interface.

Die folgenden Konfigurationen konfigurieren eine statische Ip (Router) für den RPi und dann nach werden wir ihn als Wifi-AP, DNS-Server und DHCP-Server konfigurieren.

Editiere die dhcpdc-Datei: `sudo nano /etc/dhcpcd.conf`
und lasse allen Traffik durch den AP routen indem die folgende Linie ganz unten an die Datei hinzugefügt wird:

```bash
denyinterfaces wlan0
```

Nun konfiguriere den Pi als Statische Ip: `sudo nano /etc/network/interfaces`

Füge hier folgendes hinzu:

```bash
auto log
iface lo inet loopback

allow-hotplug wlan0
iface wlan0 inet static
   address 10.45.0.1
   netmask 255.255.255.0
   network 10.45.0.0
   broadcast 10.45.0.255
```

und passe die Ip-Adressen entsprechen an.
Da der Pi allerdings ein eigenes Wifi öffnet, sollten die Vorgegebenen passend sein.
Für den Fall, dass du trotzdem eine änderung an den Ip-ADressen oder Adressräumen vornimmt, dokumentiere diese bitte.

### Konfigurieren von Hostapd

Hostapd ist die Software, welche wir nutzen um den RPi als Wifi-Access-Point zu konfigurieren.
Starte damit die Konfigurationsdatei zu erstellen: `sudo nano /etc/hostapd/hostapd.conf` und füge folgendes hinzu:

```ini
interface=wlan0
driver=nl80211
ssid={SSID}
hw_mode=g
channel=6
ieee80211n=1
wmm_enabled=1
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase={PASSWORD}
rsn_pairwise=CCMP
```

Die SSID und das Passwort müssen hier natürlich mit deinen eigenen Daten für den Access-Point ersetzt werden.
Da das Passwort hier die Hauptsicherheit gegen Fremden Zugriff, auf öffentlichen Events, übernimmt, sollte es entsprechend sicher gewählt sein. Auch dieses ist bitte irgendwo zu Dokumentieren.

Wir empfehlen als Standard SSID einfach bei `ATS-Welti-Config` zu bleiben, damit für jeden klar ist wie und wo dieses Netzwerk genutzt wird.

Weiter müssen wir nun noch Hostapt mitteilen wo seine Konfigdatei liegen. Bearbeite `sudo nano /etc/default/hostapd` und ändere `DAEMON_CONF` zu folgendem:

```ini
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

**Wichtig: Das Attribut heißt DAEMON_CONF, nicht zu verwechseln mit DAEMON_OPTS**

Damit ist die Konfiguration des RPi als Access-Point abgeschlossen und er muss somit nur noch als DNS-Server und DHCP-Server konfiguriert werden.

## Konfigurieren als DNS- und DHCP-Server

Starten wird hier erstmal damit ein Backup von der existierenden Konfiguration zu machen:

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
```

Anschließend wird die Konfigdatei `sudo nano /etc/dnsmasq.conf` mit folgendem gefüllt:

```
interface=wlan0 
listen-address=10.45.0.1
bind-interfaces
bogus-priv
dhcp-range=10.45.0.10,10.45.0.30,24h
address=/#/10.45.0.1
```

Auch hier sind natürlich die Ip-Adressen und DHCP-Range anzupassen. Auch ist bitte darauf zu achten, dass genügend Ip-Adressen zu vergeben sind.

## Beheben des DNSMasq-Neustart-Problems

Leider kann es passieren, dass `dnsmasq` und `hostapd` beim booten des RPi nicht vernünftig starten, was auch der Grund ist, dass beim Booten dies teilweise angezeigt wird.
Starten diese Programme nach dem booten allerdings neu, funktionieren sie trotzdem perfekt.

`Hostapd` ist bereits von haus aus so konfiguriert nach einem Fehler neu zu starten, aber für `dnsmasq` müssen wir das selber einstellen.

Also versichere dich, dass der Service gestoppt ist und editiere die Serivce-Datei `sudo nano /lib/systemd/system/dnsmasq.service`.

In der `[Service]`-Sektion muss folgendes hinzugefügt werden:

```ini
Restart=on-failure
RestartSec=1s
```

Die sorgen dafür, dass der Serivce bei einem Fehler nach einer Sekunde neustartet bis es funktioniert.

# Final
Um die Installation zu finalisieren können nun alle Essenziellen Software-Services in den autostart-on-boot übernommen werden:

```bash
sudo systemctl enable Welti
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
```

Nach einem neustart des Welti: `sudo reboot 0` sollte nun das Wifi des Welti erscheinen und nach Verbindung durch ein Capture-Portal auf die Konfigurationsseite weiterleiten.