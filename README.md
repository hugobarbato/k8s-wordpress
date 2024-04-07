# Tutorial Kubernets WordPress 
 
Para provisionar um site WordPress com um banco de dados MySQL no Kubernetes usando K3S, siga os passos abaixo.  

`** Este guia é projetado para ser seguido em uma VM Ubuntu e assume conhecimento básico de Kubernetes e uso de linha de comando.`

  

## 1. Preparação do Ambiente 

Para começarmos vamos preparar nosso ambiente linux para implementação desse lab. 

#### Instalação do K3S no Ubuntu 

1. **Atualize o sistema**: Antes de começar, é sempre uma boa prática atualizar os pacotes do seu sistema Ubuntu. 
```bash  
    # Atualizar o software
    sudo apt update && sudo apt upgrade -y  
``` 

2. **Instale o K3S**: O K3S é uma distribuição leve do Kubernetes fácil de instalar. Execute o seguinte comando para instalá-lo. 
```bash  
    # Instalar K3S com kubectl 
    curl -sfL https://get.k3s.io | sh - 
    # Habilitar permissão para usuário ubuntu obter configuração de acesso ao K8S 
    echo K3S_KUBECONFIG_MODE=\"644\" >> /etc/systemd/system/k3s.service.env 
    # Restart do serviço k3s para carregar nova config 
    systemctl restart k3s 
``` 

3. **Verifique a instalação**: Após a instalação, confirme se o K3S está rodando corretamente. 
```bash
    # Logar com usuário ubuntu (caso pedir senha, utilizar `ubuntu`)
    su ubuntu
    # Obter informações dos nós do cluster K8S
    kubectl get nodes
    # Observe que possuímos uma única instância de K8S
``` 
Você deve ver o nó do seu servidor listado, indicando que o K3S está rodando. 
  
--- 
## 2. Preparação dos Arquivos de Configuração do Kubernetes 

Você precisará configurar o MySQL e WordPress no Kubernets para isso precisaremos da configuração de alguns recursos. 
Você poderá clonar este repositorio ou criar os arquivos por linha de comando seguindo o padrão: 
``` bash
    nano <NOME_DO_ARQUIVO>.yml
```

### MySQL 

Para configurar o MySQL no Kubernetes envolve a criação de um conjunto de configurações, incluindo `Secret`, `PersistentVolumeClaim` (PVC), `Deployment` e `Service`. 

#### 1. Secret

Uma boa pratica em Kubernets é armazenar segredos em cofres seguros ao inves do codigo, para criar a secret execute o seguinte comando:

```bash
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=YOUR_PASSWORD
EOF
```

Esse comando criou um arquivo `kustomization.yaml` que irá ser nosso arquivo de configuração principal.
 

#### 2. PersistentVolumeClaim - mysql.yml

O PersistentVolumeClaim é um recurso que representa um volume persistente no cluster, adicione ao arquivo mysql.yml o seguinte trecho:
```yaml 
--- #PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi #Espaço disponibilizado para o MySQL
```
O PVC(PersistentVolumeClaim) é usado pelo pod do MySQL para requisitar o armazenamento físico.

#### 3. Deployment - mysql.yml

O Deployment gerencia o pod do MySQL, adicione ao arquivo mysql.yml o seguinte trecho:
```yaml 
--- # Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec: 
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "200Mi"
            cpu: "100m" #(0.1 vCPU)
          limits:
            memory: "1000Mi"
            cpu: "500m" #(0.5 vCPU)
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

#### 4. Service - mysql.yml

O Service define como expor o aplicativo MySQL, geralmente dentro do cluster, permitindo que outros serviços ou pods se conectem ao MySQL, adicione ao arquivo mysql.yml o seguinte trecho:
```yaml
--- # Service
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```
 
#### 5. Considerações

- **Segurança**: Garanta que a senha do MySQL (`MYSQL_ROOT_PASSWORD`) não seja armazenada em texto claro nos arquivos. Edite as secrets do Kubernetes utilizando `kubectl edit secret mysql-root-password`.
- **Storage**: Ajuste o tamanho do PV (`resources.requests.storage`) conforme necessário. 
- **Versão do MySQL**: Certifique-se de que a versão do MySQL (`image: mysql:8`) esteja alinhada com seus requisitos de aplicação WordPress.


### WordPress 
Para configurar o WordPress no Kubernetes envolve a criação de um conjunto de configurações, incluindo `PersistentVolumeClaim` (PV), `Deployment` e `Service`. 

#### 1. PersistentVolumeClaim - wordpress.yml
O PersistentVolume é um recurso que representa um volume persistente no cluster, adicione ao arquivo wordpress.yml o seguinte trecho:
```yaml
--- #PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi #Espaço disponibilizado para o WordPress
```
O PVC(PersistentVolumeClaim) é usado pelo pod do WordPress para requisitar o armazenamento físico.

#### 2. Deployment - wordpress.yml

O Deployment gerencia pods do WordPress, garantindo que o número desejado de instâncias esteja rodando e atualiza os pods de forma declarativa, adicione ao arquivo wordpress.yml o seguinte trecho:
```yaml
--- # Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:6.2.1-apache
        name: wordpress3
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: WORDPRESS_DB_USER
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
        resources:
          requests:
            memory: "200Mi"
            cpu: "100m" #(0.1 vCPU)
          limits:
            memory: "1000Mi"
            cpu: "500m" #(0.5 vCPU)
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

#### 3. Service - wordpress.yml

O Service define como expor o aplicativo WordPress, geralmente dentro do cluster, permitindo que outros serviços ou pods se conectem ao WordPress, adicione ao arquivo wordpress.yml o seguinte trecho:
```yaml
--- # Service 
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

### Aplicando as configurações

1. Adicione os arquivos criados ao `kustomization.yaml` como recursos a serem criados, rodando o comando a seguir: 
```bash
  cat <<EOF >>./kustomization.yaml
  resources:
    - mysql.yml
    - wordpress.yml
  EOF
```
<detail>
 <summary>Caso não tenha criado os arquivos pode optar por baixar eles por meio do seguinte comando</summary>
 ```bash
   curl -LO https://github.com/hugobarbato/k8s-wordpress/blob/main/mysql.yml
   curl -LO https://github.com/hugobarbato/k8s-wordpress/blob/main/wordpress.yml
 ```
</detail>

2. Aplique o arquivo `kustomization.yaml` atraves do comando: 
```bash
  kubectl apply -k ./
```

3. Você pode confirmar a criação dos recursos utilizando o seguinte comando: 
```bash 
  kubectl get pods,pv,pvc,service,secret,deployment 
```

## 3. Acesso ao WordPress 
Após a aplicação dos arquivos de configuração, o WordPress estará acessível através do endereço IP do Serviço. Você pode encontrar este endereço IP e Porta com o seguinte comando: 
```bash 
kubectl get service 
``` 
Agora, abra um navegador e acesse o WordPress usando o endereço IP e Porta encontrado. 

## 4. Testando a resiliencia

Ao matar qualquer pod da solução, por conta da `strategy` estar como `Recreate` nos deployments ele serão automaticamente recriados sem perder as informações por conta do `PersistentVolume`. 
