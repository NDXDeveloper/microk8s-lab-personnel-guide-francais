🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.7 Nettoyage et maintenance

## Introduction à la maintenance préventive

Un cluster Kubernetes, même dans un environnement de lab personnel, accumule au fil du temps des ressources inutilisées : images Docker orphelines, pods terminés, logs volumineux, volumes non réclamés, et configurations obsolètes. Sans maintenance régulière, ces éléments peuvent consommer un espace disque précieux, ralentir les performances, et rendre le dépannage plus difficile. Cette section vous guidera à travers les pratiques de nettoyage et de maintenance essentielles pour garder votre cluster MicroK8s propre, performant et facile à gérer.

## Comprendre ce qui s'accumule dans un cluster

### Les types de ressources à nettoyer

Dans un cluster MicroK8s, plusieurs types de ressources peuvent s'accumuler et nécessiter un nettoyage régulier :

**Images de conteneurs** : Chaque fois que vous déployez une nouvelle version d'une application, l'ancienne image reste stockée localement. Avec le temps, ces images peuvent occuper des dizaines de gigaoctets.

**Pods terminés** : Les pods en état Completed, Failed, ou Evicted restent dans le cluster jusqu'à suppression manuelle. Ils n'utilisent pas de ressources actives mais encombrent les listings et peuvent masquer des problèmes actuels.

**Logs** : Les logs des conteneurs, du système, et de MicroK8s lui-même croissent continuellement. Sans rotation appropriée, ils peuvent remplir le disque.

**Volumes orphelins** : Les PersistentVolumes dont les PersistentVolumeClaims ont été supprimés peuvent rester et occuper de l'espace.

**ConfigMaps et Secrets inutilisés** : Les configurations d'anciennes applications peuvent persister même après la suppression des applications.

**Snapshots et sauvegardes** : Les sauvegardes automatiques et les snapshots s'accumulent si aucune politique de rétention n'est en place.

### Impact de l'accumulation

L'accumulation de ressources inutilisées a plusieurs impacts négatifs :

- **Espace disque** : Le plus évident, pouvant mener à l'impossibilité de créer de nouveaux pods
- **Performance** : Plus d'objets à gérer signifie plus de charge pour l'API server
- **Visibilité** : Difficile de distinguer les ressources actives des anciennes
- **Sécurité** : Les vieux secrets et configurations peuvent contenir des informations sensibles obsolètes
- **Coût de maintenance** : Plus de temps nécessaire pour naviguer et gérer le cluster

## Nettoyage des pods et des ressources Kubernetes

### Suppression des pods terminés

Commençons par le nettoyage le plus basique et le plus fréquent :

```bash
# Voir tous les pods non-running
microk8s kubectl get pods --all-namespaces | grep -v Running

# Supprimer les pods en état Completed
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded

# Supprimer les pods en état Failed
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed

# Supprimer les pods Evicted
microk8s kubectl get pods --all-namespaces | grep Evicted | \
  awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod

# Script de nettoyage complet des pods
cat <<'EOF' > clean-pods.sh
#!/bin/bash
echo "=== Nettoyage des pods terminés ==="

# Compter avant nettoyage
COMPLETED=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded --no-headers | wc -l)
FAILED=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Failed --no-headers | wc -l)
EVICTED=$(microk8s kubectl get pods --all-namespaces | grep -c Evicted)

echo "Pods à nettoyer:"
echo "  Completed: $COMPLETED"
echo "  Failed: $FAILED"
echo "  Evicted: $EVICTED"

if [ $((COMPLETED + FAILED + EVICTED)) -eq 0 ]; then
  echo "Aucun pod à nettoyer"
  exit 0
fi

# Nettoyage
echo "Suppression en cours..."
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded 2>/dev/null
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed 2>/dev/null
microk8s kubectl get pods --all-namespaces | grep Evicted | \
  awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod 2>/dev/null

echo "✅ Nettoyage terminé"
EOF

chmod +x clean-pods.sh
```

### Nettoyage des ReplicaSets obsolètes

Les déploiements gardent par défaut l'historique des ReplicaSets :

