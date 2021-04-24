# raspberry-cluster
How to set up a RaspberryPi Kubernetes Cluster
![Raspberry Pi Cluster](https://github.com/santamm/raspberry-cluster/blob/main/raspberry-pi.jpg)

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

#### Install a CNI add-on
A CNI add-on handles configuration and cleanup of the pod networks. We will use the Flannel CNI add-on.    

```
# Download the Flannel YAML data and apply it
#
curl -sSL https://raw.githubusercontent.com/coreos/flannel/v0.12.0/Documentation/kube-flannel.yml | kubectl apply -f -

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

#### What if I want my master node to be a worker too?
I had only two Raspberry Pis at my disposal, so I did not want to commit one of them to a control plane solely. That is why I unset the taint automatically applied to the control-plane node by running the following command:

`kubectl edit node ubuntu-controlplane`

and deleted the ollowing 2 lines:

```
 - effect: NoSchedule
   key: node-role.kubernetes.io/master
```


### Access dashboards from local machine
If you need to access any dashboard running from services in the cluster you have to create an SSH tunnel from your local machine to the control-plane to forward the request:
Run the following command on your local machine:

```ssh -L 9999:control-plane-ip:port -N -f -l user control-plane-ip``


For example if you started a proxy on the control-plane to access the API server with:
``` kubectl proxy&```

as it listens to the 8001 port you can run:
```ssh -L 8001:control-plane-ip:8001 -N -f -l user control-plane-ip```
and you will be to access the API server on your local machine on port 8001




## Essential add-ons
- [Helm](https://www.digitalocean.com/community/tutorials/an-introduction-to-helm-the-package-manager-for-kubernetes): Kubernetes package manager, to install packaged applications (charts)
- [Nginx Ingress controller](https://www.nginx.com/resources/glossary/kubernetes-ingress-controller/): a specialized load balancer for Kubernetes
- [Node Exporter](https://github.com/prometheus/node_exporter): a tool that collects health info rom each node and exports in a format that Prometheus can use.
- [Prometheus](https://prometheus.io/docs/introduction/overview/): an open-source systems monitoring and alerting toolkit
- [Grafana](https://grafana.com/docs/grafana/latest/getting-started/): an open source solution for running data analytics, pulling up metrics that make sense of the massive amount of data & to monitor our apps with the help of cool customizable dashboards.
- [cert-manager](https://cert-manager.io/docs/installation/kubernetes/): allows to issue certificates
- [Kubernetes Dashboard]: control cluster resources




### Install Helm
Helm is a package manager for Kubernetes that allows developers and operators to more easily package, configure, and deploy applications and services onto Kubernetes clusters. Helm does:
- Install software.
- Automatically install software dependencies.
- Upgrade software.
- Configure software deployments.
- Fetch software packages from repositories.

To install, run the ffollowing:
`curl -s https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

Add the chart repositories:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add infobloxopen https://infobloxopen.github.io/cert-manager/
helm repo add stable https://charts.helm.sh/stable/
```

### Install Ingress controller
A Kubernetes Ingress controller is a specialized load balancer for Kubernetes environments.
- Accept traffic from outside the Kubernetes platform, and load balance it to pods (containers) running inside the platform
- Can manage egress traffic within a cluster for services which need to communicate with other services outside of a cluster
- Are configured using the Kubernetes API to deploy objects called “Ingress Resources”
- Monitor the pods running in Kubernetes and automatically update the load‑balancing rules when pods are added or removed from a service

Run the following command as from the bare metal section of the Nginx Ingress controller website:
``` 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/baremetal/deploy.yaml
```

You can now check the status:
```kubectl -n ingress-nginx get pod```

### Node-exporter

You need to run the following code in each node:

First download:
```
curl -SL https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-armv7.tar.gz > node_exporter.tar.gz && sudo tar -xvf node_exporter.tar.gz -C /usr/local/bin/ --strip-components=1
```

then create the file `/etc/systemd/system/nodeexporter.service` to make sure it restarts each time the node is rebooted:
```
[Unit]
Description=NodeExporter
[Service]
TimeoutStartSec=0
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
```

Now you can start the services:
```
sudo systemctl daemon-reload && sudo systemctl enable nodeexporter && sudo systemctl start nodeexporter
```


### Install Prometheus
Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud.
Edit the `targets` line in the prometheus.yaml file below with the addresses ot hostnames of your configuration:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yaml: |
    global:
      scrape_interval:     15s
      external_labels:
        monitor: 'k3s-monitor'
    scrape_configs:
      - job_name: 'prometheus'
        scrape_interval: 5s
        static_configs:
          - targets: ['localhost:9090']
      - job_name: 'K3S'
        static_configs:
          - targets: ['192.168.0.201:9100', '192.168.0.202:9100', '192.168.0.203:9100','192.168.0.204:9100']
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus
        volumeMounts:
          - name: config-volume
            mountPath: /etc/prometheus/prometheus.yml
            subPath: prometheus.yaml
        ports:
        - containerPort: 9090
      volumes:
        - name: config-volume
          configMap:
           name: prometheus-config
---
kind: Service
apiVersion: v1
metadata:
  namespace: monitoring
  name: prometheus-service
spec:
  selector:
    app: prometheus
  ports:
  - name: promui
    protocol: TCP
    port: 9090
    targetPort: 9090
```


To install, run the following:


```kubectl apply -f prometheus.yaml```


You have to make the service available in the control-plane with a port-forward

```kubectl port-forward service/prometheus-service -n monitoring --address 0.0.0.0 9090:9090```

### Install Grafana

``` 
kubectl apply --kustomize github.com/kubernetes/ingress-nginx/deploy/grafana/

# get admin password
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# check the service is up and running
kubectl get svc -n ingress-nginx
```
Now to access the grafana dashboard from your control plane (or from your local machine with an SSH tunnel) you need to forward the request to the node.
If the service is on port 3000 you can forward from the same port on the control-plane:
```
kubectl port-forward -n ingress-nginx service/grafana --address 0.0.0.0 3000:3000
```
Once you are logged in the grafana dashboard (admin/admin) you can  add the Prometheus datasource to grafana using the url `http://prometheus-service:9090`, and import the Node Exporter grafana dashboard.


### Install Cert-manager

```
helm repo add jetstack https://charts.jetstack.io
kubectl create ns cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.16.0 \
  --set installCRDs=true
```

You can find a guide to issuing certificates with Cert-manager [here](https://medium.com/flant-com/cert-manager-lets-encrypt-ssl-certs-for-kubernetes-7642e463bbce).


### Kubernetes Dashboard
You can install the Kubernetes dashboard:

```kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml```

To access tha dashboard you need to start a proxy on the control plane to access the API server:
```kubectl proxy```

Now you can access the dashboard:

``` http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ ```

Alternatively you can forward the port for the dashboard only:

```kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8080:443```

and access the dashboard with:

```https://localhost:8080```

Either way you will need an authentication token:
List secrets using:

```
kubectl get secrets
```

You will see a secret named `dashboard-admin...`

```kubectl describe secret dashboard-admin-sa-token-kw7vn```

Copy and past the token into the authentication page


## Appendix: Connect your Raspberry Pi to the network via WiFi
1. identify the name of your wireless network interface. You will get a list of network interfaces. Usually the wireless one starts with a 'w'

`ls /sys/class/net` or `ip link show`

2. navigate to the `/etc/netplan` directory and locate the appropriate Netplan configuration files. Since we have disables cloud config, the config file will be the same we used to assign a static IP to the eth0 network interface:

`sudo nano /etc/netplan/01-netcfg.yaml`

You will have to add something like this:
```
wifis:
    wlan0:
      access-points:
        "SSID_name":
          password: "00000000"
      dhcp4: no
      addresses: [192.168.1.102/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8,4.2.2.2]

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
        dhcp4: no
        addresses: [192.168.1.102/24]
        gateway4: 192.168.0.1
        nameservers:
        addresses: [8.8.8.8,4.2.2.2]

```




