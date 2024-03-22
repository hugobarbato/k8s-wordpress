# Tutorial Kubernets WordPress 
 
Para provisionar um site WordPress com um banco de dados MySQL no Kubernetes usando K3S, siga os passos abaixo.  

** Este guia é projetado para ser seguido em uma VM Ubuntu e assume conhecimento básico de Kubernetes e uso de linha de comando.  

  

### Preparação do Ambiente 

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
  

### 2. Arquivos de Configuração do Kubernetes 

Você precisará de alguns arquivos YAML de configuração para o WordPress e MySQL, abaixo estão os exemplos:

#### MySQL 

 Para configurar o MySQL no Kubernetes envolve a criação de um conjunto de configurações, incluindo `PersistentVolume` (PV), `Deployment` e `Service`. Vamos detalhar cada parte.

##### 1. PersistentVolume - mysql-pvc.yml

    O PersistentVolume é um recurso que representa um volume persistente no cluster.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mysql-pvc
      labels:
        app: mysql
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
    ```

    O PVC é usado pelo pod do MySQL para requisitar o armazenamento físico.

##### 2. Deployment - mysql-deployment.yml

    O Deployment gerencia pods do MySQL, garantindo que o número desejado de instâncias esteja rodando e atualiza os pods de forma declarativa.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: mysql
    spec:
    selector:
        matchLabels:
        app: mysql
    strategy:
        type: Recreate
    template:
        metadata:
        labels:
            app: mysql
        spec:
        containers:
        - name: mysql
            image: mysql:5.7
            env:
            - name: MYSQL_ROOT_PASSWORD
            value: "yourpassword"
            ports:
            - containerPort: 3306
            name: mysql
            volumeMounts:
            - name: mysql-persistent-storage
            mountPath: /var/lib/mysql
        volumes:
        - name: mysql-persistent-storage
            persistentVolumeClaim:
            claimName: mysql-pvc
    ```

##### 3. Service - mysql-service.yml

    O Service define como expor o aplicativo MySQL, geralmente dentro do cluster, permitindo que outros serviços ou pods se conectem ao MySQL.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: mysql
    labels:
        app: mysql 
    spec:
    ports:
    - name: tcp-mysql-port
    - port: 3306
    selector:
        app: mysql 
    ```

##### 4. Aplicando as configurações

    1. Salve cada uma dessas configurações em arquivos `.yml` separados.
    2. Aplique-os usando `kubectl apply -f <arquivo>.yml` para cada arquivo.

    ```bash
        kubectl apply -f mysql-pvc.yaml
        kubectl apply -f mysql-deployment.yaml
        kubectl apply -f mysql-service.yaml
    ```

##### 5. Considerações

    - **Segurança**: Garanta que a senha do MySQL (`MYSQL_ROOT_PASSWORD`) não seja armazenada em texto claro nos arquivos. Use secrets do Kubernetes.
    - **Storage**: Ajuste o tamanho do PVC (`resources.requests.storage`) conforme necessário. 
    - **Versão do MySQL**: Certifique-se de que a versão do MySQL (`image: mysql:5.7`) esteja alinhada com seus requisitos de aplicação.

    Este é um guia básico para configurar o MySQL no Kubernetes, e dependendo das necessidades específicas do seu ambiente, pode ser necessário fazer ajustes adicionais.

  

#### WordPress 

  

1. **PersistentVolume** e **PersistentVolumeClaim**: Similar ao MySQL, para persistência dos dados. 

2. **Deployment**: Define o deployment do WordPress. 

3. **Service**: Para expor o WordPress dentro do cluster. 

4. **Ingress**: Para acessar o WordPress de fora do cluster K3S. 

  

### 3. Criação dos Recursos Kubernetes 

  

Após preparar os arquivos de configuração, aplique-os usando `kubectl`, o CLI do Kubernetes. 

  

```bash 

kubectl apply -f mysql-pv.yaml 

kubectl apply -f mysql-deployment.yaml 

kubectl apply -f wordpress-pv.yaml 

kubectl apply -f wordpress-deployment.yaml 

kubectl apply -f ingress.yaml 

``` 

  

### 4. Acesso ao WordPress 

  

Após a aplicação dos arquivos de configuração, o WordPress estará acessível através do endereço IP do seu Ingress. Você pode encontrar este endereço IP com o seguinte comando: 

  

```bash 

kubectl get ingress 

``` 

  

Agora, abra um navegador e acesse o WordPress usando o endereço IP encontrado. 

  

### Observações Importantes 

  

- **Segurança**: Certifique-se de alterar as senhas padrão e aplicar práticas recomendadas de segurança. 

- **Backups**: Configure backups regulares para os seus volumes persistentes. 

- **Monitoramento e Logs**: Utilize ferramentas de monitoramento e agregação de logs para manter a saúde do seu ambiente. 

- **Atualizações**: Mantenha o seu ambiente atualizado com as últimas versões do K3S, WordPress, e MySQL. 

  

Essas instruções fornecem uma visão geral do processo de configuração de um ambiente WordPress com MySQL no Kubernetes usando K3S. Dependendo das especificidades do seu ambiente de produção, ajustes adicionais podem ser necessários. 