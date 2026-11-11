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

## 7. R4: resiliência (falhas sob carga)

Antes de cada cenário: `kubectl get pods` todos `Running` e fila `order.created` a 0
(`kubectl exec deploy/rabbitmq -- rabbitmqctl list_queues name messages_ready`).

### (a) Apagar um Pod do `tickets`

```bash
# Terminal B (observar; um ficheiro por terminal)
kubectl get pods -l app=tickets -w | tee resultados/r4a-pods-watch.txt
# Terminal C (falha aos 15 s + teste comum)
( sleep 15; kubectl get pod -l app=tickets -o name | head -1 | xargs kubectl delete ) &
bash scripts/teste-comum.sh --reduzido
```

**Porquê:** o `sleep 15` garante que a carga já está a correr; `head -1 | xargs kubectl delete` apaga um Pod; o `watch` regista o ciclo de vida.
**Resultado medido:** 0 erros em 15 167 pedidos, `OVERSOLD 0`, 500/500 notificações, `PASS` (`resultados/teste-comum-20261111-213609-reduzido.txt`). O Pod novo ficou `1/1` ao fim de 15 s; entretanto serviram 2 réplicas.
**Porquê não houve erros:** réplicas (3) + `preStop` + readiness + retries do `orders` com o mesmo `requestId`.

### (b) Apagar um Pod do `orders`

```bash
# Terminal B
kubectl get pods -l app=orders -w | tee resultados/r4b-pods-watch.txt
# Terminal C
( sleep 15; kubectl get pod -l app=orders -o name | head -1 | xargs kubectl delete ) &
bash scripts/teste-comum.sh --reduzido
```

**Porquê:** igual a (a), com `app=orders`. Aqui **ninguém repete** o pedido do cliente (o Traefik só tem `strip-prefix`, sem `retry`; o k6 não repete): a proteção é o `preStop: sleep 5` + a saída do Pod dos endpoints + as outras 2 réplicas.
**Resultado medido:** 0 erros em 20 758 pedidos, `OVERSOLD 0`, 500/500 notificações, `PASS` (`resultados/teste-comum-20261111-215029-reduzido.txt`). Pod novo `1/1` aos 9 s; o antigo acabou em `Completed`.

### (c) `notifications` a 0 réplicas, 30 s (`NOTIFY_MODE=async`)

```bash
# Terminal B: fila a cada 5 s (ficheiro próprio)
while true; do date +%T; kubectl exec deploy/rabbitmq -- rabbitmqctl list_queues name messages_ready messages_unacknowledged; sleep 5; done | tee resultados/r4c-async-fila.txt
# Terminal C
( sleep 10; kubectl scale deployment/notifications --replicas=0; sleep 30; kubectl scale deployment/notifications --replicas=1 ) &
bash scripts/teste-comum.sh --reduzido
```

**Porquê:** `scale --replicas=0` tira o consumidor; 30 s depois repõe-se. `list_queues ... messages_ready messages_unacknowledged` mostra as mensagens à espera e as entregues sem `ack`.
**Resultado medido:** 0 erros em 13 104 pedidos, `OVERSOLD 0`, 500/500, `missing 0` (`waited 2s`), `PASS` (`resultados/teste-comum-20261111-215503-reduzido.txt`). A fila **congelou em 251** durante a falha e escoou depois (0 às 21:56:43).
**Porquê não há erros:** em `async` o `orders` só publica; a fila durável + outbox + ack manual guardam as mensagens. O custo é **atraso** (falha silenciosa).

### (c') Repetição com `NOTIFY_MODE=sync` (experiência piorada de propósito)

```bash
# editar k8s/orders.yaml linha 10 -> NOTIFY_MODE: sync
kubectl apply -f k8s/orders.yaml
kubectl rollout restart deployment/orders
kubectl rollout status deployment/orders
kubectl get pods -l app=orders -o name | xargs -I{} kubectl exec {} -- printenv NOTIFY_MODE   # tem de dar sync ×3
( sleep 10; kubectl scale deployment/notifications --replicas=0; sleep 30; kubectl scale deployment/notifications --replicas=1 ) &
bash scripts/teste-comum.sh --reduzido
```

**Porquê:** o `envFrom` só é lido quando o Pod arranca, por isso o `rollout restart` é obrigatório; o `printenv` em cada Pod prova o que o processo recebeu (o cabeçalho do teste lê o ConfigMap, não os Pods).
**Resultado medido:** 0 erros em 14 268 pedidos, `OVERSOLD 0`, mas **`notifications 234, missing 266`**, `check notified: FAIL` (`resultados/teste-comum-20261111-221238-reduzido.txt`). Sem fila, as notificações da janela da falha perdem-se. Resultado **esperado**: justifica o `async`. **Repor `async` no fim** (ver abaixo).

