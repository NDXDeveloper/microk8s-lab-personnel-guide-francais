🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.14 Haute Disponibilité Prometheus

## Introduction à la haute disponibilité

La haute disponibilité (HA) pour Prometheus signifie s'assurer que votre système de monitoring continue de fonctionner même en cas de panne d'un composant. Dans un environnement de production, perdre votre monitoring est critique car vous devenez "aveugle" sur l'état de vos applications. Même dans un lab personnel, comprendre et implémenter la haute disponibilité est un excellent apprentissage qui vous prépare aux environnements professionnels.

Imaginez que votre instance Prometheus tombe en panne pendant que vous testez une nouvelle application. Sans haute disponibilité, vous perdez non seulement le monitoring en temps réel, mais potentiellement aussi l'historique des métriques si le stockage est corrompu. Avec une configuration HA, une autre instance prend automatiquement le relais.

## Pourquoi la haute disponibilité dans un lab personnel ?

### Apprentissage et expérimentation
Mettre en place une architecture HA dans votre lab MicroK8s vous permet de :
- Comprendre les concepts de résilience et de redondance
- Expérimenter avec des architectures complexes sans risque
- Acquérir des compétences valorisées en entreprise
- Tester des scénarios de panne de manière contrôlée

### Continuité du monitoring
Même dans un lab, vous pouvez avoir :
- Des applications personnelles importantes à surveiller
- Des tests de longue durée nécessitant un monitoring continu
- Des démonstrations ou présentations nécessitant de la fiabilité
- Un historique de métriques que vous ne voulez pas perdre

### Préparation aux environnements réels
Les concepts que vous apprenez s'appliquent directement aux environnements de production où la haute disponibilité est cruciale.

## Architectures de haute disponibilité pour Prometheus

### Architecture de base : Duplication simple

La forme la plus simple de HA consiste à faire fonctionner deux instances Prometheus identiques qui collectent les mêmes métriques :

```yaml
# prometheus-instance-1.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus-1
  namespace: observability
spec:
  serviceName: prometheus-1
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      instance: primary
  template:
    metadata:
      labels:
        app: prometheus
        instance: primary
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus'
          - '--storage.tsdb.retention.time=30d'
          - '--web.external-url=http://prometheus-1.observability.svc.cluster.local:9090'
        ports:
        - containerPort: 9090
          name: web
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

La deuxième instance serait identique, mais avec un nom différent :

```yaml
# prometheus-instance-2.yaml
# (Même configuration mais avec prometheus-2 comme nom)
```

### Architecture fédérée

La fédération Prometheus permet de créer une hiérarchie d'instances où un Prometheus "global" agrège les données de plusieurs Prometheus "locaux" :

```yaml
# Configuration pour Prometheus global
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-global-config
  namespace: observability
data:
  prometheus.yml: |
    global:
      scrape_interval: 60s
      evaluation_interval: 60s

    scrape_configs:
      # Fédération depuis les instances locales
      - job_name: 'federate'
        scrape_interval: 15s
        honor_labels: true
        metrics_path: '/federate'
        params:
          'match[]':
            - '{job=~".*"}'  # Récupère toutes les métriques
            # Ou soyez plus sélectif :
            # - '{__name__=~"job:.*"}' # Seulement les recording rules
            # - '{__name__=~"ALERTS"}' # Seulement les alertes
        static_configs:
          - targets:
            - 'prometheus-1.observability.svc.cluster.local:9090'
            - 'prometheus-2.observability.svc.cluster.local:9090'
```

### Architecture avec Thanos

Thanos est une solution open source qui ajoute la haute disponibilité et le stockage à long terme à Prometheus. C'est l'approche la plus robuste mais aussi la plus complexe :

```yaml
# thanos-sidecar.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus-with-thanos
  namespace: observability