```bash
# Voir tous les ReplicaSets avec 0 replicas
microk8s kubectl get replicasets --all-namespaces | grep "0         0         0"

# Script pour nettoyer les vieux ReplicaSets
cat <<'EOF' > clean-replicasets.sh
#!/bin/bash
echo "=== Nettoyage des ReplicaSets obsolètes ==="

# Pour chaque namespace
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  # Pour chaque deployment
  for deploy in $(microk8s kubectl get deployments -n $ns -o jsonpath='{.items[*].metadata.name}'); do
    # Garder seulement les 2 dernières révisions
    microk8s kubectl rollout history deployment/$deploy -n $ns --revision=0 >/dev/null 2>&1
    if [ $? -eq 0 ]; then
      REVISIONS=$(microk8s kubectl rollout history deployment/$deploy -n $ns | grep -E "^[0-9]+" | wc -l)
      if [ $REVISIONS -gt 2 ]; then
        echo "Nettoyage de $ns/$deploy (gardant 2 dernières révisions)"
        # Limiter l'historique
        microk8s kubectl patch deployment $deploy -n $ns \
          -p '{"spec":{"revisionHistoryLimit":2}}'
      fi
    fi
  done
done

# Supprimer les ReplicaSets avec 0 replicas et sans owner
echo "Suppression des ReplicaSets orphelins..."
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  microk8s kubectl get rs -n $ns -o json | \
    jq -r '.items[] | select(.spec.replicas==0 and .status.replicas==0 and .metadata.ownerReferences==null) | .metadata.name' | \
    xargs -I {} microk8s kubectl delete rs {} -n $ns 2>/dev/null
done

echo "✅ Nettoyage des ReplicaSets terminé"
EOF

chmod +x clean-replicasets.sh
```

### Nettoyage des ConfigMaps et Secrets inutilisés

Identifier et supprimer les configurations orphelines :

```bash
# Script pour trouver les ConfigMaps et Secrets non référencés
cat <<'EOF' > find-unused-configs.sh
#!/bin/bash
echo "=== Recherche des ConfigMaps et Secrets non utilisés ==="

NAMESPACE=${1:-default}

# Fonction pour vérifier si une ressource est utilisée
is_used() {
  local type=$1
  local name=$2
  local namespace=$3

  # Vérifier dans les pods
  used_in_pods=$(microk8s kubectl get pods -n $namespace -o json | \
    jq -r --arg name "$name" --arg type "$type" \
    '.items[].spec |
     (.volumes[]? | select(.[$type].name == $name) | .[$type].name),
     (.containers[].env[]? | select(.valueFrom[$type+"KeyRef"].name == $name) | .valueFrom[$type+"KeyRef"].name),
     (.containers[].envFrom[]? | select(.[$type+"Ref"].name == $name) | .[$type+"Ref"].name)' 2>/dev/null)

  [ ! -z "$used_in_pods" ]
}

# Vérifier les ConfigMaps
echo "ConfigMaps non utilisés dans namespace $NAMESPACE:"
for cm in $(microk8s kubectl get configmaps -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  # Ignorer les ConfigMaps système
  if [[ $cm == kube-* ]] || [[ $cm == *-token-* ]]; then
    continue
  fi

  if ! is_used "configMap" "$cm" "$NAMESPACE"; then
    echo "  - $cm"
  fi
done

# Vérifier les Secrets
echo -e "\nSecrets non utilisés dans namespace $NAMESPACE:"
for secret in $(microk8s kubectl get secrets -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  # Ignorer les secrets système
  secret_type=$(microk8s kubectl get secret $secret -n $NAMESPACE -o jsonpath='{.type}')
  if [[ $secret_type == "kubernetes.io/service-account-token" ]] || [[ $secret == sh.helm.* ]]; then
    continue
  fi

  if ! is_used "secret" "$secret" "$NAMESPACE"; then
    echo "  - $secret (type: $secret_type)"
  fi
done

echo -e "\nNote: Vérifiez manuellement avant de supprimer ces ressources"
EOF

chmod +x find-unused-configs.sh
```

## Nettoyage des images Docker

### Suppression des images non utilisées

Les images Docker peuvent rapidement consommer beaucoup d'espace :

```bash
# Voir l'utilisation actuelle des images
microk8s ctr images list

# Voir l'espace utilisé par les images
microk8s ctr images list | awk '{sum+=$7} END {print "Total: " sum/1024/1024/1024 " GB"}'

# Supprimer les images non utilisées (prune)
microk8s ctr images prune

# Script de nettoyage intelligent des images
cat <<'EOF' > clean-images.sh
#!/bin/bash
echo "=== Nettoyage des images Docker ==="

# Obtenir la liste des images utilisées par les pods
echo "Collecte des images actuellement utilisées..."
USED_IMAGES=$(microk8s kubectl get pods --all-namespaces -o json | \
  jq -r '.items[].spec.containers[].image' | sort -u)

# Obtenir toutes les images locales
ALL_IMAGES=$(microk8s ctr images list -q)

echo "Images présentes: $(echo "$ALL_IMAGES" | wc -l)"
echo "Images utilisées: $(echo "$USED_IMAGES" | wc -l)"

# Calculer l'espace avant nettoyage
BEFORE=$(df -h / | tail -1 | awk '{print $4}')

# Nettoyer les images non utilisées
echo "Suppression des images non utilisées..."
microk8s ctr images prune

# Pour un nettoyage plus agressif (supprimer les images non utilisées depuis X jours)
# Attention: peut supprimer des images nécessaires
if [ "$1" == "--aggressive" ]; then
  echo "Mode agressif: suppression des images non utilisées"
  for image in $ALL_IMAGES; do
    if ! echo "$USED_IMAGES" | grep -q "$image"; then
      echo "  Suppression de: $image"
      microk8s ctr images remove "$image" 2>/dev/null
    fi
  done
fi

# Calculer l'espace après nettoyage
AFTER=$(df -h / | tail -1 | awk '{print $4}')
echo "Espace disponible avant: $BEFORE"
echo "Espace disponible après: $AFTER"

echo "✅ Nettoyage des images terminé"
EOF

chmod +x clean-images.sh
```

