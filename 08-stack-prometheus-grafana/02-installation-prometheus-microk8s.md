🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.02 - Installation de Prometheus sur MicroK8s

## Vue d'ensemble des options d'installation

MicroK8s offre plusieurs méthodes pour installer Prometheus, chacune avec ses avantages. Nous allons explorer toutes les options pour que vous puissiez choisir celle qui convient le mieux à votre lab.

### Comparaison des méthodes

| Méthode | Complexité | Personnalisation | Maintenance | Recommandé pour |
|---------|------------|------------------|-------------|-----------------|
| Addon MicroK8s | ⭐ | Limitée | Automatique | Débutants, tests rapides |
| Helm Chart | ⭐⭐ | Moyenne | Semi-auto | Labs intermédiaires |
| Kube-prometheus-stack | ⭐⭐⭐ | Complète | Manuelle | Production, labs avancés |
| Manifests manuels | ⭐⭐⭐⭐ | Totale | Manuelle | Apprentissage approfondi |

## Méthode 1 : Addon Observability MicroK8s (Recommandé pour débuter)

### Vérification des prérequis

Avant d'installer l'addon, vérifions l'état de votre cluster :

```bash
# Vérifier que MicroK8s est actif
microk8s status

# Vérifier les ressources disponibles
kubectl get nodes -o wide
kubectl top nodes  # Si metrics-server est installé

# Lister les addons activés
microk8s status --addon
```

### Installation de l'addon

L'addon observability installe Prometheus, Grafana et plusieurs exporters d'un seul coup :

```bash
# Activer l'addon observability
microk8s enable observability

# L'installation peut prendre 2-5 minutes
# Surveiller le déploiement
watch microk8s kubectl get pods -n observability
```

### Que contient l'addon observability ?

L'addon déploie automatiquement :

```
Namespace: observability
├── Prometheus Server (v2.x)
│   ├── Configuration par défaut
│   ├── Service Discovery activé
│   └── Rétention: 2 semaines
├── Grafana (v9.x)
│   ├── Datasource Prometheus configurée
│   └── Dashboards Kubernetes préinstallés
├── Alertmanager
│   └── Configuration basique
├── Kube-state-metrics
│   └── Métriques des objets K8s
└── Node Exporter (via DaemonSet)
    └── Métriques système de chaque node
```

### Vérification de l'installation

```bash
# Vérifier que tous les pods sont Running
kubectl get pods -n observability

# Sortie attendue :
# NAME                                  READY   STATUS
# alertmanager-0                        2/2     Running
# grafana-5c6bbf7b4c-xxxxx             1/1     Running
# kube-state-metrics-5d6885d89-xxxxx   1/1     Running
# node-exporter-xxxxx                   1/1     Running
# prometheus-0                          2/2     Running

# Vérifier les services
kubectl get svc -n observability

# Vérifier les PersistentVolumes
kubectl get pv
kubectl get pvc -n observability
```

### Accès à Prometheus

**Option 1 : Port-forward (accès local)**

```bash
# Accéder à Prometheus UI
kubectl port-forward -n observability svc/prometheus-operated 9090:9090

# Ouvrir dans le navigateur : http://localhost:9090
```

**Option 2 : Via NodePort**

```bash
# Créer un service NodePort
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: prometheus-nodeport
  namespace: observability
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090
  selector:
    app.kubernetes.io/name: prometheus
EOF

# Accès via http://<IP-NODE>:30090
```

**Option 3 : Via Ingress (recommandé)**

```bash
# Créer un Ingress pour Prometheus
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: observability
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: prometheus-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Prometheus'
spec:
  ingressClassName: nginx
  rules:
  - host: prometheus.votredomaine.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-operated
            port:
              number: 9090
EOF
```

### Configuration post-installation

Le fichier de configuration Prometheus se trouve dans un ConfigMap :

```bash
# Voir la configuration actuelle
kubectl get configmap -n observability prometheus-config -o yaml

# Éditer la configuration (attention aux modifications)
kubectl edit configmap -n observability prometheus-config

# Recharger Prometheus après modification
kubectl rollout restart statefulset/prometheus -n observability
```

## Méthode 2 : Installation via Helm

### Prérequis Helm

```bash
# Installer Helm si nécessaire
sudo snap install helm --classic

# Ou via script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Vérifier l'installation
helm version
```

### Ajout du repository Prometheus

```bash
# Ajouter le repo officiel Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable

# Mettre à jour les repos
helm repo update

# Rechercher les charts disponibles
helm search repo prometheus
```

### Installation avec configuration personnalisée

