🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.7 Premier diagnostic du cluster

## Introduction

Maintenant que MicroK8s est installé et configuré, il est essentiel de comprendre comment diagnostiquer l'état de votre cluster. Cette compétence vous permettra d'identifier rapidement les problèmes, de comprendre le comportement de votre cluster et d'assurer son bon fonctionnement. Ce premier diagnostic établira une base de référence pour l'état "sain" de votre cluster, que vous pourrez utiliser pour comparaison lors de futurs dépannages.

**Objectifs du diagnostic initial** :
- Établir une baseline de l'état normal du cluster
- Identifier les composants critiques et leur état
- Comprendre l'utilisation des ressources
- Détecter les problèmes potentiels avant qu'ils ne deviennent critiques
- Se familiariser avec les outils de diagnostic

## Vue d'ensemble du cluster

### Architecture et composants

Avant de diagnostiquer, comprenons ce que nous examinons :

```bash
# Afficher une vue synthétique du cluster
microk8s kubectl cluster-info

# Sortie typique :
# Kubernetes control plane is running at https://127.0.0.1:16443
# CoreDNS is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Pour plus de détails
microk8s kubectl cluster-info dump > cluster-info-dump.txt
echo "Dump sauvegardé dans cluster-info-dump.txt ($(wc -l cluster-info-dump.txt | awk '{print $1}') lignes)"
```

### État général du système

```bash
# Script de diagnostic rapide
cat << 'EOF' > ~/cluster-diagnostic.sh
#!/bin/bash

echo "=========================================="
echo "     DIAGNOSTIC DU CLUSTER MICROK8S      "
echo "=========================================="
echo ""

# Informations système
echo "📊 INFORMATIONS SYSTÈME"
echo "------------------------"
echo "Hostname: $(hostname)"
echo "OS: $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2)"
echo "Kernel: $(uname -r)"
echo "Uptime: $(uptime -p)"
echo "Date: $(date)"
echo ""

# État MicroK8s
echo "🚀 ÉTAT MICROK8s"
echo "----------------"
microk8s status | head -5
echo ""

# Version Kubernetes
echo "☸️  VERSION KUBERNETES"
echo "---------------------"
microk8s kubectl version --short
echo ""

# État des nodes
echo "🖥️  ÉTAT DU NODE"
echo "----------------"
microk8s kubectl get nodes -o wide
echo ""

# Utilisation des ressources
echo "💾 RESSOURCES SYSTÈME"
echo "--------------------"
echo "CPU disponibles: $(nproc)"
echo "Mémoire totale: $(free -h | grep Mem | awk '{print $2}')"
echo "Mémoire utilisée: $(free -h | grep Mem | awk '{print $3}')"
echo "Espace disque /: $(df -h / | tail -1 | awk '{print $4}') libre sur $(df -h / | tail -1 | awk '{print $2}')"
echo ""

EOF

chmod +x ~/cluster-diagnostic.sh
~/cluster-diagnostic.sh
```

## Diagnostic des composants du Control Plane

### Vérification de l'API Server

L'API Server est le cœur de Kubernetes :

```bash
# Test de santé de l'API Server
curl -k https://127.0.0.1:16443/healthz

# Si vous avez configuré kubectl
microk8s kubectl get --raw /healthz

# Vérifier les endpoints disponibles
microk8s kubectl get --raw /healthz?verbose

# Points de vérification importants :
# etcd : Base de données du cluster
# scheduler : Planification des pods
# controller-manager : Gestion des contrôleurs
```

### État d'etcd

etcd stocke toutes les données du cluster :

```bash
# Vérifier la santé d'etcd (si accessible)
microk8s kubectl get --raw /healthz/etcd

# Voir les métriques etcd (si metrics-server activé)
microk8s kubectl get --raw /metrics | grep etcd | head -20

# Vérifier la taille de la base de données
sudo du -sh /var/snap/microk8s/current/var/kubernetes/backend/
```

### Scheduler et Controller Manager

```bash
# Vérifier les composants du control plane
microk8s kubectl get componentstatuses

# Note: Cette commande peut être dépréciée dans les versions récentes
# Alternative : vérifier directement les pods
microk8s kubectl get pods -n kube-system | grep -E "(scheduler|controller)"

# Logs du scheduler
microk8s kubectl logs -n kube-system -l component=kube-scheduler --tail=50

# Logs du controller manager
microk8s kubectl logs -n kube-system -l component=kube-controller-manager --tail=50
```

## Diagnostic du réseau

### Configuration réseau de base

