# raspberry-cluster
How to set up a RaspberryPi Kubernetes Cluster

#### Flash the initial OS
- Use Raspberry Pi Imager to flash your microSD card. Choose Raspberry Pi OS Lite (32 bit). 
- Before you boot up that RPi, make sure you create a file named ssh in the boot partition.
- Run raspi-config and change the memory split to 16mb, so that we have all the RAM for Kubernetes
- Change the password with passwd pi
#### Connect to Raspberry Pi to the network
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
Reconfigure the interface with `wpa_cli -i wlan0 reconfigure`
You can verify whether it has successfully connected using `ifconfig wlan0`
- Add the following to /boot/cmdline.txt (essential for k3s), but make sure that you don’t add new lines:
`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`
#### Copy or create an SSH key
Copy your SSH key to the Raspberry Pi with:
`ssh-copy-id pi@raspberrypi.local`
#### Install Kubernetes
We will install:
- arkade — a hassle-free way to get Kubernetes apps and CLIs
- kubectl — the Kubernetes CLI
- k3sup — the Kubernetes (k3s) installer that uses SSH to bootstrap Kubernetes

Download arkade,  a portable Kubernetes marketplace which makes it easy to install around 40 apps to your cluster:
```
curl -sSL https://dl.get-arkade.dev | sudo sh
arkade get kubectl
arkade get k3sup
```
Remember to either add them to your PATH or move them to your `/usr/local/bin` directory

Now install k3sup on the node:
```export IP="192.168.0.1" # find from ifconfig on RPi
k3sup install --ip $IP --user pi``


# Install agent node
export AGENT_IP=192.168.0.101

export SERVER_IP=192.168.0.100
export USER=root

k3sup join --ip $AGENT_IP --server-ip $SERVER_IP --user $USER
