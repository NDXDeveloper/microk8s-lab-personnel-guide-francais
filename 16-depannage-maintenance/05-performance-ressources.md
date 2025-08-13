🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.5 Performance et ressources

## Introduction aux défis de performance dans Kubernetes

La gestion des performances et des ressources est cruciale pour maintenir un cluster MicroK8s stable et réactif. Dans un environnement de lab personnel, les ressources sont souvent limitées, rendant l'optimisation encore plus importante. Cette section vous guidera à travers l'identification des problèmes de performance, la compréhension de l'utilisation des ressources, et les techniques d'optimisation pour tirer le meilleur parti de votre infrastructure.

## Comprendre les ressources dans Kubernetes

### Les types de ressources

Kubernetes gère principalement quatre types de ressources pour les conteneurs :

**CPU (processeur)** : Mesurée en "millicores" où 1000m = 1 CPU core. Un conteneur avec 500m peut utiliser jusqu'à la moitié d'un cœur de processeur. La CPU est une ressource "compressible" - si un conteneur dépasse sa limite, il est ralenti (throttled) mais pas arrêté.

**Mémoire (RAM)** : Mesurée en bytes (Ki, Mi, Gi pour kibi, mebi, gibi-octets). La mémoire est une ressource "incompressible" - si un conteneur dépasse sa limite, il est tué (OOMKilled - Out Of Memory Killed).

**Stockage éphémère** : L'espace disque utilisé par les logs, les caches, et les données temporaires. Mesuré également en bytes.

**Stockage persistant** : Géré via les Persistent Volumes (PV) et Persistent Volume Claims (PVC), séparé des limites du conteneur.

### Requests vs Limits

La distinction entre requests et limits est fondamentale :

**Requests** : La quantité minimale de ressources garantie au conteneur. Kubernetes utilise les requests pour décider sur quel nœud placer un pod. C'est ce que le conteneur est garanti d'obtenir.

**Limits** : La quantité maximale de ressources qu'un conteneur peut utiliser. Au-delà, le conteneur est throttled (CPU) ou killed (mémoire).

Un conteneur peut utiliser plus que ses requests si des ressources sont disponibles, jusqu'à ses limits. Cette flexibilité permet une meilleure utilisation des ressources du cluster.

### Quality of Service (QoS)

Kubernetes assigne automatiquement une classe QoS à chaque pod basée sur ses ressources :

1. **Guaranteed** : Requests = Limits pour toutes les ressources
2. **Burstable** : Au moins un request ou limit défini, mais pas Guaranteed
3. **BestEffort** : Aucun request ni limit défini

En cas de pression sur les ressources, Kubernetes évince d'abord les pods BestEffort, puis Burstable, et enfin Guaranteed.

## Problème 1 : Pods en état Pending ou Evicted

### Symptômes

- Pods restent en état "Pending" indéfiniment
- Message "Insufficient cpu" ou "Insufficient memory"
- Pods évincés avec le statut "Evicted"
- Events montrant "FailedScheduling"

### Diagnostic

Examinez pourquoi les pods ne peuvent pas être schedulés :

```bash
# Vérifier l'état des pods
microk8s kubectl get pods --all-namespaces | grep -E "Pending|Evicted"

# Examiner les événements d'un pod pending
microk8s kubectl describe pod <pod-name> -n <namespace>

# Vérifier les ressources disponibles sur les nœuds
microk8s kubectl describe nodes

# Voir l'utilisation actuelle vs capacité
microk8s kubectl top nodes

# Vérifier les requests totales vs disponibles
microk8s kubectl describe node | grep -A 5 "Allocated resources"

# Lister tous les pods avec leurs requests
microk8s kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): CPU:\(.spec.containers[].resources.requests.cpu // "none") Memory:\(.spec.containers[].resources.requests.memory // "none")"'
```

Analysez la pression sur les ressources :

```bash
# Vérifier les conditions du nœud
microk8s kubectl get nodes -o json | jq '.items[].status.conditions'

# Identifier les pods les plus gourmands
microk8s kubectl top pods --all-namespaces --sort-by=cpu
microk8s kubectl top pods --all-namespaces --sort-by=memory

# Calculer le total des requests
microk8s kubectl get pods --all-namespaces -o json | \
  jq '[.items[].spec.containers[].resources.requests.memory // "0Mi"] | map(gsub("[^0-9]";"") | tonumber) | add'
```

