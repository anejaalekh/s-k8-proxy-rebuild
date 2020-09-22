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