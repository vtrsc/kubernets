# KSA
Kubernetes Santo André

Repositório para documentação da implementação do Kubernetes Santo André

### Pré Requisitos:
4 Máquinas disponíveis


# Rancher HA
Instanciar 4 máquinas:
1- Load Balancer (nginx)
3- Rancher Server

## Load Balancer
### 1. Kubectl
Acessar a máquina do Load Balancer e instalar o Kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

![image](https://user-images.githubusercontent.com/24412660/140569482-0a7e4997-1658-47e8-82b6-b4b993e3a919.png)

### 2. RKE1
Instale o RKE1

```
curl -LO https://github.com/rancher/rke/releases/download/v1.3.2/rke_linux-amd64
mv rke_linux-amd64 rke
chmod +x rke
mv ./rke /usr/local/bin/rke
rke --version
```

![image](https://user-images.githubusercontent.com/24412660/140755699-de44e2d2-1cda-439f-8015-2b38820e9ce8.png)


### 3. Preparar Cluster
Para o cluster funcionar será necessário que seja configurado o acesso via chaves ssh da máquina Load Balancer aos outros 3 servidores Rancher. Isso se dá por conta que nessa arquitetura, quem envia as instruções de configuração e instanciamento é o próprio Load Balancer via SSH a partir dos comandos do RKE onde, por segurança, o RKE exige que a autenticação seja feita via chaves assimétricas. Então caso não tenha configurado ainda o acesso via chave assimétrica, verifique as instruções na seção extra dessa documentação clicando aqui: [Extra - Gerar Chave Privada para o servidor Load Balancer](#gerar-chave-privada-para-o-servidor-load-balancer) .


Crie um arquivo chamado cluster.yml na pasta raiz do diretório linux seguindo o template abaixo. Subistitua os addresses pelos IPs referentes as 3 máquinas que estarao rodando o rancher server, e o user pelo usuário do sistema operacional.

```
nodes:
  - address: 172.30.72.233
    user: root
    role:
    - controlplane
    - worker
    - etcd 
  - address: 172.30.72.234
    user: root
    role: 
    - controlplane
    - worker
    - etcd  
  - address: 172.30.72.235
    user: root
    role: 
    - controlplane
    - worker
    - etcd 

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h

# Required for external TLS termination with
# ingress-nginx v0.22+
ingress:
  provider: nginx
  options:
    use-forwarded-headers: "true"

```
**Atenção**: *Por padrão, a chave privada a ser utilizada será a que foi armazenada no endereço ~/.ssh/id_rsa através do passo anterior* **Gerar Chave Privada para o Servidor Load Balancer.** *Para outras opções de configuração acesse https://rancher.com/docs/rke/latest/en/config-options/nodes/*

Execute então o comando e aguarde alguns minutos até o cluster subir completamente.
```
rke up
```
![image](https://user-images.githubusercontent.com/24412660/140785523-4bb975ca-3cad-4285-9b73-6aa540a21969.png)

Após a execução do rke up, será criado alguns arquivos na pasta raiz. O arquivo kube_config_cluster.yml é onde contem as informações para poder habilitar o kubectl. Para poder habilitar o kubectl dentro do servidor load balancer, execute o comando export e verifique os nós conectados com o kubectl:

```
export KUBECONFIG=$(pwd)/kube_config_cluster.yml
kubectl get nodes
kubectl get pods --all-namespaces
```

![image](https://user-images.githubusercontent.com/24412660/140787217-fc51910d-7b65-4dbe-afc4-9696e91fdf11.png)

#### Instalação do Helm
O Helm é um gerenciador e instalador de pacotes semelhante ao apt-get, yum e homebrew, com a diferença que ele é voltado para aplicações que irão rodar dentro do kubernetes.

Vamos então instalar o helm e verificar a instalação:
```
curl -LO https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
tar -zxvf helm-v3.7.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm help
```
![image](https://user-images.githubusercontent.com/24412660/140789551-93f8e14c-d7f5-40e7-826d-aac0be5a33c6.png)

Iremos adicionar o repositório das aplicações do rancher e criaremos um namespace chamado cattle-system onde teremos algumas extensões do rancher rodando.

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
kubectl create namespace cattle-system
```

![image](https://user-images.githubusercontent.com/24412660/140790250-5e55bf42-5984-4f64-aecc-273185a507ef.png)

Agora adicione o repositório e namespace para o gerenciador de certificados:

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
```
![image](https://user-images.githubusercontent.com/24412660/140790642-dcdbbcba-2bba-4256-a595-ab8e6f08d8bd.png)

Entao atualize o repositório, instale o gerenciador de certificados e verifique se os pod estão rodando:
```
helm repo update
## old version: helm install  cert-manager jetstack/cert-manager --namespace cert-manager --version v0.15.0
helm install  cert-manager jetstack/cert-manager --namespace cert-manager --version v1.6.1
kubectl get pods --namespace cert-manager
```
![image](https://user-images.githubusercontent.com/24412660/140791066-85bc00dc-b8d9-4848-ab7e-5db233d16a14.png)


### Instalação do Rancher
Finalmente vamos realizar a instalação do Rancher via Helm. Para isso, basta executar o helm install, substituindo o **hostname** pelo correto e verificar se o deployment estao em execucao
```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.santoandre.sp.gov.br
  
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get deploy rancher
```

![image](https://user-images.githubusercontent.com/24412660/140795255-b9d7bfc4-81b5-4364-b10e-c6a2641231ec.png)


### Configuração do NGINX
Crie um arquivo chamado ``nginx.conf`` na pasta ``/etc/``, e inclua as informacoes abaixo adicionando os IPs corretos dos 3 servidores:

```
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream rancher_servers_http {
        least_conn;
        server 172.30.72.233:80 max_fails=3 fail_timeout=5s;
        server 172.30.72.234:80 max_fails=3 fail_timeout=5s;
        server 172.30.72.235:80 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 80;
        proxy_pass rancher_servers_http;
    }

    upstream rancher_servers_https {
        least_conn;
        server 172.30.72.233:443 max_fails=3 fail_timeout=5s;
        server 172.30.72.234:443 max_fails=3 fail_timeout=5s;
        server 172.30.72.235:443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     443;
        proxy_pass rancher_servers_https;
    }

}

```

Coloque o container nginx em execucao através do docker:
```
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  nginx:1.14
```
![image](https://user-images.githubusercontent.com/24412660/140795344-d9368d91-f589-4a2f-8b80-a3c7ae4d4e03.png)

Após esse passo o seu servidor rancher já estará pronto e disponível para acesso via navegador. Agora vamos apenas configurar o primeiro acesso com a conta de administrador. Para isso será necessário obter o *bootstrap password* executando o comando abaixo na máquina do NGINX e copiando o resultado:

```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

Agora acesse o link https://rancher.santoandre.sp.gov.br/ e cole o bootstrap password para poder realizar o primeiro acesso.

![image](https://user-images.githubusercontent.com/24412660/140824240-77cba1b7-89ce-4ce8-b951-9f5342f4cf19.png)

Por fim, crie uma senha randomica ou especifique uma senha para ser utilizada como administrador:

![image](https://user-images.githubusercontent.com/24412660/140824345-d64a7b09-2320-4a9a-9afa-86ffa3fc795c.png)

Parabéns! Você finalizou a instalação do Rancher Server em Alta Disponibilidade. 
Prossiga aos próximos passos para criar o cluster de desenvolvimento/produção.

![image](https://user-images.githubusercontent.com/24412660/140824653-66d91e3c-f1b6-4d15-a1d4-1240aad5890e.png)




# Kubernetes HA
~~Vamos criar o cluster de homologação, para isso utilizaremos a seguinte arquitetura:~~
 ~~- 3 Control Plane + ETCD~~
 ~~- 4 Workers ~~

## Ambiente de Teste
Vamos criar o cluster de testes, para isso utilizaremos a seguinte arquitetura:
- 3 Control Plane + ETCD + Workers

## Pré-requisitos:
- Docker instalado em todas as VMs (instruções [aqui](#instalar-o-docker))

## Criando o Cluster - Testes

Na tela inicial do Rancher, selecione a opção Create
![image](https://user-images.githubusercontent.com/24412660/140830232-48869f88-b930-4eb6-8ccc-c3ee3f9e03ca.png)

Selecione Custom:

![image](https://user-images.githubusercontent.com/24412660/140830541-bd3e0d0d-1ab6-46a4-9516-0569e4a92d57.png)

Em Cluster Name insira o nome ``test`` e clique em next

![image](https://user-images.githubusercontent.com/24412660/140830875-e97e4d7f-4b2e-4368-ae83-137504df35d5.png)

Agora selecione os roles que serão atribuidos para a máquina que será instanciada no Cluster. Nesse caso, como as 3 máquinas possuirão todos os roles (etcd, Control Plane e Worker), marque as 3 opções, copie o código docker gerado em baixo e clique em done.

![image](https://user-images.githubusercontent.com/24412660/140832459-34dbd3a8-6837-48c9-a14a-ca798cbe5bc6.png)

## Adicionando nodes - Testes
Agora será necessário acessar via SSH cada máquina que fará parte do cluster de testes e executar o comando Docker anterior gerado pelo rancher.

![image](https://user-images.githubusercontent.com/24412660/140833539-a0b5f762-d1ee-45ab-8c8f-7c2ec402ee42.png)


Veja que conforme as máquinas vão sendo adicionadas, elas vão aparecendo na interface do Rancher > Cluster:test > Machines

![image](https://user-images.githubusercontent.com/24412660/140833319-55f2006c-5f1a-420e-b9bd-96aa5e9686e4.png)

Repita o passo anterior nas outras 2 máquinas para serem adicionadas no cluster.


## Ambiente de Homologação
Vamos criar o cluster de homologação, para isso utilizaremos a seguinte arquitetura:
- 3 Control Plane + ETCD
- 4 Workers

## Criando o Cluster - Homologação

Na tela inicial do Rancher, selecione a opção Create
![image](https://user-images.githubusercontent.com/24412660/140830232-48869f88-b930-4eb6-8ccc-c3ee3f9e03ca.png)

Ative a versao do RKE2 e selecione Custom:
![image](https://user-images.githubusercontent.com/24412660/140942224-08ef4709-121f-49b5-a8ef-24d6be97d43c.png)

Insira ``homolog`` em Cluster Name, desmarque o System Service ``NGINX Ingress`` e clique em create

![image](https://user-images.githubusercontent.com/24412660/140943370-ed0c9ad7-68bf-436e-ab8b-b0a633dbf892.png)

Agora selecione os roles que serão atribuidos para a máquina que será instanciada no Cluster. Nesse caso, 3 máquinas possuirão o ETCD + Control Plane e as outras 4 serão apenas workers. 

![image](https://user-images.githubusercontent.com/24412660/140944428-fb3492a1-e61e-4b0c-8d10-614bc6a896dd.png)

Clique em ``Show Advanced`` e insira o nome que será atribuido a respectiva máquina.

![image](https://user-images.githubusercontent.com/24412660/140944667-8d733702-e79a-460b-8ddd-672560b3dab6.png)

Se necessário, habilite o modo inseguro para poder pular a verificação TLS.
Perceba que conforme você altera alguns parametros, os mesmos serão automaticamente adicionados ao comando curl abaixo, como é o caso da seleção de role (nos quais foram adicionados os parametros ``--etcd`` e ``--controlplane``) e também do node name (no qual foi adicionado o parametro ``--node-name controlplane1``.
Clique no comando para copia-lo.

![image](https://user-images.githubusercontent.com/24412660/140945619-b1461855-dd73-4d7b-a713-b5f1111795fd.png)

## Adicionando nodes - Homologação

Agora acesse via ssh a máquina que será instanciada com o respectivo role e node-name, cole e execute o comando curl copiado anteriormente.

![image](https://user-images.githubusercontent.com/24412660/140946984-db0eb7a2-6e7d-4934-b161-9c4273cdc8a1.png)


Você pode observar o provisionamento em tempo real das máquinas do cluster acessando ``Cluster Management > Clusters > Machines``:

![image](https://user-images.githubusercontent.com/24412660/140947509-abc79251-3568-4bf7-9c23-de78047cb5a0.png)

Repita os passos anteriores de provisionamento de máquina dentro do cluster alterando no comando os roles e node-names conforme a arquitetura desejada. Você gerar o comando isso alterando diretamente o comando copiado, ou via interface em ``Cluster Management > Clusters > Registration``

Aguarde alguns minutos e seu cluster de homologação estará provisionado:

![image](https://user-images.githubusercontent.com/24412660/140991003-ad541c61-b73d-4b82-a958-27d731a0289a.png)


## Extras:
### Gerar Chave Privada para o servidor Load Balancer
Acesse o servidor Load Balancer via SSH e execute o comando abaixo:

```
ssh-keygen -t rsa
```
![image](https://user-images.githubusercontent.com/24412660/140768637-a0e2732a-8e3f-4054-aa72-908ca2982d20.png)


Será solicitado para você inserir o caminho e nome do arquivo a ser salvo. Pressione ENTER para salvar na pasta padrão /root/.ssh/id_rsa


Em seguida será solicitado uma passphrase para segurança do certificado, pressione enter para manter em branco ou insira a passphrase seguido de enter.

![image](https://user-images.githubusercontent.com/24412660/140769152-26be08ab-8dc3-425e-b272-9f0bc3e4f91a.png)


Após gerar a chave no servidor Load Balancer, será necessário enviar a chave pública para os servidores que instalaremos o rancher server. Para isso, execute o comando ssh-copy-id subistituindo o **user** e **serverip** pelo usuário e ip de cada máquina.

```
ssh-copy-id user@serverip
```

![image](https://user-images.githubusercontent.com/24412660/140772853-df333640-9607-407b-ab97-71702c77f395.png)

Será então solicitado a senha SSH do usuário da respectiva máquina a ser enviado a chave pública, insira a senha e pressione enter.

![image](https://user-images.githubusercontent.com/24412660/140773084-aff131e9-1d34-44b6-a93a-2e633c210f08.png)

Por fim, repita o envio das chaves para as outras 2 máquinas que faltam.

### Instalar o Docker
Para instalar o docker, basta rodar a sequencia de comandos abaixo:

```
sudo apt-get install curl
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### Removendo o node do Rancher

É sempre recomendado realizar a remoção do node do cluster pela interface gráfica onde o mesmo será instanciado, mas em alguns momentos pode ser necessário remove-lo manualmente.
Para isso, basta executar a sequencia abaixo de comandos e depois remover as interfaces manualmente:

```
docker rm -f $(docker ps -qa)
docker rmi -f $(docker images -q)
docker volume rm $(docker volume ls -q)

for mount in $(mount | grep tmpfs | grep '/var/lib/kubelet' | awk '{ print $3 }') /var/lib/kubelet /var/lib/rancher; do umount $mount; done

rm -rf /etc/ceph \
       /etc/cni \
       /etc/kubernetes \
       /opt/cni \
       /opt/rke \
       /run/secrets/kubernetes.io \
       /run/calico \
       /run/flannel \
       /var/lib/calico \
       /var/lib/etcd \
       /var/lib/cni \
       /var/lib/kubelet \
       /var/lib/rancher/rke/log \
       /var/log/containers \
       /var/log/kube-audit \
       /var/log/pods \
       /var/run/calico

```

Verifique as interfaces com o comando:
```
ip address show
```

Se algum dos nomes de interfaces abaixo aparecerem:

```
flannel.1
cni0
tunl0
caliXXXXXXXXXXX (random interface names)
vethXXXXXXXX (random interface names)
```

Então remova-o com o comando, substituindo ``INTERFACE_NAME`` pelo nome da interface:

```
ip link delete INTERFACE_NAME
```
