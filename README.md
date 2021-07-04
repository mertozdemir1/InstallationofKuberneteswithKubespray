# Installation of Kubernetes with Kubespray

My Test Environment. 


![topology](https://user-images.githubusercontent.com/30117144/124351210-ad917080-dc01-11eb-91c5-bf2ec45f6277.png)
 

Apply the commands below on Kubernetes Nodes.

	Disable SELINUX

		setenforce 0 && \
		sed -i "s/^SELINUX\=enforcing/SELINUX\=disabled/g" /etc/selinux/config && \
		sed -i "s/^SELINUX\=permissive/SELINUX\=disabled/g" /etc/selinux/config && \
		systemctl disable firewalld; systemctl stop firewalld; systemctl mask firewalld


	Load "br_netfilter" module persistent for K8S 

		sudo modprobe br_netfilter
		echo "br_netfilter" >> /etc/modules-load.d/br_netfilter.conf
		echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
		echo 'net.bridge.bridge-nf-call-iptables=1' | sudo tee -a /etc/sysctl.conf
		reboot
			We can check this with to below commands outputs
				lsmod | grep -i br_netfilter
				cat /proc/sys/net/bridge/bridge-nf-call-iptables


	Create user with root permissions  for installation. (This user must create also on Host machine)

		adduser kubespray
		sed -i '/visiblepw/i Defaults   \!requiretty' /etc/sudoers
		echo 'kubespray    ALL=(ALL)       NOPASSWD: ALL' > /etc/sudoers.d/kubespray

Prepare Passwordless SSH access on host machine.
	
    Firstly, we  are generating public key

      kubespray@master-pc:~/.ssh$ ssh-keygen -t rsa
      Generating public/private rsa key pair.
      Enter file in which to save the key (/home/kubespray/.ssh/id_rsa): k8s-lisbon
      Enter passphrase (empty for no passphrase): 
      Enter same passphrase again: 
      Your identification has been saved in /home/kubespray/.ssh/k8s-lisbon
      Your public key has been saved in /home/kubespray/.ssh/k8s-lisbon.pub
      kubespray@master-pc:~/.ssh$ ls -la
      total 20
      drwx------ 2 kubespray kubespray 4096 Tem  1 00:00 .
      drwxr-xr-x 5 kubespray kubespray 4096 Haz 30 12:39 ..
      -rw------- 1 kubespray kubespray 2610 Tem  1 00:00 k8s-lisbon
      -rw-r--r-- 1 kubespray kubespray  573 Tem  1 00:00 k8s-lisbon.pub



	  and, Copy public key to kubernetes nodes.

	    kubespray@master-pc:~/.ssh$ ssh-copy-id -i k8s-lisbon.pub kubespray@<ip-address>  


Download kubespray and install ansible on the host machine

	$ su - kubespray
	$ git clone https://github.com/kubernetes-sigs/kubespray.git
	$ cd /kubespray/
	$ sudo pip install -r requirements.txt


We are on the last step before installation. Now we configure ansible

	kubespray@master-pc:~/kubespray/inventory/k8s-lisbon$ export CLUSTER_NAME="k8s-lisbon"
	kubespray@master-pc:~/kubespray/inventory/k8s-lisbon$ export CLUSTER_DOMAIN="k8s-lisbon"
	kubespray@master-pc:~/kubespray/inventory/k8s-lisbon$ export CLUSTER_FQDN="$CLUSTER_NAME.$CLUSTER_DOMAIN"


  We are creating our cluster's inventory file

    $ mkdir inventory/$CLUSTER_NAME
 	  $ cp -rfp inventory/sample/* inventory/$CLUSTER_NAME/


  Configure inventory.ini file and save as inventory.cfg
	
	  $ mv inventory/$CLUSTER_NAME/inventory.ini inventory/$CLUSTER_NAME/inventory.cfg

  On all.yml file, this parameter must set true
	
	  - kube_read_only_port: 10255


  We can change parameters in "k8s-cluster.yml" such as

    - kube_version: v1.21.1
    - kube_api_anonymous_auth: true
    - cluster_name: k8s-lisbon
    - kube_network_plugin: calico

Lastly, we are running ansible playbook

	ansible-playbook -u $(whoami) -b -i inventory/$CLUSTER_NAME/inventory.cfg cluster.yml