spec:
  serviceName: prometheus-thanos
  replicas: 2
  selector:
    matchLabels:
      app: prometheus-thanos
  template:
    metadata:
      labels:
        app: prometheus-thanos
    spec:
      containers:
      # Container Prometheus
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus'
          - '--storage.tsdb.retention.time=2h'  # Rétention courte avec Thanos
          - '--storage.tsdb.min-block-duration=2h'
          - '--storage.tsdb.max-block-duration=2h'
          - '--web.enable-lifecycle'
        ports:
        - containerPort: 9090
          name: web
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus

      # Sidecar Thanos
      - name: thanos-sidecar
        image: quay.io/thanos/thanos:v0.32.0
        args:
          - 'sidecar'
          - '--tsdb.path=/prometheus'
          - '--prometheus.url=http://localhost:9090'
          - '--grpc-address=0.0.0.0:10901'
          - '--http-address=0.0.0.0:10902'
          - '--objstore.config-file=/etc/thanos/objstore.yml'
        ports:
        - containerPort: 10901
          name: grpc
        - containerPort: 10902
          name: http
        volumeMounts:
        - name: storage
          mountPath: /prometheus
        - name: thanos-config
          mountPath: /etc/thanos

      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: thanos-config
        secret:
          secretName: thanos-objstore-config

  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Configuration du stockage pour la haute disponibilité

### Stockage local répliqué

Pour un lab MicroK8s, vous pouvez utiliser le stockage local avec réplication manuelle :

```yaml
# Configuration de volumes persistants pour chaque instance
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv-1
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/prometheus-1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv-2
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/prometheus-2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-2
```

### Stockage objet avec MinIO

Pour Thanos, vous avez besoin d'un stockage objet. MinIO est parfait pour un lab :

```yaml
# minio-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
          - server
          - /data
          - --console-address
          - ":9001"
        env:
        - name: MINIO_ROOT_USER
          value: "admin"
        - name: MINIO_ROOT_PASSWORD
          value: "changeme123"
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: minio-pvc
---
# Configuration Thanos pour utiliser MinIO
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
  namespace: observability
stringData:
  objstore.yml: |
    type: S3
    config:
      bucket: thanos
      endpoint: minio.observability.svc.cluster.local:9000
      access_key: admin
      secret_key: changeme123
      insecure: true
```

## Load Balancing et découverte de service

### Configuration avec MetalLB

Pour exposer vos instances Prometheus de manière équilibrée :

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-lb
  namespace: observability
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  type: LoadBalancer
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
    protocol: TCP
---
# Endpoints pour les deux instances
apiVersion: v1
kind: Endpoints
metadata:
  name: prometheus-lb
  namespace: observability
subsets:
  - addresses:
    - ip: 10.1.1.10  # IP du pod prometheus-1
    - ip: 10.1.1.11  # IP du pod prometheus-2
    ports:
    - port: 9090
```

### Service Discovery automatique

Configuration pour découvrir automatiquement les instances Prometheus :

```yaml
# Configuration dans prometheus.yml
scrape_configs:
  - job_name: 'prometheus-instances'
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
        - observability
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_label_app]
      action: keep
      regex: prometheus
    - source_labels: [__meta_kubernetes_pod_name]
      target_label: instance
    - source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
```

## Gestion des alertes en haute disponibilité

### Alertmanager en cluster

Pour éviter les alertes dupliquées, configurez Alertmanager en mode cluster :

```yaml
# alertmanager-ha.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: alertmanager
  namespace: observability
spec:
  serviceName: alertmanager-ha
  replicas: 3  # Nombre impair pour le consensus
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.26.0
        args:
          - '--config.file=/etc/alertmanager/config.yml'
          - '--storage.path=/alertmanager'
          - '--cluster.listen-address=0.0.0.0:9094'
          - '--cluster.peer=alertmanager-0.alertmanager-ha.observability.svc.cluster.local:9094'
          - '--cluster.peer=alertmanager-1.alertmanager-ha.observability.svc.cluster.local:9094'
          - '--cluster.peer=alertmanager-2.alertmanager-ha.observability.svc.cluster.local:9094'
        ports:
        - containerPort: 9093
          name: web
        - containerPort: 9094
          name: cluster
        volumeMounts:
        - name: config
          mountPath: /etc/alertmanager
        - name: storage
          mountPath: /alertmanager
      volumes:
      - name: config
        configMap:
          name: alertmanager-config
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