### Résolution

Libérer des ressources :

```bash
# Identifier et supprimer les pods evicted
microk8s kubectl get pods --all-namespaces | grep Evicted | \
  awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod

# Nettoyer les pods terminés
microk8s kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces
microk8s kubectl delete pods --field-selector=status.phase=Failed --all-namespaces

# Réduire les requests des deployments non critiques
microk8s kubectl edit deployment <deployment-name> -n <namespace>
# Modifier les sections resources.requests
```

Optimiser l'allocation des ressources :

```yaml
# exemple-optimized-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            memory: "64Mi"    # Minimum garanti
            cpu: "50m"        # 5% d'un CPU
          limits:
            memory: "128Mi"   # Maximum autorisé
            cpu: "100m"       # 10% d'un CPU
```

## Problème 2 : OOMKilled (Out Of Memory)

### Symptômes

- Pods redémarrent fréquemment
- Status "OOMKilled" dans les events
- Logs s'arrêtent brusquement
- Applications deviennent lentes avant de crasher

### Diagnostic

Identifiez les problèmes de mémoire :

```bash
# Vérifier les pods OOMKilled
microk8s kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.status.containerStatuses[]?.lastState.terminated.reason == "OOMKilled") | "\(.metadata.namespace)/\(.metadata.name)"'

# Examiner l'historique d'un pod
microk8s kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Last State"

# Vérifier l'utilisation mémoire actuelle
microk8s kubectl top pod <pod-name> -n <namespace> --containers

# Analyser les limites vs utilisation
microk8s kubectl get pod <pod-name> -n <namespace> -o json | \
  jq '.spec.containers[] | {name: .name, limits: .resources.limits, requests: .resources.requests}'

# Surveiller l'utilisation en temps réel
watch -n 2 "microk8s kubectl top pod <pod-name> -n <namespace> --containers"
```

Analysez les patterns de consommation :

```bash
# Script pour tracker la mémoire dans le temps
cat <<'EOF' > memory-tracker.sh
#!/bin/bash
POD=$1
NAMESPACE=${2:-default}
INTERVAL=${3:-5}

echo "Tracking memory for $POD in namespace $NAMESPACE every ${INTERVAL}s"
echo "Timestamp,Container,Memory(Mi)"

while true; do
  timestamp=$(date +%Y-%m-%d_%H:%M:%S)
  microk8s kubectl top pod $POD -n $NAMESPACE --containers --no-headers | \
    while read pod container cpu memory; do
      echo "$timestamp,$container,${memory%Mi}"
    done
  sleep $INTERVAL
done
EOF

chmod +x memory-tracker.sh
./memory-tracker.sh <pod-name> <namespace> 5 | tee memory-log.csv
```

### Résolution

Ajuster les limites de mémoire :

```bash
# Augmenter les limites mémoire
microk8s kubectl set resources deployment <deployment-name> \
  -n <namespace> \
  --limits=memory=512Mi \
  --requests=memory=256Mi

# Ou éditer directement
microk8s kubectl edit deployment <deployment-name> -n <namespace>
```

Optimiser l'application :

```yaml
# Configuration avec surveillance de santé
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-optimized-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "512Mi"
        env:
        # Variables JVM pour Java
        - name: JAVA_OPTS
          value: "-Xmx384m -Xms256m -XX:MaxRAMPercentage=75"
        # Pour Node.js
        - name: NODE_OPTIONS
          value: "--max-old-space-size=384"
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "ps aux | grep -v grep | grep java || exit 1"
          periodSeconds: 30
```

## Problème 3 : CPU Throttling

### Symptômes

- Applications lentes mais pods actifs
- Latence élevée des requêtes
- Timeouts fréquents
- Metrics montrant une utilisation CPU proche des limites

### Diagnostic

Identifiez le CPU throttling :

```bash
# Vérifier l'utilisation CPU vs limites
microk8s kubectl top pods --all-namespaces | head -20

# Examiner le throttling (nécessite accès au nœud)
# Pour chaque conteneur, vérifier les stats cgroup
cat /sys/fs/cgroup/cpu/kubepods/pod*/*/cpu.stat | grep throttled

# Via metrics-server
microk8s kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | \
  jq '.items[] | select(.containers[].usage.cpu != null) | {namespace: .metadata.namespace, name: .metadata.name, cpu: .containers[].usage.cpu}'

# Analyser les performances d'un pod spécifique
microk8s kubectl exec -it <pod-name> -n <namespace> -- top
```

