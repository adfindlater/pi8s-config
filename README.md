# pi8s cluster setup

## Prepare nodes and create cluster using Terraform

Clone the [terraform-raspberrypi-bootstrap](https://github.com/adfindlater/terraform-raspberrypi-bootstrap) repository and follow the instructions in the README to provision nodes and create a new k8s cluster.
```
git clone git@github.com:adfindlater/terraform-raspberrypi-bootstrap.git
```

After running `terraform_pis.sh`, to verify the existence of the new k8s cluster copy `~/terraform-raspberrypi-bootstrap/control-plane/kube_config` to `~/.kube/config` and then run

```
kubectl get pods --all-namespaces
```

## Deploy bare-metal load balancer

Deploy a load balancer to the cluster using helm.  Custom configuration values are provided by the `myvalues.yaml` file.
```
helm install -f pi8s-config/load-balancer/helm/myvalues.yaml metallb bitnami/metallb
```

`myvalues.yaml`
```
configInline: 
  address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.125-192.168.1.130

speaker:
  image:
    registry: docker.io
    repository: metallb/speaker
    tag: v0.9.3

controller:
  image:
    registry: docker.io
    repository: metallb/controller
    tag: v0.9.3
```

## Centralized logging with the ELK stack

### Elasticsearch (currenlty not using Helm)

Create a persistent volume in the same namespace that the ELK stack will be deployed. 
Make sure that k8s can write/read to the host path `/mnt/data` on the worker nodes.

```
kubectl apply -f pi8s-config/elk/pv.yaml -n kube-system
```

Deploy Elasticsearch to the cluster.
```
kubectl apply -f pi8s-config/elk/es/yaml -n kube-system
```


### Filebeat

Deploy using helm with customized values.

`myvalues.yaml`
```
image: "kasaoden/filebeat"
imageTag: "7.2.0-arm64"

filebeatConfig:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
    output.logstash:
      hosts: ["logstash-service:5044"]
```

```
helm install -f pi8s-config/elk/filebeat/helm/myvalues.yaml filebeat elastic/filebeat -n kube-system
```


### Logstash (currenlty not using Helm)

```
kubectl apply -f pi8s-config/elk/logstash/yaml -n kube-system
```

### Kibana

`myvalues.yaml`
```
image: "gagara/kibana-oss-arm64"
imageTag: "7.6.2"

resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "500m"
    memory: "1Gi"

elasticsearchHosts: "http://elasticsearch-master:9200"

service:
  type: LoadBalancer
```

```
helm install -f pi8s-config/elk/kibana/helm/myvalues.yaml kibana elastic/kibana -n kube-system
```