### Déduplication des alertes

Configuration pour éviter les alertes multiples :

```yaml
# Dans alertmanager config
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: observability
data:
  config.yml: |
    global:
      resolve_timeout: 5m

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'

    inhibit_rules:
      # Supprime les alertes de faible priorité si critique
      - source_match:
          severity: 'critical'
        target_match:
          severity: 'warning'
        equal: ['alertname', 'namespace', 'pod']

    receivers:
    - name: 'default'
      webhook_configs:
      - url: 'http://notification-service.observability.svc.cluster.local/webhook'
        send_resolved: true
```

## Synchronisation et cohérence des données

### Configuration de la synchronisation temporelle

La synchronisation temporelle est cruciale pour la cohérence des métriques :

```yaml
# ntp-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ntp-sync
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: ntp-sync
  template:
    metadata:
      labels:
        name: ntp-sync
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: ntp
        image: tutum/ntpd:latest
        securityContext:
          privileged: true
        env:
        - name: NTP_SERVERS
          value: "0.pool.ntp.org,1.pool.ntp.org,2.pool.ntp.org"
```

### Validation de la cohérence

Script pour vérifier la cohérence entre instances :

```yaml
# prometheus-consistency-check.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: consistency-check
  namespace: observability
data:
  check.sh: |
    #!/bin/bash

    # Récupération d'une métrique test depuis les deux instances
    METRIC="up"
    TIME=$(date +%s)

    # Requête sur instance 1
    RESULT1=$(curl -s "http://prometheus-1:9090/api/v1/query?query=${METRIC}&time=${TIME}")

    # Requête sur instance 2
    RESULT2=$(curl -s "http://prometheus-2:9090/api/v1/query?query=${METRIC}&time=${TIME}")

    # Comparaison des résultats
    if [ "$RESULT1" = "$RESULT2" ]; then
      echo "✓ Les instances sont synchronisées"
    else
      echo "✗ Divergence détectée entre les instances"
      echo "Instance 1: $RESULT1"
      echo "Instance 2: $RESULT2"
    fi
```

## Grafana avec sources multiples

### Configuration de Grafana pour la HA

Configurez Grafana pour utiliser plusieurs sources Prometheus :

```yaml
# grafana-datasources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: observability
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    # Instance principale
    - name: Prometheus-Primary
      type: prometheus
      access: proxy
      url: http://prometheus-1.observability.svc.cluster.local:9090
      isDefault: true
      editable: false

    # Instance secondaire
    - name: Prometheus-Secondary
      type: prometheus
      access: proxy
      url: http://prometheus-2.observability.svc.cluster.local:9090
      editable: false

    # Load balancer (utilise automatiquement une instance disponible)
    - name: Prometheus-HA
      type: prometheus
      access: proxy
      url: http://prometheus-lb.observability.svc.cluster.local:9090
      editable: false

    # Thanos Query (si utilisé)
    - name: Thanos
      type: prometheus
      access: proxy
      url: http://thanos-query.observability.svc.cluster.local:9090
      editable: false
```

### Variables de dashboard pour la sélection de source

Dans vos dashboards Grafana, créez une variable pour basculer entre les sources :

```json
{
  "templating": {
    "list": [
      {
        "name": "datasource",
        "type": "datasource",
        "query": "prometheus",
        "current": {
          "text": "Prometheus-HA",
          "value": "Prometheus-HA"
        },
        "refresh": 1,
        "regex": "",
        "multi": false,
        "includeAll": false
      }
    ]
  }
}
```

## Tests de résilience

### Simulation de pannes

Scripts pour tester votre configuration HA :

