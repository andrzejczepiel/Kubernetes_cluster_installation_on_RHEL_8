In this manual I will present how I was installing Docker engine and Kubernetes cluster on EC2 instance in AWS running RedHat 8.

https://andrzejczepiel.github.io/Kubernetes_cluster_setup_RHEL_8/

I will present how to add repository to operating system in order to install Docker and Kubernetes packages.
I will show what changes I made to the operating system, how cluster was initialized and how to add nodes to the cluster.

You have to be aware that Kubernetes installation requires minimum 2 virtual CPU and 2 GB of RAM.
That is why if you want to run it in AWS, it wil not work on Free Tier instance t2.micro.
In my case I was using t3a.medium instance type.
Infrastructure was build using terraform deployment. 
(you can check my other terrafrom repositories on https://github.com/andrzejczepiel

**You can use this manual for installation in cloud or on premisses. If you want to use it on different OS than RedHat, make suitable adjustments in commands.**

One of the requirement for Kubernetes is disabling swap, run below command:

    # swapoff -a

Add "kube_user" user to sudoers file without password, this user will be used to manage your Kubernetes cluser and make calls
to API server with kubectl client, I am using NOPASSWD option so I do not have to provide password, you may force it otherwise.
Keep in mind, I am running it in my personal AWS lab.

    # echo "kube_user        ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers

I am also enabling IP forwarding on my system:

    # echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
    # sysctl -p

As I did this setup in AWS Cloud, I do not want to see long EC2 names when running eg: `kubectl get nodes` command.
That is why I modified /etc/hosts file by adding entires for every node of my cluster infrastructure.
Here you see that I will have master node (control plane) and three worker nodes.
Please note that IP address range: 10.0.1.x is an example here. Your environment may differ.

    # cat <<EOF > /etc/hosts
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    10.0.1.1   c1master
    10.0.1.2   c1node1
    10.0.1.3   c1node2
    10.0.1.4   c1node3
    EOF

I change also hostname of every node which will be part of my cluster eg: 
    # hostname c1node1, hostname c1master  etc.


Let's move on to next part which is preparing repositories for Docker and Kubernetes.
I am using Docker engine and Kubernetes acts as an orchestrator of containers.

Fist we will create repository file for Kubernetes

    #  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF


Add Docker repository to system repo list and install other packages which you may need later
You may use yum or dnf.


    # yum install telnet -y
    # dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    # dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm -y
    # dnf install docker-ce-3:19.03.8-3.el7 -y
    # yum install python3-pip.noarch -y
    # pip3.6 install docker-compose
    # yum install unzip -y
    # yum install gzip -y
    # yum install dos2unix.x86_64 -y
    # yum install git.x86_64 -y
    # adduser kube_user
    # usermod -aG docker kube_user
    # yum install wget -y


Refresh repo list

    # dnf repolist -y


Install required kubernetes packages

    # yum install kubeadm kubelet kubectl kubernetes-cni -y

Enable both services and start Docker service

    # systemctl enable kubelet.service
    # systemctl enable docker.service
    # service docker start


Download network definition files, download those files to home directory of a user which will manage your cluster
NOTE:  there are many other network definitions you can use, calico, is an example here.

    # su - kube_user
    
Being in kube_user home directory: /home/kube_user/ run below commands (you may use any other user, kube_user is an example)

    $ wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
    $ wget https://docs.projectcalico.org//v3.10/manifests/calico.yaml


Initialize cluster, API server advertise address is IP of your master node, or control plane node.
192.168.0.0/16 is an example of pod network, you can change this in calico.yaml file 

    $ sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.0.1.1


Apply network (communication between pods) definition

    $ kubectl apply -f /home/kube_user/rbac-kdd.yaml
    $ kubectl apply -f /home/kube_user/calico.yaml	


Copy config file to kube_user home directory and change permissions

    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config


To join a node (minion) to a cluster run command on a node which you want to be part of your cluster.
Command which is dispayed as result of above executed kubeadm init command

    $ sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.0.1.1

will look similar to this:

    $ sudo kubeadm join 10.0.1.1:6443 --token yjp4ep.plt0maylaxemggdq --discovery-token-ca-cert-hash sha256:4ad8121486fe267e1a80100fd5ac86049b0214fe7fae3f10aa923a3d056e782a

If by any chance you do not know node join token you can generate it with below command:
Get kubernetes kubadm token:

    $ sudo kubeadm token list > node_join_token

Get CA cert hash

    $ sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //' > discovery-token-ca-cert-hash


After that you can join a node to a cluster

To verify run:

    $ kubectl get nodes
