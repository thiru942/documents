# Elastic Cloud on Kubernetes (ECK Stack)
This is a straightforward guide on deploying Elastic Stack on Kubernetes. In this guide, we’ll be using the ECK operator for creating all the Elastic related resources on Kubernetes.

Followings are the outcomes of this guide:

- An Elasticsearch cluster.
- A Kibana dashboard integrated with the ES cluster.
- Filebeat agents to ship logs of the host K8s cluster to the ES cluster. (The K8s cluster and the ES - cluster are two different clusters)
- Optionally, ingress to receive logs from outside the host cluster.

## Prerequisites:
- A Kubernetes cluster with at least 2 vCPUs and 4GB RAM per worker node. (An Elasticsearch pod requires a minimum of 1 vCPU and 2GB of RAM).
- An Ingress controller installed on the cluster. (Only if you need to expose the services via Ingress).
- Helm installed on your computer.

## Step 1 — Install the ECK operator CRDs
```
kubectl create -f https://download.elastic.co/downloads/eck/2.12.1/crds.yaml
```
The following Elastic resources have been created:
```
customresourcedefinition.apiextensions.k8s.io/agents.agent.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/apmservers.apm.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/beats.beat.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/elasticmapsservers.maps.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/elasticsearches.elasticsearch.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/enterprisesearches.enterprisesearch.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/kibanas.kibana.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/logstashes.logstash.k8s.elastic.co created
```
## Step 2 - Install the operator with its RBAC rules:
```
kubectl apply -f https://download.elastic.co/downloads/eck/2.12.1/operator.yaml
```

> The ECK operator runs by default in the elastic-system namespace. It is recommended that you choose a dedicated namespace for your workloads, rather than using the elastic-system or the default namespace.
{.is-info}

## Monitor the operator logs:
```
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```
## Step 3 - Create Elasticsearch, Kibana and Filebeat auto discover manifest file.
```
vim eck_filebeat_autodiscover.yml
```
```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: filebeat
spec:
  type: filebeat
  version: 8.12.1
  elasticsearchRef:
    name: elasticsearch
  kibanaRef:
    name: kibana
  config:
    filebeat:
      autodiscover:
        providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints:
            enabled: true
            default_config:
              type: container
              paths:
              - /var/log/containers/*${data.kubernetes.container.id}.log
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: filebeat
        automountServiceAccountToken: true
        terminationGracePeriodSeconds: 30
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true # Allows to provide richer host metadata
        containers:
        - name: filebeat
          securityContext:
            runAsUser: 0
            # If using Red Hat OpenShift uncomment this:
            #privileged: true
          volumeMounts:
          - name: varlogcontainers
            mountPath: /var/log/containers
          - name: varlogpods
            mountPath: /var/log/pods
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
        volumes:
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups: ["apps"]
  resources:
  - replicasets
  verbs:
  - get
  - list
  - watch
- apiGroups: ["batch"]
  resources:
  - jobs
  verbs:
  - get
  - list
  - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: logging
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 8.12.1
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: -Xms2g -Xmx2g
          resources:
            requests:
              memory: 4Gi
              cpu: 4
            limits:
              memory: 4Gi
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: local-path
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
spec:
  version: 8.12.1
  count: 1
  elasticsearchRef:
    name: elasticsearch
---

...

```
We'll install stack on namespace logging, create namespace first
```
kubectl create ns logging
```
Apply above manifest to create eck stack
```
kubectl apply -f eck_filebeat_autodiscover.yml
```
## Deploying Metricbeat
We'll deply Matricbeat to monitor k8s cluster resources.
Create following matricbeat manifests
```
vim metricbeat.yml
```
```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
  namespace: logging
spec:
  type: metricbeat
  version: 8.12.1
  elasticsearchRef:
    name: elasticsearch
  kibanaRef:
    name: kibana
  config:
    metricbeat:
      autodiscover:
        providers:
        - hints:
            default_config: {}
            enabled: "true"
          node: ${NODE_NAME}
          type: kubernetes
      modules:
      - module: system
        period: 10s
        metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
        process:
          include_top_n:
            by_cpu: 5
            by_memory: 5
        processes:
        - .*
      - module: system
        period: 1m
        metricsets:
        - filesystem
        - fsstat
        processors:
        - drop_event:
            when:
              regexp:
                system:
                  filesystem:
                    mount_point: ^/(sys|cgroup|proc|dev|etc|host|lib)($|/)
      - module: kubernetes
        period: 10s
        node: ${NODE_NAME}
        hosts:
        - https://${NODE_NAME}:10250
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        ssl:
          verification_mode: none
        metricsets:
        - node
        - system
        - pod
        - container
        - volume
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true # some older Beat versions are depending on this settings presence in k8s context
        containers:
        - args:
          - -e
          - -c
          - /etc/beat.yml
          - -system.hostfs=/hostfs
          name: metricbeat
          volumeMounts:
          - mountPath: /hostfs/sys/fs/cgroup
            name: cgroup
          - mountPath: /var/run/docker.sock
            name: dockersock
          - mountPath: /hostfs/proc
            name: proc
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true # Allows to provide richer host metadata
        securityContext:
          runAsUser: 0
        terminationGracePeriodSeconds: 30
        volumes:
        - hostPath:
            path: /sys/fs/cgroup
          name: cgroup
        - hostPath:
            path: /var/run/docker.sock
          name: dockersock
        - hostPath:
            path: /proc
          name: proc
---
# permissions needed for metricbeat
# source: https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-kubernetes.html
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metricbeat
  namespace: logging
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - namespaces
  - events
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
  - replicasets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  - deployments
  - replicasets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metricbeat
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metricbeat
  namespace: logging
subjects:
- kind: ServiceAccount
  name: metricbeat
  namespace: logging
roleRef:
  kind: ClusterRole
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
---
```
Apply Metricbeat manifests to deploy
```
kubectl apply -f metricmeat.yml
```
## Ingress
I'd like to expose elasticsearch and kibana via ingress so I can push logs and metrics from other clusters, dockers and hosts.