### Gestion du cache de build

Si vous construisez des images localement :

```bash
# Nettoyer le cache de build
microk8s ctr content prune

# Script pour nettoyer complètement le cache
cat <<'EOF' > clean-build-cache.sh
#!/bin/bash
echo "=== Nettoyage du cache de build ==="

# Arrêter les conteneurs en cours si nécessaire
echo "Arrêt des conteneurs orphelins..."
microk8s ctr tasks list | grep RUNNING | awk '{print $1}' | \
  xargs -I {} microk8s ctr tasks kill {}

# Nettoyer le contenu non référencé
echo "Nettoyage du contenu non référencé..."
microk8s ctr content prune

# Nettoyer les snapshots
echo "Nettoyage des snapshots..."
microk8s ctr snapshots list | tail -n +2 | awk '{print $1}' | \
  xargs -I {} microk8s ctr snapshots remove {}

echo "✅ Cache de build nettoyé"
EOF

chmod +x clean-build-cache.sh
```

## Gestion des logs

### Rotation et nettoyage des logs système

Les logs peuvent rapidement remplir le disque :

```bash
# Vérifier l'espace utilisé par les logs
journalctl --disk-usage
du -sh /var/log/*

# Nettoyer les logs système
# Garder seulement les 7 derniers jours
sudo journalctl --vacuum-time=7d

# Garder seulement 500MB de logs
sudo journalctl --vacuum-size=500M

# Configuration permanente de la rotation
cat <<EOF | sudo tee /etc/systemd/journald.conf.d/50-microk8s.conf
[Journal]
SystemMaxUse=1G
SystemKeepFree=200M
SystemMaxFileSize=100M
MaxRetentionSec=7day
EOF

sudo systemctl restart systemd-journald
```

### Gestion des logs de conteneurs

```bash
# Script pour nettoyer les logs de conteneurs
cat <<'EOF' > clean-container-logs.sh
#!/bin/bash
echo "=== Nettoyage des logs de conteneurs ==="

# Répertoire des logs de containerd
LOG_DIR="/var/snap/microk8s/common/var/lib/containerd"

# Taille avant nettoyage
echo "Espace utilisé avant: $(du -sh $LOG_DIR 2>/dev/null | cut -f1)"

# Trouver et truncate les gros fichiers de log
find $LOG_DIR -name "*.log" -size +100M -exec truncate -s 0 {} \;

# Nettoyer les logs des pods supprimés
# Les logs des pods supprimés peuvent rester
for pod_dir in $(find /var/log/pods -type d -mtime +7); do
  echo "Suppression des vieux logs: $pod_dir"
  sudo rm -rf "$pod_dir"
done

# Taille après nettoyage
echo "Espace utilisé après: $(du -sh $LOG_DIR 2>/dev/null | cut -f1)"

echo "✅ Logs nettoyés"
EOF

chmod +x clean-container-logs.sh
```

### Configuration de logrotate

Créer une configuration logrotate pour MicroK8s :

```bash
# Créer la configuration logrotate
cat <<EOF | sudo tee /etc/logrotate.d/microk8s
/var/snap/microk8s/common/var/log/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    sharedscripts
    postrotate
        # Signaler à MicroK8s de rouvrir les fichiers de log
        systemctl reload snap.microk8s.daemon-kubelite 2>/dev/null || true
    endscript
}

/var/log/pods/*/*.log {
    daily
    rotate 3
    compress
    delaycompress
    missingok
    notifempty
    maxsize 100M
}
EOF

# Tester la configuration
sudo logrotate -d /etc/logrotate.d/microk8s
```

## Nettoyage des volumes

### Identification des volumes orphelins

```bash
# Lister tous les PersistentVolumes
microk8s kubectl get pv

# Lister les PV non liés (status: Released ou Available avec ancienne claim)
microk8s kubectl get pv | grep -E "Released|Available"

# Script pour identifier les volumes orphelins
cat <<'EOF' > find-orphan-volumes.sh
#!/bin/bash
echo "=== Recherche des volumes orphelins ==="

# PersistentVolumes sans PVC
echo "PersistentVolumes non réclamés:"
microk8s kubectl get pv -o json | \
  jq -r '.items[] | select(.status.phase == "Released" or (.status.phase == "Available" and .spec.claimRef != null)) | .metadata.name'

# Répertoires de volumes locaux non utilisés
VOLUME_PATH="/var/snap/microk8s/common/default-storage"
if [ -d "$VOLUME_PATH" ]; then
  echo -e "\nRépertoires de volumes locaux potentiellement orphelins:"

  # Obtenir les PV actifs utilisant le stockage local
  ACTIVE_VOLUMES=$(microk8s kubectl get pv -o json | \
    jq -r '.items[] | select(.spec.hostPath != null) | .spec.hostPath.path' | \
    xargs -I {} basename {})

  # Comparer avec les répertoires existants
  for dir in $(ls $VOLUME_PATH 2>/dev/null); do
    if ! echo "$ACTIVE_VOLUMES" | grep -q "$dir"; then
      size=$(du -sh "$VOLUME_PATH/$dir" 2>/dev/null | cut -f1)
      echo "  - $dir (taille: $size)"
    fi
  done
fi

echo -e "\n⚠️  Vérifiez manuellement avant de supprimer ces volumes"
EOF

chmod +x find-orphan-volumes.sh
```