```yaml
# chaos-engineering.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: prometheus-chaos-test
  namespace: observability
spec:
  template:
    spec:
      containers:
      - name: chaos
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          echo "Test 1: Arrêt de prometheus-1"
          kubectl delete pod prometheus-1-0 -n observability
          sleep 30

          echo "Test 2: Vérification que prometheus-2 continue"
          wget -O- http://prometheus-2:9090/api/v1/query?query=up

          echo "Test 3: Attente du redémarrage de prometheus-1"
          kubectl wait --for=condition=ready pod/prometheus-1-0 -n observability --timeout=300s

          echo "Tests terminés avec succès"
      restartPolicy: Never
```

### Validation de la récupération

Test automatisé de récupération après panne :

```yaml
# recovery-test.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: recovery-test
  namespace: observability
data:
  test.sh: |
    #!/bin/bash

    # Fonction de test de disponibilité
    check_prometheus() {
      local instance=$1
      curl -s -o /dev/null -w "%{http_code}" \
        "http://${instance}:9090/api/v1/query?query=up"
    }

    # Test de basculement
    echo "État initial:"
    echo "Prometheus-1: $(check_prometheus prometheus-1)"
    echo "Prometheus-2: $(check_prometheus prometheus-2)"

    # Simulation de panne
    echo "Simulation panne prometheus-1..."
    kubectl scale statefulset prometheus-1 --replicas=0 -n observability

    sleep 10

    # Vérification du basculement
    echo "Après panne:"
    echo "Prometheus-1: $(check_prometheus prometheus-1)"
    echo "Prometheus-2: $(check_prometheus prometheus-2)"

    # Restauration
    echo "Restauration prometheus-1..."
    kubectl scale statefulset prometheus-1 --replicas=1 -n observability

    # Attente de la récupération
    kubectl wait --for=condition=ready pod/prometheus-1-0 -n observability

    echo "Après récupération:"
    echo "Prometheus-1: $(check_prometheus prometheus-1)"
    echo "Prometheus-2: $(check_prometheus prometheus-2)"
```

## Optimisation pour un lab personnel

### Configuration minimale HA

Pour un lab avec ressources limitées, voici une configuration HA minimale :

```yaml
# prometheus-ha-minimal.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-ha-config
  namespace: observability
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s  # Intervalle plus long pour économiser les ressources
      evaluation_interval: 30s
      external_labels:
        replica: $(POD_NAME)  # Identifie chaque réplique

    # Configuration minimale de scraping
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        # Scrape seulement les pods avec annotation
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-ha
  namespace: observability
spec:
  replicas: 2  # Seulement 2 répliques pour un lab
  selector:
    matchLabels:
      app: prometheus-ha
  template:
    metadata:
      labels:
        app: prometheus-ha
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        resources:
          requests:
            memory: "512Mi"  # Ressources minimales
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus'
          - '--storage.tsdb.retention.time=7d'  # Rétention courte
          - '--storage.tsdb.wal-compression'  # Compression WAL
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-ha-config
      - name: storage
        emptyDir:
          sizeLimit: 5Gi  # Stockage limité
```

### Monitoring de la HA elle-même

Dashboard pour surveiller votre configuration HA :

```yaml
# Requêtes PromQL pour votre dashboard HA
apiVersion: v1
kind: ConfigMap
metadata:
  name: ha-dashboard-queries
  namespace: observability
data:
  queries.txt: |
    # Nombre d'instances Prometheus actives
    count(up{job="prometheus"})

    # Temps depuis le dernier scrape par instance
    time() - prometheus_target_last_scrape_timestamp_seconds

    # Divergence de métriques entre instances
    stddev by (metric) (
      prometheus_tsdb_symbol_table_size_bytes
    )

    # Utilisation mémoire par instance
    process_resident_memory_bytes{job="prometheus"}

    # Latence de scraping
    prometheus_target_scrape_duration_seconds{quantile="0.99"}

    # État de synchronisation Alertmanager
    alertmanager_cluster_members

    # Alertes actives par instance
    count by (replica) (ALERTS)
```