Script de détection du throttling :

```bash
#!/bin/bash
# detect-throttling.sh

echo "Détection du CPU throttling..."

# Obtenir tous les pods
pods=$(microk8s kubectl get pods --all-namespaces -o json)

echo "$pods" | jq -r '.items[] | "\(.metadata.namespace) \(.metadata.name)"' | \
while read namespace pod; do
  # Obtenir l'utilisation actuelle
  usage=$(microk8s kubectl top pod $pod -n $namespace --no-headers 2>/dev/null | awk '{print $2}')

  # Obtenir la limite
  limit=$(microk8s kubectl get pod $pod -n $namespace -o json | \
    jq -r '.spec.containers[0].resources.limits.cpu // "none"')

  if [ "$limit" != "none" ] && [ ! -z "$usage" ]; then
    # Convertir en millicores
    usage_m=${usage%m}
    limit_m=${limit%m}

    if [ "$limit_m" -gt 0 ] 2>/dev/null; then
      percent=$((usage_m * 100 / limit_m))
      if [ $percent -gt 80 ]; then
        echo "⚠️  $namespace/$pod: ${percent}% de la limite CPU ($usage/$limit)"
      fi
    fi
  fi
done
```

### Résolution

Optimiser les limites CPU :

```bash
# Augmenter les limites CPU
microk8s kubectl set resources deployment <deployment-name> \
  -n <namespace> \
  --limits=cpu=500m \
  --requests=cpu=200m

# Ou supprimer les limites CPU (utiliser avec précaution)
microk8s kubectl patch deployment <deployment-name> \
  -n <namespace> \
  --type='json' \
  -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/resources/limits/cpu"}]'
```

Configuration équilibrée :

```yaml
# balanced-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
  - max:
      cpu: "2"
      memory: 2Gi
    type: Pod
```

## Problème 4 : Disque plein

### Symptômes

- Erreur "No space left on device"
- Pods en CrashLoopBackOff
- Impossible de créer de nouveaux pods
- Logs qui ne s'écrivent plus

### Diagnostic

Vérifiez l'espace disque :

```bash
# Espace disque sur le nœud
df -h

# Utilisation par répertoire
du -sh /var/snap/microk8s/*

# Trouver les gros fichiers
find /var/snap/microk8s -type f -size +100M -exec ls -lh {} \;

# Vérifier l'espace dans les conteneurs
microk8s kubectl exec -it <pod-name> -n <namespace> -- df -h

# Identifier les pods utilisant le plus d'espace éphémère
for pod in $(microk8s kubectl get pods -o name); do
  echo -n "$pod: "
  microk8s kubectl exec $pod -- du -sh / 2>/dev/null | tail -1
done

# Vérifier les volumes persistants
microk8s kubectl get pv
microk8s kubectl describe pv
```

Analyser l'utilisation des logs :

```bash
# Taille des logs des conteneurs
sudo du -sh /var/snap/microk8s/common/var/lib/containerd/

# Logs système MicroK8s
journalctl --disk-usage

# Identifier les conteneurs avec beaucoup de logs
for container in $(microk8s ctr containers list -q); do
  size=$(microk8s ctr containers info $container | grep -i log | wc -c)
  echo "$container: $size bytes of log config"
done
```

### Résolution

Nettoyer l'espace disque :

```bash
# Nettoyer les images Docker non utilisées
microk8s ctr images prune

# Supprimer les conteneurs arrêtés
microk8s ctr containers list -q | xargs -I {} microk8s ctr containers info {} | \
  grep "STOPPED" -B 5 | grep "ID" | awk '{print $2}' | \
  xargs -I {} microk8s ctr containers delete {}

# Nettoyer les logs système
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=500M

# Nettoyer les pods terminés
microk8s kubectl delete pods --field-selector=status.phase=Failed --all-namespaces
microk8s kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces

# Script de nettoyage complet
cat <<'EOF' > cleanup.sh
#!/bin/bash
echo "=== Nettoyage MicroK8s ==="

echo "1. Suppression des pods evicted/failed..."
microk8s kubectl get pods --all-namespaces | grep -E 'Evicted|Error|Completed' | \
  awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod

echo "2. Nettoyage des images non utilisées..."
microk8s ctr images prune

echo "3. Nettoyage des logs système..."
sudo journalctl --vacuum-time=3d

echo "4. Nettoyage des snapshots MicroK8s..."
sudo snap remove --purge microk8s --revision=-2

echo "Espace libéré:"
df -h /
EOF

chmod +x cleanup.sh
sudo ./cleanup.sh
```

