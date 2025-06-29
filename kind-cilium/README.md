# Usando o KIND com Cilium

Saudações! Caso você esteja querendo começar nos estudos de segurança, envolvendo um cluster kubernetes, aqui vai um breve tutorial de como iniciar esses estudos.

O objetivo desse guia é construir um cluster, usando uma ferramenta indicada pela galera mantenedora do projeto do Kubernetes (CNCF). Trata-se da ferramenta [KIND](https://kind.sigs.k8s.io/) (Kubernetes IN Docker).

Em adição, para ser permitido o estudo sobre as *NetworkPolicies*, vamos abordar a instalação do [Cilium](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/) como plugin _Container Network Interface (CNI)_.

E pra finalizar, vamos realizar um laboratório simples, mostrando o funcionamento das ferramentas! :D

Antes de começarmos, é necessário que o docker esteja instalado na máquina. Outros ContainerRuntimes não foram validados. ¯\\\_(ツ)_/¯


## 1 - Passo: Instalação do KIND

A documentação da ferramenta já é muito boa, vou deixar [aqui o link](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries) pra facilitar a navegação.

Dos métodos de instalação disponíveis, o usado para esse guia foi o de download do binário:

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Obs.: Algumas distros linux não usam o ```/usr/local/bin``` na variável ```${PATH}``` padrão. Para checar, basta executar o comando abaixo:

```bash
echo ${PATH}
```

Caso não esteja, basta adicionar a seguinte linha no arquivo ```~/.bashrc```

```bash
(...)
export PATH="$PATH:/usr/local/bin"
(...)
```

E carregar as configurações com o comando:

```bash
source ~/.bashrc
```

## 2 - Criação do cluster

Com o binário baixado, agora é necessária a criação do cluster, para esse exemplo que não é padrão duas coisas precisam ser levadas em consideração:

#### I - A instalação não usará o CNI padrão, então, é necessário desabilitá-lo.


#### II - O Cilium precisa de algumas "permissões" dentro do sistema operacional que o usuário comum não tem, sendo necessária a execução como root nos passos a seguir. 

Obs.: Caso queira contribuir com como fazer essa configuração sem usar o usuário root, fique a vontade ;)

### 2.1 - Criando o arquivo de configuração

Criar, com o editor de texto de preferência, o arquivo ```kind-config.yaml``` com o seguinte conteúdo:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster1
nodes:
- role: control-plane
  extraPortMappings: # Portas a serem expostas do cluster
  - containerPort: 30001
    hostPort: 30001
  - containerPort: 30002
    hostPort: 30002