## Dépannage de la haute disponibilité

### Problèmes courants et solutions

**Les instances ne se synchronisent pas**
- Vérifiez la configuration réseau entre les pods
- Assurez-vous que les horloges sont synchronisées
- Contrôlez les labels `external_labels` dans la configuration

**Performances dégradées avec HA**
- Réduisez la fréquence de scraping
- Limitez le nombre de métriques avec des relabel_configs
- Utilisez des recording rules pour les calculs lourds

**Stockage qui se remplit rapidement**
- Ajustez la rétention (`--storage.tsdb.retention.time`)
- Activez la compression WAL
- Configurez le compactage plus agressif

**Alertes dupliquées**
- Vérifiez la configuration du cluster Alertmanager
- Ajustez les paramètres `group_wait` et `group_interval`
- Utilisez les `inhibit_rules` correctement

### Commandes de diagnostic

```bash
# Vérifier l'état des instances Prometheus
kubectl get pods -n observability -l app=prometheus

# Logs d'une instance spécifique
kubectl logs -n observability prometheus-1-0 --tail=50

# Métriques internes de Prometheus
kubectl port-forward -n observability prometheus-1-0 9090:9090
curl localhost:9090/metrics | grep prometheus_

# État du cluster Alertmanager
kubectl exec -n observability alertmanager-0 -- \
  amtool cluster show

# Vérifier la cohérence des données
for i in 1 2; do
  echo "Instance prometheus-$i:"
  kubectl exec -n observability prometheus-$i-0 -- \
    promtool query instant \
    'http://localhost:9090' \
    'up{job="kubernetes-nodes"}'
done
```

## Migration vers une architecture HA

### Plan de migration depuis une instance unique

Si vous avez déjà Prometheus en fonctionnement, voici comment migrer vers HA :

```yaml
# Étape 1: Backup de la configuration existante
apiVersion: batch/v1
kind: Job
metadata:
  name: prometheus-backup
  namespace: observability
spec:
  template:
    spec:
      containers:
      - name: backup
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          # Sauvegarde de la configuration
          kubectl get configmap prometheus-config -n observability -o yaml > /backup/config.yaml

          # Sauvegarde des données TSDB
          kubectl exec prometheus-0 -n observability -- tar czf - /prometheus > /backup/prometheus-data.tar.gz

          echo "Backup terminé"
        volumeMounts:
        - name: backup
          mountPath: /backup
      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup-pvc
      restartPolicy: Never
```

### Checklist de migration

Avant de passer en HA, vérifiez :

1. **Ressources disponibles**
   - Au moins 2x les ressources CPU/RAM actuelles
   - Espace disque suffisant pour les répliques

2. **Configuration réseau**
   - Communication inter-pods fonctionnelle
   - Load balancer ou service de type ClusterIP configuré

3. **Données existantes**
   - Backup complet effectué
   - Plan de migration des données historiques

4. **Tests**
   - Environment de test disponible
   - Procédures de rollback documentées

## Bonnes pratiques pour la HA dans un lab

### Dimensionnement approprié

Pour un lab personnel, adaptez les ressources :

```yaml
# Recommandations pour un lab
Small Lab (1-10 services):
  - 2 instances Prometheus
  - 512MB RAM par instance
  - 5GB stockage par instance
  - Rétention: 7 jours

Medium Lab (10-50 services):
  - 2-3 instances Prometheus
  - 1GB RAM par instance
  - 10GB stockage par instance
  - Rétention: 15 jours

Large Lab (50+ services):
  - 3 instances Prometheus
  - 2GB RAM par instance
  - 20GB stockage par instance
  - Rétention: 30 jours
  - Considérez Thanos
```

### Automatisation et maintenance

Scripts de maintenance automatisée :

