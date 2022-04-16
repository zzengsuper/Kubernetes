# Helm Chart

1. Show values of helm chart and save it to yaml files

```shell
helm show values grafana/loki-stack > /tmp/loki-stack-values.yaml
```

2. Install with the value yaml file

   ```shell
   helm install loki-stack grafana/loki-stack --values /tmp/loki-stack-values.yaml -n loki --create-namespace
   ```

3. Check helm install list

   ```shell
   helm list -n loki
   ```

