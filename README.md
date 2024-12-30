# Guia de Instalação do Kubernetes com Kubeadm - Single Control Plane

Este guia tem como objetivo auxiliar na criação de um cluster Kubernetes utilizando o kubeadm em uma arquitetura Single Control Plane (sem alta disponibilidade). Serão abordados desde os requisitos até a configuração do ambiente e os passos para inicialização e configuração do cluster.

Esta abordagem é ideal para ambientes de desenvolvimento, aprendizado ou testes, onde a simplicidade é a prioridade. Porém, é importante entender as limitações desse modelo para tomar decisões informadas ao migrar para ambientes de produção.

Aqui, vamos utilizar o kubeadm. O kubeadm é uma ferramenta projetada para simplificar a configuração do Kubernetes. Ele fornece comandos padronizados para inicializar e configurar clusters, reduzindo significativamente o esforço necessário para colocar um cluster em funcionamento. Este guia detalha cada etapa, desde a preparação do ambiente até a execução de uma aplicação no cluster, fornecendo também explicações sobre os conceitos fundamentais para garantir um aprendizado sólido.

---

## Requisitos

### Requisitos de Hardware

- 3 Máquinas Linux (aqui no caso vou utilizar Ubuntu 22.04)
- 2 GB de memória RAM
- 2 CPUs
- Conexão de rede entre as máquinas
- Hostname, endereço MAC e `product_uuid` únicos para cada nó
- Swap desabilitado
- Acesso SSH a todas as máquinas

### Requisitos de Rede

#### Control Plane
O control plane requer a liberação das seguintes portas:

| Protocol | Direction | Port Range   | Purpose                     | Used By               |
|----------|-----------|--------------|-----------------------------|-----------------------|
| TCP      | Inbound   | 6443         | Kubernetes API server       | All                   |
| TCP      | Inbound   | 2379-2380    | etcd server client API      | kube-apiserver, etcd  |
| TCP      | Inbound   | 10250        | Kubelet API                 | Self, Control plane   |
| TCP      | Inbound   | 10259        | kube-scheduler              | Self                  |
| TCP      | Inbound   | 10257        | kube-controller-manager     | Self                  |

#### Worker Nodes
Os worker nodes requerem a liberação das seguintes portas:

| Protocol | Direction | Port Range   | Purpose                     | Used By             |
|----------|-----------|--------------|-----------------------------|---------------------|
| TCP      | Inbound   | 10250        | Kubelet API                 | Self, Control plane |
| TCP      | Inbound   | 10256        | kube-proxy                  | Self, Load balancers |
| TCP      | Inbound   | 30000-32767  | NodePort Services†          | All                 |

---

## Passos da Instalação

### 1. Instalação do Container Runtime (ContainerD)

Para a instalação e criação do cluster Kubernetes, vamos executar as seguintes etapas:
- Instalação do Container Runtime (ContainerD)
- Instalação do kubeadm, kubelet e kubectl
- Inicialização do cluster Kubernetes
- Instalação do CNI cálico
- Incluir os worker nodes no cluster Kubernetes

O Container Runtime é a base para executar contêineres no Kubernetes. O ContainerD é uma escolha popular devido à sua simplicidade, desempenho e suporte oficial do Kubernetes.

**OBS**: Esta etapa deve ser executada em todas as máquinas que farão parte do cluster Kubernetes.

#### Instalação dos Módulos do Kernel do Linux

Para seguir com a instalação, primeiro é preciso habilitar 2 módulos no kernel do Linux:

- overlay ⇒ Usado pra unir camadas de file system (falo isso no curso de Docker)
- br_netfilter ⇒ Modulo de rede usado pra garantir a comunicação dos containers e do Kubernetes

Comandos para habilitar:

Habilitar os módulos `overlay` e `br_netfilter`:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
### Ajustes de definições do Kernel:
```bash
# Configuração dos parâmetros do sysctl, fica mantido mesmo com reebot da máquina.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Aplica as definições do sysctl sem reiniciar a máquina
sudo sysctl --system
```

net.bridge.bridge-nf-call-iptables = 1 ⇒ Ativo as redes bridges pra passarem pelo iptables e assim as regras de firewall vão passar por essas redes também.

net.ipv4.ip_forward = 1 ⇒ Habilita o encaminhamento de pacotes IPv4 no sistema. Isso é essencial para que o host funcione como um roteador, encaminhando pacotes de rede de uma interface para outra.

net.bridge.bridge-nf-call-ip6tables = 1 ⇒ Similar ao bridge-nf-call-iptables, mas para tráfego IPv6.

#### Instalação do ContainerD