**Créer un fichier de valeurs personnalisées :**

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    # Configuration de la rétention
    retention: 30d
    retentionSize: "10GB"

    # Ressources
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 1000m
        memory: 4Gi

    # Stockage persistant
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi

    # Scrape configs additionnels
    additionalScrapeConfigs:
    - job_name: 'custom-app'
      static_configs:
      - targets: ['my-app:8080']

  # Service Monitor pour autodiscovery
  serviceMonitorSelectorNilUsesHelmValues: false
  podMonitorSelectorNilUsesHelmValues: false

alertmanager:
  enabled: true
  alertmanagerSpec:
    retention: 120h
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi

grafana:
  enabled: false  # On installera Grafana séparément

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
```

**Installer avec les valeurs personnalisées :**

```bash
# Créer le namespace
kubectl create namespace monitoring

# Installer Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

# Suivre le déploiement
kubectl get pods -n monitoring -w
```

### Mise à jour de la configuration Helm

```bash
# Voir les valeurs actuelles
helm get values prometheus -n monitoring

# Mettre à jour avec de nouvelles valeurs
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

# Voir l'historique des révisions
helm history prometheus -n monitoring

# Rollback si nécessaire
helm rollback prometheus 1 -n monitoring
```

## Méthode 3 : Installation manuelle avec manifests

### Structure des composants

Pour une installation manuelle complète, nous devons créer :

```
monitoring/
├── namespace.yaml
├── prometheus/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── clusterrole.yaml
│   └── clusterrolebinding.yaml
├── node-exporter/
│   ├── daemonset.yaml
│   └── service.yaml
└── kube-state-metrics/
    ├── deployment.yaml
    ├── service.yaml
    └── rbac.yaml
```

### Création du namespace et RBAC

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
# clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions", "networking.k8s.io"]
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
---
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

### ConfigMap Prometheus

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    # Alertmanager configuration
    alerting:
      alertmanagers:
      - static_configs:
        - targets: []

    # Scrape configurations
    scrape_configs:
    # Prometheus self-monitoring
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']

    # Kubernetes API server
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

    # Kubernetes nodes
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

    # Kubernetes pods
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
```

### Deployment Prometheus

```yaml
# prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus/'
          - '--storage.tsdb.retention.time=30d'
          - '--storage.tsdb.retention.size=10GB'
          - '--web.enable-lifecycle'
        ports:
        - containerPort: 9090
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1
            memory: 2Gi
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        persistentVolumeClaim:
          claimName: prometheus-pvc
---
# PVC pour le stockage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

### Application des manifests

```bash
# Appliquer tous les manifests
kubectl apply -f monitoring/

# Vérifier le déploiement
kubectl get all -n monitoring

# Logs de Prometheus
kubectl logs -n monitoring deployment/prometheus
```

## Vérification et tests post-installation

### Tests de connectivité

```bash
# Test 1: Vérifier que Prometheus scrape ses targets
kubectl port-forward -n monitoring svc/prometheus 9090:9090 &

# Ouvrir http://localhost:9090/targets
# Tous les targets doivent être "UP"

# Test 2: Requête simple
curl http://localhost:9090/api/v1/query?query=up

# Test 3: Vérifier les métriques
curl http://localhost:9090/api/v1/label/__name__/values | jq '.data[:10]'
```

### Diagnostic des problèmes courants

**Problème 1 : Pods en CrashLoopBackOff**

```bash
# Vérifier les logs
kubectl logs -n monitoring pod/prometheus-xxx --previous

# Causes communes:
# - Permissions insuffisantes (RBAC)
# - ConfigMap mal formaté
# - Ressources insuffisantes

# Solution: Vérifier les events
kubectl describe pod -n monitoring prometheus-xxx
```

**Problème 2 : Targets Down**

```bash
# Vérifier la configuration réseau
kubectl get endpoints -n monitoring

# Vérifier les annotations des pods
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | select(.metadata.annotations."prometheus.io/scrape"=="true") | .metadata.name'

# Tester la connectivité
kubectl exec -n monitoring prometheus-xxx -- wget -O- http://target-service:port/metrics
```

**Problème 3 : Stockage plein**

```bash
# Vérifier l'utilisation du stockage
kubectl exec -n monitoring prometheus-xxx -- df -h /prometheus

# Augmenter le PVC si possible
kubectl edit pvc -n monitoring prometheus-pvc

# Ou réduire la rétention
kubectl edit configmap -n monitoring prometheus-config
# Modifier: retention.time ou retention.size
```

## Configuration de la persistance

### Types de stockage dans MicroK8s

```bash
# Activer le storage addon si nécessaire
microk8s enable hostpath-storage

# Vérifier les StorageClasses disponibles
kubectl get storageclass

