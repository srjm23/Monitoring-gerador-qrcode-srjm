# Monitoramento do Gerador de QR Code

Stack de observabilidade Kubernetes criada para coletar, armazenar e visualizar
métricas, traces e logs da aplicação.

```text
Métricas ──► OpenTelemetry/Prometheus ──► Grafana
Traces   ──► OpenTelemetry ──► Tempo ──► Grafana
Logs     ──► Filebeat ──► Elasticsearch ──► Kibana
```

Grafana, Prometheus e Kibana foram publicados por meio do **NGINX Gateway**. O
**cert-manager**, integrado ao Let's Encrypt, foi utilizado para emitir e
renovar os certificados TLS desses endpoints.

## kube-prometheus

O kube-prometheus instala Prometheus Operator, Prometheus, Alertmanager,
Grafana, kube-state-metrics, node-exporter, regras de alerta e dashboards no
namespace `monitoring`.

```bash
git clone https://github.com/prometheus-operator/kube-prometheus
cd kube-prometheus

kubectl create -f manifests/setup
kubectl wait --for condition=Established \
  --all CustomResourceDefinition --timeout=5m
kubectl apply -f manifests/
```

O diretório `manifests/setup` cria os CRDs e o Prometheus Operator. O
`manifests/` cria os componentes responsáveis pela coleta, armazenamento,
alertas e visualização das métricas.

### Recursos complementares

| Arquivo | Recurso | Função |
| --- | --- | --- |
| `prometheus/prometheus-rbac.yml` | `Role` e `RoleBinding` | Permite ao Prometheus descobrir pods, Services e EndpointSlices da aplicação. |
| `prometheus/prometheus-net.yml` | `NetworkPolicy` | Restringe o acesso de entrada ao Prometheus pela porta `9090`. |
| `prometheus/prometheus-httpRoute.yml` | `HTTPRoute` | Encaminha o domínio público para o Service `prometheus-k8s`. |
| `grafana/datasources.yml` | `Secret` | Provisiona Prometheus e Tempo como fontes de dados do Grafana. |
| `grafana/otel-dashboard.yml` | `ConfigMap` | Armazena o dashboard de disponibilidade e telemetria do OpenTelemetry. |
| `grafana/grafana-httpRoute.yml` | `HTTPRoute` | Encaminha o domínio público para o Service do Grafana. |
| `serviceMonitor.yml` | `ServiceMonitor` | Configura a coleta das métricas da aplicação e do OpenTelemetry Collector. |
| `prometheus-rule.yml` | `PrometheusRule` | Cria alertas para consumo elevado de CPU e memória da aplicação. |

## Tempo

O Tempo armazena os traces distribuídos. Foi configurado em modo monolítico,
com recepção OTLP gRPC/HTTP, retenção de 24 horas e volume persistente de
10 GiB.

```bash
kubectl apply -f tempo/
kubectl rollout status deployment/tempo \
  --namespace monitoring --timeout=5m
```

| Arquivo | Recurso | Função |
| --- | --- | --- |
| `tempo/configmap.yml` | `ConfigMap` | Configura receivers OTLP, retenção e armazenamento local dos traces. |
| `tempo/persistent-volume-claim.yml` | `PersistentVolumeClaim` | Reserva 10 GiB para WAL e blocos de traces. |
| `tempo/service.yml` | `Service` | Expõe internamente a API `3200` e os endpoints OTLP `4317` e `4318`. |
| `tempo/deployment.yml` | `Deployment` | Executa uma instância do Tempo com probes, segurança e limites de recursos. |

## OpenTelemetry

O OpenTelemetry Collector recebe métricas e traces via OTLP. Ele também
descobre pods anotados para coleta Prometheus, disponibiliza as métricas para o
Prometheus e encaminha os traces ao Tempo.

```bash
kubectl apply -f opentelemetry/
kubectl apply -f serviceMonitor.yml
kubectl rollout status deployment/otel-collector \
  --namespace monitoring --timeout=5m
```

| Arquivo | Recurso | Função |
| --- | --- | --- |
| `opentelemetry/service-account.yml` | `ServiceAccount` | Define a identidade utilizada pelo Collector. |
| `opentelemetry/cluster-role.yml` | `ClusterRole` | Autoriza a leitura dos pods usados na descoberta de métricas. |
| `opentelemetry/cluster-role-binding.yml` | `ClusterRoleBinding` | Associa as permissões à ServiceAccount do Collector. |
| `opentelemetry/configmap.yml` | `ConfigMap` | Define receivers, processors, exporters e pipelines de métricas e traces. |
| `opentelemetry/service.yml` | `Service` | Expõe OTLP, métricas Prometheus, telemetria interna e health check. |
| `opentelemetry/deployment.yml` | `Deployment` | Executa o Collector com probes, limites de recursos e contexto de segurança. |

## Elastic Stack

O ECK gerencia o Elasticsearch, Kibana e Filebeat. O Filebeat executa como
`DaemonSet` nos nodes, coleta os logs dos containers, adiciona metadados do
Kubernetes e envia os eventos ao Elasticsearch. A política ILM mantém os
índices por dois dias.

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
helm upgrade --install elastic-operator elastic/eck-operator \
  --namespace elastic-system --create-namespace \
  --version 3.4.0 --wait

kubectl apply -f elastic/namespace.yml
kubectl apply -f elastic/filebeat-ilm-policy.yml
kubectl apply -f elastic/service-account.yml
kubectl apply -f elastic/cluster-role.yml
kubectl apply -f elastic/cluster-role-binding.yml
kubectl apply -f elastic/elasticsearch.yml
kubectl apply -f elastic/kibana.yml
kubectl apply -f elastic/filebeat.yml
```

| Arquivo | Recurso | Função |
| --- | --- | --- |
| `elastic/namespace.yml` | `Namespace` | Isola os recursos no namespace `elastic-stack`. |
| `elastic/elasticsearch.yml` | `Elasticsearch` | Armazena e indexa os logs em um volume persistente de 20 GiB. |
| `elastic/kibana.yml` | `Kibana` | Disponibiliza a interface de pesquisa e visualização dos logs. |
| `elastic/filebeat.yml` | `Beat` | Cria o DaemonSet responsável pela coleta e envio dos logs. |
| `elastic/filebeat-ilm-policy.yml` | `ConfigMap` | Define a política de exclusão dos índices após dois dias. |
| `elastic/service-account.yml` | `ServiceAccount` | Define a identidade utilizada pelo Filebeat. |
| `elastic/cluster-role.yml` | `ClusterRole` | Permite ler nodes, namespaces e pods para enriquecer os logs. |
| `elastic/cluster-role-binding.yml` | `ClusterRoleBinding` | Associa as permissões à ServiceAccount do Filebeat. |
| `elastic/http-route-kibana.yml` | `HTTPRoute` | Encaminha o domínio público para o Service do Kibana. |

## Validação

```bash
kubectl get pods --namespace monitoring
kubectl get pods --namespace elastic-stack
kubectl get servicemonitor --all-namespaces
kubectl get prometheusrule --all-namespaces
kubectl get elasticsearch,kibana,beat --namespace elastic-stack
```

Senha do usuário `elastic` para acesso ao Kibana:

```bash
kubectl get secret logs-es-elastic-user \
  --namespace elastic-stack \
  --output go-template='{{.data.elastic | base64decode}}{{"\n"}}'
```

## Referências

- [Descomplicando Prometheus](https://github.com/badtuxx/DescomplicandoPrometheus)
- [prometheus-operator/kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)
