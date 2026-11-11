# README-grupo — MEI.CDN-2627-TP

Grupo: <nomes e números de aluno>
Agentes: **1** · Perfil do teste comum: **`--reduzido`**
Máquina: 8 CPUs, 5,79 GiB de RAM no Docker Desktop · SO: macOS (Darwin), Apple Silicon (arm64)

> Todos os comandos correm em **bash**, a partir da pasta `cdn-tp/`.
> Cada passo tem uma **verificação**: só se avança quando ela confirma.

## 0. Pré-requisitos

```bash
cd cdn-tp
bash verificar-ambiente.sh          # tem de terminar com RESULT: OK
```

**Porquê:** confirma Docker, `kubectl`, `k3d`, `helm`, a porta 8080 livre e recursos suficientes, em vez de assumir que estão instalados.

## 1. Cluster (apaga o da PL05 e cria o nosso)

```bash
k3d cluster delete cdn
k3d cluster create cdn --servers 1 --agents 1 -p "8080:80@loadbalancer" --registry-config registries.yaml
k3d cluster list                    # AGENTS tem de mostrar 1/1
kubectl get nodes                   # 2 nós Ready: server-0 (control-plane) e agent-0
```

**Porquê:**
- `delete` parte de um estado limpo e igual em todos os portáteis (o cluster `cdn` da PL05 existe em todos).
- `--servers 1 --agents 1`: 1 nó de plano de controlo e 1 de trabalho (declarado no cabeçalho).
- `-p "8080:80@loadbalancer"`: a porta 8080 do portátil chega ao Traefik; é por aqui que o teste comum entra.
- `--registry-config registries.yaml`: os nós descarregam imagens do Docker Hub através de um *mirror*.

> ⚠️ O `create` tem de ir **numa só linha**: se for cortado, o k3d ignora as opções seguintes (já aconteceu: cluster criado com `AGENTS 0/0`).

## 2. Imagens (build + import para os nós)

```bash
docker build -t cdn-tickets:1.1 services/tickets
docker build -t cdn-orders:1.1 services/orders
docker build -t cdn-notifications:1.1 services/notifications
k3d image import cdn-tickets:1.1 cdn-orders:1.1 cdn-notifications:1.1 -c cdn
docker exec k3d-cdn-agent-0 crictl images | grep cdn-     # as 3 imagens com a tag 1.1
```

**Porquê:**
- `docker build -t <nome>:1.1`: constrói a imagem a partir do `Dockerfile` do serviço, com **tag fixa** (nunca `latest`), para o Kubernetes usar a imagem importada.
- `k3d image import`: cada nó é um contentor com o seu próprio containerd e **não vê** as imagens do Docker do portátil; sem registo, as imagens têm de ser copiadas para os nós.
- `crictl images`: a verificação que **prova** o import (a mensagem do k3d sozinha não prova).

> ⚠️ A tag tem de ser `:1.1` nos três builds. Se um sair como `latest` (já aconteceu com o `notifications`), o import falha e os Pods ficam em `ErrImagePull`.

## 3. Base de dados e broker (por ordem)

```bash
kubectl apply -f k8s/secrets.yaml -f k8s/postgres-init.yaml
kubectl apply -f k8s/postgres-statefulset.yaml -f k8s/rabbitmq.yaml -f k8s/redis.yaml
kubectl rollout status statefulset/postgres
kubectl rollout status deployment/rabbitmq
kubectl apply -f k8s/postgres-replica.yaml
kubectl rollout status statefulset/postgres-replica
kubectl get pods -o wide            # postgres-0, postgres-replica-0, rabbitmq, redis: Running 1/1
```

**Porquê (a ordem importa):**
- Os **Secrets primeiro**: os Pods seguintes referenciam-nos e não arrancam sem eles.
- O **primário antes da réplica**: a réplica arranca com `pg_basebackup`, que copia os dados do primário.
- `rollout status` bloqueia até o recurso estar pronto; é uma espera por **condição**, não por tempo.
- Postgres é **StatefulSet** (estado e disco próprios); RabbitMQ e Redis são Deployments.

> Nota: com 1 agente, `postgres-0` e `postgres-replica-0` ficam no mesmo nó (`agent-0`).

## 4. Serviços e entrada (configuração inicial do pacote)

