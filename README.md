# Helm chart for geoserver-cloud

![Version: 0.0.45](https://img.shields.io/badge/Version-0.0.45-informational?style=flat-square) ![AppVersion: 1.0-RC26](https://img.shields.io/badge/AppVersion-1.0--RC26-informational?style=flat-square)

A Helm chart for Geoserver

This is a fork to keep track of what I am doing to get this running on a fedora 36 box, based on the repo at https://github.com/camptocamp/helm-geoserver-cloud, and the principle of using the most unfamiliar path possible to see how far I could get and see what I could learn.

Dependencies: Using (dnf)[https://docs.fedoraproject.org/en-US/quick-docs/dnf-system-upgrade/], I brought a workstation up to fedora 36 and followed the docker (install instructions)[https://docs.docker.com/engine/install/fedora/]. k3d install and kubectl instuctions below worked fine and I can recommend them

I had to take a bit of time setting up and learning  about helm: (quickststart)[https://helm.sh/docs/intro/quickstart/]


## NOT YET WORKING DO NOT FOLLOW ##
The geoserver is not yet working, please do not follow these directions without preparing yourself for a bit of fustration or delay.
The kubernetes cluster is up and the external dependecies are running correctly, but I have a CrashLoopBackOff error for all the geoserver-cloud specific services:

    geoserver-postgresql-0                             1/1     Running            0                 12h
    geoserver-geoservercloud-gateway-f7bb958f8-jn8dc   1/1     Running            0                 12h
    geoserver-rabbitmq-0                               1/1     Running            0                 12h
    geoserver-geoservercloud-wms-6bf49d779-8b2v7       0/1     CrashLoopBackOff   144 (4m43s ago)   12h
    geoserver-geoservercloud-gwc-5c756b99cf-ltlj6      0/1     CrashLoopBackOff   144 (4m39s ago)   12h
    geoserver-geoservercloud-rest-67b84569f4-pzzkr     0/1     CrashLoopBackOff   145 (4m2s ago)    12h
    geoserver-geoservercloud-wfs-9ff77756c-nc797       0/1     CrashLoopBackOff   145 (2m28s ago)   12h
    geoserver-geoservercloud-webui-c45b795fb-4r9bj     0/1     CrashLoopBackOff   146 (101s ago)    12h
    geoserver-geoservercloud-wcs-7c4d95d9fc-dk98b      0/1     CrashLoopBackOff   146 (78s ago)     12h

So of course the web ui returns a 404:

    wget http://localhost:8085/geoserver-cloud/web/ 

    [root@fedora kevin]# wget http://localhost:8085/geoserver-cloud/web/
    --2022-11-08 07:00:05--  http://localhost:8085/geoserver-cloud/web/
    Resolving localhost (localhost)... ::1, 127.0.0.1
    Connecting to localhost (localhost)|::1|:8085... connected.
    HTTP request sent, awaiting response... 404 Not Found
    2022-11-08 07:00:05 ERROR 404: Not Found.



## include this chart as dependency of your own chart:

just include the following section in your `Chart.yaml`:

```yaml
dependencies:
  - name: geoservercloud
    repository: https://camptocamp.github.io/helm-geoserver-cloud
    version: <version-numer-here>
```

See the value file for configuration options.

## Developing on geoserver-cloud code using this chart

To develop in this chart, we recommend that you use `k3d` if you want to use your machine. Please follow the following steps to ensure that you have all the requirements for development:

### install kubectl

```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
```

or follow the steps of [the official documentation](https://v1-18.docs.kubernetes.io/docs/tasks/tools/install-kubectl/)

### install helm (version 3!!!!)

follow the steps of the [official documentation](https://helm.sh/docs/intro/install/)

### install k3d

```bash
wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

### create a cluster on your machine

The following configuration will create a single-node "cluster" on your machine, the kubelet parameters ensure that it works also if you have a development PC with a disk almost full. It will also bind the port `localhost:8085` of your machine with the ingress controller port `80`, just adapt accordingly to your needs. You also need a local image registry to push your local images into the cluster.

```bash
# create a local registry
k3d registry create registry.localhost --port 5000

# create a local cluster and enable the registry
k3d cluster create k3d-cluster-1 --k3s-arg '--kubelet-arg=eviction-soft-grace-period=imagefs.available=60s,nodefs.available=60s@all'  --k3s-arg '--kubelet-arg=eviction-hard=imagefs.available<10Mi,nodefs.available<10Mi@all' --k3s-arg '--kubelet-arg=eviction-minimum-reclaim=imagefs.available=10Mi,nodefs.available=10Mi@all'  -p "8085:80@loadbalancer"
```


### check that the configuration is available

This repository provides a default configuration thanks to a git submodule : if you don't have performed a recursive clone of the repository, don't forget to do a `git submodule update` before installing the chart !

### start the chart

```bash
# update the dependencies the first time as well
helm dependency update
# then install the chart
helm install geoserver .
```

### test that it is working.

If everything went well you should see something similar with kubectl:

```bash
~# kubectl get po
NAME                                   READY   STATUS    RESTARTS   AGE
geoserver-postgresql-0                 1/1     Running   0          6m17s
geoserver-rabbitmq-0                   1/1     Running   0          6m17s
geoserver-gateway-7d99c5c864-wpbt4     1/1     Running   0          6m17s
geoserver-webui-67dfc967d7-x8cx8       1/1     Running   0          6m17s
geoserver-rest-577ddb7854-pjhv2        1/1     Running   0          6m17s
geoserver-wms-7798c78cfd-ngwdm         1/1     Running   0          6m17s
geoserver-wcs-748c9fb64b-flxts         1/1     Running   0          6m17s
geoserver-wfs-65947f5d4d-88g26         1/1     Running   0          6m17s
```

and you should be able to access the webui on http://localhost:8085/geoserver-cloud/web/

### use a local docker image and import it into the cluster

1. build your image, tag and name it:

```bash
docker build -t k3d-registry.localhost:5000/geoserver-cloud-webui:test .
```

And push it to the localhost registry

```bash
docker push k3d-registry.localhost:5000/geoserver-cloud-webui:test
```

and then upgrade (or start) the chart with the following argument:

```bash
helm upgrade --set geoserver.webui.image.repository=k3d-registry.localhost:5000/geoserver-cloud-webui --set geoserver.webui.image.tag=test geoserver .
```

NB: this should not be the case with modern OS, but you might need to add the following to your `/etc/hosts` configuration:

```
127.0.0.1 k3d-registry.localhost
```

## Requirements

| Repository                         | Name       | Version |
| ---------------------------------- | ---------- | ------- |
| https://charts.bitnami.com/bitnami | postgresql | 10.13.8 |
| https://charts.bitnami.com/bitnami | rabbitmq   | 8.0.1   |

## Values

| Key                                                   | Type   | Default                                      | Description |
| ----------------------------------------------------- | ------ | -------------------------------------------- | ----------- |
| geoserver.config.affinity                             | object | `{}`                                         |             |
| geoserver.config.env                                  | list   | `[]`                                         |             |
| geoserver.config.image.repository                     | string | `"geoservercloud/geoserver-cloud-config"`    |             |
| geoserver.config.image.tag                            | string | `""`                                         |             |
| geoserver.config.nodeSelector                         | object | `{}`                                         |             |
| geoserver.config.podAnnotations                       | object | `{}`                                         |             |
| geoserver.config.podSecurityContext.runAsUser         | int    | `630`                                        |             |
| geoserver.config.replicaCount                         | int    | `1`                                          |             |
| geoserver.config.resources.limits.cpu                 | float  | `2`                                          |             |
| geoserver.config.resources.limits.memory              | string | `"512Mi"`                                    |             |
| geoserver.config.resources.requests.cpu               | float  | `0.1`                                        |             |
| geoserver.config.resources.requests.memory            | string | `"128Mi"`                                    |             |
| geoserver.config.securityContext                      | object | `{}`                                         |             |
| geoserver.config.service.port                         | int    | `8080`                                       |             |
| geoserver.config.service.type                         | string | `"ClusterIP"`                                |             |
| geoserver.config.tolerations                          | object | `{}`                                         |             |
| geoserver.database.dropOnDelete                       | bool   | `false`                                      |             |
| geoserver.database.extensions[0]                      | string | `"pg_stat_statements"`                       |             |
| geoserver.database.secretConfig                       | string | `"jdbcconfig"`                               |             |
| geoserver.discovery.affinity                          | object | `{}`                                         |             |
| geoserver.discovery.env                               | list   | `[]`                                         |             |
| geoserver.discovery.image.repository                  | string | `"geoservercloud/geoserver-cloud-discovery"` |             |
| geoserver.discovery.image.tag                         | string | `""`                                         |             |
| geoserver.discovery.nodeSelector                      | object | `{}`                                         |             |
| geoserver.discovery.podAnnotations                    | object | `{}`                                         |             |
| geoserver.discovery.podSecurityContext.runAsUser      | int    | `630`                                        |             |
| geoserver.discovery.replicaCount                      | int    | `1`                                          |             |
| geoserver.discovery.resources.limits.cpu              | float  | `2`                                          |             |
| geoserver.discovery.resources.limits.memory           | string | `"512Mi"`                                    |             |
| geoserver.discovery.resources.requests.cpu            | float  | `0.1`                                        |             |
| geoserver.discovery.resources.requests.memory         | string | `"128Mi"`                                    |             |
| geoserver.discovery.securityContext                   | object | `{}`                                         |             |
| geoserver.discovery.service.port                      | int    | `8761`                                       |             |
| geoserver.discovery.service.type                      | string | `"ClusterIP"`                                |             |
| geoserver.discovery.tolerations                       | object | `{}`                                         |             |
| geoserver.envVariables                                | object | `{}`                                         |             |
| geoserver.gateway.affinity                            | object | `{}`                                         |             |
| geoserver.gateway.env                                 | list   | `[]`                                         |             |
| geoserver.gateway.image.repository                    | string | `"geoservercloud/geoserver-cloud-gateway"`   |             |
| geoserver.gateway.image.tag                           | string | `""`                                         |             |
| geoserver.gateway.nodeSelector                        | object | `{}`                                         |             |
| geoserver.gateway.podAnnotations                      | object | `{}`                                         |             |
| geoserver.gateway.podSecurityContext.runAsUser        | int    | `630`                                        |             |
| geoserver.gateway.replicaCount                        | int    | `1`                                          |             |
| geoserver.gateway.resources.limits.cpu                | float  | `2`                                          |             |
| geoserver.gateway.resources.limits.memory             | string | `"512Mi"`                                    |             |
| geoserver.gateway.resources.requests.cpu              | float  | `0.1`                                        |             |
| geoserver.gateway.resources.requests.memory           | string | `"128Mi"`                                    |             |
| geoserver.gateway.securityContext                     | object | `{}`                                         |             |
| geoserver.gateway.service.port                        | int    | `8080`                                       |             |
| geoserver.gateway.service.type                        | string | `"ClusterIP"`                                |             |
| geoserver.gateway.tolerations                         | object | `{}`                                         |             |
| geoserver.ingress.baseUrl                             | string | `"/geoserver-cloud/"`                        |             |
| geoserver.ingress.deploy                              | bool   | `true`                                       |             |
| geoserver.jdbc.external                               | bool   | `false`                                      |             |
| geoserver.jdbc.postgresqlDatabase.secretKey           | string | `""`                                         |             |
| geoserver.jdbc.postgresqlDatabase.secretName          | string | `""`                                         |             |
| geoserver.jdbc.postgresqlDatabase.value               | string | `""`                                         |             |
| geoserver.jdbc.postgresqlHostName.secretKey           | string | `""`                                         |             |
| geoserver.jdbc.postgresqlHostName.secretName          | string | `""`                                         |             |
| geoserver.jdbc.postgresqlHostName.value               | string | `""`                                         |             |
| geoserver.jdbc.postgresqlPassword.secretKey           | string | `""`                                         |             |
| geoserver.jdbc.postgresqlPassword.secretName          | string | `""`                                         |             |
| geoserver.jdbc.postgresqlPassword.value               | string | `""`                                         |             |
| geoserver.jdbc.postgresqlPort.secretKey               | string | `""`                                         |             |
| geoserver.jdbc.postgresqlPort.secretName              | string | `""`                                         |             |
| geoserver.jdbc.postgresqlPort.value                   | string | `""`                                         |             |
| geoserver.jdbc.postgresqlUsername.secretKey           | string | `""`                                         |             |
| geoserver.jdbc.postgresqlUsername.secretName          | string | `""`                                         |             |
| geoserver.jdbc.postgresqlUsername.value               | string | `""`                                         |             |
| geoserver.name                                        | string | `"geoserver"`                                |             |
| geoserver.rabbitmq.port                               | int    | `5672`                                       |             |
| geoserver.serviceAccount.annotations                  | object | `{}`                                         |             |
| geoserver.serviceAccount.create                       | bool   | `true`                                       |             |
| geoserver.serviceAccount.name                         | string | `""`                                         |             |
| geoserver.services.rest.affinity                      | object | `{}`                                         |             |
| geoserver.services.rest.env                           | list   | `[]`                                         |             |
| geoserver.services.rest.image.repository              | string | `"geoservercloud/geoserver-cloud-rest"`      |             |
| geoserver.services.rest.image.tag                     | string | `""`                                         |             |
| geoserver.services.rest.nodeSelector                  | object | `{}`                                         |             |
| geoserver.services.rest.podAnnotations                | object | `{}`                                         |             |
| geoserver.services.rest.podSecurityContext.runAsUser  | int    | `630`                                        |             |
| geoserver.services.rest.replicaCount                  | int    | `1`                                          |             |
| geoserver.services.rest.resources.limits.cpu          | float  | `2`                                          |             |
| geoserver.services.rest.resources.limits.memory       | string | `"2Gi"`                                      |             |
| geoserver.services.rest.resources.requests.cpu        | float  | `0.1`                                        |             |
| geoserver.services.rest.resources.requests.memory     | string | `"512Mi"`                                    |             |
| geoserver.services.rest.securityContext               | object | `{}`                                         |             |
| geoserver.services.rest.service.port                  | int    | `8080`                                       |             |
| geoserver.services.rest.service.type                  | string | `"ClusterIP"`                                |             |
| geoserver.services.rest.tolerations                   | object | `{}`                                         |             |
| geoserver.services.wcs.affinity                       | object | `{}`                                         |             |
| geoserver.services.wcs.env                            | list   | `[]`                                         |             |
| geoserver.services.wcs.image.repository               | string | `"geoservercloud/geoserver-cloud-wcs"`       |             |
| geoserver.services.wcs.image.tag                      | string | `""`                                         |             |
| geoserver.services.wcs.nodeSelector                   | object | `{}`                                         |             |
| geoserver.services.wcs.podAnnotations                 | object | `{}`                                         |             |
| geoserver.services.wcs.podSecurityContext.runAsUser   | int    | `630`                                        |             |
| geoserver.services.wcs.replicaCount                   | int    | `1`                                          |             |
| geoserver.services.wcs.resources.limits.cpu           | float  | `4`                                          |             |
| geoserver.services.wcs.resources.limits.memory        | string | `"4Gi"`                                      |             |
| geoserver.services.wcs.resources.requests.cpu         | float  | `0.1`                                        |             |
| geoserver.services.wcs.resources.requests.memory      | string | `"512Mi"`                                    |             |
| geoserver.services.wcs.securityContext                | object | `{}`                                         |             |
| geoserver.services.wcs.service.port                   | int    | `8080`                                       |             |
| geoserver.services.wcs.service.type                   | string | `"ClusterIP"`                                |             |
| geoserver.services.wcs.tolerations                    | object | `{}`                                         |             |
| geoserver.services.webui.affinity                     | object | `{}`                                         |             |
| geoserver.services.webui.env                          | list   | `[]`                                         |             |
| geoserver.services.webui.image.repository             | string | `"geoservercloud/geoserver-cloud-webui"`     |             |
| geoserver.services.webui.image.tag                    | string | `""`                                         |             |
| geoserver.services.webui.nodeSelector                 | object | `{}`                                         |             |
| geoserver.services.webui.podAnnotations               | object | `{}`                                         |             |
| geoserver.services.webui.podSecurityContext.runAsUser | int    | `630`                                        |             |
| geoserver.services.webui.replicaCount                 | int    | `1`                                          |             |
| geoserver.services.webui.resources.limits.cpu         | float  | `2`                                          |             |
| geoserver.services.webui.resources.limits.memory      | string | `"512Mi"`                                    |             |
| geoserver.services.webui.resources.requests.cpu       | float  | `0.1`                                        |             |
| geoserver.services.webui.resources.requests.memory    | string | `"128Mi"`                                    |             |
| geoserver.services.webui.securityContext              | object | `{}`                                         |             |
| geoserver.services.webui.service.port                 | int    | `8080`                                       |             |
| geoserver.services.webui.service.type                 | string | `"ClusterIP"`                                |             |
| geoserver.services.webui.tolerations                  | object | `{}`                                         |             |
| geoserver.services.wfs.affinity                       | object | `{}`                                         |             |
| geoserver.services.wfs.env                            | list   | `[]`                                         |             |
| geoserver.services.wfs.image.repository               | string | `"geoservercloud/geoserver-cloud-wfs"`       |             |
| geoserver.services.wfs.image.tag                      | string | `""`                                         |             |
| geoserver.services.wfs.nodeSelector                   | object | `{}`                                         |             |
| geoserver.services.wfs.podAnnotations                 | object | `{}`                                         |             |
| geoserver.services.wfs.podSecurityContext.runAsUser   | int    | `630`                                        |             |
| geoserver.services.wfs.replicaCount                   | int    | `1`                                          |             |
| geoserver.services.wfs.resources.limits.cpu           | float  | `4`                                          |             |
| geoserver.services.wfs.resources.limits.memory        | string | `"4Gi"`                                      |             |
| geoserver.services.wfs.resources.requests.cpu         | float  | `0.1`                                        |             |
| geoserver.services.wfs.resources.requests.memory      | string | `"512Mi"`                                    |             |
| geoserver.services.wfs.securityContext                | object | `{}`                                         |             |
| geoserver.services.wfs.service.port                   | int    | `8080`                                       |             |
| geoserver.services.wfs.service.type                   | string | `"ClusterIP"`                                |             |
| geoserver.services.wfs.tolerations                    | object | `{}`                                         |             |
| geoserver.services.wms.affinity                       | object | `{}`                                         |             |
| geoserver.services.wms.env                            | list   | `[]`                                         |             |
| geoserver.services.wms.image.repository               | string | `"geoservercloud/geoserver-cloud-wms"`       |             |
| geoserver.services.wms.image.tag                      | string | `""`                                         |             |
| geoserver.services.wms.nodeSelector                   | object | `{}`                                         |             |
| geoserver.services.wms.podAnnotations                 | object | `{}`                                         |             |
| geoserver.services.wms.podSecurityContext.runAsUser   | int    | `630`                                        |             |
| geoserver.services.wms.replicaCount                   | int    | `1`                                          |             |
| geoserver.services.wms.resources.limits.cpu           | float  | `4`                                          |             |
| geoserver.services.wms.resources.limits.memory        | string | `"4Gi"`                                      |             |
| geoserver.services.wms.resources.requests.cpu         | float  | `0.1`                                        |             |
| geoserver.services.wms.resources.requests.memory      | string | `"512Mi"`                                    |             |
| geoserver.services.wms.securityContext                | object | `{}`                                         |             |
| geoserver.services.wms.service.port                   | int    | `8080`                                       |             |
| geoserver.services.wms.service.type                   | string | `"ClusterIP"`                                |             |
| geoserver.services.wms.tolerations                    | object | `{}`                                         |             |
| geoserver.volumeMounts                                | list   | `[]`                                         |             |
| global.annotations                                    | object | `{}`                                         |             |
| global.app_environment_name                           | string | `"master"`                                   |             |
| global.base_environment_name                          | string | `"dev"`                                      |             |
| global.cluster_environment                            | string | `"dev"`                                      |             |
| global.cors.enabled                                   | bool   | `true`                                       |             |
| global.custom_pod_labels                              | object | `{}`                                         |             |
| global.deployed_by                                    | string | `"helm"`                                     |             |
| global.image.pullPolicy                               | string | `"IfNotPresent"`                             |             |
| global.image.registry                                 | string | `"docker.io"`                                |             |
| global.revision                                       | string | `"HEAD"`                                     |             |
| postgresql.enabled                                    | bool   | `true`                                       |             |
| postgresql.fromSecret.enabled                         | bool   | `false`                                      |             |
| postgresql.persistence.enabled                        | bool   | `false`                                      |             |
| postgresql.postgresqlDatabase                         | string | `"config"`                                   |             |
| postgresql.postgresqlPassword                         | string | `"password"`                                 |             |
| postgresql.postgresqlUsername                         | string | `"username"`                                 |             |
| postgresql.servicePort                                | string | `"5432"`                                     |             |
| postgresql.tls.autoGenerated                          | bool   | `true`                                       |             |
| postgresql.tls.enabled                                | bool   | `true`                                       |             |
| postgresql.volumePermissions.enabled                  | bool   | `true`                                       |             |
| rabbitmq.auth.username                                | string | `"geoserver"`                                |             |
| rabbitmq.image.registry                               | string | `"docker.io"`                                |             |
| rabbitmq.image.repository                             | string | `"bitnami/rabbitmq"`                         |             |
| rabbitmq.image.tag                                    | string | `"3.8.9-debian-10-r37"`                      |             |
| rabbitmq.persistence.enabled                          | bool   | `false`                                      |             |
| rabbitmq.resources.limits.cpu                         | int    | `2`                                          |             |
| rabbitmq.resources.limits.memory                      | string | `"2Gi"`                                      |             |
| rabbitmq.resources.requests.cpu                       | float  | `0.1`                                        |             |
| rabbitmq.resources.requests.memory                    | string | `"512Mi"`                                    |             |

---

Autogenerated from chart metadata using [helm-docs v1.7.0](https://github.com/norwoodj/helm-docs/releases/v1.7.0)