```bash
# Vérifier le plugin CNI (Container Network Interface)
echo "Plugin CNI utilisé:"
microk8s kubectl get pods -n kube-system | grep -E "(calico|flannel|cilium)"

# Vérifier les interfaces réseau
ip addr show | grep -E "(cali|vxlan|flannel|cilium)"

# Vérifier les règles iptables (nombre)
sudo iptables -L -n | wc -l
echo "Nombre total de règles iptables: $(sudo iptables -L -n | wc -l)"

# Test de connectivité DNS
microk8s kubectl run test-dns --image=busybox:latest --rm -it --restart=Never -- nslookup kubernetes.default

# Vérifier CoreDNS
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### Test de connectivité pod-to-pod

```bash
# Créer deux pods de test
cat << EOF > ~/network-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-1
  labels:
    app: test-1
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-2
  labels:
    app: test-2
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ['sleep', '3600']
EOF

microk8s kubectl apply -f ~/network-test.yaml

# Attendre que les pods soient prêts
microk8s kubectl wait --for=condition=ready pod/test-pod-1 pod/test-pod-2 --timeout=60s

# Obtenir les IPs
POD1_IP=$(microk8s kubectl get pod test-pod-1 -o jsonpath='{.status.podIP}')
echo "Pod 1 IP: $POD1_IP"

# Test de connectivité
microk8s kubectl exec test-pod-2 -- wget -O- --timeout=5 http://$POD1_IP

# Nettoyage
microk8s kubectl delete -f ~/network-test.yaml
```

## Diagnostic du stockage

### Vérification du stockage local

```bash
# Vérifier l'espace disponible
echo "=== Espace disque pour MicroK8s ==="
df -h | grep -E "(/$|/var|snap)"

# Vérifier les StorageClasses disponibles
echo ""
echo "=== StorageClasses disponibles ==="
microk8s kubectl get storageclass

# Si le storage addon est activé
if microk8s status | grep -q "storage.*enabled"; then
    echo ""
    echo "=== PersistentVolumes ==="
    microk8s kubectl get pv

    echo ""
    echo "=== PersistentVolumeClaims ==="
    microk8s kubectl get pvc --all-namespaces
fi