I'm using letsencrypt staging certificate for https traffic but I again route my all http traffic from my main Treafik instance on docker which has prod certificate from letsencrypt.

First we'll create certificate request for my domain from cert-manager for logging namespace.
```
vim tritec_tls.yml
```
```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: staging-tritec-in
  namespace: logging
spec:
  secretName: tritec-in-staging-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.tritec.in"
  dnsNames:
  - "tritec.in"
  - "*.tritec.in"
```
Now Lets create ingress manifests form elastic search and kibana
```
vim elastic_ingress.yml
```
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elastic-ingress
  namespace: logging
spec:
  ingressClassName: traefik
  tls:
  - hosts:
      - elastic.tritec.in
    secretName: tritec-in-staging-tls
  rules:
  - host: elastic.tritec.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-es-http
            port:
              number: 9200
```
```
vim kibana_ingress.yml
```
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: logging
spec:
  ingressClassName: traefik
  tls:
  - hosts:
      - kibana.tritec.in
    secretName: tritec-in-staging-tls
  rules:
  - host: kibana.tritec.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana-kb-http
            port:
              number: 5601
```
apply manifests to create ingress
```
kubectl apply -f elastic_ingress.yml
kubectl apply -f kibana_ingress.yml
```
## External Kubernetes Cluster Monitoring and Logs via Filebeat and Metricbeat
To injest logs and metrics from external kubernetes cluster, we need to create following manifests
### Filebeat
```
vim hulk_filebit.yml
```
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups: ["apps"]
  resources:
    - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources:
    - jobs
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: filebeat
  # should be the namespace where filebeat is running
  namespace: kube-system
  labels:
    k8s-app: filebeat
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs: ["get", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: filebeat-kubeadm-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - kubeadm-config
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: filebeat
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: kube-system
roleRef:
  kind: Role
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: filebeat-kubeadm-config
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: kube-system
roleRef:
  kind: Role
  name: filebeat-kubeadm-config
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: filestream
      id: kubernetes-container-logs
      paths:
        - /var/log/containers/*.log
      parsers:
        - container: ~
      prospector:
        scanner:
          fingerprint.enabled: true
          symlinks: true
      file_identity.fingerprint: ~
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

    # To enable hints based autodiscover, remove `filebeat.inputs` configuration and uncomment this:
    # filebeat.autodiscover:
    #  providers:
    #    - type: kubernetes
    #      node: ${NODE_NAME}
    #      hints.enabled: true
    #      hints.default_config:
    #        type: filestream
    #        id: kubernetes-container-logs-${data.kubernetes.pod.name}-${data.kubernetes.container.id}
    #        paths:
    #        - /var/log/containers/*-${data.kubernetes.container.id}.log
    #        parsers:
    #        - container: ~
    #        prospector:
    #         scanner:
    #           fingerprint.enabled: true
    #           symlinks: true
    #        file_identity.fingerprint: ~

    processors:
      - add_cloud_metadata:
      - add_host_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.13.2
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: https://elastic.tritec.in
        - name: ELASTICSEARCH_PORT
          value: "443"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: autogeneratedelasticpassword
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          # If using Red Hat OpenShift uncomment this:
          #privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0640
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          # When filebeat runs as non-root user, this directory needs to be writable by group (g+w).
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
```
As you can see we have changed elastic environment variables to connect our existing ECK cluster
`following environment variables need to change only`
```
        env:
        - name: ELASTICSEARCH_HOST
          value: https://elastic.tritec.in
        - name: ELASTICSEARCH_PORT
          value: "443"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: autogeneratedelasticpassword
```
### Metricbeat
Same way we can create following manifests for metricbeat.
```
vim hulk_metricbeat.yml
```
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metricbeat
  namespace: kube-system
  labels:
    k8s-app: metricbeat
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metricbeat
  labels:
    k8s-app: metricbeat
rules:
- apiGroups: [""]
  resources:
  - nodes
  - namespaces
  - events
  - pods
  - services
  - persistentvolumes
  - persistentvolumeclaims
  verbs: ["get", "list", "watch"]
# Enable this rule only if planing to use Kubernetes keystore
#- apiGroups: [""]
#  resources:
#  - secrets
#  verbs: ["get"]
- apiGroups: ["extensions"]
  resources:
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - deployments
  - replicasets
  - daemonsets
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources:
    - storageclasses
  verbs: ["get", "list", "watch"]
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
- nonResourceURLs:
  - "/metrics"
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: metricbeat
  # should be the namespace where metricbeat is running
  namespace: kube-system
  labels:
    k8s-app: metricbeat
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs: ["get", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: metricbeat-kubeadm-config
  namespace: kube-system
  labels:
    k8s-app: metricbeat
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - kubeadm-config
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metricbeat
subjects:
- kind: ServiceAccount
  name: metricbeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metricbeat
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: metricbeat
    namespace: kube-system
roleRef:
  kind: Role
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metricbeat-kubeadm-config
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: metricbeat
    namespace: kube-system
roleRef:
  kind: Role
  name: metricbeat-kubeadm-config
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-daemonset-config
  namespace: kube-system
  labels:
    k8s-app: metricbeat
data:
  metricbeat.yml: |-
    metricbeat.config.modules:
      # Mounted `metricbeat-daemonset-modules` configmap:
      path: ${path.config}/modules.d/*.yml
      # Reload module configs as they change:
      reload.enabled: false

    metricbeat.autodiscover:
      providers:
        - type: kubernetes
          scope: cluster
          node: ${NODE_NAME}
          # In large Kubernetes clusters consider setting unique to false
          # to avoid using the leader election strategy and
          # instead run a dedicated Metricbeat instance using a Deployment in addition to the DaemonSet
          unique: true
          templates:
            - config:
                - module: kubernetes
                  hosts: ["kube-state-metrics:8080"]
                  period: 10s
                  add_metadata: true
                  metricsets:
                    - state_namespace
                    - state_node
                    - state_deployment
                    - state_daemonset
                    - state_replicaset
                    - state_pod
                    - state_container
                    - state_job
                    - state_cronjob
                    - state_resourcequota
                    - state_statefulset
                    - state_service
                    - state_persistentvolume
                    - state_persistentvolumeclaim
                    - state_storageclass
                  # If `https` is used to access `kube-state-metrics`, uncomment following settings:
                  # bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
                  # ssl.certificate_authorities:
                  #   - /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
                - module: kubernetes
                  metricsets:
                    - apiserver
                  hosts: ["https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"]
                  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
                  ssl.certificate_authorities:
                    - /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                  period: 30s
                # Uncomment this to get k8s events:
                - module: kubernetes
                  metricsets:
                    - event
        # To enable hints based autodiscover uncomment this:
        - type: kubernetes
          node: ${NODE_NAME}
          hints.enabled: true

    processors:
      - add_cloud_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-daemonset-modules
  namespace: kube-system
  labels:
    k8s-app: metricbeat
data:
  system.yml: |-
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
        #- core
        #- diskio
        #- socket
      processes: ['.*']
      process.include_top_n:
        by_cpu: 5      # include top 5 processes by CPU
        by_memory: 5   # include top 5 processes by memory

    - module: system
      period: 1m
      metricsets:
        - filesystem
        - fsstat
      processors:
      - drop_event.when.regexp:
          system.filesystem.mount_point: '^/(sys|cgroup|proc|dev|etc|host|lib|snap)($|/)'
  kubernetes.yml: |-
    - module: kubernetes
      metricsets:
        - node
        - system
        - pod
        - container
        - volume
      period: 10s
      host: ${NODE_NAME}
      hosts: ["https://${NODE_NAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
      # If there is a CA bundle that contains the issuer of the certificate used in the Kubelet API,
      # remove ssl.verification_mode entry and use the CA, for instance:
      #ssl.certificate_authorities:
        #- /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
    - module: kubernetes
      metricsets:
        - proxy
      period: 10s
      host: ${NODE_NAME}
      hosts: ["localhost:10249"]
      # If using Red Hat OpenShift should be used this `hosts` setting instead:
      # hosts: ["localhost:29101"]
---
# Deploy a Metricbeat instance per node for node metrics retrieval
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metricbeat
  namespace: kube-system
  labels:
    k8s-app: metricbeat
spec:
  selector:
    matchLabels:
      k8s-app: metricbeat
  template:
    metadata:
      labels:
        k8s-app: metricbeat
    spec:
      serviceAccountName: metricbeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: metricbeat
        image: docker.elastic.co/beats/metricbeat:8.13.2
        args: [
          "-c", "/etc/metricbeat.yml",
          "-e",
          "-system.hostfs=/hostfs",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: https://elastic.tritec.in
        - name: ELASTICSEARCH_PORT
          value: "443"
        - name: ELASTICSEARCH_USERNAME
          value: "manish"
        - name: ELASTICSEARCH_PASSWORD
          value: "Dun1ya@007"
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/metricbeat.yml
          readOnly: true
          subPath: metricbeat.yml
        - name: data
          mountPath: /usr/share/metricbeat/data
        - name: modules
          mountPath: /usr/share/metricbeat/modules.d
          readOnly: true
        - name: proc
          mountPath: /hostfs/proc
          readOnly: true
        - name: cgroup
          mountPath: /hostfs/sys/fs/cgroup
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: cgroup
        hostPath:
          path: /sys/fs/cgroup
      - name: config
        configMap:
          defaultMode: 0640
          name: metricbeat-daemonset-config
      - name: modules
        configMap:
          defaultMode: 0640
          name: metricbeat-daemonset-modules
      - name: data
        hostPath:
          # When metricbeat runs as non-root user, this directory needs to be writable by group (g+w)
          path: /var/lib/metricbeat-data
          type: DirectoryOrCreate
---
---
```
`following environment variables need to change only`
```
        env:
        - name: ELASTICSEARCH_HOST
          value: https://elastic.tritec.in
        - name: ELASTICSEARCH_PORT
          value: "443"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: autogeneratedelasticpassword
```
## Adding Docker hosts to ECK
I'd like to monitor my docker host for metrics and logs
I'm going to use following config and docker-compose to create filebeat and metricbeat docker.

### Filebeat
create following filebit config yml file
```
vim filebeat.yml
```
```
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"
  - add_cloud_metadata: ~ 
#filebeat.config.modules:
#  path: ${path.config}/modules.d/*.yml
#  reload.enabled: false
 
#output.logstash:
#  hosts: ["192.168.16.226:5044"]
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["elastic.tritec.in:443"]

  # Performance preset - one of "balanced", "throughput", "scale",
  # "latency", or "custom".
  preset: balanced

  # Protocol - either `http` (default) or `https`.
  protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  username: "elastic"
  password: "autogeneratedelasticpass"

```
Create following docker-compose.yml file
```
name: filebeat
services:
    filebeat:
        container_name: filebeat
        user: root
        volumes:
            - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
            - /var/lib/docker/containers:/var/lib/docker/containers:ro
            - /var/run/docker.sock:/var/run/docker.sock:ro
        image: docker.elastic.co/beats/filebeat:7.9.2
        command: filebeat -e --strict.perms=false

```
Start docker compose
```
docker compose up -d
```
### Metricbeat
create following metricbeat config yml file
```
vim metricbeat.yml
```
```
metricbeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    # Reload module configs as they change:
    reload.enabled: false

metricbeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

metricbeat.modules:
- module: docker
  metricsets:
    - "container"
    - "cpu"
    - "diskio"
    - "healthcheck"
    - "info"
    #- "image"
    - "memory"
    - "network"
  hosts: ["unix:///var/run/docker.sock"]
  period: 10s
  enabled: true

processors:
  - add_cloud_metadata: ~

output.elasticsearch:
  hosts: 'https://elastic.tritec.in:443'
  username: 'elastic'
  password: 'autogeneratedelasticpass'
```
Create following docker-compose.yml file
```
services:
  metricbeat:
    image: docker.elastic.co/beats/metricbeat:8.13.2
    user: root
    hostname: "superman"
    configs:
      - source: metricbeat-config
        target: /usr/share/metricbeat/metricbeat.yml
    command:
      - -e
      - --strict.perms=false
      - --system.hostfs=/hostfs
    volumes:
      - type: bind
        source: /
        target: /hostfs
        read_only: true
      - type: bind
        source: /sys/fs/cgroup
        target: /hostfs/sys/fs/cgroup
        read_only: true
      - type: bind
        source: /proc
        target: /hostfs/proc
        read_only: true
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
        read_only: true
    deploy:
      mode: global

configs:
  metricbeat-config:
    file: ./metricbeat.yml
```
Start docker compose
```
docker compose up -d
```



