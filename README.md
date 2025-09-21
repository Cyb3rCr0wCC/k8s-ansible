### Step-by-Step Guide: Initializing a Kubernetes Cluster on Ubuntu Bare-Metal Servers Using Ansible.

![img](https://cdn-images-1.medium.com/max/800/1*7RYKaNTIjK-gmnPkvz5pxA.png)

Setting up a Kubernetes cluster from scratch can feel daunting, especially on bare-metal servers where every configuration matters. Fortunately, with the power of **Ansible**, you can automate much of the setup, making the process faster, more reliable, and reproducible. In this guide, we’ll walk through the steps to **initialize a Kubernetes cluster on Ubuntu bare-metal servers**, covering everything from preparing your servers to running your first Kubernetes commands. Whether you’re a sysadmin, DevOps engineer, or Kubernetes enthusiast, this tutorial will help you streamline cluster deployment and management with minimal manual intervention.

🖥 Prepare environment

To follow along with this tutorial, we’ll use a simple lab environment:

🟢 **Ubuntu Serverx2(worker, controller)**

- RAM: 6 GB
- CPU: 4 cores
- Disk Storage: 50 GB

💻 **VirtualBox**

Used to create and manage virtual machines for testing the cluster.

⚙️ **Ansible**

Automates the installation and configuration of Kubernetes across nodes.

💡 Tip: Even with minimal resources, this setup is sufficient for a small test Kubernetes cluster for learning and experimentation.

If you want to dive deeper or see **every Ansible pipeline and role used in this setup**, feel free to check out my GitHub repository:

https://github.com/Cyb3rCr0wCC/k8s-ansible

### Preparing servers:

Initialize ansible role

```
mkdir -p ~/k8s-ansible/roles && cd ~/k8s-ansible/roles
ansible-galaxy init kubernetes
```

Inside `k8s-ansible/kubernetes/tasks `folder , create a file named `dependencies.yml`. This will include all the tasks to prepare the server (like disabling swap, loading kernel modules, installing containerd, etc.).

```
k8s-ansible/roles/kubernetes/tasks/dependencies.yml
- name: Updates
      apt:
        update_cache: yes
    - name: Disable SWAP
      shell: |
        swapoff -a
    - name: Disable SWAP in fstab
      replace:
        path: /etc/fstab
        regexp: '^([^#].*?\sswap\s+sw\s+.*)$'
        replace: '# \1'
    - name: Create an empty file for the containerd module
      copy:
        content: ""
        dest: /etc/modules-load.d/containerd.conf
        force: no
    - name: Configure modules for containerd
      blockinfile:
        path: /etc/modules-load.d/containerd.conf
        block: |
          overlay
          br_netfilter
    - name: Create an empty file for K8S sysctl parameters
      copy:
        content: ""
        dest: /etc/sysctl.d/99-kubernetes-cri.conf
        force: no
    - name: Configure sysctl parameters for K8S
      lineinfile:
        path: /etc/sysctl.d/99-kubernetes-cri.conf
        line: "{{ item }}"
      with_items:
        - "net.bridge.bridge-nf-call-iptables  = 1"
        - "net.ipv4.ip_forward                 = 1"
        - "net.bridge.bridge-nf-call-ip6tables = 1"
    - name: Apply sysctl parameters
      command: sysctl --system
    - name: Install APT Transport HTTPS
      apt:
        name: apt-transport-https
        state: present
    - name: Add Docker apt-key
      get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker-apt-keyring.asc
        mode: "0644"
        force: true
    - name: Add Docker's APT repo
      apt_repository:
        repo: "deb [arch={{ 'amd64' if ansible_architecture == 'x86_64' else 'arm64' }} signed-by=/etc/apt/keyrings/docker-apt-keyring.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
        update_cache: yes
    - name: Add Kubernetes apt-ke
      get_url:
        url: https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key
        dest: /etc/apt/keyrings/kubernetes-apt-keyring.asc
        mode: "0644"
        force: true
    - name: Add Kubernetes APT repository
      apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.asc] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /"
        state: present
        update_cache: yes
    - name: Install containerd
      apt:
        name: containerd.io
        state: present
    - name: Create containerd directory
      file:
        path: /etc/containerd
        state: directory
    - name: Add containerd configuration
      shell: /usr/bin/containerd config default > /etc/containerd/config.toml
    - name: Configuring Systemd cgroup driver for containerd
      lineinfile:
        path: /etc/containerd/config.toml
        regexp: "            SystemdCgroup = false"
        line: "            SystemdCgroup = true"
    - name: Enable the containerd service and start service
      systemd:
        name: containerd
        state: restarted
        enabled: yes
        daemon-reload: yes
    - name: Install Kubelet
      apt:
        name: kubelet
        state: present
        update_cache: true
    - name: Install Kubeadm
      apt:
        name: kubeadm
        state: present
    - name: Enable the Kubelet service
      service:
        name: kubelet
        enabled: yes
    - name: Load br_netfilter kernel module
      modprobe:
        name: br_netfilter
        state: present
    - name: Set bridge-nf-call-iptables
      sysctl:
        name: net.bridge.bridge-nf-call-iptables
        value: 1
    - name: Set ip_forward
      sysctl:
        name: net.ipv4.ip_forward
        value: 1
    - name: Reboot
      reboot:
```

Now, reference the dependency file inside your role’s main task file which is located `k8s-ansible/kubernetes/tasks/main.yml` with tag `kube_dependencies`.

```
k8s-ansible/roles/kubernetes/tasks/main.yml:
---
- import_tasks: dependencies.yml
  tags: kube_dependencies
```

At the project root, create a playbook file `server.yml`.

```
k8s-ansible/server.yml:
---
- name: Kubernetes # This starts the first (and only) play in this playbook
  hosts: all       # Indented 2 spaces from "name"
  become: yes             # Indented 2 spaces from "name"
  tasks:                  # Indented 2 spaces from "name"
    - include_role:
        name: kubernetes
      tags:
        - "kube_dependencies"
```

This tells Ansible to apply the `kubernetes` role to all servers listed in your **inventory file**.

Create inventory file in your

```
k8s-ansible/inventory.yml:
all:
  children:
    master:
      hosts:
        192.168.1.160:
    worker:
      hosts:
        192.168.1.150:
```

Here we have two hosts:

- **master — this is the ip address of master node(control-plane)**
- **worker — this is the ip address of worker node.**

If you don’t know what is worker and control plane node here’s great explanation for that https://spot.io/resources/kubernetes-architecture/11-core-components-explained/.

Create an `ansible.cfg` configuration file to specify the location of the `inventory.yml` file and define the default user Ansible will use to connect to the servers.

```
k8s-ansible/ansible.cfg:
[defaults]
inventory = inventory.yml
remote_user = ubuntu
```

Finally run playbook against inventory.

```
ansible-playbook server.yml --ask-pass --ask-become-pass --tag "kube_dependencies"
```

Above playbooks prepare our server to initalize kubernetes cluster.

Sample output after success:

![img](https://cdn-images-1.medium.com/max/800/1*VNttjbSIKWfgqThB8mKtaw.png)

### Initialize kubernetes cluster and control-plane node:

Now our servers are ready to initialize our kubernetes cluster.

First we need to initialize our control-plane and kubernetes cluster. Create new file named `master.yml` in tasks folder.

```
k8s-ansible/roles/kubernetes/tasks/master.yml:
- name: Ensure /etc/kubernetes directory exists
      ansible.builtin.file:
        path: /etc/kubernetes
        state: directory
        owner: root
        group: root
        mode: '0755'
    
    - name: Create an Empty file for Kubeadm configuring
      copy:
        content: ""
        dest: /etc/kubernetes/kubeadm-config.yaml
        force: no
    - name: Configure container runtime
      blockinfile:
        path: /etc/kubernetes/kubeadm-config.yaml
        block: |
          kind: ClusterConfiguration
          apiVersion: kubeadm.k8s.io/v1beta3
          networking:
            podSubnet: "10.244.0.0/16"
          ---
          kind: KubeletConfiguration
          apiVersion: kubelet.config.k8s.io/v1beta1
          runtimeRequestTimeout: "15m"
          cgroupDriver: "systemd"
          systemReserved:
            cpu: 100m
            memory: 350M
          kubeReserved:
            cpu: 100m
            memory: 50M
          enforceNodeAllocatable:
          - pods
    - name: Reset kubeadm (cleanup previous cluster)
      shell: kubeadm reset -f
      ignore_errors: yes
    - name: Remove old Kubernetes data
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - /var/lib/etcd
        - /var/lib/cni
        - /etc/cni/net.d
    - name: Create .kube directory
      become_user: ubuntu
      file:
        path: "/home/{{ ansible_user }}/.kube"
        state: directory
        mode: 0755
    - name: Initialize the cluster
      shell: "kubeadm init --config /etc/kubernetes/kubeadm-config.yaml"
      args:
        chdir: "/home/{{ ansible_user }}/.kube"
        
    - name: Copy admin.conf to User's kube config
      copy:
        src: /etc/kubernetes/admin.conf
        dest: "/home/{{ ansible_user }}/.kube/config"
        remote_src: yes
        owner: ubuntu
    - name: Install Pod Network
      become_user: ubuntu
      shell: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml >> pod_network_setup.log
      args:
        chdir: "/home/{{ ansible_user }}"
        creates: pod_network_setup.log
```

Add above `master.yml `into main.yml with tag `kube_master`

```
k8s-ansible/roles/kubernetes/tasks/main.yml:
---
- import_tasks: dependencies.yml
  tags: kube_dependencies
- import_tasks: master.yml
  tags: kube_master
```

Add new tag `kube_master` to server.yml playbook.

```
k8s-ansible/server.yml:
---
- name: Kubernetes  
  hosts: master       
  become: yes           
  tasks:                 
    - include_role:
        name: kubernetes
      tags:
        - "kube_dependencies"
        - "kube_master"
```

Note here we need to specify master, to run that playbook in a server we want to use it as control-plane. Now Run playbook again with tag `kube_master`.

```
ansible-playbook server.yml --ask-pass --ask-become-pass --tag "kube_master"
```

Here’s sample output:

![img](https://cdn-images-1.medium.com/max/800/1*opLzzfQw88sercm-Ca626g.png)

### Configure cni and loadbalancer:

Since we are using bare-metal servers, we need to manually configure the network components in Kubernetes. This involves setting up a CNI (Container Network Interface) for pod networking and deploying MetalLB to provide external IP addresses for kubernetes services — tasks that are typically handled automatically in cloud environments. If you are using a cloud-based Kubernetes cluster, this manual configuration is not required. Each of these components deserves a dedicated blog post, but for now, we’ll continue.

Create file called `network.yml` in tasks folder.

```
k8s-ansible/roles/kubernetes/tasks/network.yml:
- name: Copy calico.yml to remote host
  copy:
    src: calico.yaml
    dest: /tmp/calico.yml
    mode: '0644'
- name: Apply calico.yml using kubectl
  shell: kubectl apply -f /tmp/calico.yml
  environment:
    KUBECONFIG: /etc/kubernetes/admin.conf
- name: Copy metallb-native.yml to remote host
  copy:
    src: metallb-native.yaml
    dest: /tmp/metallb-system.yml
    mode: '0644'
- name: Apply metallb-system.yml using kubectl
  shell: kubectl apply -f /tmp/metallb-system.yml
  environment:
    KUBECONFIG: /etc/kubernetes/admin.conf
```

Now we need metallb and calico manifest file which we can download it from github.

```
cd k8s-ansible/roles/kubernetes/files
wget https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.28.3/manifests/calico.yaml
```

Update `main.yml` in tasks

```
k8s-ansible/roles/kubernetes/tasks/main.yml:
---
- import_tasks: dependencies.yml
  tags: kube_dependencies
- import_tasks: master.yml
  tags: kube_master
- import_tasks: network.yml
  tags: kube_network
```

Update `server.yml` with `kube_network` tag.

```
k8s-ansible/server.yml:
---
- name: Kubernetes  
  hosts: master       
  become: yes           
  tasks:                 
    - include_role:
        name: kubernetes
      tags:
        - "kube_dependencies"
        - "kube_master"
        - "kube_network"
```

Run pipeline

```
ansible-playbook server.yml --ask-pass --ask-become-pass --tag "kube_network"
```

### Join worker node to kubernetes cluster

With the cluster up and running, it’s time to expand by adding worker nodes.

Create `worker.yml` inside tasks folder.

```
k8s-ansible/roles/kubernetes/tasks/worker.yml:
- name: Retrieve Join Command from master
  shell: kubeadm token create --print-join-command --ttl 0
  register: join_command_raw
  delegate_to: "{{ MASTER_NODE }}"
  run_once: true
- name: Set Join Command fact 
  set_fact:
    join_command: "{{ join_command_raw.stdout }}"
    run_once: true
    cacheable: true
- name: Wait until master API server is accessible
  wait_for:
    host: "{{ MASTER_NODE }}"
    port: 6443
    timeout: 60
- name: Reset kubeadm on worker if previously joined
  shell: kubeadm reset -f
  failed_when: false
- name: Clean kubelet directories (workers only)
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /etc/kubernetes
    - /var/lib/kubelet
  when: inventory_hostname != MASTER_NODE
- name: Remove node if it exists in cluster
  command: kubectl delete node {{ ansible_hostname }}
  environment:
    KUBECONFIG: /etc/kubernetes/admin.conf
  delegate_to: "{{ MASTER_NODE }}"
  ignore_errors: true

- name: Join worker to cluster
  shell: "{{ join_command }}"
  args:
    creates: /etc/kubernetes/kubelet.conf
```

Looking closely at the YAML file, you’ll find a variable named `MASTER_NODE` that connects to the control-plane node and retrieves the join command for worker nodes. We haven’t defined this variable yet, so let’s add it. In Ansible, role-specific variables typically live in the `vars` folder within the role’s directory.

```
k8s-ansible/roles/kubernetes/vars/main.yml:
---
MASTER_NODE: 192.168.1.160
```

Add above `worker.yml` into `main.yml `with tag `kube_worker.`

```
k8s-ansible/roles/kubernetes/tasks/main.yml:
---
- import_tasks: dependencies.yml
  tags: kube_dependencies
- import_tasks: master.yml
  tags: kube_master
- import_tasks: network.yml
  tags: kube_network
- import_tasks: worker.yml
  tags: kube_worker
```

Add new tag `kube_worker` to `server.yml` playbook.

```
k8s-ansible/server.yml:
---
- name: Kubernetes  
  hosts: worker       
  become: yes           
  tasks:                 
    - include_role:
        name: kubernetes
      tags:
        - "kube_dependencies"
        - "kube_master"
        - "kube_network"
        - "kube_worker"
```

Note here we need to specify `worker `in `hosts:`, to run that playbook in a server we want to join to cluster. Now Run playbook again with tag `kube_worker`.

```
ansible-playbook server.yml --ask-pass --ask-become-pass --tag "kube_worker"
```

Sample output:

![img](https://cdn-images-1.medium.com/max/800/1*m0vlKaT5m0_nTTf7YuMeVg.png)

After that kubernetes cluster is ready to operate with control-plane and one worker node. You can check nodes by:

```
master> kubectl get nodes
```

![img](https://cdn-images-1.medium.com/max/800/1*k3wuN8FJvX1MNpwn9OPNfg.png)

That’s all. Again all yml files are in my github repo:

https://github.com/Cyb3rCr0wCC/k8s-ansible