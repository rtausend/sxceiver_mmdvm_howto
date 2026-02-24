# **MMDVM & SXCeiver Installations-Guide**

Diese Anleitung beschreibt die vollständige Einrichtung eines MMDVM-Systems auf Basis der SXCeiver-SDR-Hardware unter Linux (z. B. Raspberry Pi OS). Die Frequenz muss im Code geändert werden da sie NICHT! aus der MMDVM.ini ausgelesen wird

---

## **1. libgpiod Downgrade & Sperre**

Aktuelle Linux-Distributionen nutzen `libgpiod` v2.x. Der SXCeiver-Treiber benötigt jedoch die Version 1.x. Wir installieren die Bullseye-Version und sperren sie gegen automatische Updates.

```Bash
# Alte Version entfernen  
sudo apt remove --purge libgpiod-dev -y  
sudo apt autoremove -y

# Bullseye Repository kurzzeitig hinzufügen  
echo "deb http://deb.debian.org/debian bullseye main" | sudo tee /etc/apt/sources.list.d/bullseye.list  
sudo apt update

# Version 1.6.2 installieren  
sudo apt install libgpiod-dev=1.6.2-1 libgpiod2=1.6.2-1 -y

# Automatische Updates verhindern (Hold)  
sudo apt-mark hold libgpiod-dev libgpiod2

# Bullseye Repo wieder entfernen  
sudo rm /etc/apt/sources.list.d/bullseye.list  
sudo apt update

```

## **2. Abhängigkeiten & Build-Tools**

Installation der notwendigen Bibliotheken für SDR, Signalverarbeitung und Kommunikation.

```Bash  
sudo apt install -y --no-install-recommends \  
  git make g++ cmake libsoapysdr-dev soapysdr-tools \  
  libasound2-dev clang llvm-dev libclang-dev \  
  libzmq3-dev cppzmq-dev libliquid-dev

```

## **3. Hardware-Treiber (SoapySX)**

Kompilierung des spezifischen Treibers für das SXCeiver-Board.

```Bash  
cd ~  
git clone "https://github.com/tejeez/sxxcvr.git"  
cd sxxcvr/SoapySX  
mkdir build && cd build  
cmake ..  
make -j4  
sudo make install  
sudo ldconfig

### **Hardware-Check**

Bevor du fortfährst, prüfe, ob das Board erkannt wird:

* `ls -l /proc/device-tree/hat`  
* `SoapySDRUtil --find`

```

## **4. MMDVM Modem (Linuxport)**

Das Modem emuliert die Hardware-Schnittstelle. Hier nehmen wir eine manuelle Frequenzkorrektur im Code vor.

```Bash  
cd ~  
git clone https://github.com/tejeez/MMDVM.git  
cd MMDVM  
git checkout linuxport  
git submodule init && git submodule update

# Frequenz-Fix (RX 438.3625 MHz / TX 430.7625 MHz)  
sed -i 's/double rxFreq = 434.0e6, txFreq = 434.0e6;/double rxFreq = 430762500.0, txFreq = 438362500.0;/g' IOLinux.cpp

# Kompilieren  
make -f Makefile.Linux clean  
make -f Makefile.Linux -j4

```

## **5. MMDVMHost-SDR (Steuerung)**

Die Host-Software verarbeitet die digitalen Modi (DMR, P25, M17 etc.).

```Bash  
cd ~  
git clone https://github.com/qradiolink/MMDVMHost-SDR.git  
cd MMDVMHost-SDR

# Build-Fixes für fehlende Header  
sed -i '22i #include <cstdint>' M17Utils.cpp  
sed -i '22i #include <cstdint>' NullController.cpp

# Kompilieren  
make clean  
make -j4

```

## **6. Automatisierung (Systemd Services)**

Damit die Dienste beim Booten automatisch im Hintergrund starten, erstellen wir zwei Units.

### **Modem Service**

Datei erstellen: `sudo nano /etc/systemd/system/mmdvm-modem.service`