### Nettoyage des volumes

```bash
# Script de nettoyage des volumes
cat <<'EOF' > clean-volumes.sh
#!/bin/bash
echo "=== Nettoyage des volumes ==="

# Supprimer les PV en état Released
echo "Suppression des PV Released..."
for pv in $(microk8s kubectl get pv | grep Released | awk '{print $1}'); do
  echo "  Suppression de $pv"
  microk8s kubectl delete pv $pv
done

# Nettoyer les PVC en état Terminating
echo "Nettoyage des PVC bloqués..."
for pvc in $(microk8s kubectl get pvc --all-namespaces | grep Terminating | awk '{print $1 ":" $2}'); do
  namespace=$(echo $pvc | cut -d: -f1)
  name=$(echo $pvc | cut -d: -f2)
  echo "  Forçage de la suppression de $namespace/$name"
  microk8s kubectl patch pvc $name -n $namespace -p '{"metadata":{"finalizers":null}}'
done

# Optionnel: nettoyer les répertoires de volumes locaux orphelins
if [ "$1" == "--clean-local" ]; then
  VOLUME_PATH="/var/snap/microk8s/common/default-storage"
  echo "Nettoyage des volumes locaux orphelins..."

  # Liste des volumes actifs
  ACTIVE=$(microk8s kubectl get pv -o json | \
    jq -r '.items[] | select(.spec.hostPath != null) | .spec.hostPath.path')

  for dir in $(find $VOLUME_PATH -maxdepth 1 -type d); do
    if [ "$dir" != "$VOLUME_PATH" ] && ! echo "$ACTIVE" | grep -q "$dir"; then
      echo "  Suppression de $dir"
      sudo rm -rf "$dir"
    fi
  done
fi

echo "✅ Nettoyage des volumes terminé"
EOF

chmod +x clean-volumes.sh
```

## Maintenance des composants système

### Maintenance de etcd

La base de données etcd peut avoir besoin de maintenance :

```bash
# Vérifier la santé d'etcd
microk8s kubectl get --raw /healthz/etcd

# Script de maintenance etcd
cat <<'EOF' > maintain-etcd.sh
#!/bin/bash
echo "=== Maintenance de etcd ==="

# Défragmentation d'etcd (récupère l'espace)
echo "Défragmentation d'etcd..."
ETCDCTL_API=3 microk8s.etcdctl \
  --endpoints=https://127.0.0.1:12379 \
  --cacert=/var/snap/microk8s/current/certs/ca.crt \
  --cert=/var/snap/microk8s/current/certs/server.crt \
  --key=/var/snap/microk8s/current/certs/server.key \
  defrag

# Vérifier l'espace utilisé
echo "Statistiques d'etcd:"
ETCDCTL_API=3 microk8s.etcdctl \
  --endpoints=https://127.0.0.1:12379 \
  --cacert=/var/snap/microk8s/current/certs/ca.crt \
  --cert=/var/snap/microk8s/current/certs/server.crt \
  --key=/var/snap/microk8s/current/certs/server.key \
  endpoint status --write-out=table

# Compaction (supprime les vieilles révisions)
LATEST_REV=$(ETCDCTL_API=3 microk8s.etcdctl \
  --endpoints=https://127.0.0.1:12379 \
  --cacert=/var/snap/microk8s/current/certs/ca.crt \
  --cert=/var/snap/microk8s/current/certs/server.crt \
  --key=/var/snap/microk8s/current/certs/server.key \
  endpoint status --write-out=json | jq -r '.[0].Status.header.revision')

if [ ! -z "$LATEST_REV" ]; then
  echo "Compaction jusqu'à la révision $LATEST_REV..."
  ETCDCTL_API=3 microk8s.etcdctl \
    --endpoints=https://127.0.0.1:12379 \
    --cacert=/var/snap/microk8s/current/certs/ca.crt \
    --cert=/var/snap/microk8s/current/certs/server.crt \
    --key=/var/snap/microk8s/current/certs/server.key \
    compact $LATEST_REV
fi

echo "✅ Maintenance etcd terminée"
EOF

chmod +x maintain-etcd.sh
```

### Optimisation des performances système

