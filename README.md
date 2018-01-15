## 使用kuberspay无坑安装生产级Kubernetes集群

本文档参考网上大神wiselyman的文章 http://www.wisely.top/2017/07/01/no-problem-kubernetes-kuberspay/， 感谢大神的无私奉献。此文档大部分内容copy大神的著作。
`kuberspay`是`kargo`更名后的名称，[使用kargo快速自动化搭建kubernetes集群](http://www.wisely.top/2017/05/16/kargo-ansible-kubernetes/)(各节点的准备信息也请参考该文)，上篇文章的部署方式的缺陷还是需要科学上网，所以还是比较麻烦的。另外一篇文章[无坑畅玩minikube(利用阿里云镜像编译minikube)](http://www.wisely.top/2017/06/27/no-problems-minikube/)，本文的原理与此文一致，使用阿里云里的镜像来安装Kubernetes集群。

### 1. 安装ansible

使用自动化运维工具ansible进行安装，我本机是MacOS，使用`homebrew`安装`ansible`:

```shell
brew install ansible

```
linux上安装ansible的方法请google/baidu


### 2. 修改kubespray代码

代码修改分别在以下的文件里，请查看源码，修改源码时主要参考阿里云里对应的镜像和版本，以防阿里云无此镜像，查看阿里云镜像请访问[https://dev.aliyun.com/search.html](https://dev.aliyun.com/search.html)。
- `kubespray/roles/kubernetes-apps/ansible/defaults/main.yml`
- `kubespray/roles/download/defaults/main.yml`
- `kubespray/extra_playbooks/roles/download/defaults/main.yml`
- `kubespray/inventory/group_vars/k8s-cluster.yml`
- `kubespray/roles/dnsmasq/templates/dnsmasq-autoscaler.yml`

此代码中所需要的镜像地址已经修改成阿里云镜像。

如需要`kubespray`官方源码，请自行下载，地址为:[https://github.com/kubernetes-incubator/kubespray](https://github.com/kubernetes-incubator/kubespray)。

### 3. inventory.cfg
在`kubespray/inventory/inventory.cfg`，添加内容:
```
[all]
master ansible_host=192.168.205.88    ansible_user=root
node1  ansible_host=192.168.205.89    ansible_user=root
node2  ansible_host=192.168.205.90    ansible_user=root

[kube-master]
master

[etcd]
master
node1
node2

[kube-node]
node1
node2

[k8s-cluster:children]
kube-node
kube-master
```
### 4. 修改docker options, 允许docker下载非官方的镜像
文件路径：inventory/group_vars/k8s-cluster.yml
docker_options中添加自己的insecure-registry和--registry-mirror，如下面的例子：
docker_options: "--insecure-registry={{ kube_service_addresses }} --insecure-registry=192.168.205.0/24 --registry-mirror=http://ed85cf14.m.daocloud.io --graph={{ docker_daemon_graph }}  {{ docker_log_opts }}"


### 5. 使用ansible安装

在kubespray根目录，执行:
```shell
 ansible-playbook -u centos -b -i inventory/inventory.cfg cluster.yml
```

### 6. 验证安装

- 登录master
- 查看node:`kubectl get node`
    ```shell
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1h        v1.9.0+coreos.0
node1     Ready     node      1h        v1.9.0+coreos.0
node2     Ready     node      1h        v1.9.0+coreos.0
    ```
- 查看pod:`kubectl get pod --all-namespaces`

    ```shell
    NAMESPACE     NAME                                  READY     STATUS    RESTARTS   AGE
    kube-system   kube-apiserver-node1                  1/1       Running   0          49m
    kube-system   kube-controller-manager-node1         1/1       Running   0          49m
    kube-system   kube-dns-69997447-783dz               3/3       Running   0          48m
    kube-system   kube-proxy-node1                      1/1       Running   0          49m
    kube-system   kube-proxy-node2                      1/1       Running   0          49m
    kube-system   kube-proxy-node3                      1/1       Running   0          49m
    kube-system   kube-scheduler-node1                  1/1       Running   0          49m
    kube-system   kubedns-autoscaler-2506230242-1vcgk   1/1       Running   0          48m
    kube-system   nginx-proxy-node2                     1/1       Running   0          48m
    kube-system   nginx-proxy-node3                     1/1       Running   0          49m
    ```


## kubespray安装kubernetes完成后kubectl客户端配置

接上篇[使用kuberspay无坑安装生产级Kubernetes集群](http://www.wisely.top/2017/07/01/no-problem-kubernetes-kuberspay/),在使用`kubespray`安装好了`kubernetes`之后，我们需要在自己的客户端电脑配置kubectl，如何将集群的配置信息在本地配置呢，我们使用下面的脚本，放在`scripts\copy-kubeconfig.yaml`下，内容为:

```Yaml
---
- hosts: kube-master[0]
  gather_facts: no
  become: yes
  tasks:
  - fetch:
      src: "/etc/kubernetes/ssl/{{ item }}.pem"
      dest: "{{ playbook_dir }}/kubectl/{{ item }}.pem"
      flat: True
    with_items:
      - admin-{{ inventory_hostname }}-key
      - admin-{{ inventory_hostname }}
      - ca
  - name: export hostname
    set_fact:
      kubectl_name: "{{ inventory_hostname }}"

- hosts: localhost
  connection: local
  vars:
    kubectl_name: "{{ hostvars[groups['kube-master'][0]].kubectl_name }}"
    cluster_name: "{{ hostvars[groups['kube-master'][0]].cluster_name }}"
    kube_apiserver_port: "{{ hostvars[groups['kube-master'][0]].kube_apiserver_port }}"
    system_namespace: "{{ hostvars[groups['kube-master'][0]].system_namespace }}"
  tasks:
  - name: "check if context admin@{{ cluster_name }} exists"
    command: kubectl config get-contexts admin@{{ cluster_name }}
    register: kctl
    failed_when: kctl.rc == 0

  - block:
    - name: "create cluster {{ cluster_name }}"
      command: >
        kubectl config set-cluster {{ cluster_name }}
        --server=https://{{ kubectl_name }}:{{ kube_apiserver_port }}
        --certificate-authority={{ playbook_dir }}/kubectl/ca.pem
        --embed-certs

    - name: "create credentials admin"
      command: >
        kubectl config set-credentials admin
        --certificate-authority={{ playbook_dir }}/kubectl/ca.pem
        --client-key={{ playbook_dir }}/kubectl/admin-{{ kubectl_name }}-key.pem
        --client-certificate={{ playbook_dir }}/kubectl/admin-{{ kubectl_name }}.pem
        --embed-certs

    - name: "create context admin@{{ cluster_name }}"
      command: >
        kubectl config set-context admin@{{ cluster_name }}
        --cluster={{ cluster_name }}
        --namespace={{ system_namespace }}
        --user=admin

    - name: "use context admin@{{ cluster_name }}"
      command: kubectl config use-context admin@{{ cluster_name }}
    when: kctl.rc != 0

  - name: "clean up fetched certificates"
    file:
      state: absent
      path: "{{ playbook_dir }}/kubectl"

```



在`kubespray`根目录执行：

`ansible-playbook -i inventory/inventory.cfg scripts/copy-kubeconfig.yaml`

执行完成后，如果你本机的`hosts`文件已经配置了`node1`对应的ip，配置已经完成；如果没有配置，编辑`vi /Users/wangyunfei/.kube/config `，将`node1`修改为`192.168.1.130`。此时再执行：

```
#kubespray wangyunfei$ kubectl get node
NAME      STATUS                     AGE       VERSION
node1     Ready,SchedulingDisabled   3d        v1.6.1+coreos.0
node2     Ready                      3d        v1.6.1+coreos.0
node3     Ready                      3d        v1.6.1+coreos.0
```



## kubenetes-dashboard安装

在安装完成后，若需安装`kubernetes-dashboard`，请进行下面操作：

- 下载描述文件`curl https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml -o kubernetes-dashboard.yaml`

- 将` gcr.io/google_containers/kubernetes-dashboard-amd64`修改为`registry.cn-hangzhou.aliyuncs.com/google_images/kubernetes-dashboard-amd64`

- 执行:`kubectl create -f kubernetes-dashboard.yaml `

- 查看执行结果

  ```
  #kubespray wangyunfei$ kubectl get pod
  NAME                                  READY     STATUS    RESTARTS   AGE
  kube-apiserver-node1                  1/1       Running   0          3d
  kube-controller-manager-node1         1/1       Running   0          3d
  kube-dns-69997447-783dz               3/3       Running   0          3d
  kube-proxy-node1                      1/1       Running   0          3d
  kube-proxy-node2                      1/1       Running   0          3d
  kube-proxy-node3                      1/1       Running   0          3d
  kube-scheduler-node1                  1/1       Running   0          3d
  kubedns-autoscaler-2506230242-1vcgk   1/1       Running   0          3d
  kubernetes-dashboard-27199923-qqgrq   1/1       Running   0          1m
  nginx-proxy-node2                     1/1       Running   0          3d
  nginx-proxy-node3                     1/1       Running   0          3d
  ```

  ​

- 查看页面，执行`kubectl proxy`，访问`http://127.0.0.1:8001/ui`

  ![https://raw.githubusercontent.com/wiselyman/kubespray/master/kubernetes-dashboard.png](https://raw.githubusercontent.com/wiselyman/kubespray/master/kubernetes-dashboard.png)