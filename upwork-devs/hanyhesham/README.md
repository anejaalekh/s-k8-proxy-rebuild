# Purpose

Building a K8s cluster using a portable option to be used across any cloud providers or on-prem machines.

There are multiple ways and tools to implement K8s including:

  ### Learning environment:

   - [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube)

   - [kind](https://kubernetes.io/docs/setup/learning-environment/kind)

   - [k8s the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

  ### Production environment:

   - [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm)

   - [kops](https://kubernetes.io/docs/setup/production-environment/tools/kops)

   - [kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray).

  ### Third party tools:

   - [microk8s](https://microk8s.io)

# Kubernetes deployment

## kubespray

Kubespray is a composition of Ansible playbooks that helps to install a Kubernetes cluster hosted on GCE, Azure, OpenStack, AWS, vSphere, Packet (bare metal), Oracle Cloud Infrastructure (Experimental) or Baremetal.

### Requirements

1- Ansible v2.9 and python-netaddr is installed on the machine that will run Ansible commands.

2- Jinja 2.11 (or newer) is required to run the Ansible Playbooks.

3- The target servers must have access to the Internet in order to pull docker images.

4- The target servers are configured to allow IPv4 forwarding.

5- Your ssh key must be copied to all the servers part of your inventory.

6- The firewalls are not managed, you'll need to implement your own rules the way you used to. in order to avoid any issue during deployment you should disable your firewall.

7- If kubespray is ran from non-root user account, correct privilege escalation method should be configured in the target servers. Then the ansible_become flag or command parameters --become or -b should be specified.

### Installation

1- The VM running K8s will be hosted on GCP with the following specs:

- Ubuntu 20.04
- e2-medium (2 vCPUs, 4 GB memory)

3- Install needed packages:

```
sudo apt update
sudo apt install ansible python3-pip
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray/
sudo pip3 install -r requirements.txt
```

4- Build the inventory file ([ansible](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible.md)):

  - Generate the inventory file
 
    ```
    declare -a IPS=($k8s_node)
    CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
    ```

  - Remove access_ip from the _inventory/mycluster/hosts.yaml_ file:

	```
	all:
	  hosts:
	    node1:
	      ansible_host: $k8s_node
	      ip: $k8s_node
	  children:
	    kube-master:
	      hosts:
	        node1:
	    kube-node:
	      hosts:
	        node1:
	    etcd:
	      hosts:
	        node1:
	    k8s-cluster:
	      children:
	        kube-master:
	        kube-node:
	    calico-rr:
	      hosts: {}
	```

5- Add Ingress addon by editing the following lines to _inventory/mycluster/group_vars/k8s-cluster/addons.yml_:

```
ingress_nginx_enabled: true
ingress_nginx_nodeselector:
  node-role.kubernetes.io/master: ""
```

6- Generate SSH key and copy the public key to _~/.ssh/authorized_keys_:

```
ssh-keygen
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 
```

7- Run the ansible playbook:

  `ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml`

8- Test the cluster deployment:

- SSH to the node and run the following command:
  ```
  sudo su - 
  kubectl get nodes
  kubectl get all --all-namespaces
  ``` 

# Apps deployment

### Configure registry

We are using github repo for this demo as a private registry.

- Create a personal token with (_read:packages_, _write:packages_) permissions in your github account (Settings --> Developer settings --> personal access token)
 
- Log in to the repo using your GitHub Id and the token.

` docker login docker.pkg.github.com` 

### Build Images

Follow the guide in [k8-reverse-proxy](https://github.com/k8-proxy/k8-reverse-proxy/tree/develop/stable-src) to build ICAP and Squid images.

For the Nginx images, build the image from _nginx_ directory:

```
cd images/nginx
docker build . -t nginx:v1 
```

**Note: This step should be done using GitHub actions to automated the process of images build and push in the prodcution environment**


### Fix Squid-ICAP problem

Squid server fails to connect to ICAP server if hostname is used instead of IP, to fix this, we need to edit entrypoint in squid image.

To build the new image:

```
cd images/squid
docker build . -t squid-reverse:v1
```

**Note: This is a temporary solution to fix the issue, a PR should be requested in the [k8-reverse-proxy](https://github.com/k8-proxy/k8-reverse-proxy/tree/develop/stable-src) to edit the code.**

### Allow Ingress passthrough

- Edit ingress daemonset to add new argument:

  `kubectl edit daemonset.apps/ingress-nginx-controller -n ingress-nginx`

- Add `- --enable-ssl-passthrough` to `- args:`:

```
    spec:
      containers:
      - args:
        - --enable-ssl-passthrough
```

- Wait until new pod is up and running:

  `kubectl get pods -n ingress-nginx`


### Deploy apps manifests

In order to deploy our stack to the K8s cluster, we need to apply some manifests as following:

```
cd k8s-manifests
kubectl apply -f ns.yaml
kubectl create secret generic entrypoint --from-file=./secrets/entrypoint.sh -n reverse-proxy
kubectl create secret generic subfilter --from-file=./secrets/subfilter.sh -n reverse-proxy
kubectl create secret generic cert --from-file=./secrets/full.pem -n reverse-proxy
kubectl apply -f icap.yaml
kubectl apply -f squid.yaml
kubectl apply -f nginx.yaml
kubectl apply -f ingress.yaml
```

### Test Application

- Add the following values to your machine hosts file:

```
$k8s_node_ip gov.uk.glasswall-icap.com www.gov.uk.glasswall-icap.com assets.publishing.service.gov.uk.glasswall-icap.com
```

- Open your browser and navigate to https://www.gov.uk.glasswall-icap.com


### Next Steps

- Use Github actions for image creation.