# Créer un PV personnalisé si nécessaire
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /var/snap/microk8s/common/prometheus-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - your-node-name
EOF
```

### Backup des données Prometheus

```bash
# Snapshot manuel
kubectl exec -n monitoring prometheus-xxx -- \
  curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Récupérer le snapshot
kubectl cp monitoring/prometheus-xxx:/prometheus/snapshots /backup/prometheus/

# Script de backup automatique
cat <<'EOF' > /usr/local/bin/backup-prometheus.sh
#!/bin/bash
DATE=$(date +%Y%m%d-%H%M%S)
NAMESPACE="monitoring"
POD=$(kubectl get pod -n $NAMESPACE -l app=prometheus -o jsonpath='{.items[0].metadata.name}')

# Créer snapshot
kubectl exec -n $NAMESPACE $POD -- curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Copier snapshot
kubectl cp $NAMESPACE/$POD:/prometheus/snapshots /backup/prometheus/$DATE/

# Nettoyer les vieux backups (garder 7 jours)
find /backup/prometheus/ -type d -mtime +7 -exec rm -rf {} \;
EOF

chmod +x /usr/local/bin/backup-prometheus.sh

# Ajouter au crontab
echo "0 2 * * * /usr/local/bin/backup-prometheus.sh" | crontab -
```

## Optimisation pour un lab personnel

### Ajustements des ressources

Pour un lab avec ressources limitées :

```yaml
# prometheus-light.yaml
resources:
  requests:
    cpu: 100m      # Au lieu de 500m
    memory: 256Mi  # Au lieu de 1Gi
  limits:
    cpu: 500m      # Au lieu de 1000m
    memory: 512Mi  # Au lieu de 2Gi

# Réduire la fréquence de scraping
global:
  scrape_interval: 30s     # Au lieu de 15s
  evaluation_interval: 30s  # Au lieu de 15s

# Réduire la rétention
storage:
  tsdb:
    retention.time: 7d    # Au lieu de 30d
    retention.size: 5GB   # Au lieu de 10GB
```

### Désactiver les composants non essentiels

```bash
# Si vous n'utilisez pas Alertmanager
kubectl scale deployment alertmanager -n monitoring --replicas=0

# Réduire les métriques collectées
# Éditer le ConfigMap pour commenter les jobs non nécessaires
kubectl edit configmap -n monitoring prometheus-config
```

## Intégration avec l'écosystème MicroK8s

### Activation des métriques pour les addons MicroK8s

```bash
# Pour l'Ingress Controller
microk8s kubectl annotate service -n ingress nginx-ingress-controller \
  prometheus.io/scrape="true" \
  prometheus.io/port="10254" \
  prometheus.io/path="/metrics"

# Pour le DNS (CoreDNS)
microk8s kubectl annotate service -n kube-system kube-dns \
  prometheus.io/scrape="true" \
  prometheus.io/port="9153" \
  prometheus.io/path="/metrics"

# Pour le Registry
microk8s kubectl annotate service -n container-registry registry \
  prometheus.io/scrape="true" \
  prometheus.io/port="5000" \
  prometheus.io/path="/metrics/prometheus"
```

## Points de vérification finale

### Checklist post-installation

- [ ] Prometheus est accessible (UI disponible)
- [ ] Tous les pods sont en état Running
- [ ] Le stockage persistant est configuré
- [ ] Les targets principaux sont UP (nodes, pods, services)
- [ ] Les métriques de base sont collectées (CPU, mémoire)
- [ ] La configuration est sauvegardée
- [ ] Les ressources sont adaptées à votre lab
- [ ] L'accès externe est sécurisé (si configuré)

### Commandes utiles pour le quotidien

```bash
# Voir l'état général
kubectl get all -n monitoring

# Recharger la configuration sans redémarrer
kubectl exec -n monitoring prometheus-xxx -- kill -HUP 1

# Voir les métriques de consommation
kubectl top pods -n monitoring

# Examiner la configuration active
kubectl exec -n monitoring prometheus-xxx -- cat /etc/prometheus/prometheus.yml

# Tester une requête PromQL
curl -G http://localhost:9090/api/v1/query --data-urlencode 'query=rate(container_cpu_usage_seconds_total[5m])'
```

## Préparation pour la suite

Avec Prometheus installé et opérationnel, vous êtes maintenant prêt à :
- Configurer les targets et le service discovery (section 8.03)
- Apprendre PromQL pour interroger vos métriques (section 8.04)
- Ajouter des exporters spécialisés (section 8.05)
- Installer et connecter Grafana pour la visualisation (section 8.06)

L'installation de Prometheus est la fondation de votre stack d'observabilité. Prenez le temps de bien comprendre sa configuration avant de passer aux étapes suivantes.

⏭️
