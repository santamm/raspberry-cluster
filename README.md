# raspberry-cluster
How to set up a RaspberryPi Kubernetes Cluster

### Hosts Setup (repeat on control-plane an each worker)
I used Raspberry Pi Imager to flash your microSD card, using `Ubuntu Server 20.04.2.LTS 64-bit` as OS.
Boot your host, as Ubuntu comes with ssh already installed you can ssh into your host from your Mac or other system you prefer to work from. I decided to install under the default user ubuntu. However if you want you can create a different user, just remember to add it to the sudoers group with:
``` sudo usermod -aG sudo newuser```

It is better to assign your hosts static IPs, to avoind trouble when you reboot aas your router might assign a different IP
#### Assigning a static IP
If your Ubuntu cloud instance is provisioned with cloud-init, you’ll need to disable it. To do so create the following file:
`sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`

network: {config: disabled}
To assign a static IP to your network interface, edit the file `/etc/netplan/01-netcfg.yaml` with your favourite text editor:

`sudo nano /etc/netplan/01-netcfg.yaml`
and add the fllowing:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens3:
      dhcp4: no
      addresses:
        - 192.168.0.221/24
      gateway4: 192.168.0.1
      nameservers:
          addresses: [8.8.8.8, 1.1.1.1]
```
where 192.168.0.221 is the IP you want to assign to your host and 192.168.0.1 is the IP of your router.



### Kubernetes Insta


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
