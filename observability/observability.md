# Production-Ready OpenTelemetry Observability Stack

For MERN Restaurant App

### Install Order

````bash
# 1. Namespace
kubectl create namespace observability

# 2. Helm repos (run once)
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add elastic https://helm.elastic.co
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# 3. Elasticsearch (HA)
helm upgrade --install elasticsearch elastic/elasticsearch -n observability \
  --set replicas=3 --set minimumMasterNodes=2 \
  --set resources.requests.memory=4Gi \
  --set volumeClaimTemplate.resources.requests.storage=100Gi

# 4. Kibana
helm upgrade --install kibana elastic/kibana -n observability \
  --set elasticsearchHosts="http://elasticsearch-master:9200"

# 5. Logstash
helm upgrade --install logstash elastic/logstash -n observability -f observability/logstash-values.yaml

# 6. Jaeger
helm upgrade --install jaeger jaegertracing/jaeger -n observability -f observability/jaeger-values.yaml

# 7. OpenTelemetry Operator
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace observability --wait

# 8. Instrumentation
kubectl apply -f observability/instrumentation-prod.yaml

# 9. Collector
helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  -n observability -f observability/collector-prod.yaml



kubectl patch deployment backend -n <your-app-ns> -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-nodejs": "observability/"
        }
      }
    }
  }
}'
kubectl patch deployment frontend -n <your-app-ns> -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-java": "observability/"
        }
      }
    }
  }
}'

Jaeger: kubectl port-forward svc/jaeger-query -n observability 16686:16686
Kibana: kubectl port-forward svc/kibana-kibana -n observability 5601:5601
```
````