# Vérifier l'utilisation du storage par les conteneurs
echo ""
echo "=== Top 10 répertoires par taille ==="
sudo du -sh /var/snap/microk8s/common/var/lib/containerd/* 2>/dev/null | sort -rh | head -10
```

### Test de création de volume

```bash
# Créer un PVC de test (si storage addon activé)
cat << EOF > ~/test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Appliquer et vérifier
microk8s kubectl apply -f ~/test-pvc.yaml
microk8s kubectl get pvc test-pvc

# Nettoyer
microk8s kubectl delete -f ~/test-pvc.yaml
```

## Analyse des performances

### Métriques du système

```bash
# Activer metrics-server si pas déjà fait
microk8s enable metrics-server

# Attendre que metrics-server soit prêt (environ 1 minute)
sleep 60

# Métriques du node
echo "=== Utilisation des ressources du node ==="
microk8s kubectl top node

# Métriques des pods système
echo ""
echo "=== Top 10 pods par CPU ==="
microk8s kubectl top pods --all-namespaces | sort -k3 -rn | head -11

echo ""
echo "=== Top 10 pods par Mémoire ==="
microk8s kubectl top pods --all-namespaces | sort -k4 -rn | head -11
```

### Analyse de la latence

```bash
# Test de latence de l'API
echo "=== Test de latence API (10 requêtes) ==="
for i in {1..10}; do
    time microk8s kubectl get nodes > /dev/null 2>&1
done 2>&1 | grep real | awk '{sum+=$2; count++} END {print "Latence moyenne:", sum/count*1000, "ms"}'

# Test de création/suppression de pod
echo ""
echo "=== Test de cycle de vie d'un pod ==="
time (
    microk8s kubectl run test-latency --image=busybox --restart=Never -- echo "test"
    microk8s kubectl wait --for=condition=ready pod/test-latency --timeout=30s
    microk8s kubectl delete pod test-latency --wait=true
) 2>&1 | grep real
```

## Analyse des logs et événements

### Événements du cluster

```bash
# Script pour analyser les événements
cat << 'EOF' > ~/analyze-events.sh
#!/bin/bash

echo "=== ANALYSE DES ÉVÉNEMENTS DU CLUSTER ==="
echo ""

# Événements récents (dernière heure)
echo "📅 Événements de la dernière heure:"
microk8s kubectl get events --all-namespaces \
    --sort-by='.lastTimestamp' \
    -o custom-columns=TIME:.lastTimestamp,NAMESPACE:.namespace,NAME:.involvedObject.name,TYPE:.type,REASON:.reason,MESSAGE:.message \
    | head -20

echo ""
echo "⚠️  Événements Warning:"
microk8s kubectl get events --all-namespaces --field-selector type=Warning \
    -o custom-columns=TIME:.lastTimestamp,NAMESPACE:.namespace,NAME:.involvedObject.name,REASON:.reason \
    | head -10

echo ""
echo "📊 Statistiques des événements:"
echo "Total: $(microk8s kubectl get events --all-namespaces --no-headers | wc -l)"
echo "Warnings: $(microk8s kubectl get events --all-namespaces --field-selector type=Warning --no-headers | wc -l)"
echo "Normal: $(microk8s kubectl get events --all-namespaces --field-selector type=Normal --no-headers | wc -l)"

EOF

chmod +x ~/analyze-events.sh
~/analyze-events.sh
```

### Analyse des logs système

```bash
# Fonction pour analyser les logs
analyze_logs() {
    local component=$1
    local namespace=${2:-kube-system}

    echo "=== Analyse des logs: $component ==="

    # Compter les erreurs
    local errors=$(microk8s kubectl logs -n $namespace -l $component --tail=1000 2>/dev/null | grep -iE "(error|fail|panic)" | wc -l)
    local warnings=$(microk8s kubectl logs -n $namespace -l $component --tail=1000 2>/dev/null | grep -iE "(warn|warning)" | wc -l)

    echo "Erreurs trouvées: $errors"
    echo "Warnings trouvés: $warnings"

    if [ $errors -gt 0 ]; then
        echo "Dernières erreurs:"
        microk8s kubectl logs -n $namespace -l $component --tail=1000 2>/dev/null | grep -iE "(error|fail|panic)" | tail -5
    fi
    echo ""
}

# Analyser les composants principaux
analyze_logs "k8s-app=kube-dns"
analyze_logs "k8s-app=calico-node"
```

## Diagnostic de sécurité

### Vérification des certificats

```bash
# Vérifier l'expiration des certificats
echo "=== État des certificats ==="
for cert in /var/snap/microk8s/current/certs/*.crt; do
    if [ -f "$cert" ]; then
        expiry=$(openssl x509 -in "$cert" -noout -enddate 2>/dev/null | cut -d= -f2)
        name=$(basename "$cert")
        echo "$name expire le: $expiry"
    fi
done

# Vérifier les jours restants avant expiration
echo ""
echo "=== Jours avant expiration ==="
for cert in /var/snap/microk8s/current/certs/*.crt; do
    if [ -f "$cert" ]; then
        days=$(( ($(date -d "$(openssl x509 -in "$cert" -noout -enddate 2>/dev/null | cut -d= -f2)" +%s) - $(date +%s)) / 86400 ))
        name=$(basename "$cert")
        if [ $days -lt 30 ]; then
            echo "⚠️  $name: $days jours (ATTENTION: renouvellement nécessaire)"
        else
            echo "✅ $name: $days jours"
        fi
    fi
done
```

### Audit des permissions

```bash
# Vérifier les ServiceAccounts
echo "=== ServiceAccounts par namespace ==="
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    count=$(microk8s kubectl get sa -n $ns --no-headers 2>/dev/null | wc -l)
    if [ $count -gt 0 ]; then
        echo "$ns: $count ServiceAccounts"
    fi
done

# Vérifier les ClusterRoleBindings
echo ""
echo "=== ClusterRoleBindings critiques ==="
microk8s kubectl get clusterrolebindings -o json | \
    jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# Vérifier les pods avec privileged
echo ""
echo "=== Pods avec privileges élevés ==="
microk8s kubectl get pods --all-namespaces -o json | \
    jq -r '.items[] | select(.spec.containers[].securityContext.privileged==true) | .metadata.namespace + "/" + .metadata.name' 2>/dev/null
```

## Création d'un rapport de diagnostic

### Script de rapport complet

```bash
cat << 'EOF' > ~/generate-diagnostic-report.sh
#!/bin/bash

REPORT_DIR="$HOME/microk8s-diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$REPORT_DIR"

echo "Génération du rapport de diagnostic dans $REPORT_DIR..."

# Fonction pour capturer une commande
capture() {
    local title=$1
    local command=$2
    local file=$3

    echo "Capture: $title..."
    echo "=== $title ===" >> "$REPORT_DIR/$file"
    echo "Date: $(date)" >> "$REPORT_DIR/$file"
    echo "Commande: $command" >> "$REPORT_DIR/$file"
    echo "" >> "$REPORT_DIR/$file"
    eval $command >> "$REPORT_DIR/$file" 2>&1
    echo "" >> "$REPORT_DIR/$file"
}

# Informations système
capture "Informations système" "uname -a" "system-info.txt"
capture "Version OS" "cat /etc/os-release" "system-info.txt"
capture "Utilisation CPU" "top -bn1 | head -20" "system-info.txt"
capture "Utilisation mémoire" "free -h" "system-info.txt"
capture "Utilisation disque" "df -h" "system-info.txt"

# État MicroK8s
capture "Status MicroK8s" "microk8s status" "microk8s-status.txt"
capture "Version MicroK8s" "microk8s version" "microk8s-status.txt"
capture "Inspect MicroK8s" "microk8s inspect" "microk8s-status.txt"

# État Kubernetes
capture "Nodes" "microk8s kubectl get nodes -o wide" "kubernetes-state.txt"
capture "Namespaces" "microk8s kubectl get namespaces" "kubernetes-state.txt"
capture "Tous les pods" "microk8s kubectl get pods --all-namespaces -o wide" "kubernetes-state.txt"
capture "Services" "microk8s kubectl get services --all-namespaces" "kubernetes-state.txt"
capture "Deployments" "microk8s kubectl get deployments --all-namespaces" "kubernetes-state.txt"

# Événements et logs
capture "Événements récents" "microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp' | head -50" "events.txt"
capture "Événements Warning" "microk8s kubectl get events --all-namespaces --field-selector type=Warning" "events.txt"

# Métriques (si disponible)
capture "Top nodes" "microk8s kubectl top nodes" "metrics.txt" 2>/dev/null
capture "Top pods" "microk8s kubectl top pods --all-namespaces" "metrics.txt" 2>/dev/null

# Réseau
capture "Configuration réseau" "ip addr show" "network.txt"
capture "Routes" "ip route show" "network.txt"
capture "Règles iptables (count)" "sudo iptables -L -n | wc -l" "network.txt"

# Générer un résumé
cat << SUMMARY > "$REPORT_DIR/SUMMARY.txt"
RAPPORT DE DIAGNOSTIC MICROK8S
==============================
Date: $(date)
Hostname: $(hostname)

RÉSUMÉ EXÉCUTIF
---------------
État MicroK8s: $(microk8s status | grep "microk8s is" | head -1)
Nodes: $(microk8s kubectl get nodes --no-headers | wc -l)
Pods Running: $(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Running --no-headers | wc -l)
Pods avec problèmes: $(microk8s kubectl get pods --all-namespaces --field-selector=status.phase!=Running --no-headers | wc -l)
Événements Warning: $(microk8s kubectl get events --all-namespaces --field-selector type=Warning --no-headers | wc -l)

POINTS D'ATTENTION
-----------------
$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase!=Running --no-headers 2>/dev/null | head -5)

RECOMMANDATIONS
--------------
- Vérifier les pods non-Running listés ci-dessus
- Examiner les événements Warning dans events.txt
- Contrôler l'utilisation des ressources dans metrics.txt

FICHIERS GÉNÉRÉS
---------------
$(ls -la "$REPORT_DIR" | tail -n +2)

SUMMARY

echo ""
echo "✅ Rapport généré avec succès!"
echo "📁 Emplacement: $REPORT_DIR"
echo "📊 Résumé: $REPORT_DIR/SUMMARY.txt"
echo ""
echo "Pour créer une archive:"
echo "  tar -czf microk8s-diagnostic.tar.gz -C $HOME $(basename $REPORT_DIR)"

EOF

chmod +x ~/generate-diagnostic-report.sh
```

## Tableau de référence rapide

### Indicateurs de santé

| Composant | Commande de vérification | État sain | État problématique |
|-----------|-------------------------|-----------|-------------------|
| API Server | `curl -k https://127.0.0.1:16443/healthz` | `ok` | Timeout ou erreur |
| Node | `kubectl get nodes` | `Ready` | `NotReady`, `Unknown` |
| Pods système | `kubectl get pods -n kube-system` | Tous `Running 1/1` | `Pending`, `CrashLoopBackOff` |
| CoreDNS | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | `Running` | Redémarrages fréquents |
| Network | Test de connectivité pod-to-pod | Succès | Timeout |
| Certificats | Vérifier expiration | >30 jours | <30 jours |
| Événements | `kubectl get events --field-selector type=Warning` | <10 warnings | Nombreux warnings |
| Ressources | `kubectl top nodes` | CPU <80%, Mem <80% | >90% utilisation |

### Actions correctives communes

```bash
# Créer un script d'actions correctives
cat << 'EOF' > ~/fix-common-issues.sh
#!/bin/bash

echo "=== ACTIONS CORRECTIVES COMMUNES ==="
echo ""

# Fonction pour demander confirmation
confirm() {
    read -p "$1 (y/n): " response
    [[ "$response" == "y" ]]
}

# Redémarrer MicroK8s
if confirm "Redémarrer MicroK8s?"; then
    echo "Redémarrage de MicroK8s..."
    microk8s stop
    microk8s start
    microk8s status --wait-ready
fi

# Redémarrer CoreDNS
if confirm "Redémarrer CoreDNS?"; then
    echo "Redémarrage de CoreDNS..."
    microk8s kubectl rollout restart deployment/coredns -n kube-system
fi

# Nettoyer les pods terminés
if confirm "Nettoyer les pods terminés?"; then
    echo "Nettoyage des pods..."
    microk8s kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces
    microk8s kubectl delete pods --field-selector=status.phase=Failed --all-namespaces
fi

# Régénérer les certificats (si nécessaire)
if confirm "Régénérer les certificats?"; then
    echo "Régénération des certificats..."
    microk8s refresh-certs
fi

echo ""
echo "Actions correctives terminées!"

EOF

chmod +x ~/fix-common-issues.sh
```

## Monitoring continu

### Configuration d'un monitoring basique

```bash
# Script de monitoring en temps réel
cat << 'EOF' > ~/monitor-cluster.sh
#!/bin/bash

# Fonction pour afficher avec couleur
print_status() {
    local status=$1
    local message=$2

    if [ "$status" = "OK" ]; then
        echo -e "\033[32m✓\033[0m $message"
    elif [ "$status" = "WARNING" ]; then
        echo -e "\033[33m⚠\033[0m $message"
    else
        echo -e "\033[31m✗\033[0m $message"
    fi
}

while true; do
    clear
    echo "==================================="
    echo "   MONITORING CLUSTER MICROK8S    "
    echo "   $(date)"
    echo "==================================="
    echo ""

    # État MicroK8s
    if microk8s status | grep -q "is running"; then
        print_status "OK" "MicroK8s is running"
    else
        print_status "ERROR" "MicroK8s is not running"
    fi

    # État du node
    node_status=$(microk8s kubectl get nodes --no-headers | awk '{print $2}')
    if [ "$node_status" = "Ready" ]; then
        print_status "OK" "Node is Ready"
    else
        print_status "ERROR" "Node status: $node_status"
    fi

    # Pods avec problèmes
    problem_pods=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase!=Running --no-headers | wc -l)
    if [ $problem_pods -eq 0 ]; then
        print_status "OK" "All pods are running"
    else
        print_status "WARNING" "$problem_pods pods with issues"
    fi

    # Utilisation des ressources
    if command -v microk8s kubectl top node &>/dev/null; then
        cpu_usage=$(microk8s kubectl top node --no-headers | awk '{print $3}' | sed 's/%//')
        mem_usage=$(microk8s kubectl top node --no-headers | awk '{print $5}' | sed 's/%//')

        if [ "$cpu_usage" -lt 80 ]; then
            print_status "OK" "CPU usage: $cpu_usage%"
        else
            print_status "WARNING" "CPU usage: $cpu_usage%"
        fi

        if [ "$mem_usage" -lt 80 ]; then
            print_status "OK" "Memory usage: $mem_usage%"
        else
            print_status "WARNING" "Memory usage: $mem_usage%"
        fi
    fi

    # Événements récents
    recent_warnings=$(microk8s kubectl get events --all-namespaces --field-selector type=Warning --no-headers | wc -l)
    if [ $recent_warnings -lt 5 ]; then
        print_status "OK" "$recent_warnings warning events"
    else
        print_status "WARNING" "$recent_warnings warning events"
    fi

    echo ""
    echo "Rafraîchissement dans 30 secondes... (Ctrl+C pour quitter)"
    sleep 30
done

EOF

chmod +x ~/monitor-cluster.sh
```

## Points de vérification finale

Après ce premier diagnostic, vous devriez avoir :

- ✅ Compris l'architecture de votre cluster
- ✅ Identifié tous les composants critiques
- ✅ Établi une baseline de performance
- ✅ Créé des scripts de diagnostic réutilisables
- ✅ Généré un rapport de référence
- ✅ Mis en place un monitoring basique
- ✅ Identifié et résolu les problèmes éventuels

## Prochaines étapes

Avec ce diagnostic initial complété, vous êtes prêt à :

1. Configurer le réseau et les domaines (Section 3)
2. Activer les addons nécessaires (Section 4)
3. Déployer vos premières applications
4. Mettre en place un monitoring plus avancé

Ce diagnostic initial servira de référence pour tous vos futurs dépannages et optimisations du cluster.

⏭️