```yaml
# cronjob-maintenance.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: prometheus-ha-maintenance
  namespace: observability
spec:
  schedule: "0 2 * * 0"  # Chaque dimanche à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: maintenance
            image: prom/prometheus:v2.45.0
            command:
            - /bin/sh
            - -c
            - |
              # Compactage des données TSDB
              for i in 0 1; do
                echo "Compactage prometheus-$i"
                kubectl exec prometheus-$i-0 -n observability -- \
                  promtool tsdb compact /prometheus
              done

              # Nettoyage des métriques obsolètes
              kubectl exec prometheus-0-0 -n observability -- \
                curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones

              echo "Maintenance terminée"
          restartPolicy: OnFailure
```

## Conclusion

La haute disponibilité de Prometheus dans un lab MicroK8s est un excellent moyen d'apprendre les concepts de résilience et de redondance applicables en production. Même avec des ressources limitées, vous pouvez implémenter une architecture HA fonctionnelle qui vous permettra de :

- Maintenir votre monitoring opérationnel en cas de panne
- Expérimenter avec des architectures complexes
- Préparer vos compétences pour des environnements professionnels
- Garantir la continuité de vos métriques importantes

Commencez par une configuration simple avec deux instances Prometheus, puis évoluez progressivement vers des architectures plus sophistiquées comme Thanos quand vos besoins et vos ressources le permettent.

## Évolution progressive de votre architecture HA

### Roadmap de maturité HA

Voici un chemin d'évolution recommandé pour votre lab :

**Phase 1 : Duplication simple (Semaines 1-2)**
- Deux instances Prometheus identiques
- Scraping des mêmes targets
- Grafana avec les deux sources configurées
- Test manuel de basculement

**Phase 2 : Load balancing (Semaines 3-4)**
- Ajout d'un service LoadBalancer
- Configuration de health checks
- Basculement automatique en cas de panne
- Monitoring de la disponibilité

**Phase 3 : Alerting HA (Semaines 5-6)**
- Cluster Alertmanager à 3 nœuds
- Déduplication des alertes
- Tests de résilience automatisés
- Documentation des procédures

**Phase 4 : Stockage distribué (Mois 2)**
- Introduction de Thanos ou Cortex
- Stockage objet avec MinIO
- Requêtes globales cross-instances
- Rétention long terme

**Phase 5 : Automatisation complète (Mois 3)**
- GitOps pour la configuration
- Tests de chaos engineering
- Auto-healing avec operators
- Dashboards de santé HA

### Métriques de succès HA

Indicateurs pour mesurer l'efficacité de votre HA :

```yaml
# Requêtes PromQL pour mesurer la HA
apiVersion: v1
kind: ConfigMap
metadata:
  name: ha-sli-queries
  namespace: observability
data:
  sli.yml: |
    # Disponibilité du service (SLI)
    - name: prometheus_availability
      query: |
        avg_over_time(
          (count(up{job="prometheus"} == 1) >= 1)[5m:]
        ) * 100
      target: 99.9  # Objectif de disponibilité

    # Temps de récupération après incident
    - name: mean_time_to_recovery
      query: |
        histogram_quantile(0.5,
          rate(prometheus_recovery_duration_seconds_bucket[7d])
        )
      target: 300  # MTTR cible en secondes

    # Cohérence des données entre instances
    - name: data_consistency
      query: |
        1 - (
          stddev by (metric) (prometheus_tsdb_symbol_table_size_bytes)
          /
          avg by (metric) (prometheus_tsdb_symbol_table_size_bytes)
        ) * 100
      target: 99  # % de cohérence cible

    # Latence de requête en HA
    - name: query_latency_p99
      query: |
        histogram_quantile(0.99,
          rate(prometheus_engine_query_duration_seconds_bucket[5m])
        )
      target: 1  # Latence max en secondes
```

## Scénarios avancés de test

### Test de split-brain

Simulation d'une partition réseau (split-brain) :

