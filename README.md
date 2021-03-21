# raspberry-cluster
How to set up a RaspberryPi Kubernetes Cluster

- Use Raspberry Pi Imager to flash your microSD card. Choose Raspberry Pi OS Lite (32 bit). 
- Before you boot up that RPi, make sure you create a file named ssh in the boot partition.
- Run raspi-config and change the memory split to 16mb, so that we have all the RAM for Kubernetes
- Now connect to the Raspberry Pi over your local network, either with an ethernet cable (better) or WiFi.
- To connect to wifi, you need to edit the Pi's wpa_supplicanf.conf file under /etc/wpa_supplicant. If you log to the network via WPA2, then add the following:

`sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`
and add the following:
```
network={
    ssid="WI-FI_NAME_HERE"
    psk="Password"
}

```
The password can be configured either as the ASCII representation, in quotes as per the example above, or as a pre-encrypted 32 byte hexadecimal number. You can use the wpa_passphrase utility to generate an encrypted PSK. This takes the SSID and the password, and generates the encrypted PSK. 
Otherwise if connecting through a WiFi hotspot, add the following lines:
```
network={
  ssid="WI-FI_NAME_HERE"
  identity="yourUsername@yourInstitution" 
  password="YOUR_PASSWORD_HERE"
  }
```