```bash
# Script d'optimisation système
cat <<'EOF' > optimize-system.sh
#!/bin/bash
echo "=== Optimisation du système pour MicroK8s ==="

# Nettoyer le cache système
echo "Nettoyage du cache système..."
sync
echo 3 | sudo tee /proc/sys/vm/drop_caches

# Optimiser les paramètres réseau
echo "Optimisation réseau..."
sudo sysctl -w net.core.netdev_max_backlog=5000
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sudo sysctl -w net.core.default_qdisc=fq

# Désactiver le swap si activé (recommandé pour Kubernetes)
if [ $(swapon -s | wc -l) -gt 1 ]; then
  echo "Désactivation du swap..."
  sudo swapoff -a
fi

# Nettoyer les paquets système inutilisés
echo "Nettoyage des paquets système..."
sudo apt-get autoremove -y
sudo apt-get autoclean -y

# Nettoyer le cache snap
echo "Nettoyage du cache snap..."
sudo sh -c 'rm -rf /var/lib/snapd/cache/*'

# Garder seulement les 2 dernières versions de chaque snap
LANG=C snap list --all | awk '/disabled/{print $1, $3}' | \
  while read snapname revision; do
    sudo snap remove "$snapname" --revision="$revision"
  done

echo "✅ Optimisation terminée"
EOF

chmod +x optimize-system.sh
```

## Scripts de maintenance automatisée

### Script de maintenance complète

Créons un script principal qui combine toutes les tâches :

```bash
cat <<'EOF' > microk8s-maintenance.sh
#!/bin/bash
#
# Script de maintenance complète pour MicroK8s
# Usage: ./microk8s-maintenance.sh [--full|--quick|--dry-run]
#

set -e

# Configuration
LOG_FILE="/var/log/microk8s-maintenance.log"
DRY_RUN=false
MODE="quick"

# Parsing des arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    --full)
      MODE="full"
      shift
      ;;
    --quick)
      MODE="quick"
      shift
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    *)
      echo "Usage: $0 [--full|--quick|--dry-run]"
      exit 1
      ;;
  esac
done

# Fonction de logging
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Fonction d'exécution (respecte dry-run)
execute() {
  if [ "$DRY_RUN" == "true" ]; then
    echo "[DRY-RUN] $@"
  else
    eval "$@"
  fi
}

# Début de la maintenance
log "=== Début de la maintenance MicroK8s (mode: $MODE, dry-run: $DRY_RUN) ==="

# 1. Collecte des métriques avant
log "Collecte des métriques initiales..."
DISK_BEFORE=$(df -h / | tail -1 | awk '{print $4}')
PODS_BEFORE=$(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
IMAGES_BEFORE=$(microk8s ctr images list -q | wc -l)

# 2. Nettoyage des pods
log "Nettoyage des pods terminés..."
execute "microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded"
execute "microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed"
EVICTED=$(microk8s kubectl get pods --all-namespaces | grep Evicted | awk '{print $2 " -n " $1}')
if [ ! -z "$EVICTED" ]; then
  echo "$EVICTED" | while read pod; do
    execute "microk8s kubectl delete pod $pod"
  done
fi

# 3. Nettoyage des images (mode quick)
log "Nettoyage des images Docker..."
execute "microk8s ctr images prune"

if [ "$MODE" == "full" ]; then
  # 4. Nettoyage approfondi des images
  log "Nettoyage approfondi des images..."
  USED_IMAGES=$(microk8s kubectl get pods --all-namespaces -o json | \
    jq -r '.items[].spec.containers[].image' | sort -u)

  for image in $(microk8s ctr images list -q); do
    if ! echo "$USED_IMAGES" | grep -q "$image"; then
      execute "microk8s ctr images remove $image"
    fi
  done

  # 5. Maintenance etcd
  log "Maintenance de etcd..."
  if [ "$DRY_RUN" == "false" ]; then
    ETCDCTL_API=3 microk8s.etcdctl \
      --endpoints=https://127.0.0.1:12379 \
      --cacert=/var/snap/microk8s/current/certs/ca.crt \
      --cert=/var/snap/microk8s/current/certs/server.crt \
      --key=/var/snap/microk8s/current/certs/server.key \
      defrag
  fi

  # 6. Nettoyage des volumes orphelins
  log "Nettoyage des volumes..."
  for pv in $(microk8s kubectl get pv | grep Released | awk '{print $1}'); do
    execute "microk8s kubectl delete pv $pv"
  done
fi

# 7. Nettoyage des logs
log "Nettoyage des logs..."
execute "sudo journalctl --vacuum-time=7d"
execute "sudo journalctl --vacuum-size=500M"

# 8. Nettoyage du cache système
if [ "$MODE" == "full" ]; then
  log "Nettoyage du cache système..."
  execute "sync"
  execute "echo 3 | sudo tee /proc/sys/vm/drop_caches"
  execute "sudo apt-get autoremove -y"
  execute "sudo apt-get autoclean -y"
fi

# 9. Collecte des métriques après
log "Collecte des métriques finales..."
DISK_AFTER=$(df -h / | tail -1 | awk '{print $4}')
PODS_AFTER=$(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
IMAGES_AFTER=$(microk8s ctr images list -q | wc -l)

# 10. Rapport
log "=== Rapport de maintenance ==="
log "Espace disque: $DISK_BEFORE → $DISK_AFTER"
log "Nombre de pods: $PODS_BEFORE → $PODS_AFTER"
log "Nombre d'images: $IMAGES_BEFORE → $IMAGES_AFTER"
log "=== Maintenance terminée avec succès ==="

# Envoi de notification (optionnel)
# echo "Maintenance MicroK8s terminée" | mail -s "MicroK8s Maintenance Report" admin@example.com
EOF

chmod +x microk8s-maintenance.sh
```