```yaml
# network-partition-test.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: split-brain-test
  namespace: observability
data:
  test.sh: |
    #!/bin/bash

    echo "=== Test de Split-Brain Prometheus HA ==="

    # Fonction pour isoler un pod réseau
    isolate_pod() {
      local pod=$1
      echo "Isolation réseau de $pod..."

      # Ajout de règles iptables pour bloquer le trafic
      kubectl exec -n observability $pod -- sh -c "
        iptables -A INPUT -s 10.0.0.0/8 -j DROP
        iptables -A OUTPUT -d 10.0.0.0/8 -j DROP
      "
    }

    # Fonction pour restaurer la connectivité
    restore_pod() {
      local pod=$1
      echo "Restauration réseau de $pod..."

      kubectl exec -n observability $pod -- sh -c "
        iptables -D INPUT -s 10.0.0.0/8 -j DROP
        iptables -D OUTPUT -d 10.0.0.0/8 -j DROP
      "
    }

    # Test 1: Isolation d'une instance
    echo "Test 1: Isolation de prometheus-1..."
    isolate_pod "prometheus-1-0"

    sleep 30

    # Vérification que prometheus-2 continue
    echo "Vérification de prometheus-2..."
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://prometheus-2:9090/api/v1/query?query=up)
    if [ "$STATUS" = "200" ]; then
      echo "✓ Prometheus-2 fonctionne correctement"
    else
      echo "✗ Prometheus-2 ne répond pas"
    fi

    # Vérification des divergences de données
    echo "Attente pour créer une divergence..."
    sleep 60

    # Restauration
    restore_pod "prometheus-1-0"

    echo "Test de split-brain terminé"
```

### Test de saturation des ressources

Simulation de saturation CPU/mémoire :

```yaml
# stress-test-ha.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: prometheus-stress-test
  namespace: observability
spec:
  template:
    spec:
      containers:
      - name: stress
        image: progrium/stress
        args:
          - '--cpu'
          - '2'
          - '--io'
          - '1'
          - '--vm'
          - '2'
          - '--vm-bytes'
          - '128M'
          - '--timeout'
          - '60s'
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
      - name: monitor
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "Monitoring pendant le stress test..."
          for i in $(seq 1 12); do
            echo "Check $i/12:"
            curl -s http://prometheus-lb:9090/api/v1/query?query=up | grep -o '"status":"[^"]*"'
            sleep 5
          done
      restartPolicy: Never
```

## Intégration avec l'écosystème cloud-native

### Prometheus Operator pour simplifier la HA

L'utilisation de Prometheus Operator simplifie grandement la gestion HA :

```yaml
# prometheus-operator-ha.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-ha
  namespace: observability
spec:
  replicas: 2  # Nombre de répliques
  version: v2.45.0
  serviceAccountName: prometheus

  # Configuration HA native
  enableAdminAPI: true

  # Sharding automatique pour distribuer la charge
  shards: 2

  # Configuration de la rétention
  retention: 30d
  retentionSize: "10GB"

  # Ressources par réplique
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1000m

  # Stockage persistant
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi

  # Anti-affinité pour distribution sur différents nœuds
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: prometheus
            operator: In
            values:
            - prometheus-ha
        topologyKey: kubernetes.io/hostname

  # Configuration des ServiceMonitors à scraper
  serviceMonitorSelector:
    matchLabels:
      monitoring: "true"

  # Configuration Alertmanager
  alerting:
    alertmanagers:
    - namespace: observability
      name: alertmanager-ha
      port: web
```

### Intégration avec Service Mesh (Istio/Linkerd)

Pour une observabilité avancée avec un service mesh :

```yaml
# istio-prometheus-ha.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-istio-config
  namespace: observability
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      external_labels:
        mesh: istio
        replica: $(POD_NAME)

    scrape_configs:
    # Métriques Istio Telemetry v2
    - job_name: 'istio-mesh'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: istio-telemetry
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: prometheus

    # Métriques des sidecars Envoy
    - job_name: 'envoy-stats'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        action: keep
        regex: '.*-envoy-prom'
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

    # Métriques du control plane Istio
    - job_name: 'istio-control-plane'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app]
        regex: istiod
        action: keep
```