Configurer la rotation des logs :

```yaml
# log-rotation-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-rotation
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: log-rotation
  template:
    metadata:
      labels:
        name: log-rotation
    spec:
      containers:
      - name: log-rotator
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            find /var/log -name "*.log" -size +100M -exec truncate -s 0 {} \;
            sleep 3600
          done
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      hostPID: true
      hostIPC: true
```

## Problème 5 : Performances réseau dégradées

### Symptômes

- Latence élevée entre services
- Timeouts fréquents
- Débit réseau faible
- Connexions qui se perdent

### Diagnostic

Testez les performances réseau :

```bash
# Test de bande passante avec iperf3
# Serveur
microk8s kubectl run iperf-server --image=networkstatic/iperf3 -- -s

# Client
microk8s kubectl run iperf-client --rm -it --image=networkstatic/iperf3 -- \
  iperf3 -c $(microk8s kubectl get pod iperf-server -o jsonpath='{.status.podIP}')

# Test de latence
microk8s kubectl run ping-test --rm -it --image=busybox -- \
  ping -c 100 $(microk8s kubectl get pod iperf-server -o jsonpath='{.status.podIP}')

# Analyser les statistiques réseau du nœud
netstat -s | grep -i drop
netstat -s | grep -i retrans

# Vérifier la charge réseau
iftop -i any
```

Analysez les connexions :

```bash
# Nombre de connexions par pod
for pod in $(microk8s kubectl get pods -o name); do
  echo -n "$pod: "
  microk8s kubectl exec $pod -- netstat -an 2>/dev/null | wc -l
done

# Connexions TIME_WAIT
ss -tan state time-wait | wc -l

# Vérifier les limites de connexion
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
```

### Résolution

Optimiser les paramètres réseau :

```bash
# Augmenter les buffers réseau
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# Optimiser pour la latence
sudo sysctl -w net.ipv4.tcp_low_latency=1
sudo sysctl -w net.ipv4.tcp_nodelay=1

# Augmenter les limites de connexion
sudo sysctl -w net.core.somaxconn=65535
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# Rendre permanent
cat <<EOF | sudo tee -a /etc/sysctl.d/99-kubernetes.conf
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 87380 134217728
net.ipv4.tcp_wmem=4096 65536 134217728
net.core.somaxconn=65535
net.ipv4.tcp_max_syn_backlog=8192
EOF

sudo sysctl -p /etc/sysctl.d/99-kubernetes.conf
```

## Monitoring et prévention

### Mise en place d'alertes

Configuration Prometheus pour les alertes de ressources :

```yaml
# resource-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: resource-alerts
  namespace: monitoring
data:
  alerts.yaml: |
    groups:
    - name: resources
      interval: 30s
      rules:
      - alert: HighMemoryUsage
        expr: |
          (sum(container_memory_working_set_bytes) by (pod, namespace)
          / sum(container_spec_memory_limit_bytes) by (pod, namespace)) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} memory usage > 80%"

      - alert: HighCPUUsage
        expr: |
          sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} CPU usage > 80%"

      - alert: PodOOMKilled
        expr: |
          kube_pod_container_status_terminated_reason{reason="OOMKilled"} > 0
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} was OOMKilled"

      - alert: NodeDiskPressure
        expr: |
          kube_node_status_condition{condition="DiskPressure",status="true"} > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.node }} has disk pressure"
```

### Dashboard de monitoring

Script pour un dashboard de ressources en temps réel :

```bash
#!/bin/bash
# resource-dashboard.sh

clear
while true; do
  echo "=== MicroK8s Resource Dashboard - $(date) ==="
  echo ""

  echo "NODE RESOURCES:"
  microk8s kubectl top nodes
  echo ""

  echo "TOP 10 PODS BY CPU:"
  microk8s kubectl top pods --all-namespaces --sort-by=cpu | head -11
  echo ""

  echo "TOP 10 PODS BY MEMORY:"
  microk8s kubectl top pods --all-namespaces --sort-by=memory | head -11
  echo ""

  echo "PROBLEMATIC PODS:"
  microk8s kubectl get pods --all-namespaces | grep -v Running | grep -v Completed | head -10
  echo ""

  echo "DISK USAGE:"
  df -h / | tail -1
  echo ""

  echo "Press Ctrl+C to exit. Refreshing in 10 seconds..."
  sleep 10
  clear
done
```