### Automatisation avec cron

Mettre en place une maintenance régulière :

```bash
# Créer les tâches cron pour la maintenance
cat <<EOF | sudo tee /etc/cron.d/microk8s-maintenance
# Maintenance quotidienne légère à 2h du matin
0 2 * * * root /usr/local/bin/microk8s-maintenance.sh --quick >> /var/log/microk8s-maintenance.log 2>&1

# Maintenance complète hebdomadaire le dimanche à 3h
0 3 * * 0 root /usr/local/bin/microk8s-maintenance.sh --full >> /var/log/microk8s-maintenance.log 2>&1

# Nettoyage des logs tous les jours à minuit
0 0 * * * root journalctl --vacuum-time=7d --vacuum-size=500M

# Vérification de l'espace disque toutes les heures
0 * * * * root df -h / | awk '\$5+0 > 80 {print "Warning: Disk usage is " \$5}' | logger -t microk8s-disk
EOF

# Copier le script de maintenance dans /usr/local/bin
sudo cp microk8s-maintenance.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/microk8s-maintenance.sh

# Vérifier que les tâches sont bien créées
sudo crontab -l | grep microk8s
```

### Script de monitoring et alerting

```bash
cat <<'EOF' > monitor-health.sh
#!/bin/bash
#
# Script de monitoring de santé pour MicroK8s
#

# Configuration
ALERT_DISK_THRESHOLD=80  # Pourcentage
ALERT_MEMORY_THRESHOLD=90  # Pourcentage
ALERT_CPU_THRESHOLD=80  # Pourcentage
LOG_FILE="/var/log/microk8s-health.log"

# Fonction d'alerte
send_alert() {
  local severity=$1
  local message=$2

  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$severity] $message" >> "$LOG_FILE"

  # Notifications (à personnaliser)
  # echo "$message" | mail -s "MicroK8s Alert: $severity" admin@example.com
  # curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK \
  #   -H 'Content-Type: application/json' \
  #   -d "{\"text\":\"MicroK8s Alert ($severity): $message\"}"

  logger -t microk8s-health -p user.$severity "$message"
}

# 1. Vérifier l'espace disque
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
if [ $DISK_USAGE -gt $ALERT_DISK_THRESHOLD ]; then
  send_alert "warning" "Espace disque critique: ${DISK_USAGE}% utilisé"

  # Tentative de nettoyage automatique
  /usr/local/bin/microk8s-maintenance.sh --quick
fi

# 2. Vérifier la mémoire
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
if [ $MEMORY_USAGE -gt $ALERT_MEMORY_THRESHOLD ]; then
  send_alert "warning" "Utilisation mémoire élevée: ${MEMORY_USAGE}%"
fi

# 3. Vérifier l'état du cluster
if ! microk8s status --wait-ready --timeout 30 >/dev/null 2>&1; then
  send_alert "critical" "Cluster MicroK8s non disponible"
fi

# 4. Vérifier les pods en erreur
FAILED_PODS=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Failed --no-headers 2>/dev/null | wc -l)
if [ $FAILED_PODS -gt 5 ]; then
  send_alert "warning" "$FAILED_PODS pods en état Failed"
fi

# 5. Vérifier les pods CrashLoopBackOff
CRASH_PODS=$(microk8s kubectl get pods --all-namespaces | grep -c CrashLoopBackOff)
if [ $CRASH_PODS -gt 0 ]; then
  send_alert "warning" "$CRASH_PODS pods en CrashLoopBackOff"
fi

# 6. Vérifier l'âge des images
OLD_IMAGES=$(microk8s ctr images list --quiet | xargs -I {} microk8s ctr images info {} 2>/dev/null | \
  grep -c "created.*[0-9]\{4\}-[0-9]\{2\}" | wc -l)
if [ $OLD_IMAGES -gt 50 ]; then
  send_alert "info" "$OLD_IMAGES images Docker présentes, considérez un nettoyage"
fi

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Health check completed" >> "$LOG_FILE"
EOF

chmod +x monitor-health.sh

# Ajouter au cron pour exécution toutes les 30 minutes
echo "*/30 * * * * root /usr/local/bin/monitor-health.sh" | sudo tee -a /etc/cron.d/microk8s-maintenance
```