### (d) Apagar o Pod do RabbitMQ (`OUTBOX=on`, `async`)

```bash
# Terminal B
kubectl get pods -l app=rabbitmq -w | tee resultados/r4d2-pods-watch.txt
# Terminal D: outbox pendente, lida do próprio orders a cada segundo
while true; do date +%T; kubectl exec deploy/orders -- node -e "fetch('http://localhost:3002/metrics').then(r=>r.text()).then(t=>console.log(t.split('\n').filter(l=>l.startsWith('orders_outbox_pending')).join('\n')))"; sleep 1; done | tee resultados/r4d2-outbox.txt
# Terminal C: falha aos 3 s (as encomendas entram nos primeiros segundos)
( sleep 3; kubectl delete pod -l app=rabbitmq ) &
bash scripts/teste-comum.sh --reduzido
```

**Porquê:** `/metrics` do `orders` (`lib.js:99`) expõe `orders_outbox_pending` (linhas da outbox ainda por publicar); a falha tem de vir cedo, porque com 500 bilhetes as encomendas aceites entram nos primeiros segundos.
**Resultado medido:** 0 erros em 17 542 pedidos, `OVERSOLD 0`, 500/500, `PASS` (`resultados/teste-comum-20261111-223843-reduzido.txt`). A outbox subiu a **500** (22:38:54–22:39:11) e voltou a **0** (22:39:14) quando o RabbitMQ voltou (`1/1` aos 19 s).
**Porquê:** encomenda e mensagem na mesma transação; o `/ready` do `orders` não exige o broker; o relay só publica quando o broker responde. Nota: a 1.ª tentativa (falha aos 15 s, sem medir a outbox) não prova o mecanismo.

### (f) Apagar `postgres-replica-0` e decisão sobre a réplica (liga ao R5)

**Problema medido:** com `READ_DATABASE_URL` definido no `tickets`, o `/ready` inclui a réplica (`services/tickets/src/index.js:49`). Ao apagar a réplica, os 3 Pods do `tickets` deram `503` na readiness ao mesmo tempo e o Service ficou sem endpoints (falha parcial → total), embora o `POST /orders` não use a réplica.

**Decisão (configuração final):** retirar `READ_DATABASE_URL` do `tickets`. Em `k8s/tickets.yaml`, **não** existe o bloco:
```yaml
            - name: READ_DATABASE_URL
              valueFrom:
                secretKeyRef: { name: postgres, key: TICKETS_READ_DATABASE_URL }
```
(no pacote original está nas linhas 53-55; foi apagado). Depois de alterar: `kubectl apply -f k8s/tickets.yaml`, `kubectl rollout restart deployment/tickets`, `kubectl rollout status deployment/tickets`.

**Verificação:**
```bash
kubectl get pods -l app=tickets -o name | xargs -I{} kubectl exec {} -- sh -c 'printenv READ_DATABASE_URL || echo "(sem READ_DATABASE_URL)"'   # ×3 sem variável
curl -s -w '\n' http://localhost:8080/tickets/events/<eventId>    # "readFrom":"primary"
```

**Teste da falha:**
```bash
kubectl get pods -w | grep --line-buffered -E 'tickets|postgres-replica' | tee resultados/r4f3-pods-watch.txt   # Terminal B
kubectl get endpoints tickets -w | tee resultados/r4f3-endpoints-watch.txt                                       # Terminal D
( sleep 15; kubectl delete pod postgres-replica-0 ) &                                                           # Terminal C
bash scripts/teste-comum.sh --reduzido
```
**Resultado medido (máquina libertada):** 0 erros em 27 720 pedidos, 614 req/s, p95 50,3 ms, `OVERSOLD 0`, `PASS` (`resultados/teste-comum-20261111-232018-reduzido.txt`); endpoints do `tickets` sempre 3. Custo da decisão: as leituras `GET /events/:id` vão ao primário (o alívio de leituras é nulo no teste comum).

> ⚠️ Medir desempenho só com o portátil libertado (apps fechadas, ≥ ~35% de memória livre): com 8 GiB de RAM e 5,79 GiB no Docker, as corridas com a máquina cheia deram 85-110 req/s contra 548-614 req/s.
