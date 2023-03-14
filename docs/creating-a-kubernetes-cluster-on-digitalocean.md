# 使用 Python 和 Fabric 在 DigitalOcean 上创建 Kubernetes 集群

> 原文：<https://testdriven.io/blog/creating-a-kubernetes-cluster-on-digitalocean/>

在本教程中，我们将使用 Ubuntu 20.04[digital ocean](https://m.do.co/c/d8f211a4b4c2)droplets 构建一个三节点 Kubernetes 集群。我们还将看看如何使用 Python 和 [Fabric](http://www.fabfile.org/) 来自动设置 Kubernetes 集群。

> 请随意将 DigitalOcean 更换为不同的云托管提供商或您自己的内部环境。

*依赖关系*:

1.  Docker v20.10
2.  Kubernetes v1.21 版

## 什么是面料？

[Fabric](https://www.fabfile.org/) 是一个 Python 库，用于自动化 SSH 上的例行 shell 命令，我们将使用它来自动化 Kubernetes 集群的设置。

安装:

```
`$ pip install fabric==2.6.0` 
```

验证版本:

```
`$ fab --version

Fabric 2.6.0
Paramiko 2.8.0
Invoke 1.6.0` 
```

通过将以下代码添加到名为 *fabfile.py* 的新文件中进行测试:

```
`from fabric import task

@task
def ping(ctx, output):
    """Sanity check"""
    print("pong!")
    print(f"hello {output}!")` 
```

尝试一下:

```
`$ fab ping --output="world"

pong!
hello world!` 
```

> 更多信息，请查看官方网站[文档](http://docs.fabfile.org/)。

## 水滴设置

首先，[在 DigitalOcean 上注册](https://m.do.co/c/d8f211a4b4c2)账户(如果你还没有的话)，[给你的账户添加](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)一个公共 SSH 密钥，然后[生成](https://www.digitalocean.com/docs/apis-clis/api/)一个访问令牌，这样你就可以访问 DigitalOcean API 了。

将令牌添加到您的环境中:

```
`$ export DIGITAL_OCEAN_ACCESS_TOKEN=<YOUR_DIGITAL_OCEAN_ACCESS_TOKEN>` 
```

接下来，为了以编程方式与 API 交互，安装 [python-digitalocean](https://github.com/koalalorenzo/python-digitalocean) 模块:

```
`$ pip install python-digitalocean==1.17.0` 
```

现在，让我们创建另一个任务来旋转三个 droplets:一个用于 Kubernetes master，两个用于 workers。更新 *fabfile.py* 这样:

```
`import os

from digitalocean import Droplet, Manager
from fabric import task

DIGITAL_OCEAN_ACCESS_TOKEN = os.getenv("DIGITAL_OCEAN_ACCESS_TOKEN")

# tasks

@task
def ping(ctx, output):
    """Sanity check"""
    print("pong!")
    print(f"hello {output}!")

@task
def create_droplets(ctx):
    """
 Create three new DigitalOcean droplets -
 node-1, node-2, node-3
 """
    manager = Manager(token=DIGITAL_OCEAN_ACCESS_TOKEN)
    keys = manager.get_all_sshkeys()
    for num in range(3):
        node = f"node-{num + 1}"
        droplet = Droplet(
            token=DIGITAL_OCEAN_ACCESS_TOKEN,
            name=node,
            region="nyc3",
            image="ubuntu-20-04-x64",
            size="s-2vcpu-4gb",
            tags=[node],
            ssh_keys=keys,
        )
        droplet.create()
        print(f"{node} has been created.")` 
```

注意传递给 Droplet 类的参数。本质上，这将在 NYC3 区域创建三个 Ubuntu 20.04 droplets，每个都有 4 GB 的内存。它还会给每个 droplet 添加 *[所有](https://github.com/koalalorenzo/python-digitalocean#creating-a-new-droplet-with-all-your-ssh-keys)* SSH 密钥。您可能希望更新它，以便只包含您专门为此项目创建的 SSH 密钥:

```
`@task
def create_droplets(ctx):
    """
 Create three new DigitalOcean droplets -
 node-1, node-2, node-3
 """
    manager = Manager(token=DIGITAL_OCEAN_ACCESS_TOKEN)
    # Get ALL SSH keys
    all_keys = manager.get_all_sshkeys()
    keys = []
    for key in all_keys:
        if key.name == "<ADD_YOUR_KEY_NAME_HERE>":
            keys.append(key)
    for num in range(3):
        node = f"node-{num + 1}"
        droplet = Droplet(
            token=DIGITAL_OCEAN_ACCESS_TOKEN,
            name=node,
            region="nyc3",
            image="ubuntu-20-04-x64",
            size="s-2vcpu-4gb",
            tags=[node],
            ssh_keys=keys,
        )
        droplet.create()
        print(f"{node} has been created.")` 
```

创造水滴:

```
`$ fab create-droplets

node-1 has been created.
node-2 has been created.
node-3 has been created.` 
```

接下来，让我们添加一个任务来检查每个 droplet 的状态，以确保在开始安装 Docker 和 Kubernetes 之前，每个 droplet 都已启动并准备就绪:

```
`@task
def wait_for_droplets(ctx):
    """Wait for each droplet to be ready and active"""
    for num in range(3):
        node = f"node-{num + 1}"
        while True:
            status = get_droplet_status(node)
            if status == "active":
                print(f"{node} is ready.")
                break
            else:
                print(f"{node} is not ready.")
                time.sleep(1)` 
```

添加`get_droplet_status`辅助函数:

```
`def get_droplet_status(node):
    """Given a droplet's tag name, return the status of the droplet"""
    manager = Manager(token=DIGITAL_OCEAN_ACCESS_TOKEN)
    droplet = manager.get_all_droplets(tag_name=node)
    return droplet[0].status` 
```

不要忘记重要的一点:

在我们测试之前，添加另一个任务来销毁液滴:

```
`@task
def destroy_droplets(ctx):
    """Destroy the droplets - node-1, node-2, node-3"""
    manager = Manager(token=DIGITAL_OCEAN_ACCESS_TOKEN)
    for num in range(3):
        node = f"node-{num + 1}"
        droplets = manager.get_all_droplets(tag_name=node)
        for droplet in droplets:
            droplet.destroy()
        print(f"{node} has been destroyed.")` 
```

摧毁我们刚刚创造的三个液滴:

```
`$ fab destroy-droplets

node-1 has been destroyed.
node-2 has been destroyed.
node-3 has been destroyed.` 
```

然后，调出三个新液滴，并验证它们是否可以运行:

```
`$ fab create-droplets

node-1 has been created.
node-2 has been created.
node-3 has been created.

$ fab wait-for-droplets

node-1 is not ready.
node-1 is not ready.
node-1 is not ready.
node-1 is not ready.
node-1 is not ready.
node-1 is not ready.
node-1 is ready.
node-2 is not ready.
node-2 is not ready.
node-2 is ready.
node-3 is ready.` 
```

## 供应机器

需要在每个微滴上运行以下任务...

### 设置地址

首先添加一个任务，在`hosts`环境变量中设置主机地址:

```
`@@task
def get_addresses(ctx, type):
    """Get IP address"""
    manager = Manager(token=DIGITAL_OCEAN_ACCESS_TOKEN)
    if type == "master":
        droplet = manager.get_all_droplets(tag_name="node-1")
        print(droplet[0].ip_address)
        hosts.append(droplet[0].ip_address)
    elif type == "workers":
        for num in range(2, 4):
            node = f"node-{num}"
            droplet = manager.get_all_droplets(tag_name=node)
            print(droplet[0].ip_address)
            hosts.append(droplet[0].ip_address)
    elif type == "all":
        for num in range(3):
            node = f"node-{num + 1}"
            droplet = manager.get_all_droplets(tag_name=node)
            print(droplet[0].ip_address)
            hosts.append(droplet[0].ip_address)
    else:
        print('The "type" should be either "master", "workers", or "all".')
    print(f"Host addresses - {hosts}")` 
```

在顶部定义以下变量，就在`DIGITAL_OCEAN_ACCESS_TOKEN = os.getenv('DIGITAL_OCEAN_ACCESS_TOKEN')`下面:

运行:

```
`$ fab get-addresses --type=all

165.227.96.238
134.122.8.106
134.122.8.204
Host addresses - ['165.227.96.238', '134.122.8.106', '134.122.8.204']` 
```

这样，我们就可以开始安装 Docker 和 Kubernetes 依赖项了。

### 安装依赖项

安装 Docker 和-

1.  kubeadm -引导一个 Kubernetes 集群
2.  配置容器在主机上运行
3.  用于管理集群的命令行工具

添加一个将 Docker 安装到 fabfile 的任务:

```
`@task
def install_docker(ctx):
    """Install Docker"""
    print(f"Installing Docker on {ctx.host}")
    ctx.sudo("apt-get update && apt-get install -qy docker.io")
    ctx.run("docker --version")
    ctx.sudo("systemctl enable docker.service")` 
```

让我们禁用交换文件:

```
`@task
def disable_selinux_swap(ctx):
    """
 Disable SELinux so kubernetes can communicate with other hosts
 Disable Swap https://github.com/kubernetes/kubernetes/issues/53533
 """
    ctx.sudo('sed -i "/ swap / s/^/#/" /etc/fstab')
    ctx.sudo("swapoff -a")` 
```

安装库:

```
`@task
def install_kubernetes(ctx):
    """Install Kubernetes"""
    print(f"Installing Kubernetes on {ctx.host}")
    ctx.sudo("apt-get update && apt-get install -y apt-transport-https")
    ctx.sudo(
        "curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -"
    )
    ctx.sudo(
        'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | \
 tee -a /etc/apt/sources.list.d/kubernetes.list && apt-get update'
    )
    ctx.sudo(
        "apt-get update && apt-get install -y kubelet=1.21.1-00 kubeadm=1.21.1-00 kubectl=1.21.1-00"
    )
    ctx.sudo("apt-mark hold kubelet kubeadm kubectl")` 
```

与其分别运行这些任务，不如创建一个主`provision_machines`任务:

```
`@task
def provision_machines(ctx):
    for conn in get_connections(hosts):
        install_docker(conn)
        disable_selinux_swap(conn)
        install_kubernetes(conn)` 
```

添加`get_connections`辅助函数:

```
`def get_connections(hosts):
    for host in hosts:
        yield Connection(
            f"{user}@{host}",
        )` 
```

更新导入:

```
`from fabric import Connection, task` 
```

运行:

```
`$ fab get-addresses --type=all provision-machines` 
```

安装所需的软件包需要几分钟时间。

## 配置主节点

初始化 Kubernetes 集群并部署[法兰绒](https://github.com/coreos/flannel)网络:

```
`@task
def configure_master(ctx):
    """
 Init Kubernetes
 Set up the Kubernetes Config
 Deploy flannel network to the cluster
 """
    ctx.sudo("kubeadm init")
    ctx.sudo("mkdir -p $HOME/.kube")
    ctx.sudo("cp -i /etc/kubernetes/admin.conf $HOME/.kube/config")
    ctx.sudo("chown $(id -u):$(id -g) $HOME/.kube/config")
    ctx.sudo(
        "kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
    )` 
```

保存加入令牌:

```
`@task
def get_join_key(ctx):
    sudo_command_res = ctx.sudo("kubeadm token create --print-join-command")
    token = re.findall("^kubeadm.*$", str(sudo_command_res), re.MULTILINE)[0]

    with open("join.txt", "w") as f:
        with stdout_redirected(f):
            print(token)` 
```

添加以下导入内容:

```
`import re
import sys
from contextlib import contextmanager` 
```

创建`stdout_redirected`上下文管理器:

```
`@contextmanager
def stdout_redirected(new_stdout):
    save_stdout = sys.stdout
    sys.stdout = new_stdout
    try:
        yield None
    finally:
        sys.stdout = save_stdout` 
```

再次添加一个父任务来运行这些任务:

```
`@task
def create_cluster(ctx):
    for conn in get_connections(hosts):
        configure_master(conn)
        get_join_key(conn)` 
```

运行它:

```
`$ fab get-addresses --type=master create-cluster` 
```

这将需要一两分钟来运行。一旦完成，join token 命令应该输出到屏幕并保存到一个 *join.txt* 文件:

```
`kubeadm join 165.227.96.238:6443 --token mvk32y.7z7i5x3viga4f4kn --discovery-token-ca-cert-hash sha256:f358dfc00ae7160fff3cb8fa3e3a3c8865f3c5b83c1f242fc9e51efe94108960` 
```

## 配置工作节点

使用上面保存的 join 命令，添加一个任务，将 workers“加入”到 master:

```
`@task
def configure_worker_node(ctx):
    """Join a worker to the cluster"""
    with open("join.txt") as f:
        join_command = f.readline()
        for conn in get_connections(hosts):
            conn.sudo(f"{join_command}")` 
```

在两个工作节点上运行此命令:

```
`$ fab get-addresses --type=workers configure-worker-node` 
```

## 健全性检查

最后，为了确保集群启动并运行，添加一个任务来查看节点:

```
`@task
def get_nodes(ctx):
    for conn in get_connections(hosts):
        conn.sudo("kubectl get nodes")` 
```

运行:

```
`$ fab get-addresses --type=master get-nodes` 
```

您应该会看到类似如下的内容:

```
`NAME     STATUS   ROLES                  AGE    VERSION
node-1   Ready    control-plane,master   3m6s   v1.21.1
node-2   Ready    <none>                 84s    v1.21.1
node-3   Ready    <none>                 77s    v1.21.1` 
```

完成后清除水滴:

```
`$ fab destroy-droplets

node-1 has been destroyed.
node-2 has been destroyed.
node-3 has been destroyed.` 
```

## 自动化脚本

最后一件事:添加一个 *create.sh* 脚本来自动化整个过程:

```
`#!/bin/bash

echo "Creating droplets..."
fab create-droplets
fab wait-for-droplets
sleep 20

echo "Provision the droplets..."
fab get-addresses --type=all provision-machines

echo "Configure the master..."
fab get-addresses --type=master create-cluster

echo "Configure the workers..."
fab get-addresses --type=workers configure-worker-node
sleep 20

echo "Running a sanity check..."
fab get-addresses --type=master get-nodes` 
```

尝试一下:

就是这样！

* * *

您可以在 GitHub 上的 [kubernetes-fabric](https://github.com/testdrivenio/kubernetes-fabric) repo 中找到这些脚本。