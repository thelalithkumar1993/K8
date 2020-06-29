# Provide a highly available Jenkins cluster using Kubernetes

Let me create Kubernetes clusters before installing Jenkins. For creating Kubernetes Cluster, I am going to be using Ansible machines created from the Google Cloud Platform.

I can directly bring up Kubernetes clusters using GKE or EKS, let's say, I need these clusters in my local servers and therefore going to do Ansible to create clusters for me. Let us go with the hard way!

## Installation of Ansible Master

Let us take a CentOS 7 machine for this project.

### 1. Install epel-release

```bash
sudo yum install epel-release -y
```
### 2. Update packages 

```bash
sudo yum update -y
```
### 3. Install Ansible 

```bash
sudo yum install ansible -y
```
### 4. Switch to root user

```bash
 sudo su
```
### 5. Create an ansible user and provide password

```bash
useradd ansible
passwd ansible
```
 Note: I set my password as 'q1azw2sx' as it needs to be a mixture of numbers and letters.

### 6. Give permissions to ansible user

```bash
visudo
```
#### Go to the end and you will see some root permission privilege. 
Below " root  ALL=(ALL) ALL ", give permissions to ansible user as follows:
```bash
ansible ALL=(ALL)  NOPASSWD:ALL
```
### 7. Setup Password Authentication

```bash
cd ../../../
vi /etc/sshd/sshd_config
```
 Set passwordauthentication yes
#### By default, it will be 'no'
 Now, restart the service using the command

```bash
service sshd restart
```


#### Now, we have an ansible machine with ansible user given sudo privileges.

### 8. Switch to ansible user and generate ssh-key 
```bash
su ansible
ssh-keygen
```

### 9. Establish connection with the localhost
```bash
ssh-copy-id localhost
```
 Check the connection using "ssh localhost". You should be able to access ansible user without password.

## Installation for Ansible Nodes

### 1. Follow steps 1 to 7 above.

### 2. Switch back to Ansible master and copy ssh key to the node

```bash
ssh-copy-id <server id>
```

#### Note: To find the hostname of the node, use 
```bash
hostname -f
```  
#### Now, we have a working master and node with Ansible up and running.

## Creating Kubernetes Cluster on Master and Nodes
### 1. Creating hosts file
```bash 
vi hosts
```
Paste the commands:
```bash
[masters]
master ansible_host=ip_address ansible_user=ansible

[workers]
node1 ansible_host=ip_address ansible_user=ansible
node2 ansible_host=ip_address ansible_user=ansible
node3 ansible_host=ip_address ansible_user=ansible
```
### 2. Create a directory named kube_cluster_playbooks
```bash
mkdir kube_cluster_playbooks
cd kube_cluster_playbooks
```
We will be writing all the playbooks inside this directory

### 3. Setting up dependencies for K8s cluster installation
```bash
vi kube-dependencies.yml
```
Copy this instruction:
```bash 
- hosts: all  
  become: true
  tasks:
   - name: install Docker ##installing docker
     yum:
       name: docker
       state: present
       update_cache: true

   - name: start Docker ##starting docker service
     service:
       name: docker
       state: started

   - name: disable SELinux ##disabling SELinux as it is not fully supported by kubernetes
     command: setenforce 0

   - name: disable SELinux on reboot
     selinux:
       state: disabled

   - name: ensure net.bridge.bridge-nf-call-ip6tables is set to 1 ##setting up IPTables for networking
     sysctl:
      name: net.bridge.bridge-nf-call-ip6tables
      value: 1
      state: present

   - name: ensure net.bridge.bridge-nf-call-iptables is set to 1
     sysctl:
      name: net.bridge.bridge-nf-call-iptables
      value: 1
      state: present

   - name: add Kubernetes' YUM repository ##adding kubernetes YUM repository in remote server' repository list
     yum_repository:
      name: Kubernetes
      description: Kubernetes YUM repository
      baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
      gpgkey: https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
      gpgcheck: yes

   - name: install kubelet ##installing kubelet
     yum:
        name: kubelet-1.14.0
        state: present
        update_cache: true

   - name: install kubeadm ##installing kubeadm
     yum:
        name: kubeadm-1.14.0
        state: present

   - name: start kubelet #starting kubelet service
     service:
       name: kubelet
       enabled: yes
       state: started

- hosts: master
  become: yes
  tasks:
   - name: install kubectl ##installing kubectl on master node
     yum:
        name: kubectl-1.14.0
        state: present
        allow_downgrade: yes
```

### 4. Master installation
```bash
- hosts: master
  become: true
  tasks:
    - name: initialize the cluster ##initializing kubernetes cluster on master 
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

    - name: create .kube directory ##creating /.kube directory for storing configuration informations
      become: true
      become_user: ansible
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

    - name: copy admin.conf to user's kube config ##copying files created by "kubeadm init" to ansible user 
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ansible/.kube/config
        remote_src: yes
        owner: ansible

    - name: install Pod network ##installing flannel network as Pod Networking plugin for pods communication
      become: true
      become_user: ansible
      shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml >> pod_network_setup.txt
      args:
        chdir: $HOME
        creates: pod_network_setup.txt
  ```
  ### 5. Worker installation
```bash
- hosts: master
  become: true
  gather_facts: false
  tasks:
    - name: get join command ##joining worker nodes to the master node by creating tokens in master and then setting join command
      shell: kubeadm token create --print-join-command
      register: join_command_raw

    - name: set join command
      set_fact:
        join_command: "{{ join_command_raw.stdout_lines[0] }}"


- hosts: workers
  become: yes
  tasks:
    - name: join cluster ##joining nodes to master
      shell: "{{ hostvars['master'].join_command }} --ignore-preflight-errors all  >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```
### 7. Create a playbook which executes all the installations one by one
```bash 
cd ..
vi k8s-cluster.yml
```
Paste these commands:
```bash
---
- import_playbook: kube_cluster_playbooks/kube-dependencies.yml
- import_playbook: kube_cluster_playbooks/master.yml
- import_playbook: kube_cluster_playbooks/workers.yml 
```
### 8. Execute the playbook
```bash
ansible-playbook k8s-cluster.yml -i hosts
```
### 9. Check the clusters
```bash
kubectl get nodes
```
You will now see something like this
```bash 
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   3h   v1.14.0
node1    Ready    <none>   3h   v1.14.0
node2    Ready    <none>   3h   v1.14.0
```
Now, we have an active kubernetes cluster with master and nodes.

Let us install Jenkins on top of Kubernetes and let it orchestrate Jenkins for us!
## Jenkins Installation on Kubernetes
### 1. Create jenkins-deployment.yml
```bash
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: master
    spec:
      containers:
      - name: master
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 2
          failureThreshold: 5
        env:
        - name: JAVA_OPTS
          value: "-Djenkins.install.runSetupWizard=false"
```
### 2. Create jenkins-service.yml
```bash
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: master
  type: LoadBalancer
```
### 3. Execute Jenkins cluster
```bash 
kubectl apply -f jenkins-deployment.yml
kubectl apply -f jenkins-service.yml
```
Check the status using:
```bash 
kubectl get pod -o wide
```
You should be able to see
```bash
$ kubectl get pod -o wide
NAME                                 READY     STATUS    RESTARTS   AGE
jenkins-72205450-mc851               1/1       Running   0          25s
```
### 4. Port forward to access Jenkins UI
```bash
kubectl port-forward jenkins-72205450-mc851 8080
```

You should be seeing
```bash
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Access it using https://ip_address_of_pod:8080

Terminate the session using Ctrl + C


        





## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.


