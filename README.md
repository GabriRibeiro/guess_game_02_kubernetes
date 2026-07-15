# Guess Game no Kubernetes (k3d)

Reimplementação em Kubernetes da versão em Docker Compose do jogo de adivinhação. O sistema
tem um backend Flask, um frontend React servido por NGINX e um banco PostgreSQL. O backend
escala automaticamente por HPA. Todos os objetos de Kubernetes ficam em `k8s/`.

As imagens já estão publicadas no Docker Hub, então não é necessário reconstruir nada:

- `gabribeiro/guess-backend:1.0.0`
- `gabribeiro/guess-frontend:1.0.0`

## Componentes

Tudo é criado no namespace `guess-game`.

### Banco (PostgreSQL)
- **Secret `guess-postgres`**: usuário, senha e nome do banco.
- **PersistentVolumeClaim `guess-postgres-data`**: 1Gi no storage padrão do k3d (`local-path`), para os dados sobreviverem ao reinício do pod.
- **Deployment `guess-postgres`**: 1 réplica de `postgres:16-alpine`, com probe `pg_isready`.
- **Service `guess-postgres`** (ClusterIP:5432): endereço interno do banco.

### Backend (Flask)
- **ConfigMap `guess-backend-config`**: variáveis não sensíveis (tipo do banco, host, porta, nome e usuário).
- **Deployment `guess-backend`**: réplicas da API (imagem `gabribeiro/guess-backend`). Tem um initContainer que espera o Postgres aceitar conexão antes de subir, requests/limits de CPU (necessário para o HPA) e probes em `/health`. A senha do banco vem do Secret.
- **Service `guess-backend`** (ClusterIP:5000): endereço interno da API. O NGINX faz proxy para cá.
- **HorizontalPodAutoscaler `guess-backend`**: escala de 2 a 6 réplicas com alvo de 60% de CPU. Usa o metrics-server que o k3d já traz embutido.

### Frontend (React + NGINX)
- **Deployment `guess-frontend`**: imagem `gabribeiro/guess-frontend`. O NGINX serve o React e faz proxy de `/create`, `/guess/` e `/health` para o Service do backend.
- **Service `guess-frontend`** (NodePort:30080): ponto de entrada do sistema.

## Pré-requisitos
- Docker em execução.
- k3d e kubectl instalados.
- helm (apenas para o caminho com Helm; opcional).

## Passo a passo

### 1. Criar o cluster k3d
Mapeia a porta do NodePort do frontend (30080) para a porta `8080` da sua máquina:

```
k3d cluster create guess-game -p "8080:30080@server:0"
```

### 2. Fazer o deploy

Opção A - Helm (recomendado):

```
helm install guess-game ./k8s/chart -n guess-game --create-namespace
```

Opção B - manifests planos (kubectl):

```
kubectl apply -f k8s/manifests/
```

### 3. Acompanhar a subida

```
kubectl get pods -n guess-game -w
```

Espere todos os pods ficarem `Running` e `Ready`. O backend só sobe depois que o Postgres estiver aceitando conexão.

### 4. Acessar
Abra no navegador:

```
http://localhost:8080
```

Se o cluster tiver sido criado sem o mapeamento de porta, use port-forward:

```
kubectl port-forward -n guess-game svc/guess-frontend 8080:80
```

### 5. Jogar
- Na tela inicial, crie um jogo com uma frase secreta e guarde o `game_id`.
- Vá para a opção de adivinhar (breaker), informe o `game_id` e tente adivinhar.

### 6. Ver o autoscaling (HPA)
Em um terminal, observe o HPA:

```
kubectl get hpa -n guess-game -w
```

Em outro, gere carga no backend:

```
kubectl run -n guess-game load --image=busybox:1.36 --restart=Never --rm -it -- \
  /bin/sh -c "while true; do wget -q -O- http://guess-backend:5000/health >/dev/null; done"
```

Depois de um a dois minutos o número de réplicas do backend sobe. Ao parar a carga (Ctrl+C) ele volta ao mínimo depois de alguns minutos.

### 7. Limpar

```
helm uninstall guess-game -n guess-game     # se usou Helm
kubectl delete -f k8s/manifests/            # se usou kubectl
k3d cluster delete guess-game
```

## Estrutura do repositório

```
Dockerfile.backend        imagem do backend Flask
Dockerfile.frontend       imagem do frontend React + NGINX
nginx/default.conf        config do NGINX (serve o React e faz proxy para o backend)
run.py, guess/, repository/, requirements.txt   código do backend
frontend/                 código do frontend React
k8s/chart/                Helm chart (fonte de verdade)
k8s/manifests/            YAML plano gerado do chart (kubectl apply)
```

## Rebuild das imagens (opcional)
Não é necessário, as imagens já estão no Docker Hub. Para reconstruir e publicar:

```
docker build -t gabribeiro/guess-backend:1.0.0 -f Dockerfile.backend .
docker build -t gabribeiro/guess-frontend:1.0.0 -f Dockerfile.frontend .
docker push gabribeiro/guess-backend:1.0.0
docker push gabribeiro/guess-frontend:1.0.0
```

Se alterar o chart, regenere os manifests planos:

```
helm template guess-game ./k8s/chart -n guess-game > k8s/manifests/all.yaml
```
