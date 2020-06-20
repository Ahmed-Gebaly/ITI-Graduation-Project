# Monitoring On Kubernetes Cluster using Prometheus

## Prometheus is a free software application used for event monitoring and alerting

### Our setup consists of 4 parts
<!-- OL -->
* Prometheus
* Kube State Metrics
* Alert Manager
* Grafana
  
<!-- Images -->
![drawio](https://drive.google.com/file/d/1cUlj5R3cgwmMCCOBII-LVjg5mSF_WTKr/view?usp=sharing)
# Prometheus

## 1. Create a Namespace

```bash
$ kubectl create namespace monitoring
```

## 2. Create ClusterRole with read permissions to all cluster objects

```bash
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

## 3. Assign ClusterRoleBinding to monitoring namespace

```bash
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
  ```

## 4. Create a Config Map with all the prometheus config and alerting rules

* Alerting Rules
  
```bash
  prometheus.rules: |-
    groups:
    - name: alert group
      rules:

      - alert: Target Inctance Down
        expr: up == 0
        for: 5m
        labels:
          severity: slack
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{$labels.instance}} of job {{$labels.job}} has been down for more than 5 minutes."

      - alert: High Container Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Container Memory Usage
          description: "Pod {{$labels.pod_name}} have high memory usage"

      - alert: High Container CPU
        expr: sum(container_cpu_usage_seconds_total) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Container CPU Usage
          description: "Pod {{$labels.pod_name}} have high CPU usage"

      - alert: High Container SWAP
        expr: sum(container_memory_swap) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Container SWAP Usage
          description: "Pod {{$labels.pod_name}} have high SWAP usage"

      - alert: HTTP Requests
        expr: sum(http_request_total) > 1000
        for: 10m
        labels:
          severity: slack
        annotations:
          summary: High HTTP Request
          description: "{{$labels.handler}} of {{$labels.kubernetes_name}} have high {{$lables.method}} requests"

      - alert: Deploymnet Replicas
        expr: kube_deployment_status_replicas > 5
        labels:
          severity: slack
        annotations:
          summary: Deploymnet Replicas
          description: "{{$labels.deployment}} had more than 5 replicas"
```

* Prometheus Configuration
  
```bash
prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics
      
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name

      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      
      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
```

### Notes

* kubernetes-apiservers: It gets all the metrics from the API servers
* kubernetes-nodes: All Kubernetes node metrics will be collected with this job
* kubernetes-pods: All the pod metrics will be discovered
* kubernetes-cadvisor: Collects all cAdvisor metrics
* kubernetes-service-endpoints: All the Service endpoints will be scrapped
  
## 5. Create a Prometheus Deployment

```bash
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:v2.12.0
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/

      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
        - name: prometheus-storage-volume
          emptyDir: {}
```

## 6. Exposing the Prometheus deployment as a service on port 30000

```bash
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus-server
  type: NodePort  
  ports:
    - port: 8080
      targetPort: 9090 
      nodePort: 30000
```

<!-- Images -->
![Prometheus Dashboard](https://drive.google.com/file/d/1qtsM-r1BHP8ofnaq90ROdc-NywNXQKzy/view?usp=sharing)

# Kube State Metrics

### Kube State metrics is s service which talks to Kubernetes API server to get all the details about all the API objects

## 1. Create a Service Account

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: v1.8.0
  name: kube-state-metrics
  namespace: kube-system
```

## 2. Create ClusterRole with read permissions

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: v1.8.0
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - replicasets
  - ingresses
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - list
  - watch
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs:
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - list
  - watch
```

## 3. Assign ClusterRoleBinding to the Service Account

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: v1.8.0
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
```

## 4. Create Kube State Metrics Deployment
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: v1.8.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/version: v1.8.0
    spec:
      containers:
      - image: quay.io/coreos/kube-state-metrics:v1.8.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
      nodeSelector:
        beta.kubernetes.io/os: linux
      serviceAccountName: kube-state-metrics
```

## 5.  Internal exposing the Kube State Metrics deployment as a service

```bash
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: v1.8.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
```

# Alert Manager

### AlertManager is an opensource alerting system which works with Prometheus Monitoring system

## 1. A config map for Alert Manager configuration

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  config.yml: |-
    global:
    templates:
    - '/etc/alertmanager/*.tmpl'
    route:
      receiver: slack_api
      group_by: ['alertname', 'priority']
      match:
        severity: slack
      group_wait: 10s
      repeat_interval: 1m

    receivers:
    - name: slack_api
      slack_configs:
      - api_url: curl -X POST -H 'Content-type: application/json' --data '{"text":"Hello, World!"}' https://hooks.slack.com/services/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
        channel: '#project'
```

## 2. A config Map for Alert Manager alert templates

## 3. Alert Manager Deployment

```bash
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.19.0
        args:
          - "--config.file=/etc/alertmanager/config.yml"
          - "--storage.path=/alertmanager"
        ports:
        - name: alertmanager
          containerPort: 9093
        volumeMounts:
        - name: config-volume
          mountPath: /etc/alertmanager
        - name: templates-volume
          mountPath: /etc/alertmanager-templates
        - name: alertmanager
          mountPath: /alertmanager
      volumes:
      - name: config-volume
        configMap:
          name: alertmanager-config
      - name: templates-volume
        configMap:
          name: alertmanager-templates
      - name: alertmanager
        emptyDir: {}
```

## 4. Alert Manager service

```bash
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '9093'
spec:
  selector: 
    app: alertmanager
  type: NodePort  
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 31000
```

<!-- Images -->
![Alert Manager](https://drive.google.com/file/d/1CdN84Nu9RpAk2cHIjZN-z-GGtv2-hSZD/view?usp=sharing)

# Grafana 

### Grafana is an open-source lightweight dashboard tool. It can be integrated with many data sources like Prometheus, AWS cloud watch, Stackdriver, etc.

## 1. A config map for Grafana configuration

```bash
data:
  prometheus.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
               "access":"proxy",
                "editable": true,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus-service.monitoring.svc:8080",
                "version": 1
            }
        ]
    }
```

## 2. Grafana Deployment

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - name: grafana
          containerPort: 3000
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests: 
            memory: "1Gi"
            cpu: "500m"
        volumeMounts:
          - mountPath: /var/lib/grafana
            name: grafana-storage
          - mountPath: /etc/grafana/provisioning/datasources
            name: grafana-datasources
            readOnly: false
      volumes:
        - name: grafana-storage
          emptyDir: {}
        - name: grafana-datasources
          configMap:
              defaultMode: 420
              name: grafana-datasources
```

## 3. Grafana service

```bash
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '3000'
spec:
  selector: 
    app: grafana
  type: NodePort  
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 32000
```

## 4. Access the Grafana dashboard using any node IP on port 32000

```
User: admin
Pass: admin
```

## 5. Setup Kubernetes Dashbaords

<!-- Links -->
[Kubernetes Dashbaords](https://grafana.com/grafana/dashboards/8588)

<!-- Images -->
![Alert Manager](https://drive.google.com/file/d/1hg6NVK_0xgjopregnG_5uVTGZZX7uk1l/view?usp=sharing)
