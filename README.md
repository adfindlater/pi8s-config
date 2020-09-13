# pi8s cluster setup

![](https://github.com/adfindlater/pi8s-config/blob/master/images/pi8s-logo.png?raw=true)
<img src="https://github.com/adfindlater/pi8s-config/blob/master/images/pi8s-logo.png?raw=true width="200" height="200">
[<img src="https://github.com/adfindlater/pi8s-config/blob/master/images/pi8s-logo.png?raw=true" width="250"/>]("https://github.com/adfindlater/pi8s-config/blob/master/images/pi8s-logo.png?raw=true)
Deploy a kubernetes cluster across two or more raspberry pi's using Terraform.  This repository also contains config and instructions for deploying:
1. on-cluster load balancer (metallb)
2. Argo CD continuous delivery tool
3. centralized logging (ELK stack)

The deployment strategy currently taken is to patch helm charts using `helm install -f myvalues.yaml` when it is convenient.  Otherwise, explicit deployment
yaml is used.  Helm charts require patching because the x86 images need to be swapped out for corresponding arm64 images (as well as any other custom config that is required).

Requirements:
- Each raspberry pi has an SD card flashed with Ubuntu 20.04 ARM64
- The ubuntu user password is the same on each raspberry pi

## Prepare nodes and create cluster using Terraform

Clone the [terraform-raspberrypi-bootstrap](https://github.com/adfindlater/terraform-raspberrypi-bootstrap) repository and follow the instructions in the README to provision nodes and create a new k8s cluster.
```console
$ git clone git@github.com:adfindlater/terraform-raspberrypi-bootstrap.git
```

After running `terraform_pis.sh`, verify the existence of the new k8s cluster by coping `~/terraform-raspberrypi-bootstrap/control-plane/kube_config` to `~/.kube/config` and then running

```console
$ kubectl get pods --all-namespaces
```

## Deploy bare-metal load balancer

Deploy a load balancer to the cluster using helm.  Custom configuration values are provided by the `myvalues.yaml` file.
```console
$ helm install -f pi8s-config/load-balancer/helm/myvalues.yaml metallb bitnami/metallb
```

`myvalues.yaml`
```yaml
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

## Argo CD 

```console
$ helm install -f pi8s-config/argo-cd/helm/myvalues.yaml argo-cd argo/argo-cd
```

`myvalues.yaml`
```yaml
global:
  image:
    # https://github.com/argoproj/argo-cd/issues/2167#issuecomment-584936692
    repository: phillebaba/argocd
    tag: v1.3.6

server:
  service:
    type: LoadBalancer
```

## Centralized logging with the ELK stack

### Elasticsearch
TODO: switch to helm chart

Create a persistent volume in the same namespace that the ELK stack will be deployed. 
Make sure that k8s can write/read to the host path `/mnt/data` on the worker nodes.

```console
$ kubectl apply -f pi8s-config/elk/pv.yaml -n kube-system
```

Deploy Elasticsearch to the cluster.
```console
$ kubectl apply -f pi8s-config/elk/es/yaml -n kube-system
```


### Filebeat

Deploy using helm with customized values.

`myvalues.yaml`
```yaml
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

```console
$ helm install -f pi8s-config/elk/filebeat/helm/myvalues.yaml filebeat elastic/filebeat -n kube-system
```


### Logstash
TODO: switch to helm chart

```
kubectl apply -f pi8s-config/elk/logstash/yaml -n kube-system
```

### Kibana

`myvalues.yaml`
```yaml
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

```console
$ helm install -f pi8s-config/elk/kibana/helm/myvalues.yaml kibana elastic/kibana -n kube-system
```


