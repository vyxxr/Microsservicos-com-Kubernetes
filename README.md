# Kubernetes com Microsserviços

Projeto de estudo (curso Alura) que executa uma aplicação de microsserviços Spring Boot em um cluster Kubernetes.

## Arquitetura

| Serviço      | Porta  | Descrição                                                        |
|--------------|--------|------------------------------------------------------------------|
| `server`     | 8081   | Eureka Server (service discovery), 2 réplicas (StatefulSet)      |
| `gateway`    | 8082   | Spring Cloud Gateway, roteia via Eureka (`/{nome-do-servico}/…`) |
| `pagamentos` | 40000  | Microsserviço `pagamentos-ms` (MySQL `alurafood-pagamento`)      |
| `pedidos`    | 40001  | Microsserviço `pedidos-ms` (MySQL `alurafood-pedidos`)           |
| `mysql`      | 3306   | MySQL 8 (StatefulSet com PVC de 8Gi)                             |

`pagamentos` chama `pedidos` via Feign (`PUT /pedidos/{id}/pago`) protegido por Resilience4j (circuit breaker).

Stack: Java 17, Spring Boot 2.6.7, Spring Cloud (Eureka, Gateway, OpenFeign), Flyway, MySQL, Docker, Kubernetes.

## Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Um cluster Kubernetes local, p.ex. [minikube](https://minikube.sigs.k8s.io/) ou kind
- [Helm 3](https://helm.sh/docs/intro/install/) (para a instalação via Helm)
- (Opcional, para rodar sem Docker) JDK 17 e MySQL

## Executando com Helm (recomendado)

O chart em `helm/` instala o Eureka Server, gateway, pagamentos, pedidos, os Services/LoadBalancers e, como dependência, o MySQL do chart da Bitnami (`mysql` 12.3.5, imagem `bitnamilegacy/mysql`).

### 1. Suba o cluster

```bash
minikube start
```

O MySQL do chart usa a StorageClass padrão do cluster (no minikube, `standard`), então não é preciso habilitar addons.

### 2. Baixe as dependências do chart

O MySQL não está versionado no repositório: ele é baixado para `helm/charts/` a partir do `Chart.yaml`/`Chart.lock`.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm dependency update helm/
```

Isso cria `helm/charts/mysql-12.3.5.tgz`. Confira com:

```bash
helm dependency list helm/
```

O status deve aparecer como `ok`. Sem esse passo, o `helm install` falha com `found in Chart.yaml, but missing in charts/ directory: mysql`.

### 3. Instale o chart

```bash
helm install alura-foods-app helm/
```

> **Importante:** use o nome de release `alura-foods-app`. O host do banco (`DB_HOST: alura-foods-app-mysql` em `values.yaml`) e o Secret com a senha (`secret.name: alura-foods-app-mysql`) são gerados pelo chart do MySQL a partir do nome do release. Se usar outro nome, ajuste os dois valores, p.ex. `--set configMap.data.DB_HOST=<release>-mysql --set secret.name=<release>-mysql`.

A senha de `root` do MySQL é gerada automaticamente e guardada no Secret `alura-foods-app-mysql` (chave `mysql-root-password`), que `pagamentos` e `pedidos` leem. Os bancos `alurafood-pagamento` e `alurafood-pedidos` são criados pelos próprios serviços (`createDatabaseIfNotExist=true`). Para ver a senha:

```bash
kubectl get secret alura-foods-app-mysql -o jsonpath='{.data.mysql-root-password}' | base64 -d
```

Para validar o que será gerado antes de instalar: `helm template alura-foods-app helm/`.

### 4. Aguarde os pods e exponha os serviços

```bash
kubectl get pods -w
```

Esperado: `alura-foods-app-mysql-0`, `server-0`, `server-1` e 2 réplicas de `gateway`, `pagamentos` e `pedidos`, todos `Running`. Reinícios iniciais de `pagamentos`/`pedidos` são normais enquanto o MySQL sobe.

Depois siga o passo [Exponha os serviços](#4-exponha-os-serviços) (`minikube tunnel` + `kubectl get svc loadbalancer-eureka loadbalancer-gateway`) e [Testando a API](#testando-a-api).

### Atualizando e removendo

```bash
helm upgrade alura-foods-app helm/     # após alterar values.yaml ou templates
helm uninstall alura-foods-app
```

O `helm uninstall` não remove o PVC do MySQL. Se for reinstalar do zero, apague-o também, senão o banco antigo continua com a senha anterior e não bate com o novo Secret:

```bash
kubectl delete pvc data-alura-foods-app-mysql-0
```

## Executando no Kubernetes (manifestos puros)

Alternativa sem Helm. Os manifestos estão em `k8s/` e usam imagens já publicadas no Docker Hub (`vyxxr/java-*-k8s`).

### 1. Suba o cluster

```bash
minikube start
```

O `PersistentVolumeClaim` (`k8s/volumes.yaml`) usa a StorageClass `csi-hostpath-sc`. No minikube, habilite-a:

```bash
minikube addons enable csi-hostpath-driver
minikube addons enable volumesnapshots
```

Em outro cluster, ajuste `storageClassName` em `k8s/volumes.yaml` para uma classe disponível (`kubectl get storageclass`).

### 2. Aplique os manifestos

```bash
# Configuração e armazenamento
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/volumes.yaml
kubectl apply -f k8s/services.yaml

# Banco de dados
kubectl apply -f k8s/mysql.yaml

# Eureka Server, gateway, pagamentos e pedidos
kubectl apply -f k8s/app.yaml

# Acesso externo (LoadBalancer)
kubectl apply -f k8s/loadbalancer.yaml
```

### 3. Aguarde os pods ficarem prontos

```bash
kubectl get pods -w
```

Esperado: `mysql-0`, `server-0`, `server-1` e 2 réplicas de `gateway`, `pagamentos` e `pedidos`, todos `Running`.
Se `pagamentos`/`pedidos` reiniciarem no início, é normal: eles aguardam o MySQL subir.

### 4. Exponha os serviços

No minikube, em um terminal separado (mantenha aberto):

```bash
minikube tunnel
```

Depois consulte os IPs externos:

```bash
kubectl get svc loadbalancer-eureka loadbalancer-gateway
```

- Eureka: `http://<EXTERNAL-IP>:8081`
- Gateway: `http://<EXTERNAL-IP>:8082`

Sem `tunnel`, use `minikube service loadbalancer-gateway --url`.

## Testando a API

Todas as chamadas passam pelo gateway (o nome do serviço no Eureka, em minúsculas, é o prefixo):

```bash
GATEWAY=http://<EXTERNAL-IP>:8082

# Pagamentos
curl $GATEWAY/pagamentos-ms/pagamentos
curl -X POST $GATEWAY/pagamentos-ms/pagamentos \
  -H 'Content-Type: application/json' \
  -d '{"valor":50.00,"nome":"Fulano","numero":"1234567890123456","expiracao":"2030-12","codigo":"123","formaDePagamentoId":1,"pedidoId":1}'
curl -X PATCH $GATEWAY/pagamentos-ms/pagamentos/1/confirmar

# Pedidos
curl $GATEWAY/pedidos-ms/pedidos
curl $GATEWAY/pedidos-ms/pedidos/porta     # mostra qual réplica respondeu
```

Endpoints:

- `pagamentos`: `GET /pagamentos`, `GET /pagamentos/{id}`, `POST /pagamentos`, `PUT /pagamentos/{id}`, `DELETE /pagamentos/{id}`, `PATCH /pagamentos/{id}/confirmar`
- `pedidos`: `GET /pedidos`, `GET /pedidos/{id}`, `GET /pedidos/porta`, `POST /pedidos`, `PUT /pedidos/{id}/status`, `PUT /pedidos/{id}/pago`

## Construindo suas próprias imagens

Cada serviço tem um `Dockerfile` multi-stage (Maven + JRE 17):

```bash
docker build -t <seu-usuario>/java-server-k8s:v1     server/
docker build -t <seu-usuario>/java-gateway-k8s:v1    gateway/
docker build -t <seu-usuario>/java-pagamentos-k8s:v1 pagamentos/
docker build -t <seu-usuario>/java-pedidos-k8s:v1    pedidos/

docker push <seu-usuario>/java-server-k8s:v1   # idem para os demais
```

Depois atualize o campo `image:` correspondente em `k8s/app.yaml` ou em `helm/values.yaml` (`gateway.image`, `pagamentos.image`, …).

## Executando localmente (sem Kubernetes)

Os serviços leem variáveis de ambiente: `SERVER_HOST` (URL do Eureka), `DB_HOST`, `DB_USER`, `DB_PASSWORD`.

```bash
# MySQL
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=12345 mysql:8

export SERVER_HOST=http://localhost:8081/eureka
export DB_HOST=localhost DB_USER=root DB_PASSWORD=12345

# Em terminais separados, nesta ordem:
(cd server     && ./mvnw spring-boot:run)
(cd gateway    && ./mvnw spring-boot:run)
(cd pagamentos && ./mvnw spring-boot:run)
(cd pedidos    && ./mvnw spring-boot:run)
```

## Limpeza

```bash
# Instalação via Helm
helm uninstall alura-foods-app
kubectl delete pvc data-alura-foods-app-mysql-0

# Instalação via manifestos
kubectl delete -f k8s/

minikube delete   # opcional
```

> **Atenção:** `k8s/secrets.yaml` contém credenciais de exemplo (`root` / `12345`). Não use em ambientes reais.
