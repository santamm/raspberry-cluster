# raspberry-cluster
How to set up a RaspberryPi Kubernetes Cluster

### Hosts Setup (repeat on control-plane an each worker)
I used Raspberry Pi Imager to flash your microSD card, using `Ubuntu Server 20.04.2.LTS 64-bit` as OS.
I am assuming the raspberry Pi will be connected to your local network through an ethernet cable, however if you want to use WiFi you can follow the instructions below.
Boot your host, as Ubuntu comes with ssh already installed you can ssh into your host from your Mac or other system you prefer to work from. I decided to install under the default user ubuntu. However if you want you can create a different user, just remember to add it to the sudoers group with:

`sudo usermod -aG sudo newuser`

It is better to assign your hosts static IPs, to avoind trouble when you reboot aas your router might assign a different IP
#### Assigning a static IP
If your Ubuntu cloud instance is provisioned with cloud-init, you’ll need to disable it. To do so create the following file:

`sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`
```
network: {config: disabled}
```
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
        - 192.168.0.201/24
      gateway4: 192.168.0.1
      nameservers:
          addresses: [8.8.8.8, 1.1.1.1]
```
where 192.168.0.201 is the IP you want to assign to your host and 192.168.0.1 is the IP of your router.

You can now save the new configuration:

`sudo netplan apply`

#### Set Up SSH Keys on Ubuntu 
SSH keys provide an easy, secure way of logging into your server and are recommended for all users. You will log in to your hosts from your machine without even having to enter a password.
If you already have an ssh key on your machine, all you have to do is:

`ssh-copy-id username@remote_host`

You can check if you have one by running:

`ls ~/.ssh`

if have 2 files `id.rsa` and `id.rsa_pub` then you are set. Otherwise you will have to first create an RSA key pai by running:

`ssh-keygen`

Now you can login into yout hosts:

`ssh ubuntu@remote_host`

#### Install docker

```
sudo apt-get update
sudo apt install -y docker.io


sudo cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

You need to change the default cgroups driver Docker uses from cgroups to systemd to allow systemd to act as the cgroups manager and ensure there is only one cgroup manager in use.
```
sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt
```
Enable docker to start at boot:
```
sudo systemctl enable docker.service
```
You will have to reboot your host and check if docker has been installed correctly by running:
```
sudo docker info
```
#### Allow iptables to see bridged traffic
Kubernetes needs iptables to be configured to see bridged network traffic. You can do this by changing the sysctl config:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### Install Kubernetes packages for Ubuntu
```
# Add the packages.cloud.google.com atp key
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Add the Kubernetes repo
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Update the apt cache and install kubelet, kubeadm, and kubectl
# (Output omitted)
sudo apt update && sudo apt install -y kubelet kubeadm kubectl

# Disable (mark as held) updates for the Kubernetes packages
$ sudo apt-mark hold kubelet kubeadm kubectl

```
That's it for the host configuration. Remember you have to do this for your control-plane host and for all your worker nodes.

### Config the Control Plane
Generate a bootstrap token:
```
# Generate a bootstrap token to authenticate nodes joining the cluster
TOKEN=$(sudo kubeadm token generate)
echo $TOKEN
```

Initialize the Control Plane:
```
sudo kubeadm init --token=${TOKEN} --kubernetes-version=v1.21.0 --pod-network-cidr=10.244.0.0/16
```
If everything went ok, you should see a message like this one:
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.201:6443 --token zqqoy7.9oi8dpkfmqkop2p5 \
    --discovery-token-ca-cert-hash sha256:71270ea137214422221319c1bdb9ba6d4b76abfa2506753703ed654a90c4982b
```
You can now validate that the Control Plane has been installed with the kubectl get nodes command:    
```
kubectl get nodes
```
    
### Join the workers nodes to the control plane
To join a worker node to the cluster, login into the node and run the following:

```
# Add a worker nodes
# Get join token
kubeadm token list
# Get Discovery Token CA cert Hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
# Get your API server endpoint with kubectl command:
kubectl cluster-info


# Now we have everything we need to join a worker node to the Cluster
kubeadm join \
  <control-plane-host>:<control-plane-port> \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
  
  
sudo kubeadm join 192.168.0.201:6443 --token w4r8s6.w8yzb2pvvnlhb9cp --discovery-token-ca-cert-hash sha256:f6d91b2f0bf4bd26a3c66894c5ea9db5fe6bc7499ef817d8847e074477261d10
```



#### Connect your Raspberry Pi to the network via WiFi
1. identify the name of your wireless network interface. You will get a list of network interfaces. Usually the wireless one starts with a 'w'

`ls /sys/class/net`

2. navigate to the `/etc/netplan` directory and locate the appropriate Netplan configuration files.

`sudo nano /etc/netplan/50-cloud-init.yaml`

You will have to add something like this:
```
wifis:
    wlan0:
        dhcp4: true
        optional: true
        access-points:
            "SSID_name":
                password: "WiFi_password"

```



```

Otherwise if connecting through a WiFi hotspot, add instead the following lines:

```
wifis:
    wlan0:
        dhcp4: true
        optional: true
        access-points:
            "SSID_name":
                identity: "yourUsername@yourInstitution"
                password: "YOUR_password_here"

```