- role: worker
- role: worker
- role: worker
networking:
  disableDefaultCNI: true # Desativação do CNI padrão
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.245.0.0/16"
```

Obs.: A configuração acima usa um control-plane e três workers, caso julgue interessante, pode reduzir a quantidade de workers para menor uso dos recursos.

### 2.2 - Criando o cluster

Após a criação do arquivo de configuração com as definições do cluster é necessário aplicar para subir os containers que executarão o cluster.

O comando para criação, usando o arquivo de configuração, é o seguinte:

```bash
kind create cluster --config=kind-config.yaml
```

Depois da criação, ao executar o ```docker ps```, os containers serão mostrados como abaixo:

```bash
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                                                             NAMES
74e6730e90ca   kindest/node:v1.33.1   "/usr/local/bin/entr…"   2 hours ago   Up 2 hours   0.0.0.0:30001-30002->30001-30002/tcp, 127.0.0.1:36843->6443/tcp   cluster1-control-plane
81adc969c9c9   kindest/node:v1.33.1   "/usr/local/bin/entr…"   2 hours ago   Up 2 hours                                                                     cluster1-worker2
d7f5c7b43a6d   kindest/node:v1.33.1   "/usr/local/bin/entr…"   2 hours ago   Up 2 hours                                                                     cluster1-worker
c54aa0d50534   kindest/node:v1.33.1   "/usr/local/bin/entr…"   2 hours ago   Up 2 hours                                                                     cluster1-worker3
```

### 2.3 - Configurando o kubectl

Para manuseio dos objetos dentro desse cluster, é necessária a instalação do binário kubectl. 

Existem repositórios de distros RHEL e Debian like e opção do uso direto do binário. Para ambientes produtivos, é recomendado o uso do gerenciador de pacotes das distros, através dos repositórios. Mas para esse guia, será abordado o uso do binário simples. O guia oficial pode ser lido através desse [link](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).


```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv kubectl /usr/local/bin/
```

Após a instalação, ao executar o comando ```kubectl get pods -A```, que lista todos os *pods* de todos os *namespaces*, será observado o seguinte resultado:

```bash
NAMESPACE            NAME                                             READY   STATUS    RESTARTS   AGE
kube-system          coredns-674b8bbfcf-4kmgg                         0/1     Pending   0          12s
kube-system          coredns-674b8bbfcf-pv9gf                         0/1     Pending   0          12s
kube-system          etcd-cluster1-control-plane                      1/1     Running   0          18s
kube-system          kube-apiserver-cluster1-control-plane            1/1     Running   0          18s
kube-system          kube-controller-manager-cluster1-control-plane   1/1     Running   0          18s
kube-system          kube-proxy-dl2ml                                 1/1     Running   0          10s
kube-system          kube-proxy-qq4vf                                 1/1     Running   0          10s
kube-system          kube-proxy-r9jv2                                 1/1     Running   0          12s
kube-system          kube-proxy-sw24m                                 1/1     Running   0          10s
kube-system          kube-scheduler-cluster1-control-plane            1/1     Running   0          19s
local-path-storage   local-path-provisioner-7dc846544d-7vxzw          0/1     Pending   0          12s
``` 

Evidenciando que alguns pods estão em STATUS **Pending**, o que será tratado a seguir.

## 3 - Configurando o Cilium

A "pendência" do passo anterior se dá pela não existência, ainda, da interface de rede local do cluster, que é responsável pela comunicação entre os objetos.

Para esse questão, basta realizar a instalação do cilium que, como em passos anteriores, possui mais de um método para instalação, referenciado no começo desse guia.

Nesse laboratório que será construído, será adotada a instalação através da CLI, que usa o [HELM](https://helm.sh/) em backgroud para a tarefa. Seguem os comandos para instalação da CLI:

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
Com a CLI instalada, agora é necessário instalar o *CNI* no cluster:

```bash
cilium install --version=$(curl -s https://raw.githubusercontent.com/cilium/cilium/refs/heads/main/stable.txt)
```

Depois dessa execução, os status dos pods mudarão, alem da criação de pods adicionais para o funcionamento do *CNI*.

```
NAMESPACE            NAME                                             READY   STATUS              RESTARTS   AGE
kube-system          cilium-5lppt                                     0/1     Init:0/6            0          9s
kube-system          cilium-envoy-58z4s                               0/1     ContainerCreating   0          9s
kube-system          cilium-envoy-8t72t                               0/1     ContainerCreating   0          9s
kube-system          cilium-envoy-rc8wn                               0/1     ContainerCreating   0          9s
kube-system          cilium-envoy-tht98                               0/1     ContainerCreating   0          9s
kube-system          cilium-g5rmb                                     0/1     Init:0/6            0          9s
kube-system          cilium-j6d5d                                     0/1     Init:0/6            0          9s
kube-system          cilium-lxnzl                                     0/1     Init:0/6            0          9s
kube-system          cilium-operator-686959b66d-fjj45                 0/1     ContainerCreating   0          9s
kube-system          coredns-674b8bbfcf-4kmgg                         0/1     Pending             0          13m
kube-system          coredns-674b8bbfcf-pv9gf                         0/1     Pending             0          13m
kube-system          etcd-cluster1-control-plane                      1/1     Running             0          13m
kube-system          kube-apiserver-cluster1-control-plane            1/1     Running             0          13m
kube-system          kube-controller-manager-cluster1-control-plane   1/1     Running             0          13m
kube-system          kube-proxy-dl2ml                                 1/1     Running             0          13m
kube-system          kube-proxy-qq4vf                                 1/1     Running             0          13m
kube-system          kube-proxy-r9jv2                                 1/1     Running             0          13m
kube-system          kube-proxy-sw24m                                 1/1     Running             0          13m
kube-system          kube-scheduler-cluster1-control-plane            1/1     Running             0          13m
local-path-storage   local-path-provisioner-7dc846544d-7vxzw          0/1     Pending             0          13m
```
Nos testes efetuados, por volta de sete minutos os pods mudaram o STATUS de **Pending** para **Running**:
```
NAMESPACE            NAME                                             READY   STATUS    RESTARTS   AGE
kube-system          cilium-5lppt                                     1/1     Running   0          8m14s
kube-system          cilium-envoy-58z4s                               1/1     Running   0          8m14s
kube-system          cilium-envoy-8t72t                               1/1     Running   0          8m14s
kube-system          cilium-envoy-rc8wn                               1/1     Running   0          8m14s
kube-system          cilium-envoy-tht98                               1/1     Running   0          8m14s
kube-system          cilium-g5rmb                                     1/1     Running   0          8m14s
kube-system          cilium-j6d5d                                     1/1     Running   0          8m14s
kube-system          cilium-lxnzl                                     1/1     Running   0          8m14s
kube-system          cilium-operator-686959b66d-fjj45                 1/1     Running   0          8m14s
kube-system          coredns-674b8bbfcf-4kmgg                         1/1     Running   0          21m
kube-system          coredns-674b8bbfcf-pv9gf                         1/1     Running   0          21m
kube-system          etcd-cluster1-control-plane                      1/1     Running   0          21m
kube-system          kube-apiserver-cluster1-control-plane            1/1     Running   0          21m
kube-system          kube-controller-manager-cluster1-control-plane   1/1     Running   0          21m
kube-system          kube-proxy-dl2ml                                 1/1     Running   0          21m
kube-system          kube-proxy-qq4vf                                 1/1     Running   0          21m
kube-system          kube-proxy-r9jv2                                 1/1     Running   0          21m
kube-system          kube-proxy-sw24m                                 1/1     Running   0          21m
kube-system          kube-scheduler-cluster1-control-plane            1/1     Running   0          21m
local-path-storage   local-path-provisioner-7dc846544d-7vxzw          1/1     Running   0          21m
```

## 4 - Bonus Track (Completions)

Nesse guia foram abordados 3 binários principais:

- kind
- kubectl
- cilium

Para adicionar o _bash completion_, ou o completion de outro shell de preferência, basta adicionar as linhas abaixo dentro do arquivo ~/.bashrc.

```bash
source <(kind completion bash)
source <(kubectl completion bash)
source <(cilium completion bash)
```

E depois recarregar com o comando ```source ~/.bashrc``` :)

## 5 - Laboratório simples

Já com o ambiente configurado, será feita a criação de alguns objetos para validação e criação de políticas de rede (*NetworkPolicies*) internas do cluster.

Por padrão, **todos** os containers de **todos** os namespaces se comunicam, contudo, e se fosse necessário limitar o acesso do *container A* a apenas membros do *namespace A*?

O cenário para o experimento será construído da seguinte forma:

| Nome | Namespace | Serviço 
-----|------------|----------
| web-server | ns-a | 80/TCP
| debuga | ns-a | x
| debugb | ns-a | x

O pod **debug1** realizará um acesso ao serviço HTTP do pod **web-server** no namespace **ns-a**.

Para o cenário, serão necessários alguns passos, descritos a seguir:

### 5.1 - Criação dos namespaces

No cluster previamente criado, é necessária a criação dos namespaces, através dos comandos:

```bash
kubectl create namespace ns-a 
kubectl create namespace ns-b
```

### 5.2 - Criação dos pods

Após a criação dos namespaces, é necessário criar os pods para validação do ambiente. Esses pods serão criados por meio de deployments, descritos abaixo (**Importante salvá-los no mesmo diretório, para facilitar na aplicação mais a frente**):

- **WEB-SERVER**

```web-server.yaml```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-server
    lab: teste
  name: web-server
  namespace: ns-a
spec:
  selector:
    matchLabels:
      app: web-server
      lab: teste
  template:
    metadata:
      labels:
        app: web-server
        lab: teste
    spec:
      containers:
      - image: httpd
        name: httpd
```
 - **DEBUGA**

```debuga.yaml```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: debuga
    lab: teste
  name: debuga
  namespace: ns-a
spec:
  selector:
    matchLabels:
      app: debuga
      lab: teste
  template:
    metadata:
      labels:
        app: debuga
        lab: teste
    spec:
      containers:
      - command:
        - sleep
        - "200000"
        image: debian
        name: debian
```

- **DEBUGB**

```debugb.yaml```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: debugb
    lab: teste
  name: debugb
  namespace: ns-b
spec:
  selector:
    matchLabels:
      app: debugb
      lab: teste
  template:
    metadata:
      labels:
        app: debugb
        lab: teste
    spec:
      containers:
      - command:
        - sleep
        - "200000"
        image: debian
        name: debian
```

Em seguida, criar os objetos através do comando 

```bash
kubectl create -f .
```

### 5.3 - Testando as conexões antes da política de rede

Para validação da conexão, esperamos duas condições:

- Namespace do serviço: ✅
- Qualquer namespace: ✅

Antes de acessar qualquer pod, é importante checar os endereços de IP de cada pod, para validar as conexões. 

```bash
kubectl get pods -A -l lab=teste -o wide
```

A saída será algo similar ao abaixo:

```bash
NAMESPACE   NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE               NOMINATED NODE   READINESS GATES
ns-a        debuga-c8c59775f-5zvgs        1/1     Running   0          12m   10.244.2.139   cluster1-worker3   <none>           <none>
ns-a        web-server-59cc86b959-lchmd   1/1     Running   0          12m   10.244.3.20    cluster1-worker    <none>           <none>
ns-b        debugb-58f6779fdb-zldw2       1/1     Running   0          12m   10.244.1.231   cluster1-worker2   <none>           <none>
```

Como se pode observar, nesse exemplo, o endereço de IP do pod web-server é ```10.244.3.20```

Acessar os pods de debug, através do comando ```kubectl exec```, que permite, assim como o Docker, execução de um comando dentro do container:

- DEBUGA
```bash
kubectl exec -it -n ns-a deployments/debuga -- bash
```
- DEBUGB
```bash
kubectl exec -it -n ns-b deployments/debugb -- bash
```

Uma vez acessando o terminal do container, executar os seguintes comandos:

```bash
apt update &&
apt install curl -y && \
curl 10.244.3.20
```
Caso o comando traga um retorno similar ao abaixo, a conexão se deu de forma esperada, para ambos os pods.
```html
<html><body><h1>It works!</h1></body></html>
```

### 5.4 - Aplicando a política de rede

Como visto no passo anterior, ambos os namespaces tem acesso ao pode que está executando o webserver. 

Agora será aplicada uma política de rede que permitirá apenas acesso do mesmo namespace da aplicação, ```ns-a```.

Criar o arquivo com o seguinte conteúdo:

```np.yaml```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-http
  namespace: ns-a
spec:
  podSelector:
    matchLabels:
      app: web-server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ns-a
    ports:
    - protocol: TCP
      port: 80
```

Em seguida, criar a política:

```bash
kubectl create -f np.yaml
```

### 5.5 - Testando as conexões após a política de rede

Da mesma forma que a validação anterior, esperamos duas condições:

- Namespace do serviço: ✅
- Qualquer namespace: 🔴

Como os pacotes ja foram instalados nos pods no passo 5.3, o comando para validação se dará de forma mais simples (**lembrar de alterar o endereço de IP do web-server para seu ambiente**):

- DEBUGA

```bash
kubectl exec -it -n ns-a deployments/debuga -- curl 10.244.3.20
```

- DEBUGB

```bash
kubectl exec -it -n ns-b deployments/debugb -- curl 10.244.3.20
```

Com a execução dos comandos, o ```debuga``` deve trazer o html, já o ```debugb``` deve apresentar timeout na requisição. 

```bash

root@server:/# kubectl exec -it -n ns-a deployments/debuga -- curl 10.244.3.20
<html><body><h1>It works!</h1></body></html>

root@server:/# kubectl exec -it -n ns-b deployments/debugb -- curl 10.244.3.20
curl: (28) Failed to connect to 10.244.3.20 port 80 after 130339 ms: Couldn't connect to server
command terminated with exit code 28
```

## 6 - Removendo o cluster

Caso deseje encerrar o ambiente e liberar os recursos utilizados pelos containers do KIND, é possível deletar todo o cluster com um único comando.

Se você usou o nome ```cluster1``` no seu arquivo de configuração (como no exemplo deste guia), execute:

```bash
kind delete cluster --name cluster1
```

Isso irá:

- Remover todos os containers relacionados ao cluster KIND.

- Excluir o contexto do cluster do kubectl config.

- Liberar as portas mapeadas no host.

- Evitar consumo desnecessário de recursos caso você não esteja mais utilizando o ambiente.

- Caso queira verificar se ainda há clusters KIND existentes antes de deletar, use:

```bash
kind get clusters
```

💡 Dica: Sempre remova o cluster quando terminar seus testes, principalmente em máquinas com recursos limitados ou ambientes compartilhados.

Bem, por esse guia é isso, espero que ajude! <o