```Ini, TOML  
[Unit]  
Description=MMDVM Modem Linuxport  
After=network.target

[Service]  
User=dk5rta  
WorkingDirectory=/home/dk5rta/MMDVM  
ExecStart=/home/dk5rta/MMDVM/bin_linux/mmdvm  
Restart=always  
RestartSec=3

[Install]  
WantedBy=multi-user.target
```

### **Host Service**

Datei erstellen: `sudo nano /etc/systemd/system/mmdvm-host.service`

```Ini, TOML  
[Unit]  
Description=MMDVM Host SDR  
After=mmdvm-modem.service  
Requires=mmdvm-modem.service

[Service]  
User=dk5rta  
WorkingDirectory=/home/dk5rta/MMDVMHost-SDR  
ExecStart=/home/dk5rta/MMDVMHost-SDR/MMDVMHost MMDVM.ini  
Restart=always  
RestartSec=5

[Install]  
WantedBy=multi-user.target

```

## **7. Aktivierung & Kontrolle**

```Bash  
sudo systemctl daemon-reload  
sudo systemctl enable --now mmdvm-modem.service  
sudo systemctl enable --now mmdvm-host.service

# Live-Logs einsehen  
journalctl -u mmdvm-host.service -f  
```

## **8. Funktionierende MMDVM.ini**

```Bash  
[General]
Callsign=DK5RTA
Id=2627000
Timeout=120
Duplex=1
ModeHang=7
# RFModeHang=8
NetModeHang=7
Display=None
Daemon=0

[Info]
RXFrequency=430762500
TXFrequency=438362500
Power=1
Latitude=0.0
Longitude=0.0
Height=0
Location=Nowhere
Description=Multi-Mode Repeater
URL=www.db0rta.de

[Log]
# Logging levels, 0=No logging
DisplayLevel=1
FileLevel=0
FilePath=..
FileRoot=MMDVM
FileRotate=1

[CW Id]
Enable=0
Time=10

[Modem]
Protocol=uart
UARTPort=/tmp/MMDVM_PTS
UARTSpeed=115200
# The port and address for an I2C connection
I2CPort=/dev/i2c
I2CAddress=0x22

TXInvert=1
RXInvert=1
PTTInvert=0
TXDelay=0
RXOffset=200
TXOffset=200
DMRDelay=0
RXLevel=50
TXLevel=50
RXDCOffset=0
TXDCOffset=0
RFLevel=100
RSSIMappingFile=../RSSI.dat
UseCOSAsLockout=0
Trace=0
Debug=1

[DMR]
Enable=1
Beacons=0
BeaconInterval=60
BeaconDuration=3
ColorCode=1
SelfOnly=0
EmbeddedLCOnly=0
DumpTAData=0
CallHang=7
TXHang=7
ModeHang=7
# OVCM Values, 0=off, 1=rx_on, 2=tx_on, 3=both_on, 4=force off
OVCM=0
debug=0

[P25]
Enable=1
NAC=293
SelfOnly=0
OverrideUIDCheck=1
RemoteGateway=0
TXHang=7
ModeHang=180

[POCSAG]
Enable=0
Frequency=439987500

[DMR Network]
Enable=1
Type=Direct
RemoteAddress=2621.master.brandmeister.network
LocalPort=62031
RemotePort=62031
Password=xxxxxxxx
Jitter=0
Slot1=1
Slot2=1
ModeHang=7
Debug=0

[P25 Network]
Enable=1
LocalAddress=127.0.0.1
LocalPort=32010
GatewayAddress=127.0.0.1
GatewayPort=42020
ModeHang=180
Debug=0

[POCSAG Network]
Enable=0
LocalAddress=127.0.0.1
LocalPort=3800
GatewayAddress=127.0.0.1
GatewayPort=4800
ModeHang=1
Debug=0

[Lock File]
Enable=0
File=/tmp/MMDVM_Active.lck

[Remote Control]
Enable=0
Address=127.0.0.1
Port=7642
```