## Optimisations spécifiques pour MicroK8s

### Configuration pour Raspberry Pi ou hardware limité

Si votre lab tourne sur des Raspberry Pi ou du hardware limité :

```yaml
# prometheus-ha-rpi.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rpi-config
  namespace: observability
data:
  prometheus.yml: |
    global:
      scrape_interval: 60s  # Intervalle plus long
      scrape_timeout: 10s
      evaluation_interval: 60s

      # Labels externes pour identifier le hardware
      external_labels:
        hardware: rpi
        replica: $(HOSTNAME)

    # Optimisations pour ARM
    storage:
      tsdb:
        wal_compression: true
        max_block_duration: 2h
        min_block_duration: 2h
        retention.time: 3d  # Rétention courte
        retention.size: 2GB  # Limite stricte
---
apiVersion: apps/v1
kind: DaemonSet  # Un Prometheus par nœud RPi
metadata:
  name: prometheus-ha-rpi
  namespace: observability
spec:
  selector:
    matchLabels:
      app: prometheus-rpi
  template:
    metadata:
      labels:
        app: prometheus-rpi
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64  # ou armhf selon votre architecture
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0-arm64  # Image ARM
        resources:
          limits:
            memory: 512Mi  # Limite stricte pour RPi
            cpu: 1000m
          requests:
            memory: 256Mi
            cpu: 100m
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus'
          - '--storage.tsdb.retention.time=3d'
          - '--storage.tsdb.retention.size=2GB'
          - '--storage.tsdb.wal-compression'
          - '--query.max-concurrency=2'  # Limite les requêtes parallèles
          - '--query.max-samples=50000'  # Limite les échantillons par requête
```

### Utilisation des addons MicroK8s

Intégration avec les addons MicroK8s pour la HA :

```bash
# Script d'installation HA avec addons MicroK8s
#!/bin/bash

echo "Configuration Prometheus HA pour MicroK8s"

# Activation des addons nécessaires
microk8s enable dns storage metallb:192.168.1.100-192.168.1.110

# Attente que les addons soient prêts
microk8s kubectl wait --for=condition=available --timeout=600s \
  deployment/coredns -n kube-system

# Création du namespace
microk8s kubectl create namespace observability

# Installation de Prometheus avec HA
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: prometheus-ha-lb
  namespace: observability
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  type: LoadBalancer
  ports:
  - port: 9090
    targetPort: 9090
    protocol: TCP
    name: http
  selector:
    app: prometheus-ha
EOF

echo "Configuration HA terminée"
echo "Accès Prometheus HA: http://192.168.1.100:9090"
```

## Conclusion finale

La haute disponibilité de Prometheus dans un environnement MicroK8s représente un excellent terrain d'apprentissage pour maîtriser les concepts de résilience et de fiabilité des systèmes de monitoring. En commençant par des configurations simples et en évoluant progressivement vers des architectures plus complexes, vous développez des compétences précieuses tout en maintenant un système de monitoring robuste pour votre lab.

Les points clés à retenir :

1. **Commencez simple** : Deux instances Prometheus avec un load balancer suffisent pour débuter
2. **Évoluez progressivement** : Ajoutez des fonctionnalités HA au fur et à mesure de vos besoins
3. **Testez régulièrement** : Les tests de résilience sont essentiels pour valider votre configuration
4. **Adaptez aux ressources** : Optimisez pour votre hardware spécifique
5. **Documentez tout** : Vos configurations, procédures et apprentissages

La haute disponibilité n'est pas qu'une question technique, c'est aussi une approche méthodologique qui vous prépare à gérer des systèmes critiques en production. Votre lab MicroK8s est le terrain de jeu idéal pour expérimenter, échouer, apprendre et maîtriser ces concepts sans risque.

⏭️
