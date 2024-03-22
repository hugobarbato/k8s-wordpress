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

Para configurar o MySQL no Kubernetes envolve a criação de um conjunto de configurações, incluindo `Secret`, `PersistentVolume` (PV), `Deployment` e `Service`. 

#### 1. Secret - mysql/secret.yml

Uma boa pratica em Kubernets é armazenar segredos em cofres seguros ao inves do codigo, podemos criar um component do K8S chamado Secret para isso.

```yaml
--- # Secrets
apiVersion: v1
kind: Secret
type: Opaque
data: 
  password: eW91cnBhc3N3b3Jk # Valor da Secret
metadata:  
  name: mysql-root-password-secret # Nome da Secret
```

Outra opção para essa etapa é setar essa configuração por meio do seguinte comando: 
```bash
kubectl create secret generic mysql-root-password-secret --from-literal=password='MYSQL_ROOT_PASSWORD'
```

#### 2. PersistentVolume - mysql/pv.yml

O PersistentVolume é um recurso que representa um volume persistente no cluster.
```yaml
--- # PV
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
      storage: 8Gi #Espaço disponibilizado para o MySQL
```
O PVC(PersistentVolumeClaim) é usado pelo pod do MySQL para requisitar o armazenamento físico.

#### 3. Deployment - mysql/deployment.yml

O Deployment gerencia pods do MySQL, garantindo que o número desejado de instâncias esteja rodando e atualiza os pods de forma declarativa.
```yaml
--- # Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql   
  labels:
    app: wordpress
spec:
  replicas: 3 # Numero de replicas
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
      - name: mysql # Nome do Container/Pod
        image: mysql:latest # Imagem Docker escolhida para o MYSQL
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-root-password-secret #Nome da Secret criada
              key: password
        livenessProbe:
          tcpSocket:
            port: 3306
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "64M"  # vMemoria minima alocada
            cpu: "250m"    # vCPU minima alocada
          limits:
            memory: "128M" # vMemoria maxima alocada
            cpu: "500m"    # vCPU maxima alocada
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim

```

#### 4. Service - mysql/service.yml

O Service define como expor o aplicativo MySQL, geralmente dentro do cluster, permitindo que outros serviços ou pods se conectem ao MySQL.
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

#### 5. Aplicando as configurações

1. Se optou por criar cada arquivo usando linha de comando você poderá aplicar essas configurações usando o comando `kubectl apply -f <arquivo>.yml`, conforme exemplo:
```bash
    kubectl apply -f mysql/secret.yml
    kubectl apply -f mysql/pv.yml
    kubectl apply -f mysql/deployment.yml
    kubectl apply -f mysql/service.yml
```
2. Se optou por clonar o repositorio basta rodar o comando: 
```bash
    kubectl apply -f mysql.yml
```


#### 6. Considerações

- **Segurança**: Garanta que a senha do MySQL (`MYSQL_ROOT_PASSWORD`) não seja armazenada em texto claro nos arquivos. Edite as secrets do Kubernetes utilizando `kubectl edit secret mysql-root-password`.
- **Storage**: Ajuste o tamanho do PV (`resources.requests.storage`) conforme necessário. 
- **Versão do MySQL**: Certifique-se de que a versão do MySQL (`image: mysql:lastest`) esteja alinhada com seus requisitos de aplicação WordPress.


### WordPress 
Para configurar o WordPress no Kubernetes envolve a criação de um conjunto de configurações, incluindo `PersistentVolume` (PV), `Deployment`, `Service` e `Ingress`. 

#### 1. PersistentVolume - wordpress/pv.yml
O PersistentVolume é um recurso que representa um volume persistente no cluster.
```yaml
--- # PV
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
      storage: 5Gi #Espaço disponibilizado para o WordPress
```
O PVC(PersistentVolumeClaim) é usado pelo pod do WordPress para requisitar o armazenamento físico.

#### 2. Deployment - wordpress/deployment.yml

O Deployment gerencia pods do WordPress, garantindo que o número desejado de instâncias esteja rodando e atualiza os pods de forma declarativa.
```yaml
--- # Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec: 
  replicas: 3 # Numero de replicas
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
      - name: wordpress  # Nome do Container/Pod
        image: wordpress:lastest # Imagem Docker escolhida para o WordPress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-root-password-secret
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
        resources:
          requests:
            memory: "64M"  # vMemoria minima alocada
            cpu: "250m"    # vCPU minima alocada
          limits:
            memory: "128M" # vMemoria maxima alocada
            cpu: "500m"    # vCPU maxima alocada
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

#### 3. Service - wordpress/service.yml

O Service define como expor o aplicativo WordPress, geralmente dentro do cluster, permitindo que outros serviços ou pods se conectem ao WordPress.
```yaml
--- # Service
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  type: LoadBalancer 
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
```

#### 4. Ingress - wordpress/ingress.yml

Para acessar o WordPress de fora do cluster K3S
```yaml
--- # Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
spec:
  rules:
  - http:
      paths:
      - path: / 
        pathType: Prefix 
        backend:
          service:
            name: wordpress  
            port:
              number: 80 
```

#### 5. Aplicando as configurações

1. Se optou por criar cada arquivo usando linha de comando você poderá aplicar essas configurações usando o comando `kubectl apply -f <arquivo>.yml`, conforme exemplo:
```bash
    kubectl apply -f wordpress/pv.yml 
    kubectl apply -f wordpress/deployment.yml
    kubectl apply -f wordpress/service.yml
    kubectl apply -f wordpress/ingress.yml
```
2. Se optou por clonar o repositorio basta rodar o comando: 
```bash
    kubectl apply -f wordpress.yml
```



  

## 3. Acesso ao WordPress 
Após a aplicação dos arquivos de configuração, o WordPress estará acessível através do endereço IP do seu Ingress. Você pode encontrar este endereço IP com o seguinte comando: 
```bash 
kubectl get wordpress-ingress 
``` 
Agora, abra um navegador e acesse o WordPress usando o endereço IP encontrado. 
