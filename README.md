### Setting up a Raspberry Pi as an autonomous WIFI Access Point with an MQTT broker###



#### Initial Raspberry Pi setup ####

* I started with a new Rapsberry Pi 4, 4Gb, installed with the NOOBS operating system available from the [official site](https://www.raspberrypi.org/downloads/noobs/). During the initial setup, configure a wired Ethernet connection, but do not configure WIFI.

* Start with making sure you run the latest versions of all things

```
$ sudo apt-get update
$ sudo apt-get full-upgrade
```

#### Setup the WIFI Access point ####


* Setup raspi as an access point following the official guide for setting up a Raspberry Pi as an [access point](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md). Install DNSMasq and HostAPD and stop those services for now.

```
    $ sudo apt install dnsmasq hostapd
    $ sudo systemctl stop dnsmasq
    $ sudo systemctl stop hostapd
```

* Configure a static IP address for the access point by editing the configuration file.

```
    $ sudo nano /etc/dhcpcd.conf
```


* Navigate to the end of the file and copy/paste the following lines. This will be the IP address of your access point.

```
    interface wlan0
        static ip_address=192.168.4.1/24
        nohook wpa_supplicant
```



* Restart the dhcpcd service:


```
    $ sudo service dhcpcd restart
```


* Make a backup copy of dnsmasq configuration file and create a new one with just the info needed:


```
    $ sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    $ sudo nano /etc/dnsmasq.conf
```


* In this empty file, copy/paste the following lines defining the dhcp range of addresses and the lease time, here 24 hours.


    interface=wlan0      # Use the require wireless interface - usually wlan0
    dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h


* Reload dnsmasq to use the updated configuration:


```
    $ sudo systemctl reload dnsmasq
```


* Configure the access point by editing the hostapd.conf file.

 ```
   $ sudo nano /etc/default/hostapd.conf
```


* Add the following lines to the file, mind the network name (ssid) and password (wpa_passphrase).


```
    interface=wlan0
    driver=nl80211
    ssid=MyAccessPoint
    hw_mode=g
    channel=7
    wmm_enabled=0
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=MySecretPassword
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
```


* Update hostapd to let it know the location of your configuration file


```
   $ sudo nano /etc/default/hostapd
```


* Find the line with #DAEMON_CONF, and replace it with this:


```
   DAEMON_CONF="/etc/hostapd/hostapd.conf"
```


* Enable and start hostapd, and check their status:


```
    $ sudo systemctl unmask hostapd
    $ sudo systemctl enable hostapd
    $ sudo systemctl start hostapd
    $ sudo systemctl status hostapd
    $ sudo systemctl status dnsmasq
```

* Add routing and masquerade. First add enable IP forwarding by editing sysctl.conf file:


```
    $ sudo nano /etc/sysctl.conf
```


* Uncomment the following line:


```
    net.ipv4.ip_forward=1
```


* Then,


```
   $ sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
   $ sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```


* Edit rc.local to install the rules on reboot


```
    $ sudo nano /etc/rc.local
```


* Add this line to rc.local:


```
    iptables-restore < /etc/iptables.ipv4.nat
```


* Use raspi-config to enable ssh.

* Reboot and you're done with the access point.


```
   $ sudo reboot
```


#### Install MQQT Broker ####


* Now setup the MQTT broker and a test client. This is explained in details [here](
https://randomnerdtutorials.com/how-to-install-mosquitto-broker-on-raspberry-pi/).


```
   $ sudo apt install -y mosquitto mosquitto-clients
```


* Enable mosquitto, check version, hostname and subscribe to a test topic.


```
    $ sudo systemctl enable mosquitto.service
    $ mosquitto -v
    $ hostname -I
    $ mosquitto_sub -d -t testTopic
```



#### Test your installation with you phone####



* __With your phone__, install the free "MQTT Dashboard" app. Connect your wifi to your access point using the following setup or whatever credentials you used.


```
    ssid: MyAccessPoint
    password: MySecretPassword
```


* In "MQTT Dashboard" press the (+) button and add a connection, using:


```
    Client ID: YouChooseTheClientID
    Server: 192.168.4.1
    Port: 1883
    Username: leave empty
    Password: leave empty
```


* Press "Create". Select the server you just configured, then go to "Publish", presse the (+) sign to create a new MQTT publish widget. Choose "Button" and setup widget properties:


```
    Friendly name: MyButton
    Topic: testTopic
    Button text: Press Me!
    Value to publish: button_was_pressed
```


* Press "Create"

* Press your button, check your Raspberry Pi to see it received the message.