## Outils de maintenance avancés

### Dashboard de maintenance

Créer un dashboard interactif pour la maintenance :

```bash
cat <<'EOF' > maintenance-dashboard.sh
#!/bin/bash
#
# Dashboard interactif de maintenance MicroK8s
#

# Couleurs pour l'affichage
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Fonction pour afficher le menu
show_menu() {
  clear
  echo "╔════════════════════════════════════════════╗"
  echo "║     MicroK8s Maintenance Dashboard          ║"
  echo "╚════════════════════════════════════════════╝"
  echo ""
  echo "État actuel:"
  echo "─────────────"

  # Espace disque
  DISK_USAGE=$(df / | tail -1 | awk '{print $5}')
  echo -n "📁 Espace disque: "
  if [ ${DISK_USAGE%\%} -gt 80 ]; then
    echo -e "${RED}$DISK_USAGE utilisé${NC}"
  else
    echo -e "${GREEN}$DISK_USAGE utilisé${NC}"
  fi

  # Pods
  TOTAL_PODS=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | wc -l)
  RUNNING_PODS=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
  echo "🔷 Pods: $RUNNING_PODS/$TOTAL_PODS running"

  # Images
  IMAGES=$(microk8s ctr images list -q 2>/dev/null | wc -l)
  echo "🖼️  Images Docker: $IMAGES"

  # Logs
  LOG_SIZE=$(journalctl --disk-usage 2>/dev/null | grep -oP '\d+\.?\d*[MG]')
  echo "📝 Logs système: $LOG_SIZE"

  echo ""
  echo "Actions disponibles:"
  echo "────────────────────"
  echo "1) Nettoyage rapide (pods et images basique)"
  echo "2) Nettoyage complet (inclut volumes et système)"
  echo "3) Analyser l'utilisation de l'espace"
  echo "4) Voir les pods problématiques"
  echo "5) Nettoyer les logs"
  echo "6) Optimiser etcd"
  echo "7) Générer un rapport de santé"
  echo "8) Planifier une maintenance automatique"
  echo "9) Afficher les statistiques détaillées"
  echo "0) Quitter"
  echo ""
  echo -n "Votre choix: "
}

# Fonction pour l'analyse de l'espace
analyze_space() {
  echo "Analyse de l'utilisation de l'espace..."
  echo ""

  echo "Top 10 des répertoires:"
  du -h /var/snap/microk8s/ 2>/dev/null | sort -rh | head -10

  echo -e "\nEspace par type:"
  echo "- Images: $(microk8s ctr images list | awk '{sum+=$7} END {printf "%.2f GB", sum/1024/1024/1024}')"
  echo "- Logs: $(journalctl --disk-usage | grep -oP '\d+\.?\d*[MG]')"
  echo "- Volumes: $(du -sh /var/snap/microk8s/common/default-storage 2>/dev/null | cut -f1)"

  read -p "Appuyez sur Entrée pour continuer..."
}

# Fonction pour les statistiques détaillées
show_stats() {
  echo "Statistiques détaillées du cluster:"
  echo "──────────────────────────────────"

  echo -e "\n📊 Ressources par namespace:"
  for ns in $(microk8s kubectl get ns -o name | cut -d/ -f2); do
    pods=$(microk8s kubectl get pods -n $ns --no-headers 2>/dev/null | wc -l)
    if [ $pods -gt 0 ]; then
      echo "  $ns: $pods pods"
    fi
  done

  echo -e "\n💾 Top 10 images par taille:"
  microk8s ctr images list | sort -k7 -rn | head -11 | tail -10 | awk '{printf "  %.2f MB\t%s\n", $7/1024/1024, $1}'

  echo -e "\n⚠️  Pods avec redémarrages:"
  microk8s kubectl get pods --all-namespaces | awk '$5>0 {print "  " $1 "/" $2 ": " $5 " restarts"}'

  read -p "Appuyez sur Entrée pour continuer..."
}

# Fonction pour générer un rapport
generate_report() {
  REPORT_FILE="microk8s-report-$(date +%Y%m%d-%H%M%S).txt"
  echo "Génération du rapport dans $REPORT_FILE..."

  {
    echo "MicroK8s Health Report"
    echo "Generated: $(date)"
    echo "========================"
    echo ""
    echo "System Information:"
    echo "-------------------"
    uname -a
    echo ""
    echo "MicroK8s Version:"
    microk8s version
    echo ""
    echo "Cluster Status:"
    microk8s status
    echo ""
    echo "Node Status:"
    microk8s kubectl describe node
    echo ""
    echo "Pod Summary:"
    microk8s kubectl get pods --all-namespaces
    echo ""
    echo "Resource Usage:"
    microk8s kubectl top nodes 2>/dev/null || echo "Metrics not available"
    echo ""
    echo "Disk Usage:"
    df -h
    echo ""
    echo "Recent Events:"
    microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp' | head -50
  } > "$REPORT_FILE"

  echo "Rapport généré: $REPORT_FILE"
  read -p "Appuyez sur Entrée pour continuer..."
}

# Boucle principale
while true; do
  show_menu
  read choice

  case $choice in
    1)
      echo "Exécution du nettoyage rapide..."
      /usr/local/bin/microk8s-maintenance.sh --quick
      read -p "Appuyez sur Entrée pour continuer..."
      ;;
    2)
      echo "Exécution du nettoyage complet..."
      /usr/local/bin/microk8s-maintenance.sh --full
      read -p "Appuyez sur Entrée pour continuer..."
      ;;
    3)
      analyze_space
      ;;
    4)
      echo "Pods problématiques:"
      microk8s kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
      read -p "Appuyez sur Entrée pour continuer..."
      ;;
    5)
      echo "Nettoyage des logs..."
      sudo journalctl --vacuum-time=7d
      sudo journalctl --vacuum-size=500M
      read -p "Appuyez sur Entrée pour continuer..."
      ;;
    6)
      echo "Optimisation d'etcd..."
      ./maintain-etcd.sh
      read -p "Appuyez sur Entrée pour continuer..."
      ;;
    7)
      generate_report
      ;;
    8)
      echo "Configuration de la maintenance automatique..."
      sudo cp microk8s-maintenance.sh /usr/local/bin/
      echo "Tâches cron actuelles:"
      sudo crontab -l | grep microk8s || echo "Aucune tâche configurée"
      read -p "Appuyez sur Entrée pour continuer..."
      ;;
    9)
      show_stats
      ;;
    0)
      echo "Au revoir!"
      exit 0
      ;;
    *)
      echo "Option invalide"
      sleep 2
      ;;
  esac
done
EOF

chmod +x maintenance-dashboard.sh
```

