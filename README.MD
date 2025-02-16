# Implantação do Redis com Worker Pod e ConfigMap

## Descrição

Este projeto contém uma configuração do Kubernetes que implanta um serviço Redis junto com um Worker Pod para interagir com ele. A configuração inclui:
- Um Deployment do Redis
- Um Worker Pod que consulta o serviço Redis
- Um ConfigMap para armazenar as configurações e dados do Redis
- Um serviço Redis para acesso interno ao Deployment do Redis

## Componentes

### 1. **Redis Deployment**

O Redis Deployment define um pod Redis que irá rodar um container Redis, expor a porta `6379` e usar um ConfigMap para suas configurações.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-pod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-config-volume
              mountPath: /etc/redis
              readOnly: true
          env:
            - name: redis_minha_chave
              valueFrom:
                configMapKeyRef:
                  name: redis-config
                  key: minha-chave
            - name: redis_outra_chave
              valueFrom:
                configMapKeyRef:
                  name: redis-config
                  key: outra-chave
      volumes:
        - name: redis-config-volume
          configMap:
            name: redis-config
            items:
              - key: redis.co

### 2. **Worker Deployment** 

O Worker Pod consulta continuamente o serviço Redis, buscando as chaves e imprimindo o valor de uma chave específica em um arquivo. Este pod usa a imagem redis:alpine e um loop simples para interagir com o Redis.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-pod
  labels:
    app: worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: alpine
        image: redis:alpine
        command: ["/bin/sh", "-c"]
        args:
          - while true; do
              redis-cli -h redis-service keys "*" > /data/keys.txt;
              sleep 1800;
            done;
          - redis-cli -h redis-service GET minha-chave > /data/print.txt;
            cat /data/print.txt;

### 3. **ConfigMap Redis**

O ConfigMap armazena os valores e chaves do Redis. Ele inclui duas chaves (minha-chave e outra-chave) e um arquivo redis.conf para configurar o Redis.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  minha-chave: "valor1-da-minha-chave"
  outra-chave: "valor2-da-outra-chave"
  redis.conf: |
    timeout 604800
    maxmemory-policy allkeys-lru
    appendonly yes


### 4. **Service Redis**

O Redis Service permite a comunicação interna entre o Worker Pod e o Redis Pod usando a porta do Redis (6379).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis

## Uso

Implantar os Pods Redis e Worker
Para implantar o Redis e o Worker Pod, use os seguintes comandos:
kubectl apply -f redis-configmap.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
kubectl apply -f worker-deployment.yaml

