# Instalando Cluster Kubernetes com 3 VMs Vagrant

### Tenha o Vagrant instalado em Instalado em sua máquina e o VirtualBox

Caso não tenha:

- [https://developer.hashicorp.com/vagrant/install](https://developer.hashicorp.com/vagrant/install)
- [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)

Configuração Vagrant para subir 3 máquinas Ubuntu, implemente dentro de um arquivo Vagrantfile

```bash
Vagrant.configure("2") do |config|
    (1..1).each do |i|
        config.vm.define "master-#{i}" do |k8s|
            k8s.vm.box = "ubuntu/focal64"
            k8s.vm.hostname = "master-#{i}"
            #k8s.vm.network "private_network", ip: "172.89.0.1#{i}"
            k8s.vm.network "private_network", ip: "192.168.56.10#{i}"

            #k8s.vm.network "private_network", type: "static", ip: "192.168.56.11#{i}"

            k8s.ssh.insert_key = false
            k8s.ssh.private_key_path = ['~/.vagrant.d/insecure_private_key', '~/.ssh/id_rsa']
            k8s.vm.provision "file", source: "~/.ssh/id_rsa.pub", destination: "~/.ssh/authorized_keys"

            k8s.vm.provider "virtualbox" do |vb|
              vb.gui = false
              vb.cpus = 2
              vb.memory = "2048"
            end
        end
    end

    (1..2).each do |i|
        config.vm.define "worker-#{i}" do |k8s|
            k8s.vm.box = "ubuntu/focal64"
            k8s.vm.hostname = "worker-#{i}"
            #k8s.vm.network "private_network", ip: "172.89.0.2#{i}"
            k8s.vm.network "private_network", ip: "192.168.56.11#{i}"
            #k8s.vm.network "private_network", type: "static", ip: "192.168.56.12#{i}"

            k8s.ssh.insert_key = false
            k8s.ssh.private_key_path = ['~/.vagrant.d/insecure_private_key', '~/.ssh/id_rsa']
            k8s.vm.provision "file", source: "~/.ssh/id_rsa.pub", destination: "~/.ssh/authorized_keys"

            k8s.vm.provider "virtualbox" do |vb|
              vb.gui = false
              vb.cpus = 1
              vb.memory = "2048"
            end
        end
    end
end
```

Execute o seguinte comando dentro do diretório onde está localizado o arquivo Vagrantfile

```bash
vagrant up
```

Após a finalização, abra um terminal ou um tilix da vida (sudo apt install tilix) e crie 3 divisórias do terminal.

assim:

![Untitled](Instalando%20Cluster%20Kubernetes%20com%203%20VMs%20Vagrant%20db2286d693c64b89a0a21c4b2b07a85f/Untitled.png)

No terminal, vá até diretório onde se encontra o arquivo Vagrantfile, e execute o seguinte comando:

```bash
vagrant ssh <nome-da-maquina>

# Ex:
vagrant ssh master-1
vagrant ssh worker-1
vagrant ssh worker-2
```

![Untitled](Instalando%20Cluster%20Kubernetes%20com%203%20VMs%20Vagrant%20db2286d693c64b89a0a21c4b2b07a85f/Untitled%201.png)

![Untitled](Instalando%20Cluster%20Kubernetes%20com%203%20VMs%20Vagrant%20db2286d693c64b89a0a21c4b2b07a85f/Untitled%202.png)

atualize o seu sistema com os comandos abaixo e siga cada um dos passos descritos a seguir:

```bash
sudo apt update
sudo apt upgrade -y
```

### Passo 1: Desative o `swap` e adicione as configurações de kernel necessárias:

⚠️ **Esse passo deve ser realizado em todos os Nodes.**

```
#Desativando o swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Agora vamos carregar os módulos do kernel necessários em todos nos `Nodes`:

```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

Após carregar os módulos, vamos configurar parâmetros do Kernel para o Kubernetes:

```
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Para finalizar, execute o comando abaixo para que as modificações tenham efeito:

```bash
sudo sysctl --system
```

### Passo 2: Instalando e configurando o containerd runtime

⚠️ **Esse passo deve ser realizado em todos os Nodes.**

Execute o comando abaixo para instalar as dependências:

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

Ative o repositório do Docker:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Agora podemos instalar o **containerd**:

```bash
sudo apt update
sudo apt install -y containerd.io
```

Após a instalação, vamos configurar o **containerd** para que ele inicie usando o `systemd` como `cgroup`:

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

Para finalizar esse passo, reinicie e ative o serviço do **continerd**:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Passo 3: Adicionando o repositório apt do Kubernetes e instalando os componentes Kubectl, kubeadm e kubelet:

⚠️ **Esse passo deve ser realizado em todos os Nodes.**

Baixe a chave de assinatura pública para os repositórios de pacotes Kubernetes. A mesma chave de assinatura é usada para todos os repositórios, portanto você pode desconsiderar a versão na URL:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Adicione o repositório apt apropriado do Kubernetes. Observe que este repositório possui pacotes apenas para Kubernetes 1.28; para outras versões secundárias do Kubernetes, você precisa alterar a versão secundária do Kubernetes no URL para corresponder à versão secundária desejada (você também deve verificar se está lendo a documentação da versão do Kubernetes que planeja instalar).

```bash
# Isso substitui qualquer configuração existente em /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Atualize o apt:

```bash
sudo apt-get update
```

Agora podemos finalizar esse passo instalando os componentes:

```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Passo 4: Configurando o control plane e obtendo informações importantes

⚠️ **Esse passo deve ser realizado apenas no nó que será responsável pelo control plane.**

Para iniciar, entre no `Node` que será responsável pelo control plane (o principal `Node`) e execute o comando abaixo:

```bash
kubeadm init --apiserver-advertise-address <ip do no master> --pod-network-cidr 10.244.0.0/16

#ex 192.168.56.101
kubeadm init --apiserver-advertise-address 192.168.56.101 --pod-network-cidr 10.244.0.0/16

```

Após o kubeadm init for executado com sucesso aparecerá essas mensagens em seu terminal:

![Untitled](Instalando%20Cluster%20Kubernetes%20com%203%20VMs%20Vagrant%20db2286d693c64b89a0a21c4b2b07a85f/Untitled%203.png)

```bash
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Passo 5: Adicionando os nós *workers*

⚠️ **Esse passo deve ser realizado nos Nodes que vão ser adicionados ao cluster.**

Na última figura foi apresentada a saída do comando `kubeadm init` onde foi indicado onde está o comando para adicionar um novo nó. Esse comando deve ser executado nos `Nodes` que vão ser adicionados ao cluster e é semelhante ao apresentado abaixo:

```bash
kubeadm join 192.168.56.101:6443 --token 573rcw.7jlcjp87n7ocm8bh \
	--discovery-token-ca-cert-hash sha256:224ac2b8aca19f032ff8599204463426802b97c6deb73dfcacbea56c0589803b
```

Obs.: Será necessário executar o comando com permissão de administrador (sudo) no Node *worker* para adiciona-lo ao cluster!

Se tudo correu bem você verá uma mensagem de sucesso semelhante a apresentada abaixo:

![Untitled](Instalando%20Cluster%20Kubernetes%20com%203%20VMs%20Vagrant%20db2286d693c64b89a0a21c4b2b07a85f/Untitled%204.png)

Pronto, agora podemos verificar o STATUS dos `Nodes` adicionados com o comando abaixo:

```bash
kubectl get nodes
```

A saída esperada para 3 `Nodes` vai ser semelhante a apresentada abaixo:

```bash
NAME       STATUS     ROLES           AGE   VERSION
master-1   NotReady   control-plane   24m   v1.28.4
worker-1   NotReady   <none>          19m   v1.28.4
worker-2   NotReady   <none>          19m   v1.28.4
```

Analisando a saída, você provavelmente viu que tem algo errado, os `Nodes` do nosso cluster estão com o STATUS `NotReady`. Para que esses `Nodes` fiquem "prontos" ainda precisamos instalar e configurar um componente, a "rede" do Kubernetes.

### Passo 6: Flannel Pod Network Add-on:

⚠️ **Esse passo deve ser realizado apenas no nó que será responsável pelo control plane.**

Existem várias opções de **pod network**, mas nesse tutorial vamos instalar [Flannel Pod Network](https://github.com/flannel-io/flannel#deploying-flannel-manually)

Para instalar o Flannel execute os comandos abaixo no seu principal `Node`:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Após a instalação, verifique o status dos pods no namespace kube-system com o comando `kubectl get pods -n kube-flannel`. Observe se os pods do Flannel estão com o `STATUS` Running e também, o coredns no namespace do kube-system, pois não estava em running pois não tinha instalado Addons:

```bash
root@master-1:/home/vagrant# kubectl get po -n kube-flannel -w
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-58trb   1/1     Running   0          5m53s
kube-flannel-ds-b6c5r   1/1     Running   0          5m53s
kube-flannel-ds-t9bqn   1/1     Running   0          5m53s
```

```bash
root@master-1:/home/vagrant# kubectl get po -n kube-system -w
NAME                               READY   STATUS    RESTARTS        AGE
coredns-5dd5756b68-pvn6x           1/1     Running   0               45m
coredns-5dd5756b68-vq2j5           1/1     Running   0               45m
etcd-master-1                      1/1     Running   0               45m
kube-apiserver-master-1            1/1     Running   0               45m
kube-controller-manager-master-1   1/1     Running   2 (8m25s ago)   45m
kube-proxy-872nl                   1/1     Running   0               40m
kube-proxy-wxcc7                   1/1     Running   0               45m
kube-proxy-z2phk                   1/1     Running   0               39m
kube-scheduler-master-1            1/1     Running   2 (8m40s ago)   45m
```

Após checar os pods, aguarde uns segundos e verifique se os `Nodes` do seu cluster estão com o status Ready como apresentado abaixo:

```bash
root@master-1:/home/vagrant# kubectl get nodes
NAME       STATUS   ROLES           AGE    VERSION
master-1   Ready    control-plane   113m   v1.28.4
worker-1   Ready    <none>          108m   v1.28.4
worker-2   Ready    <none>          107m   v1.28.4
```

Se sua saída for semelhante a apresentada acima, está tudo certo, já pode começar a quebradeira!

**OBS!!!** Antes de instalar o Flannel o coredns se encontra em pending mostrado no exemplo abaixo, e execute o seguinte comando para vizualizar o coredns do cluster: 👇🏻

```bash
root@master-1:/home/vagrant# kubectl get po -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-5dd5756b68-pvn6x           0/1     Pending   0          8m31s
coredns-5dd5756b68-vq2j5           0/1     Pending   0          8m31s
etcd-master-1                      1/1     Running   0          8m41s
kube-apiserver-master-1            1/1     Running   0          8m41s
kube-controller-manager-master-1   1/1     Running   0          8m46s
kube-proxy-872nl                   1/1     Running   0          3m39s
kube-proxy-wxcc7                   1/1     Running   0          8m31s
kube-proxy-z2phk                   1/1     Running   0          3m20s
kube-scheduler-master-1            1/1     Running   0          8m41s
```

Para mais informações sobre os Addons você irá encontrar no link abaixo:

[Installing Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/)