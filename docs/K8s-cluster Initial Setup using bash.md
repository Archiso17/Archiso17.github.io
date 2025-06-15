```bash
#!/bin/bash

#Bash script for k8s cluster setup using kubeadm on Linux OS


LINUXOS=`cat /etc/os-release | awk -F'=' '/^ID=/ { print $2 }'|tr -d '"'`
VCPU=`nproc`
VMEM=`cat /proc/meminfo |awk '/MemTotal/ {printf "%.1f GB", $2/1024/1024}'` 
HOSTNAME=`hostname`
MAC_ADDRESS=`cat /sys/class/net/$(ip route | awk '/default/ {print $5}')/address`
PRODUCT_ID=`cat /sys/class/dmi/id/product_uuid`

if [ `id -u` -ne 0 ];then

  echo "Please run the script as root user!!!"
  exit 0
fi
banner_info() {

  echo "##################################################"
  echo "Distro: $LINUXOS"
  echo "vCPU: $VCPU"
  echo "vMem: $VMEM"
  echo "HostName: $HOSTNAME"
  echo "Mac Address: $MAC_ADDRESS"
  echo "Product Id: $PRODUCT_ID"
  echo "###################################################"
}

system_update() {
	if [ "$LINUXOS" == "almalinux" ];then
		echo -e "\n[+]Updating $LINUXOS... \n"
		dnf update -y
	else 
		echo -e "\n[+]Updating $LINUXOS...\n"
		apt update -y
	fi
}

disable_security() {
	if [ "$LINUXOS" == "almalinux" ];then
		setenforce 0
		sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
	else
		systemctl disable apparmor --now
		ufw disable
	fi
}


config_swap() {
	echo -e "\n[*]Checking swap...\n"
	if swapon -s || cat /etc/fstab|grep "swap";then
		echo -e "\n Swap found; Removing it...\n"
		swapoff -a
		sed -i '/swap/d' /etc/fstab
	else
		echo -e "\n[+]No swap found.\n"
	fi
}

sysctl_config() {
	cat <<-EOF | sudo tee /etc/sysctl.d/k8s.conf
	net.ipv4.ip_forward = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1
	EOF
	echo -e "\n[+]Applying conf...\n"
	sysctl --system 1> /dev/null
}

kernel_modules() {
	if lsmod | grep "overlay" 1> /dev/null && lsmod | grep "br_netfilter" 1> /dev/null;then
		echo -e "[*] Already Exists.\n"
	else
		modprobe overlay
		modprobe br_netfilter
	fi
}

docker_remove() {
	if [ "$LINUXOS" == "almalinux" ];then
		echo -e "\n[+]Removing docker package in almalinux...\n"
		for pkg in docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine;do
			dnf remove $pkg;
		done
	else
		echo -e "\n[+]Removing docker package in Ubuntu...\n"
		for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc;do
			apt remove $pkg;
		done
	fi
}

update_repo() {
	if [ "$LINUXOS" == "almalinux" ];then
		echo -e "\n[+]Installing Docker Repo in almalinux..."
		dnf -y install dnf-plugins-core
		dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
	else
		echo -e "[+]Insalling Docker Repo in Ubuntu...\n"
		# Add Docker's official GPG Key:
		apt-get update
		apt-get install ca-certificates curl
		install -m 0755 -d /etc/apt/keyrings
		curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
		chmod a+r /etc/apt/keyrings/docker.asc
		# Add the repository to Apt sources:
		echo \
  			"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  			$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  		 	tee /etc/apt/sources.list.d/docker.list > /dev/null
		apt update
	fi
}

install_docker() {
	if [ "$LINUXOS" == "almalinux" ];then
		echo -e "\n[+]Installing Docker on almalinux\n"
		dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
		systemctl enable docker --now
	else
		echo -e "\n[+]Installing Docker on Ubuntu\n"
		apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
		systemctl enable docker --now
	fi
}

install_k8s_repo() {
	if [ "$LINUXOS" == "ubuntu" ];then
		apt update
		apt install -y apt-transport-https ca-certificates curl gpg
		curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
		echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
	else
		cat <<-EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
		[kubernetes]
		name=Kubernetes
		baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
		enabled=1
		gpgcheck=1
		gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
		exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
		EOF
	fi
}

install_k8s_pkg() {
	if [ "$LINUXOS" == "ubuntu" ];then
		echo -e "\n[+]Installing k8s package in ubuntu...\n"
		apt update
		apt install -y kubelet kubeadm kubectl
		apt-mark hold kubelet kubeadm kubectl
		systemctl enable --now kubelet

	else
		echo -e "\n[+]Installing k8s pacakge in almalinux...\n"
		yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
		systemctl enable --now kubelet
	fi
}

config_cgroup() {
	echo -e "\n[+]adding Cgroup to systemd...\n"
	containerd config default > /etc/containerd/config.toml
	sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
	systemctl restart containerd
}

banner_info
system_update
disable_security
config_swap
sysctl_config
kernel_modules
docker_remove
update_repo
install_docker
install_k8s_repo
install_k8s_pkg
config_cgroup
```



