# Aulas práticas de Kubernetes via Kubectl

Este repositório pressupõe que o cluster Kubernetes já está instalado e configurado.
Caso ainda não esteja, clique aqui **inserir link** para as instruções de como criar e configurar o cluster.

## Pods

### K8s imperativo
É possível rodar pods diretamente do terminal e realizando configurações simples utilizando o comando ``kubectl run NOME_DO_POD --image=NOME_DA_IMAGEM
Veja o exemplo abaixo:

```
kubectl run nginx-pod --image=nginx:latest
```

Para verificar se o mesmo foi criado utilize o comando ``kubectl get pods`` ou ``kubectl get pods --watch`` para ir atualizando automaticamente o status.

Para verificar mais informações sobre o pod utilize o comando ``kubectl describe pod NOME_DO_POD``

Para editar um pod que já esteja rodando, utilize o comando ``kubectl edit pod NOME_DO_POD``

### K8s declarativo

Crie um arquivo em seu computador chamado ``primeiro-pod.yaml`` seguindo o template abaixo:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-declarativo
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
```

Utilize o comando ``kubectl apply -f primeiro-pod.yaml`` para poder aplicar o pod e iniciar sua execucao. Em seguida, utilize o comando ``kubectl get pods --watch`` para aguardar o pod entrar em status de running.

#### Editando o pod declarativo

Acesse o arquivo ``primeiro-pod.yaml`` e edite o campo ``image: nginx:latest`` substituindo o ``latest`` por ``stable``. O template final deve ficar igual a código abaixo:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-declarativo
spec:
  containers:
    - name: nginx-container
      image: nginx:stable
```
Salve o arquivo, execute o ``kubectl apply -f primeiro-pod.yaml`` e veja com o ``kubectl describe pods`` que agora a imagem utilizada no pod será outra.


## Services
Os pods, a princípio, são isolados entre si e também isolados do ambiente externo do cluster. Para poder permitir a comunicação entre eles e/ou com o ambiente externo ao cluster, precisamos utilizar os Services. 

De forma bem simplificada, os services atuam como "switches/placas de rede" nas quais podemos alocar suas conexões de acordo com as regras que criarmos.

Temos 3 principais tipos de services:

- ClusterIP - para permitir a comunicação de diferentes pods dentro de um mesmo cluster
- NodePort - para expor a porta de um node externamente
- ~~LoadBalancer~~
- Ingress

Vamos criar dois novos pods chamados ``pod-1.yaml`` e ``pod-2.yaml``. Veja que dessa vez teremos alguns parâmetros a mais: 
- Labels: Será utilizado pelo service para identificar em quais pods devem ser aplicadas as regras
- Ports: Apesar de não ser obrigatório, é uma boa prática sempre declarar a porta para facilitar a identificação para futuras alterações. 

pod-1.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-1-SEU-USER
  ##namespace: default
  labels:
    app: primeiro-pod-SEU-USER
spec:
  containers:
    - name: container-pod-1
      image: nginx:latest
      ports:
        - containerPort: 80
```

pod-2.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-2-SEU-USER
  ##namespace: default
  labels:
    app: segundo-pod-SEU-USER
spec:
  containers:
    - name: container-pod-2
      image: nginx:latest
      ports:
        - containerPort: 80
```

Agora vamos coloca-los em execução utilizando o ``kubectl apply -f pod-1.yaml`` e ``kubectl apply -f pod-2.yaml``

Utilize o ``kubectl get pods -o wide`` para conseguir as informações de IP dos pods. Acesse o pod-1 internamente utilize o comando ``kubectl exec -it pod-1 -- bash``, e tente acessar o segundo pod como comando ``curl IP_DO_POD_2``, verá que ainda não é possível pois não existe um ClusterIP, vamos cria-lo agora.

### Cluster IP

Crie um arquivo chamado ``svc-pod1.yaml`` e outro ``svc-pod2.yaml`` seguindo os templates abaixo.
Veja que dessa vez estaremos utilizando o parâmetro ``selector`` para poder identificar a label do pod na qual iremos aplicar o SVC. O Label precisa seguir a mesma estrutura do que foi definindo anteriormente na declaração do pod, nesse caso ``app: primeiro-pod``.

Aplique com ``kubectl apply -f svc-pod1.yaml`` e ``kubectl apply -f svc-pod2.yaml``:

```
apiVersion: v1
kind: Service
metadata:
  name: svc-pod-1-SEU-USER
spec:
  type: ClusterIP
  selector:
    app: primeiro-pod-SEU-USER
  ports:
    - port: 9000 ## porta a ser escutada
      targetPort: 80 ### porta a ser despachada
```

```
apiVersion: v1
kind: Service
metadata:
  name: svc-pod-2-SEU-USER
spec:
  type: ClusterIP
  selector:
    app: segundo-pod-SEU-USER
  ports:
    - port: 9000 ## porta a ser escutada
      targetPort: 80 ### porta a ser despachada
```



### Node Port
Vamos expor as portas de um pod através do Node Port para ele poder ser acessado externamente pelo nosso computador.


Para isso, crie um arquivo chamado ``nodeport-pod2.yaml`` seguindo o template abaixo, alterando selector app e o nodePort (é possível utilizar quaisquer portas entre 30000 e 32767, por isso converse com os outros usuários que estão participando do treinamento para decidirem quais portas cada um irá alocar). Coloque-o em execução com ``kubectl apply -f nodeport-pod2.yaml``:

```
apiVersion: v1
kind: Service
metadata:
  name: nodeport-pod-2-SEU-USER
  ##namespace: default
spec:
  type: NodePort
  ports:
    - port: 80
      #targetPort: 80
      nodePort: 30000
  selector:
    app: segundo-pod-SEU-USER
```

Verifique os services que estao sendo executados com ``kubectl get svc``. Note que na coluna ``PORT(S)`` é possível verificar a regra da porta interna que foi exposta:

![image](https://user-images.githubusercontent.com/24412660/142284203-8d13d89d-162b-458c-a3cd-1ad8358c0cf0.png)

#### Acessando a aplicação

Agora iremos acessar a nossa página web. Repare que no comando executado anteriormente foi exibido o ``CLUSTER-IP``, porém o mesmo serve apenas para acesso interno ao cluster. Para acessar a aplicação externamente teremos então que descobrir em qual worker o pod está rodando, e depois qual o ip desse worker.

Execute o comando ``kubectl get pods -o wide`` e verifique em qual worker o seu pod está sendo executado:

![image](https://user-images.githubusercontent.com/24412660/142283628-f3efbffe-77e1-41e5-be5d-466e0bdcc349.png)

Com o comando ``kubectl get nodes -o wide`` verifique então o ip do node que está em execução

![image](https://user-images.githubusercontent.com/24412660/142284030-4fbc85e1-8182-4ba6-9bfe-ed7d96ccd1d6.png)

Abra seu navegador e digite ``IP-DO-WORKER:PORTA`` para poder ver a página:

![image](https://user-images.githubusercontent.com/24412660/142285497-b9e2b673-a9fd-4fb8-b59a-c73799e9a2ce.png)
