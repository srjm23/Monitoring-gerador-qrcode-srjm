# Elastic Stack para logs do EKS

O ECK gerencia um Elasticsearch, um Kibana e um Filebeat por nó. O Filebeat
coleta os logs de todos os containers em `/var/log/containers` e os envia com
metadados do Kubernetes ao Elasticsearch. Os índices usam retenção de 7 dias.

O pod do Elasticsearch requer que o NodePool do Karpenter aceite instâncias
com pelo menos 4 GiB. No cluster `eks-srjm`, o NodePool `spot-only` permite
`t3.small` e `t3.medium`, respeitando seu limite total de 16 GiB.

## Instalação

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
helm upgrade --install elastic-operator elastic/eck-operator \
  --namespace elastic-system --create-namespace \
  --version 3.4.0 --wait

kubectl apply -f elastic/namespace.yml
kubectl apply -f elastic/stack.yml
```

## Acesso ao Kibana

```bash
kubectl -n elastic-stack get secret logs-es-elastic-user \
  -o go-template='{{.data.elastic | base64decode}}{{"\n"}}'
kubectl -n elastic-stack port-forward service/logs-kb-http 5601:5601
```

Acesse `https://localhost:5601`, aceite o certificado local e entre com o
usuário `elastic`. Em **Discover**, crie o data view `filebeat-*`.