OBS: A partir da versão 1.26 do Kubernetes, foi removido o suporte ao CRI v1alpha2 e ao Containerd 1.5. E até o momento que escrevo esse guia, o repositório oficial do Ubuntu não tem o Containerd 1.6, então precisamos usar o repositório do Docker pra instalar o ContainerD.
Kubernetes v1.26: Electrifying: https://kubernetes.io/blog/2022/12/09/kubernetes-v1-26-release/#cri-v1alpha2-removed

```bash
# Instalação de pré requisitos
sudo apt-get update -y
sudo apt-get install ca-certificates curl gnupg --yes
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Configurando o repositório
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt update -y && sudo apt install containerd.io -y
```

#### Configuração padrão do Containerd
```bash
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml
```
Alterar o arquivo de configuração pra configurar o systemd cgroup driver.
Sem isso o Containerd não gerencia corretamente os recursos computacionais e vai reiniciar em loop
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

#Alterar a imagem do sandbox
sudo sed -i 's|sandbox_image = ".*"|sandbox_image = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml
```
Agora é preciso reiniciar o containerd
```bash
sudo systemctl restart containerd
```

#### Instalação do kubeadm, kubelet and kubectl

Com o containerd instalado, agora é preciso instalar o kubeadm, kubelet e o kubectl.
Instalação dos pacotes necessários
```bash
sudo apt-get update && \\
sudo apt-get install -y apt-transport-https ca-certificates curl
```
Download da chave pública do Repositório do Kubernetes

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Adicionando o repositório apt do Kubernetes

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Atualização do repositório apt e instalação das ferramentas

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
#### Iniciando o Cluster Kubernetes
OBS: Esse comando precisa ser executado apenas no control plane
Inicialização do cluster Kubernetes

```bash
sudo kubeadm init \
    --apiserver-advertise-address=10.0.0.4 \ # Endereço que o API Server vai utilizar
    --apiserver-cert-extra-sans=52.136.116.85 \ # Endereços alternativos pra adicionar no certificado e garantir o acesso ao API Server ( pode ser ip ou domínio )
    --pod-network-cidr=192.168.0.0/16 # Especifica um intervalo de endereços IP para a rede do Pod. Quando especificado, a camada de gerenciamento irá automaticamente alocar CIDRs para cada nó.**
```
Parâmetros:

--apiserver-advertise-address => Endereço que o API Server vai utilizar

--apiserver-cert-extra-sans => Endereços alternativos pra adicionar no certificado e garantir o acesso ao API Server (pode ser ip ou domínio )

--pod-network-cidr => Especifica um intervalo de endereços IP para a rede do Pod. Quando especificado, a camada de gerenciamento irá automaticamente alocar CIDRs para cada nó.

Fases do Kubeadm init:

Durante o processo de inicialização do cluster, alguns estágios são executados:

preflight ⇒ Valida o sistema e verifica se é possível fazer a instalação. Ele pode exibir alertas ou erros, no caso de erro, ele sai da inicialização.

kubelet-start ⇒ Essa fase escreve o arquivo de configuração do kubelet e o arquivo de ambiente e, em seguida, iniciará o kubelet.

certs ⇒ Gera uma autoridade de certificação auto assinada para cada componente do Kubernetes. Isso garante a segurança na comunicação com o cluster.

kubeconfig ⇒ Gera o kubeconfig no diretório /etc/kubernetes e os arquivos utilizados pelo kubelet, controller-manager e o scheduler pra conectar no api-server.

kubelet-start, control-plane e etcd ⇒ Configura o kubelet pra executar os pods com o api-server, controller-manager, o scheduller e o etcd. Depois inicia o kubelet.

mark-control-plane ⇒ Aplica labels e tains no control plane pra garantir que não vai ser executado nenhum pod dentro dele.

bootstrap-token ⇒ Gera tokens de bootstrap usados para adicionar um nó a um cluster.

kubelet-finalize ⇒ Use a seguinte fase para atualizar as configurações relevantes para o kubelet após o bootstrap TLS.

addons ⇒ Adiciona o CoreDNS e o kube-proxy

Configuração do kubectl
Com o Kubernetes iniciado, agora é preciso configurar o kubectl para se comunicar com o Api Server:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Joins dos Worker Nodes

Agora, o cluster Kubernetes tem apenas o control plane fazendo parte dele, então você deve executam o comando de join nos worker nodes para incluir eles no cluster. Mas antes, você precisa do comando. Então execute o comando abaixo no control plane e o comando de resultado execute nos worker nodes:

```bash
kubeadm token create --print-join-command
```

#### Instalação do CNI
Agora, é preciso adicionar o Calico para gerenciar a rede dos pods e dos containers:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml
```
Rodando a aplicação
Teste o cluster Kubernetes usando o manifesto abaixo:
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

Referencias para o material:

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/reference/networking/ports-and-protocols/

https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/

