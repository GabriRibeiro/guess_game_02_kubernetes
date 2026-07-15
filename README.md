# Guess Game no Kubernetes

O jogo de adivinhação (backend em Flask, frontend em React servido por NGINX e banco Postgres)
rodando no Kubernetes com k3d. É a mesma aplicação da versão em Docker Compose, só que agora com
os objetos do Kubernetes no lugar do compose e com autoscaling no backend.

As imagens já estão publicadas no Docker Hub, então não precisa buildar nada:

- gabribeiro/guess-backend:1.0.0
- gabribeiro/guess-frontend:1.0.0

Tudo que é do Kubernetes fica em `k8s/`: o Helm chart em `k8s/chart` e os mesmos objetos já
renderizados em `k8s/manifests`, para quem preferir aplicar direto com kubectl.

## O que sobe no cluster

Roda tudo no namespace `guess-game`.

Postgres é um Deployment com uma réplica do postgres:16-alpine. A senha e o nome do banco ficam
num Secret e os dados num volume (um PersistentVolumeClaim de 1Gi), para não se perderem quando o
pod reinicia. Um Service ClusterIP na porta 5432 dá o endereço interno do banco.

O backend é o Deployment do Flask. Ele começa com duas réplicas e, antes de subir, espera o
Postgres ficar pronto através de um initContainer que roda pg_isready. Os dados de conexão vêm de
um ConfigMap e a senha vem do Secret do Postgres. O acesso interno é por um Service ClusterIP na
5000. O autoscaling fica por conta de um HorizontalPodAutoscaler, que varia de 2 a 6 réplicas
conforme o uso de CPU passa de 60%. Para isso o backend declara request de CPU, e o metrics-server
já vem junto no k3d.

O frontend é o Deployment do NGINX servindo o React buildado. Além das páginas, o NGINX repassa as
chamadas de /create, /guess/ e /health para o backend. O Service dele é do tipo NodePort na 30080,
que é a porta de entrada do sistema.

## Antes de começar

Precisa do Docker rodando, mais k3d e kubectl instalados. Helm só é necessário se você for pelo
caminho do chart.

## Subindo

Cria o cluster já mapeando a porta do NodePort (30080) para a 8080 da sua máquina:

```
k3d cluster create guess-game -p "8080:30080@server:0"
```

Faz o deploy. Com Helm:

```
helm install guess-game ./k8s/chart -n guess-game --create-namespace
```

Ou, sem Helm, aplicando os manifests prontos:

```
kubectl apply -f k8s/manifests/
```

Acompanha os pods até todos ficarem prontos (o backend só sobe depois do Postgres):

```
kubectl get pods -n guess-game -w
```

Quando estiverem de pé, abre no navegador:

```
http://localhost:8080
```

Se tiver criado o cluster sem o mapeamento de porta, dá para chegar no frontend por port-forward:

```
kubectl port-forward -n guess-game svc/guess-frontend 8080:80
```

## Jogando

Na tela inicial você cria um jogo com uma frase secreta e guarda o game_id que aparece. Depois vai
na parte de adivinhar, informa o game_id e tenta acertar a frase.

## Vendo o autoscaling

Deixa o HPA aberto num terminal:

```
kubectl get hpa -n guess-game -w
```

Em outro, sobe um pod que joga bastante carga em cima do backend (vários acessos ao mesmo tempo):

```
kubectl run load -n guess-game --image=busybox:1.36 --restart=Never -- \
  sh -c 'i=0; while [ $i -lt 40 ]; do (while true; do wget -q -O- http://guess-backend:5000/health >/dev/null 2>&1; done) & i=$((i+1)); done; wait'
```

Depois de um ou dois minutos o uso de CPU passa dos 60% e o número de réplicas do backend começa a
subir (até o máximo de 6). Para parar a carga, apaga o pod:

```
kubectl delete pod load -n guess-game
```

Sem carga, o HPA devolve o backend para o mínimo de réplicas depois de alguns minutos.

## Limpando

```
helm uninstall guess-game -n guess-game     # se usou Helm
kubectl delete -f k8s/manifests/            # se usou kubectl
k3d cluster delete guess-game
```

## Refazendo as imagens

Não precisa, elas já estão no Docker Hub. Mas se quiser reconstruir e publicar de novo:

```
docker build -t gabribeiro/guess-backend:1.0.0 -f Dockerfile.backend .
docker build -t gabribeiro/guess-frontend:1.0.0 -f Dockerfile.frontend .
docker push gabribeiro/guess-backend:1.0.0
docker push gabribeiro/guess-frontend:1.0.0
```

Se mexer no chart, regera os manifests com:

```
helm template guess-game ./k8s/chart -n guess-game > k8s/manifests/all.yaml
```