## Stratégies de maintenance par environnement

### Maintenance minimale (développement)

Pour un environnement de développement avec ressources limitées :

```bash
# Configuration minimale de maintenance
cat <<'EOF' > maintenance-dev.sh
#!/bin/bash
# Maintenance minimale pour environnement de développement

# Garder seulement les 3 derniers jours de logs
journalctl --vacuum-time=3d

# Nettoyer agressivement les images
microk8s ctr images prune

# Supprimer tous les pods non-running
microk8s kubectl delete pods --all-namespaces --field-selector='status.phase!=Running'

# Nettoyer le cache
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

echo "Maintenance dev terminée"
EOF
```

### Maintenance équilibrée (lab personnel)

Pour un lab personnel avec services importants :

```bash
# Configuration équilibrée
cat <<'EOF' > maintenance-lab.sh
#!/bin/bash
# Maintenance équilibrée pour lab personnel

# Sauvegarder avant maintenance
BACKUP_DIR="$HOME/backups/microk8s-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"
microk8s kubectl get all --all-namespaces -o yaml > "$BACKUP_DIR/resources.yaml"

# Maintenance avec précaution
./microk8s-maintenance.sh --quick

# Vérifier que tout fonctionne
if ! microk8s kubectl get pods --all-namespaces | grep -q Running; then
  echo "⚠️  Attention: Aucun pod running après maintenance"
fi

# Garder 7 jours de logs
journalctl --vacuum-time=7d

echo "Maintenance lab terminée"
EOF
```

## Checklist de maintenance

### Maintenance quotidienne
- [ ] Vérifier l'espace disque disponible
- [ ] Nettoyer les pods terminés
- [ ] Vérifier les pods en erreur
- [ ] Rotation des logs si nécessaire

### Maintenance hebdomadaire
- [ ] Nettoyage complet des images Docker
- [ ] Vérification des volumes orphelins
- [ ] Analyse de l'utilisation des ressources
- [ ] Mise à jour de la documentation

### Maintenance mensuelle
- [ ] Défragmentation d'etcd
- [ ] Revue des configurations inutilisées
- [ ] Test de restauration depuis backup
- [ ] Mise à jour de MicroK8s si disponible
- [ ] Optimisation des paramètres système

## Conclusion

La maintenance régulière de votre cluster MicroK8s est essentielle pour maintenir des performances optimales et éviter les problèmes d'espace disque. Les scripts et procédures présentés dans cette section vous permettent d'automatiser la majorité des tâches de maintenance, depuis le simple nettoyage des pods jusqu'à l'optimisation complète du système.

L'important est d'établir une routine adaptée à votre utilisation : maintenance légère quotidienne pour un environnement de développement actif, ou maintenance hebdomadaire plus approfondie pour un lab stable. Les outils de monitoring vous alerteront des problèmes avant qu'ils ne deviennent critiques, et les scripts de nettoyage automatisés vous feront gagner du temps tout en gardant votre cluster en excellente santé.

N'oubliez pas que la meilleure maintenance est préventive : surveillez régulièrement les métriques, automatisez les tâches répétitives, et documentez vos procédures. Avec ces pratiques en place, votre cluster MicroK8s restera performant et fiable pour tous vos projets.

⏭️