```bash
kubectl apply -f k8s/tickets.yaml -f k8s/orders.yaml -f k8s/notifications.yaml
kubectl rollout status deployment/tickets
kubectl rollout status deployment/orders
kubectl rollout status deployment/notifications
kubectl apply -f k8s/traefik-middleware.yaml -f k8s/ingress.yaml
kubectl -n kube-system patch deployment traefik --patch-file k8s/traefik-patch.yaml
kubectl -n kube-system rollout status deployment/traefik
kubectl get pods -o wide            # todos Running 1/1
kubectl get ingress                 # bilheteira, classe traefik
curl -i http://localhost:8080/tickets/health     # HTTP/1.1 200 OK
```

**Porquê:**
- Os **ficheiros são listados um a um**, nunca `kubectl apply -f k8s/`: o `traefik-patch.yaml` não é um recurso novo, é um *patch* a um Deployment do `kube-system` que o k3s já instalou.
- O Middleware `strip-prefix` remove `/tickets` ou `/orders` antes de passar o pedido, para os serviços servirem as mesmas rotas que na sua própria porta.
- O patch faz o Traefik preferir o nó onde corre o Postgres (útil no R4(g)).
- O `curl` prova o caminho completo: portátil → balanceador → Traefik → Ingress → Service → Pod.

> Nota: `tickets` e `orders` têm 3 réplicas e há 2 nós: nada impõe uma réplica por nó, o scheduler reparte-as.

## 5. Observabilidade (Helm)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
kubectl create configmap cdn-dashboards --from-file=helm/dashboards/ --dry-run=client -o yaml | kubectl apply -f -
helm install prometheus prometheus-community/prometheus --version 29.39.0 -f helm/prometheus-values.yaml --wait --timeout 5m
helm install grafana grafana-community/grafana --version 14.2.0 -f helm/grafana-values.yaml --wait --timeout 5m
kubectl port-forward svc/prometheus-server 9090:80 > ~/pf-prom.log 2>&1 &
kubectl port-forward svc/grafana 3000:80 > ~/pf-grafana.log 2>&1 &
kubectl get pods                    # + prometheus-server (2/2) e grafana (1/1), Running
```

Prometheus em `http://localhost:9090`; Grafana em `http://localhost:3000` (admin / cdn-lab, só para laboratório), painel "cdn golden signals".

**Porquê:**
- **Helm** instala aplicações complexas (chart = pacote de manifestos) com a configuração nos ficheiros `values`.
- `--version` **fixa as versões** (29.39.0 e 14.2.0): o Helm instala sempre o mesmo.
- O painel vai num **ConfigMap** que o Grafana lê; o `--dry-run=client -o yaml | kubectl apply -f -` cria-o ou atualiza-o, por isso o comando é repetível.
- `--wait --timeout 5m` bloqueia até os Pods estarem prontos.
- O `port-forward` abre um túnel do portátil para o Service; o `&` deixa-o em segundo plano.

> Prometheus e Grafana **não guardam dados em disco**: se os Pods forem recriados, o histórico perde-se.
> Os `port-forward` morrem se o terminal fechar; há que os relançar.

## 6. Verificação

```bash
curl -s -w '\n' -X POST http://localhost:8080/tickets/events -H 'Content-Type: application/json' -d '{"id":"show1","name":"Concerto","capacity":50}'
curl -s -w '\n' -X POST http://localhost:8080/orders/orders -H 'Content-Type: application/json' -d '{"eventId":"show1","qty":2}'
curl -s -w '\n' http://localhost:8080/tickets/events/show1/audit
bash scripts/teste-comum.sh --reduzido
```

**Resultado esperado:**
- 1.º `curl`: o evento criado (`capacity 50`, `sold 0`).
- 2.º `curl`: `"status":"confirmed"` e `"notification":"queued"` (o e-mail vai pela fila do RabbitMQ).
- 3.º `curl`: `sold 2` e `oversold 0`.
- `teste-comum.sh --reduzido` (12 utilizadores, 45 s, 500 bilhetes): termina com `status: PASS` e grava a saída em `cdn-tp/resultados/`.

**Porquê:**
- O fluxo manual prova a cadeia inteira (Ingress → `orders` → `tickets` → Postgres → RabbitMQ → `notifications`).
- O `PASS` verifica duas regras: **nenhum bilhete vendido acima da capacidade** (consistência, T05) e **exatamente uma notificação por encomenda** (mensagens/idempotência, T04).
- Corre-se sempre com o **mesmo perfil** (`--reduzido`) e sem alterar os parâmetros do script.
