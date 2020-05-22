My Smarthome is documented in this repository. The base is a Raspberry Pi with containers.

![version](https://img.shields.io/badge/version-0.5.0-blue)

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Hardware requirements](#hardware-requirements)
- [Raspberry Pi installation](#raspberry-pi-installation)
- [Install docker environment](#install-docker-environment)
  - [Install Docker Compose](#install-docker-compose)
  - [Test docker environment](#test-docker-environment)
- [Docker volumes](#docker-volumes)
- [Basic containers that I use](#basic-containers-that-i-use)
  - [Portainer](#portainer)
  - [RPI-Monitor](#rpi-monitor)
  - [RPI dockerui](#rpi-dockerui)
  - [NTOP](#ntop)
  - [cAdvisor](#cadvisor)
- [Smarthome containers that I use](#smarthome-containers-that-i-use)
  - [openHAB](#openhab)
  - [Logviwer for openHAB](#logviwer-for-openhab)
  - [Node-RED](#node-red)
  - [Mosquitto](#mosquitto)
  - [Zigbee2Mqtt](#zigbee2mqtt)
- [openHAB Configuration](#openhab-configuration)
  - [Used bindings](#used-bindings)
- [MQTT](#mqtt)
  - [Sonoff S20](#sonoff-s20)
    - [Connection example to openHAB](#connection-example-to-openhab)
  - [Gosung SP111](#gosung-sp111)
    - [Connection example to openHAB](#connection-example-to-openhab-1)

## Hardware requirements
The basic system consists of:

- Raspberry Pi 4 with 4GB Ram
- Zigbee CC2531 with antenna and housing

Smarthome components:

- Homematic CCU2
- Homematic Thermostat
- Sonoff S20 (with Tasmota firmware)
- Gosund smart plug SP111 (with Tasmota firmware)
- Aqara Door & Window Sensor
- Aqara Motion Sensor

## Raspberry Pi installation
Install pi with the current raspian version (I currently use Buster) and Update your pi
```
sudo apt-get update
sudo apt-get upgrade
```

## Install docker environment
Install docker
```
curl -sSL https://get.docker.com | sh
``` 
Using docker as a non-root user
```
sudo usermod -aG docker pi
```
You can verify this with the following command
```
groups pi
```
Ensure docker is listed as one of the groups
### Install Docker Compose
I will use pip3 to install Docker Compose. Note that you need to have python3 and pip3 installed already in order to use pip3.

If you don't have python3 installed already, you can run the following commands:
```
sudo apt-get install libffi-dev libssl-dev
sudo apt install python3-dev
sudo apt-get install -y python3 python3-pip
```
Once you have python and pip installed just run the following command:
```
sudo pip3 install docker-compose
```
### Test docker environment
To test docker, we'll run the hello-world image.
```
docker run hello-world
```
If Docker is installed properly, you'll see a "Hello from Docker!" message.
##  Docker volumes
Since there are always permission problems when the docker container creates the folders for the volumes in the HOME directory. Should you ever do this manually.
```
cd ~
mkdir docker
mkdir docker/volumes
```

## Basic containers that I use
### Portainer
I always put the volumes in the home directory of the user pi, but of course this is up to everyone. But then the commands have to be adapted.

For basic administration and ease of use I use Portainer. Which can be installed with the following command:
```
docker run -d \
  --name=portainer \
  -p 8000:8000 \
  -p 9000:9000 \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer
```
Then you can access to portainer with http://raspberrypi:9000/

Further information about portainer can be found at https://www.portainer.io/

_Do not forget to set the endpoint to the local IP address. Then you can click the ports directly and you can open the application directly in your browser!_

### RPI-Monitor
RPi Monitor displays a variety of system-critical data. All relevant data is clearly displayed on the so-called "Health Page
```
docker run -d \
  --name=rpi-monitor \
  --device=/dev/vchiq \
  --device=/dev/vcsm \
  --volume=/opt/vc:/opt/vc \
  --volume=/boot:/boot \
  --volume=/sys:/dockerhost/sys:ro \
  --volume=/etc:/dockerhost/etc:ro \
  --volume=/proc:/dockerhost/proc:ro \
  --volume=/usr/lib:/dockerhost/usr/lib:ro \
  -p=8888:8888 \
  --restart=always michaelmiklis/rpi-monitor:latest
```
### RPI dockerui
This container is a UI for dockas in a interface.
```
docker run -d \
  --name rpi-dockerui \
  -p 9999:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart=always hypriot/rpi-dockerui
```
### NTOP
High-speed web-based traffic analysis.
```
docker run -d \
  --name=ntopng -t \
  -p 3000:3000 \
  --restart=always lelmarir/ntopng-docker-raspberrypi
```
### cAdvisor
The tool has native support for Docker & enables us to track historical resource usage with histograms & stuff. This helps in understanding the resource consumption, memory footprint of the code running on the servers.
```
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=7000:8080 \
  --detach=true \
  --name=cadvisor budry/cadvisor-arm:latest
```
## Smarthome containers that I use
### openHAB
The core system I use is openHAB. All other docker containers provide additional functionality.

_I am currently using version 2.5.2, but you can also use the latest one._
```
docker run -d \
  --name openhab \
  --net=host \
  -v /etc/localtime:/etc/localtime:ro \
  -v /etc/timezone:/etc/timezone:ro \
  -v /home/pi/docker/volumes/openhab_conf:/openhab/conf \
  -v /home/pi/docker/volumes/openhab_userdata:/openhab/userdata \
  -v /home/pi/docker/volumes/openhab_addons:/openhab/addons \
  -e USER_ID=1000 \
  -e GROUP_ID=1000 \
  --restart=always openhab/openhab:2.5.5
```
### Logviwer for openHAB
To see the log output in openHAB, I use the following container.
```
docker run -d \
  --name frontail-openhab \
  -p 9001:9001 \
  -v /home/pi/docker/volumes/openhab_userdata/logs:/var/log/openhab2:ro welteki/frontail-openhab:latest
```
### Node-RED
I use Node-RED as alternative rule engine for openHAB.

Je nachdem wo man das volume für den node-RED container ablegen muss, muss eventuell der Ordner manuell für den Benutzer pi angelegt werden. Da sonst der container nicht startet. Deshalb sollte man ordner für das volumen manuell anlegen mit:
```
mkdir ~/docker/volumes/node_red
```
Then start the container
```
docker run -d \
  --name node-red \
  -p 1880:1880 \
  -v /home/pi/docker/volumes/node_red:/data \
  --restart=always nodered/node-red
```
I use the following node-RED module:
```
node-red-contrib-openhab2
node-red-contrib-telegrambot
node-red-contrib-bigtimer
node-red-contrib-telegrambot-home
node-red-contrib-fritzbox-presence
```
### Mosquitto
I use mosquitto as MQTT broker.
```
docker run -d \
  --name mosquitto \
  -p 1883:1883 \
  -v /home/pi/docker/volumes/mosquitto_config:/mosquitto/config \
  -v /home/pi/docker/volumes/mosquitto_data:/mosquitto/data \
  -v /home/pi/docker/volumes/mosquitto_log:/mosquitto/log \
  --restart=always eclipse-mosquitto:latest
```
### Zigbee2Mqtt
I use the image as Zigbee gateway for my CC2531, which I connected as USB to the Raspberry Pi. So I don't have to use a gateway of a certain manufacturer and I am independent.
```
docker run -d \
  --name zigbee2mqtt \
  -v /home/pi/docker/volumes/zigbee2mqtt:/app/data \
  --device=/dev/ttyACM0 \
  --restart=always \
  -e TZ=Europe/Berlin \
  -v /run/udev:/run/udev:ro \
  --privileged=true koenkk/zigbee2mqtt
```
## openHAB Configuration
### Used bindings
I use the following binding. The configuration can be taken from the official openHAB documentation: [openHAB docs](https://www.openhab.org/docs/)
- Amazon Echo Steuerung Binding amazonechocontrol
- Astro Binding astro
- FritzboxTR064 Binding fritzboxtr064
- Homematic Binding homematic
- NTP Binding ntp

I also use the openHAB Cloud Service: https://myopenhab.org/

## MQTT
I use the MQTT protocol for many SmartHome components. Here I tell you a few things about it.
To read the commands from MQTT I use [MQTT.fx](http://mqttfx.org/).
### Sonoff S20
The Sonoff S20 socket comes with its own software with cloud connectivity. Those who don't want to do that can play Tasmota on many devices. Usually I use [Tasmota](https://tasmota.github.io/docs/) for all devices, if that is possible. 
For this device you have to solder something to get an alternative firmware.
#### Connection example to openHAB
Once you have successfully flashed the socket you can directly adjust the MQTT settings. In my case I create the topic **SonoffS20_1**.

![SonoffS20 MQTT Settings](https://github.com/r77s/My-Smart-Home/blob/master/images/SonoffS20.png)

Then you can create it as an item in openHAB. For example like this:
```
Switch soS20_1 "Steckdose" ["Switchable"] {mqtt=">[broker:cmnd/SonoffS20_1/POWER:command:*:default], <[broker:stat/SonoffS20_1/POWER:state:default]"}
```
### Gosung SP111
I also like to use the Gosung SP111 socket. It is cheap everywhere and you don't have to solder to use it with Tasmota. A good help is the [ct Repository](https://github.com/ct-Open-Source/tuya-convert)
#### Connection example to openHAB
For the Gosung sockets, the topic can be created in the same way as for the Sonoff S20. The item for openHAB would look like this:
```
Switch Gosund_SP111_1 "Steckdose" ["Switchable"] {mqtt=">[broker:cmnd/Gosund-SP111-1/POWER:command:*:default], 
<[broker:stat/Gosund-SP111-1/POWER:state:default]"}
```