### Recommandations de dimensionnement

Pour un lab personnel MicroK8s optimal :

```yaml
# recommendations.yaml
# Minimum recommandé pour le système
System Requirements:
  CPU: 2 cores minimum, 4 cores recommandé
  RAM: 4GB minimum, 8GB recommandé
  Disk: 20GB minimum, 50GB recommandé

# Template de ressources par type d'application
Application Templates:

  # Application web légère
  lightweight-web:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi

  # Base de données
  database:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 2Gi

  # Application métier
  business-app:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 1Gi

  # Service de cache
  cache:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 100m
      memory: 512Mi
```

### Script d'optimisation automatique

```bash
#!/bin/bash
# auto-optimize.sh

echo "=== Optimisation automatique des ressources ==="

# 1. Nettoyer les ressources inutilisées
echo "Nettoyage des ressources..."
microk8s kubectl delete pods --field-selector=status.phase=Failed --all-namespaces 2>/dev/null
microk8s kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces 2>/dev/null
microk8s ctr images prune 2>/dev/null

# 2. Identifier les pods sans limites
echo -e "\nPods sans limites de ressources:"
microk8s kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits == null) | "\(.metadata.namespace)/\(.metadata.name)"'

# 3. Suggérer des optimisations
echo -e "\nAnalyse des optimisations possibles:"

# Pods utilisant moins de 50% de leurs requests
microk8s kubectl top pods --all-namespaces --no-headers | while read ns pod cpu mem; do
  requests=$(microk8s kubectl get pod $pod -n $ns -o json 2>/dev/null | \
    jq -r '.spec.containers[0].resources.requests.cpu // "0"')

  if [ "$requests" != "0" ]; then
    cpu_num=${cpu%m}
    req_num=${requests%m}
    if [ $req_num -gt 0 ] 2>/dev/null; then
      percent=$((cpu_num * 100 / req_num))
      if [ $percent -lt 50 ]; then
        echo "  ℹ️  $ns/$pod utilise seulement ${percent}% de ses CPU requests"
      fi
    fi
  fi
done

# 4. Vérifier l'overcommit
echo -e "\nRatio d'overcommit:"
total_requests=$(microk8s kubectl get pods --all-namespaces -o json | \
  jq '[.items[].spec.containers[].resources.requests.cpu // "0m"] | map(gsub("m";"") | tonumber) | add')

total_capacity=$(microk8s kubectl get nodes -o json | \
  jq '[.items[].status.capacity.cpu | tonumber * 1000] | add')

if [ $total_capacity -gt 0 ]; then
  ratio=$(echo "scale=2; $total_requests / $total_capacity" | bc)
  echo "  CPU Requests/Capacity: ${ratio}"

  if (( $(echo "$ratio > 1" | bc -l) )); then
    echo "  ⚠️  Attention: Overcommit détecté!"
  fi
fi
```

## Bonnes pratiques pour la performance

### Stratégies d'optimisation

1. **Toujours définir des requests et limits** appropriées
2. **Surveiller régulièrement** l'utilisation des ressources
3. **Utiliser l'autoscaling** pour gérer la charge variable
4. **Nettoyer régulièrement** les ressources inutilisées
5. **Optimiser les images** Docker (multi-stage builds, images minimales)

### Configuration de l'Horizontal Pod Autoscaler

```yaml
# hpa-example.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

## Conclusion

La gestion des performances et des ressources dans MicroK8s est un équilibre délicat entre fournir suffisamment de ressources pour que les applications fonctionnent correctement et éviter le gaspillage. Les problèmes de performance peuvent cascade rapidement - un manque de mémoire peut causer des OOMKills, qui causent des redémarrages, qui augmentent la charge CPU, créant un cercle vicieux. La clé est la surveillance proactive, la configuration appropriée des ressources, et la maintenance régulière. Avec les outils et techniques présentés dans cette section, vous pouvez maintenir votre cluster MicroK8s performant même avec des ressources limitées. N'oubliez pas que l'optimisation est un processus continu - surveillez, analysez, ajustez, et répétez.

⏭️
