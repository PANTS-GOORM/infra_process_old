# Kubernetes Node 공통 환경 구성
○	apt 업데이트   
■ ``` sudo apt -y update ```

○	ubuntu 방화벽 해제 설정   
■	``` sudo ufw disable ```   
■	``` sudo ufw status ```

○	스왑의 비활성화
■	kubelet이 제대로 작동하게 하려면 반드시 스왑을 사용하지 않도록 설정한다.   
■	``` sudo swapoff -a ```   
&ensp;●	swap 기능 off

■	free   
&ensp;●	스왑 off가 잘 되었는지 확인

■	``` sudo echo 0 > /proc/sys/vm/swappiness ```   
&ensp;●	kernel 속성의 swap를 disable, root 사용자로 전환후 수행   
■	``` sudo sed -e '/swap/ s/^#*/#/' -i /etc/fstab ```   
&ensp;●	swap을 하는 파일시스템을 찾아 삭제   

■	``` sudo vi /etc/fstab ```   
&ensp;●	/swapfile 항목 #으로 주석처리   

○	NTP( Network Time Protocol ) 설정   
■	Kubernetes cluster는 보통 여러 개의 VM이나 서버로 구성되기 때문에 cluster 내의 모든 Node 들은 NTP를 통해서 시간 동기화 요구   

■	``` sudo apt -y install ntp ```   
■ ``` sudo systemctl restart ntp ```   
■	``` sudo systemctl status ntp ```   
■	``` sudo ntpq -p ```   

○	네트워크 패킷을 올바르게 포워딩하기 위해 kernel에서 IP 포워딩을 활성화 시켜야 함   
■	``` sudo -i ``` 또는 ``` sudo su - ```   

■	``` echo '1' > /proc/sys/net/ipv4/ip_forward ```   
■	``` cat /proc/sys/net/ipv4/ip_forward ```   

○	containerd를 이용한 container runtime 구성   
■	
```
sudo cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf

overlay
br_netfilter
EOF
```

○	modprobe 프로그램은 요청된 모듈이 동작할 수 있도록 부수적인 모듈을 depmod 프로그램을 이용하여 검색해 필요한 모듈을 커널에 차례로 등록  
■	``` sudo modprobe overlay ```  
■	``` sudo modprobe br_netfilter ```  

○	Node간 통신을 위한 iptables에 bridge 관련 설정 추가  
■	
```
sudo cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf

net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1 
net.bridge.bridge-nf-call-ip6tables = 1 
EOF
```

● 
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf  

br_netfilter 
EOF
```

●	
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf 

net.bridge.bridge-nf-call-ip6tables = 1 net.bridge.bridge-nf-call-iptables = 1 net.ipv4.ip_forward = 1 
EOF
```

■	``` sudo sysctl --system ```  

○	Kubernetes runtime 준비  
■	containerd, docker, CRI-O 중 선택이며 containerd 사용  
■	apt가 HTTP로 repository를 사용하는 것을 허용하기 위한 패키지 및 Docker에 필요한 패키지 설치  

■	``` sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2 ```  

○	도커 공식 GPG 키 추가 (https://docs.docker.com/engine/install/ubuntu/)  
■	``` sudo install -m 0755 -d /etc/apt/keyrings ```  
■	``` curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg ```  
■	
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

■	``` sudo apt-get update ```  

○	docker runtime 주의  
■	Docker와 containerd가 모두 감지되면 Docker 우선 이는 Docker 18.09부터 containerd와 함께 제공되고 Docker만 설치한 경우에도 둘 다 감지할 수 있기 때문에 필요  
■	다른 두 개 이상의 런타임이 감지되면 kubeadm은 오류와 함께 종료  
&ensp;●	docker는 image 개발 및 테스트 용도  
&ensp;●	cluster container runtime은 containerd 사용  
&ensp;●	https://v1-28.docs.kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

○	docker-ce version 확인  
■	``` apt-cache policy docker-ce ```  

○	docker-ce와 관련 도구및 containerd 설치  
■	``` sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin ```

■	``` sudo docker version ```

○	containerd 설정  
■	``` sudo sh -c "containerd config default > /etc/containerd/config.toml" ```  

■	``` sudo vi /etc/containerd/config.toml ```
&ensp;●	disabled_plugins = []  ⇒ [] CRI 제거 확인

■	``` sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml ```

■	``` sudo systemctl restart containerd.service ```

○	docker daemon 설정
■	``` sudo vi /etc/docker/daemon.json ```
```json
{ 
"exec-opts": ["native.cgroupdriver=systemd"],  "log-driver": "json-file", 
"log-opts": { 
"max-size": "100m" 
}, 
"storage-driver": "overlay2" 
}
```

■	``` sudo mkdir -p /etc/systemd/system/docker.service.d ```

■	``` sudo usermod -aG docker worker ```  
■	``` sudo systemctl daemon-reload ```  

■	``` sudo systemctl enable docker ```

■	``` sudo systemctl restart docker ```

■	``` sudo systemctl status docker ```

■	``` sudo systemctl restart containerd.service ```

■	``` sudo systemctl status containerd.service ```

■	``` sudo reboot ```

■	``` docker version ```  
■	``` docker info ```  
&ensp;●	Cgroup Driver: systemd  ⇒ 확인

○	kubernetes 도구설치 (1.28)  
■	``` curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg ```

■	``` echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list ```

■	``` sudo apt update ```

■	``` sudo apt-cache policy kubeadm ```

■	``` sudo apt -y install kubelet kubeadm kubectl ```

■	``` kubeadm version ```
■	``` kubectl version ```
■	``` kubelet --version ```

○	설치된 Kubernetes tool이 자동으로 업데이트 되는 것을 방지해야 하므로 hold 시킨다.  
■	``` sudo apt-mark hold kubelet kubeadm kubectl ```

○	모든 노드에 설치되는 kubelet은 항상 start 상태 유지  
■	``` sudo systemctl daemon-reload ```  
■	``` sudo systemctl restart kubelet.service ```
■	``` sudo systemctl enable --now kubelet.service ```
