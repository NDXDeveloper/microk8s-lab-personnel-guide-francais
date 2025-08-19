🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3 Snapshots de volumes

## Introduction aux snapshots de volumes

Un snapshot de volume est comme une photographie instantanée de vos données à un moment précis. Imaginez que vous travaillez sur un document important : un snapshot serait l'équivalent de faire "Enregistrer sous" avec un nouveau nom pour garder une version de sauvegarde, sauf que cela se fait automatiquement et instantanément pour tout le volume de stockage.

### Différence entre backup et snapshot

**Backup traditionnel :**
- Copie complète des données vers un autre emplacement
- Prend du temps proportionnel à la taille des données
- Consomme de l'espace équivalent aux données
- Peut être stocké hors site
- Processus souvent lourd

**Snapshot :**
- Capture instantanée de l'état du volume
- Création quasi-immédiate (quelques secondes)
- Utilise peu d'espace initial (copy-on-write)
- Généralement stocké sur le même système
- Idéal pour les rollbacks rapides

### Pourquoi les snapshots dans MicroK8s ?

**Cas d'usage typiques pour un lab :**
- **Avant une mise à jour risquée** : Snapshot rapide avant de tester une nouvelle version
- **Tests destructifs** : Expérimentations qui peuvent corrompre les données
- **Points de restauration** : Créer des checkpoints durant le développement
- **Clonage rapide** : Dupliquer un environnement pour des tests parallèles
- **Protection contre les erreurs** : Récupération rapide après une mauvaise manipulation

## Technologies de snapshot disponibles

### Vue d'ensemble des options pour MicroK8s

```
┌─────────────────────────────────────────────────────────┐
│                     MicroK8s Storage                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   HostPath   │  │   OpenEBS    │  │     LVM      │ │
│  │   (Défaut)   │  │  (CSI addon) │  │  (External)  │ │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤ │
│  │ ✗ Pas de     │  │ ✓ Snapshots  │  │ ✓ Snapshots  │ │
│  │   snapshot   │  │   natifs     │  │   LVM natifs │ │
│  │ ✓ Simple     │  │ ✓ Clones     │  │ ✓ Très       │ │
│  │ ✓ Rapide     │  │ ✓ Intégré K8s│  │   rapide     │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │     ZFS      │  │    Btrfs     │  │   Ceph/Rook  │ │
│  │  (Advanced)  │  │  (Advanced)  │  │  (Complexe)  │ │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤ │
│  │ ✓ Snapshots  │  │ ✓ Snapshots  │  │ ✓ Snapshots  │ │
│  │ ✓ Compression│  │ ✓ CoW natif  │  │ ✓ Distribué  │ │
│  │ ✓ Dédup      │  │ ✓ Simple     │  │ ✗ Complexe   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Comparaison pour un lab personnel

| Technologie | Complexité | Performance | Fonctionnalités | Recommandé pour |
|------------|------------|-------------|-----------------|-----------------|
| HostPath + Scripts | Faible | Moyenne | Basique | Débutants, tests simples |
| OpenEBS | Moyenne | Bonne | Complète | Usage général, CSI standard |
| LVM | Moyenne | Excellente | Snapshots natifs | Linux expérimentés |
| ZFS | Élevée | Excellente | Très complète | Utilisateurs avancés |
| Btrfs | Moyenne | Très bonne | Bonne | Systèmes modernes |

## Solution 1 : Snapshots manuels avec HostPath

MicroK8s utilise HostPath par défaut. Bien qu'il ne supporte pas les snapshots natifs, nous pouvons créer notre propre système.

### Localisation des données

```bash
# Les volumes HostPath sont stockés dans :
/var/snap/microk8s/common/default-storage/

# Lister les PersistentVolumes
microk8s kubectl get pv

# Voir où un PV spécifique est stocké
microk8s kubectl get pv <pv-name> -o jsonpath='{.spec.hostPath.path}'
```

### Script de snapshot manuel

```bash
#!/bin/bash
# snapshot-hostpath.sh

STORAGE_PATH="/var/snap/microk8s/common/default-storage"
SNAPSHOT_PATH="/backup/snapshots"
DATE=$(date +%Y%m%d-%H%M%S)

# Fonction pour créer un snapshot
create_snapshot() {
    local PVC_NAME=$1
    local NAMESPACE=$2

    echo "🔍 Recherche du PV pour PVC: $PVC_NAME dans namespace: $NAMESPACE"

    # Obtenir le nom du PV
    PV_NAME=$(microk8s kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.volumeName}')

    if [ -z "$PV_NAME" ]; then
        echo "❌ PVC non trouvé"
        return 1
    fi

    # Obtenir le chemin du volume
    PV_PATH=$(microk8s kubectl get pv $PV_NAME -o jsonpath='{.spec.hostPath.path}')

    echo "📁 Chemin du volume: $PV_PATH"

    # Créer le répertoire de snapshot
    SNAP_DIR="$SNAPSHOT_PATH/${NAMESPACE}-${PVC_NAME}/${DATE}"
    mkdir -p "$SNAP_DIR"

    # Créer le snapshot avec rsync (plus sûr que cp)
    echo "📸 Création du snapshot..."
    rsync -avz --delete "$PV_PATH/" "$SNAP_DIR/"

    # Créer les métadonnées
    cat > "$SNAP_DIR.metadata" <<EOF
{
    "timestamp": "$(date -Iseconds)",
    "namespace": "$NAMESPACE",
    "pvc": "$PVC_NAME",
    "pv": "$PV_NAME",
    "source": "$PV_PATH",
    "size": "$(du -sh $PV_PATH | cut -f1)"
}
EOF

    echo "✅ Snapshot créé: $SNAP_DIR"
}

# Fonction pour lister les snapshots
list_snapshots() {
    echo "📋 Snapshots disponibles:"
    find $SNAPSHOT_PATH -name "*.metadata" -type f | while read meta; do
        echo "---"
        cat "$meta" | python3 -m json.tool
    done
}

# Fonction pour restaurer un snapshot
restore_snapshot() {
    local SNAPSHOT_DIR=$1

    if [ ! -d "$SNAPSHOT_DIR" ]; then
        echo "❌ Snapshot non trouvé: $SNAPSHOT_DIR"
        return 1
    fi

    # Lire les métadonnées
    META_FILE="${SNAPSHOT_DIR}.metadata"
    if [ ! -f "$META_FILE" ]; then
        META_FILE="${SNAPSHOT_DIR%/}.metadata"
    fi

    PV_PATH=$(cat "$META_FILE" | python3 -c "import sys, json; print(json.load(sys.stdin)['source'])")

    echo "⚠️  ATTENTION: Restauration vers $PV_PATH"
    echo "Cela écrasera les données actuelles. Continuer? (yes/no)"
    read -r confirmation

    if [ "$confirmation" != "yes" ]; then
        echo "❌ Restauration annulée"
        return 1
    fi

    # Sauvegarder l'état actuel avant restauration
    BACKUP_CURRENT="${PV_PATH}.backup-$(date +%Y%m%d-%H%M%S)"
    echo "💾 Sauvegarde de l'état actuel vers: $BACKUP_CURRENT"
    mv "$PV_PATH" "$BACKUP_CURRENT"

    # Restaurer le snapshot
    echo "♻️  Restauration en cours..."
    rsync -avz --delete "$SNAPSHOT_DIR/" "$PV_PATH/"

    echo "✅ Restauration terminée"
}

# Menu principal
case "$1" in
    create)
        create_snapshot "$2" "$3"
        ;;
    list)
        list_snapshots
        ;;
    restore)
        restore_snapshot "$2"
        ;;
    *)
        echo "Usage: $0 {create|list|restore} [arguments]"
        echo "  create <pvc-name> <namespace> - Créer un snapshot"
        echo "  list                          - Lister les snapshots"
        echo "  restore <snapshot-path>       - Restaurer un snapshot"
        ;;
esac
```

### Automatisation avec CronJob Kubernetes

```yaml
# snapshot-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: volume-snapshot
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Toutes les 6 heures
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa
          containers:
          - name: snapshot
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Script de snapshot inline
              NAMESPACES="default production development"
              for ns in $NAMESPACES; do
                echo "Processing namespace: $ns"
                # Obtenir tous les PVCs
                kubectl get pvc -n $ns -o jsonpath='{.items[*].metadata.name}' | \
                while read pvc; do
                  echo "Snapshot PVC: $pvc"
                  # Appeler votre script de snapshot
                  /scripts/snapshot-hostpath.sh create $pvc $ns
                done
              done
            volumeMounts:
            - name: scripts
              mountPath: /scripts
            - name: storage
              mountPath: /var/snap/microk8s/common/default-storage
            - name: snapshots
              mountPath: /backup/snapshots
          volumes:
          - name: scripts
            configMap:
              name: snapshot-scripts
              defaultMode: 0755
          - name: storage
            hostPath:
              path: /var/snap/microk8s/common/default-storage
          - name: snapshots
            hostPath:
              path: /backup/snapshots
          restartPolicy: OnFailure
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: snapshot-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims", "persistentvolumes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: snapshot-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: snapshot-role
subjects:
- kind: ServiceAccount
  name: snapshot-sa
  namespace: kube-system
```

## Solution 2 : OpenEBS avec snapshots CSI

OpenEBS est une solution de stockage cloud-native qui s'intègre parfaitement avec MicroK8s.

### Installation d'OpenEBS

```bash
# Activer l'addon OpenEBS dans MicroK8s
microk8s enable openebs

# Attendre que tous les pods soient prêts
microk8s kubectl wait --for=condition=ready pods -n openebs --all --timeout=300s

# Vérifier l'installation
microk8s kubectl get pods -n openebs
microk8s kubectl get sc  # StorageClasses
```

### Configuration de la StorageClass avec snapshots

```yaml
# openebs-snapshot-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-snapshot-enabled
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
    cas.openebs.io/config: |
      - name: StorageType
        value: "hostpath"
      - name: SnapshottableVolume
        value: "true"
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# VolumeSnapshotClass pour OpenEBS
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: openebs-snapshot-class
driver: openebs.io/local
deletionPolicy: Delete
```

### Créer un PVC avec support snapshot

```yaml
# pvc-with-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-snapshot-enabled
  resources:
    requests:
      storage: 5Gi
---
# Application utilisant le PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-pvc
```

### Créer un snapshot avec CSI

```yaml
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot-$(date +%Y%m%d-%H%M%S)
  namespace: default
spec:
  volumeSnapshotClassName: openebs-snapshot-class
  source:
    persistentVolumeClaimName: data-pvc
```

```bash
# Créer le snapshot via kubectl
microk8s kubectl apply -f volume-snapshot.yaml

# Vérifier le status du snapshot
microk8s kubectl get volumesnapshot
microk8s kubectl describe volumesnapshot data-snapshot-20240315-120000

# Lister tous les snapshots
microk8s kubectl get volumesnapshot -A
```

### Restaurer depuis un snapshot CSI

```yaml
# restore-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-snapshot-enabled
  resources:
    requests:
      storage: 5Gi
  dataSource:
    name: data-snapshot-20240315-120000
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

## Solution 3 : LVM Snapshots

Pour les utilisateurs Linux expérimentés, LVM offre des snapshots très performants.

### Prérequis LVM

```bash
# Vérifier si LVM est installé
sudo lvs
sudo vgs
sudo pvs

# Installer LVM si nécessaire
sudo apt-get update
sudo apt-get install lvm2

# Créer un volume group pour MicroK8s
# (Supposons que /dev/sdb est disponible)
sudo pvcreate /dev/sdb
sudo vgcreate microk8s-vg /dev/sdb

# Créer un logical volume
sudo lvcreate -L 20G -n microk8s-data microk8s-vg

# Formater et monter
sudo mkfs.ext4 /dev/microk8s-vg/microk8s-data
sudo mkdir -p /var/snap/microk8s/common/default-storage
sudo mount /dev/microk8s-vg/microk8s-data /var/snap/microk8s/common/default-storage

# Ajouter au fstab pour montage permanent
echo "/dev/microk8s-vg/microk8s-data /var/snap/microk8s/common/default-storage ext4 defaults 0 0" | sudo tee -a /etc/fstab
```

### Créer des snapshots LVM

```bash
#!/bin/bash
# lvm-snapshot.sh

VG_NAME="microk8s-vg"
LV_NAME="microk8s-data"
SNAP_SIZE="5G"  # Taille du snapshot (copy-on-write)

create_lvm_snapshot() {
    local SNAP_NAME="snap-$(date +%Y%m%d-%H%M%S)"

    echo "📸 Création du snapshot LVM: $SNAP_NAME"

    # Créer le snapshot
    sudo lvcreate -L $SNAP_SIZE -s -n $SNAP_NAME /dev/$VG_NAME/$LV_NAME

    if [ $? -eq 0 ]; then
        echo "✅ Snapshot créé avec succès"

        # Afficher les informations
        sudo lvs $VG_NAME/$SNAP_NAME

        # Sauvegarder les métadonnées
        echo "{
            \"name\": \"$SNAP_NAME\",
            \"timestamp\": \"$(date -Iseconds)\",
            \"source\": \"/dev/$VG_NAME/$LV_NAME\",
            \"size\": \"$SNAP_SIZE\"
        }" > /backup/lvm-snapshots/$SNAP_NAME.json
    else
        echo "❌ Échec de création du snapshot"
        return 1
    fi
}

list_lvm_snapshots() {
    echo "📋 Snapshots LVM disponibles:"
    sudo lvs --noheadings -o lv_name,lv_size,lv_time,origin $VG_NAME | grep "microk8s-data"
}

restore_lvm_snapshot() {
    local SNAP_NAME=$1

    echo "⚠️  ATTENTION: Restauration depuis $SNAP_NAME"
    echo "Cette opération nécessite l'arrêt des pods"
    echo "Continuer? (yes/no)"
    read -r confirmation

    if [ "$confirmation" != "yes" ]; then
        echo "❌ Restauration annulée"
        return 1
    fi

    # Arrêter les pods utilisant le stockage
    echo "🛑 Arrêt des applications..."
    microk8s kubectl scale deployment --all --replicas=0 --all-namespaces

    # Démonter le volume actuel
    echo "💾 Démontage du volume..."
    sudo umount /var/snap/microk8s/common/default-storage

    # Merger le snapshot (restauration)
    echo "♻️  Restauration en cours..."
    sudo lvconvert --merge /dev/$VG_NAME/$SNAP_NAME

    # Remonter le volume
    echo "🔄 Remontage du volume..."
    sudo mount /dev/$VG_NAME/$LV_NAME /var/snap/microk8s/common/default-storage

    # Redémarrer les applications
    echo "▶️  Redémarrage des applications..."
    microk8s kubectl scale deployment --all --replicas=1 --all-namespaces

    echo "✅ Restauration terminée"
}

# Monitoring des snapshots LVM
monitor_lvm_snapshots() {
    echo "📊 État des snapshots LVM:"
    sudo lvs -o lv_name,lv_size,data_percent,snap_percent $VG_NAME

    # Alerter si un snapshot est plein à plus de 80%
    sudo lvs --noheadings -o snap_percent $VG_NAME | while read percent; do
        if [ "${percent%.*}" -gt 80 ] 2>/dev/null; then
            echo "⚠️  ALERTE: Snapshot utilise plus de 80% de son espace!"
        fi
    done
}
```

### Automatisation avec systemd

```ini
# /etc/systemd/system/microk8s-lvm-snapshot.service
[Unit]
Description=MicroK8s LVM Snapshot Service
After=snap.microk8s.daemon-kubelite.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/lvm-snapshot.sh create
User=root

# /etc/systemd/system/microk8s-lvm-snapshot.timer
[Unit]
Description=MicroK8s LVM Snapshot Timer
Requires=microk8s-lvm-snapshot.service

[Timer]
OnCalendar=daily
OnCalendar=02:00
Persistent=true

[Install]
WantedBy=timers.target
```

## Solution 4 : ZFS Snapshots

ZFS offre des fonctionnalités avancées parfaites pour un lab.

### Installation et configuration ZFS

```bash
# Installer ZFS
sudo apt-get update
sudo apt-get install zfsutils-linux

# Créer un pool ZFS (exemple avec fichier pour test)
sudo truncate -s 50G /var/lib/zfs-disk.img
sudo zpool create microk8s-pool /var/lib/zfs-disk.img

# Créer un dataset pour MicroK8s
sudo zfs create microk8s-pool/volumes
sudo zfs set mountpoint=/var/snap/microk8s/common/default-storage microk8s-pool/volumes

# Configurer les propriétés ZFS
sudo zfs set compression=lz4 microk8s-pool/volumes
sudo zfs set atime=off microk8s-pool/volumes
sudo zfs set dedup=off microk8s-pool/volumes  # Économise la RAM
```

### Gestion des snapshots ZFS

```bash
#!/bin/bash
# zfs-snapshot.sh

POOL="microk8s-pool"
DATASET="microk8s-pool/volumes"

# Créer un snapshot
create_zfs_snapshot() {
    local SNAP_NAME="@$(date +%Y%m%d-%H%M%S)"

    echo "📸 Création du snapshot ZFS: $DATASET$SNAP_NAME"
    sudo zfs snapshot $DATASET$SNAP_NAME

    # Afficher les informations
    sudo zfs list -t snapshot -o name,used,refer,creation $DATASET$SNAP_NAME
}

# Snapshot récursif (tous les sous-datasets)
create_recursive_snapshot() {
    local SNAP_NAME="@backup-$(date +%Y%m%d-%H%M%S)"

    echo "📸 Snapshot récursif de $POOL"
    sudo zfs snapshot -r $POOL$SNAP_NAME
}

# Lister les snapshots
list_zfs_snapshots() {
    echo "📋 Snapshots ZFS disponibles:"
    sudo zfs list -t snapshot -o name,used,refer,creation | grep $DATASET
}

# Rollback vers un snapshot
rollback_zfs_snapshot() {
    local SNAP_NAME=$1

    echo "⚠️  Rollback vers: $SNAP_NAME"
    echo "Cela supprimera tous les snapshots plus récents!"
    echo "Continuer? (yes/no)"
    read -r confirmation

    if [ "$confirmation" != "yes" ]; then
        echo "❌ Rollback annulé"
        return 1
    fi

    # Effectuer le rollback
    sudo zfs rollback -r $SNAP_NAME
    echo "✅ Rollback effectué"
}

# Cloner un snapshot (création d'un nouveau volume)
clone_zfs_snapshot() {
    local SNAP_NAME=$1
    local CLONE_NAME=$2

    echo "🔄 Clonage de $SNAP_NAME vers $CLONE_NAME"
    sudo zfs clone $SNAP_NAME $POOL/$CLONE_NAME

    # Monter le clone
    sudo zfs set mountpoint=/mnt/$CLONE_NAME $POOL/$CLONE_NAME
    echo "✅ Clone créé et monté sur /mnt/$CLONE_NAME"
}

# Envoyer un snapshot vers un autre système (backup)
send_zfs_snapshot() {
    local SNAP_NAME=$1
    local DEST_FILE=$2

    echo "📤 Envoi du snapshot vers $DEST_FILE"
    sudo zfs send $SNAP_NAME | gzip > $DEST_FILE
    echo "✅ Snapshot exporté ($(du -h $DEST_FILE | cut -f1))"
}

# Politique de rétention automatique
apply_retention_policy() {
    # Garder : 24 hourly, 7 daily, 4 weekly, 12 monthly
    echo "🗑️  Application de la politique de rétention..."

    # Utiliser zfs-auto-snapshot si installé
    if command -v zfs-auto-snapshot &> /dev/null; then
        sudo zfs-auto-snapshot --quiet --syslog --label=hourly --keep=24 $DATASET
        sudo zfs-auto-snapshot --quiet --syslog --label=daily --keep=7 $DATASET
        sudo zfs-auto-snapshot --quiet --syslog --label=weekly --keep=4 $DATASET
        sudo zfs-auto-snapshot --quiet --syslog --label=monthly --keep=12 $DATASET
    else
        echo "Installer zfs-auto-snapshot pour la rétention automatique"
        echo "sudo apt-get install zfs-auto-snapshot"
    fi
}
```

### Configuration automatique avec zfs-auto-snapshot

```bash
# Installer zfs-auto-snapshot
sudo apt-get install zfs-auto-snapshot

# Activer les snapshots automatiques pour notre dataset
sudo zfs set com.sun:auto-snapshot=true microk8s-pool/volumes
sudo zfs set com.sun:auto-snapshot:hourly=true microk8s-pool/volumes
sudo zfs set com.sun:auto-snapshot:daily=true microk8s-pool/volumes
sudo zfs set com.sun:auto-snapshot:weekly=true microk8s-pool/volumes
sudo zfs set com.sun:auto-snapshot:monthly=true microk8s-pool/volumes

# Vérifier les timers systemd
systemctl list-timers | grep zfs-auto-snapshot
```

## Intégration avec Kubernetes

### Annotations pour snapshots automatiques

```yaml
# pod-with-snapshot-annotation.yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
  annotations:
    snapshot.alpha.kubernetes.io/frequency: "hourly"
    snapshot.alpha.kubernetes.io/retention: "24"
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc
```

### Operator pour snapshots automatiques

```yaml
# snapshot-operator.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: snapshot-config
  namespace: kube-system
data:
  config.yaml: |
    snapshot_policies:
      - name: hourly
        schedule: "0 * * * *"
        retention: 24
        namespaces: ["production", "staging"]
      - name: daily
        schedule: "0 2 * * *"
        retention: 7
        namespaces: ["production"]
      - name: weekly
        schedule: "0 3 * * 0"
        retention: 4
        namespaces: ["production"]

    storage_backends:
      - type: hostpath
        script: /scripts/snapshot-hostpath.sh
      - type: lvm
        script: /scripts/lvm-snapshot.sh
      - type: zfs
        script: /scripts/zfs-snapshot.sh
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: snapshot-operator
  namespace: kube-system
spec:
  schedule: "*/15 * * * *"  # Vérifier toutes les 15 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: operator
            image: python:3.9-slim
            command:
            - python
            - -c
            - |
              import kubernetes
              import yaml
              import subprocess
              from datetime import datetime

              # Charger la configuration
              with open('/config/config.yaml') as f:
                  config = yaml.safe_load(f)

              # Logic pour appliquer les politiques
              for policy in config['snapshot_policies']:
                  print(f"Processing policy: {policy['name']}")
                  # Implémenter la logique de snapshot
                  # ...
            volumeMounts:
            - name: config
              mountPath: /config
          volumes:
          - name: config
            configMap:
              name: snapshot-config
          restartPolicy: OnFailure
```

## Monitoring et alerting

### Dashboard pour les snapshots

```yaml
# grafana-dashboard-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: snapshot-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Volume Snapshots Monitor",
        "panels": [
          {
            "title": "Snapshots par heure",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(volume_snapshots_created_total[1h])"
              }
            ]
          },
          {
            "title": "Espace utilisé par les snapshots",
            "type": "stat",
            "targets": [
              {
                "expr": "sum(volume_snapshot_size_bytes) / 1073741824"
              }
            ],
            "unit": "GB"
          },
          {
            "title": "Âge du dernier snapshot",
            "type": "stat",
            "targets": [
              {
                "expr": "(time() - volume_snapshot_last_created_timestamp) / 3600"
              }
            ],
            "unit": "heures"
          },
          {
            "title": "Snapshots échoués (24h)",
            "type": "stat",
            "targets": [
              {
                "expr": "increase(volume_snapshot_failed_total[24h])"
              }
            ],
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": 0, "color": "green" },
                { "value": 1, "color": "yellow" },
                { "value": 5, "color": "red" }
              ]
            }
          }
        ]
      }
    }
```

### Métriques Prometheus personnalisées

```python
#!/usr/bin/env python3
# snapshot-exporter.py
# Exporter Prometheus pour les métriques de snapshots

from prometheus_client import start_http_server, Gauge, Counter
import subprocess
import json
import time
import os

# Définir les métriques
snapshot_count = Gauge('volume_snapshots_total', 'Nombre total de snapshots', ['type', 'namespace'])
snapshot_size = Gauge('volume_snapshot_size_bytes', 'Taille des snapshots en bytes', ['name', 'type'])
snapshot_age = Gauge('volume_snapshot_age_seconds', 'Âge du snapshot en secondes', ['name'])
snapshot_failed = Counter('volume_snapshot_failed_total', 'Nombre de snapshots échoués')
snapshot_last_created = Gauge('volume_snapshot_last_created_timestamp', 'Timestamp du dernier snapshot créé')

def collect_hostpath_metrics():
    """Collecter les métriques pour les snapshots HostPath"""
    snapshot_path = "/backup/snapshots"

    try:
        # Compter les snapshots
        count = 0
        total_size = 0

        for root, dirs, files in os.walk(snapshot_path):
            for file in files:
                if file.endswith('.metadata'):
                    count += 1
                    # Lire les métadonnées
                    with open(os.path.join(root, file)) as f:
                        meta = json.load(f)
                        # Calculer la taille
                        snap_dir = file.replace('.metadata', '')
                        if os.path.exists(os.path.join(root, snap_dir)):
                            size = subprocess.check_output(
                                ['du', '-sb', os.path.join(root, snap_dir)]
                            ).split()[0].decode('utf-8')
                            snapshot_size.labels(
                                name=snap_dir,
                                type='hostpath'
                            ).set(int(size))

        snapshot_count.labels(type='hostpath', namespace='all').set(count)

    except Exception as e:
        print(f"Erreur collecte HostPath: {e}")
        snapshot_failed.inc()

def collect_lvm_metrics():
    """Collecter les métriques pour les snapshots LVM"""
    try:
        # Obtenir la liste des snapshots LVM
        result = subprocess.run(
            ['sudo', 'lvs', '--noheadings', '--units', 'b', '-o',
             'lv_name,lv_size,origin,lv_time', '--json'],
            capture_output=True,
            text=True
        )

        if result.returncode == 0:
            data = json.loads(result.stdout)
            snapshots = [lv for lv in data['report'][0]['lv']
                        if lv.get('origin')]

            snapshot_count.labels(type='lvm', namespace='all').set(len(snapshots))

            for snap in snapshots:
                # Taille en bytes
                size = int(snap['lv_size'].replace('B', ''))
                snapshot_size.labels(
                    name=snap['lv_name'],
                    type='lvm'
                ).set(size)

    except Exception as e:
        print(f"Erreur collecte LVM: {e}")
        snapshot_failed.inc()

def collect_zfs_metrics():
    """Collecter les métriques pour les snapshots ZFS"""
    try:
        # Obtenir la liste des snapshots ZFS
        result = subprocess.run(
            ['zfs', 'list', '-t', 'snapshot', '-H', '-p',
             '-o', 'name,used,creation'],
            capture_output=True,
            text=True
        )

        if result.returncode == 0:
            lines = result.stdout.strip().split('\n')
            snapshot_count.labels(type='zfs', namespace='all').set(len(lines))

            for line in lines:
                if line:
                    parts = line.split('\t')
                    name = parts[0].split('@')[1]
                    size = int(parts[1])
                    creation = int(parts[2])

                    snapshot_size.labels(name=name, type='zfs').set(size)
                    age = time.time() - creation
                    snapshot_age.labels(name=name).set(age)

    except Exception as e:
        print(f"Erreur collecte ZFS: {e}")
        snapshot_failed.inc()

def collect_metrics():
    """Collecter toutes les métriques"""
    while True:
        collect_hostpath_metrics()
        collect_lvm_metrics()
        collect_zfs_metrics()

        # Mettre à jour le timestamp du dernier snapshot
        snapshot_last_created.set(time.time())

        # Attendre avant la prochaine collecte
        time.sleep(60)

if __name__ == '__main__':
    # Démarrer le serveur HTTP pour Prometheus
    start_http_server(9090)
    print("Snapshot exporter démarré sur le port 9090")

    # Collecter les métriques
    collect_metrics()
```

### Alertes Prometheus

```yaml
# prometheus-snapshot-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: snapshot-alerts
  namespace: monitoring
data:
  alerts.yml: |
    groups:
    - name: snapshot_alerts
      interval: 30s
      rules:

      # Alerte si aucun snapshot depuis 24h
      - alert: NoRecentSnapshot
        expr: (time() - volume_snapshot_last_created_timestamp) > 86400
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Aucun snapshot créé depuis 24 heures"
          description: "Le dernier snapshot date de {{ $value | humanizeDuration }}"

      # Alerte si trop de snapshots échoués
      - alert: HighSnapshotFailureRate
        expr: rate(volume_snapshot_failed_total[1h]) > 0.1
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Taux d'échec élevé des snapshots"
          description: "{{ $value | humanizePercentage }} des snapshots ont échoué dans la dernière heure"

      # Alerte si espace snapshots > 100GB
      - alert: SnapshotSpaceHigh
        expr: sum(volume_snapshot_size_bytes) > 107374182400
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Espace utilisé par les snapshots élevé"
          description: "Les snapshots utilisent {{ $value | humanize1024 }}B"

      # Alerte si snapshot LVM > 80% plein
      - alert: LVMSnapshotNearlyFull
        expr: lvm_snapshot_usage_percent > 80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Snapshot LVM presque plein"
          description: "Le snapshot {{ $labels.lv_name }} est utilisé à {{ $value }}%"
```

### Script de monitoring complet

```bash
#!/bin/bash
# monitor-snapshots.sh

# Couleurs pour l'output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Fonction pour afficher le statut avec couleur
print_status() {
    local status=$1
    local message=$2

    case $status in
        "OK")
            echo -e "${GREEN}✓${NC} $message"
            ;;
        "WARNING")
            echo -e "${YELLOW}⚠${NC} $message"
            ;;
        "ERROR")
            echo -e "${RED}✗${NC} $message"
            ;;
        *)
            echo "$message"
            ;;
    esac
}

# Vérifier les snapshots HostPath
check_hostpath_snapshots() {
    echo "=== Snapshots HostPath ==="

    SNAPSHOT_DIR="/backup/snapshots"

    if [ -d "$SNAPSHOT_DIR" ]; then
        # Compter les snapshots
        COUNT=$(find "$SNAPSHOT_DIR" -name "*.metadata" 2>/dev/null | wc -l)

        if [ $COUNT -gt 0 ]; then
            print_status "OK" "Nombre de snapshots: $COUNT"

            # Vérifier l'âge du dernier snapshot
            LATEST=$(find "$SNAPSHOT_DIR" -name "*.metadata" -printf '%T@\n' 2>/dev/null | sort -n | tail -1)
            if [ ! -z "$LATEST" ]; then
                AGE=$(($(date +%s) - ${LATEST%.*}))
                HOURS=$((AGE / 3600))

                if [ $HOURS -lt 24 ]; then
                    print_status "OK" "Dernier snapshot: il y a $HOURS heures"
                elif [ $HOURS -lt 48 ]; then
                    print_status "WARNING" "Dernier snapshot: il y a $HOURS heures"
                else
                    print_status "ERROR" "Dernier snapshot: il y a $HOURS heures"
                fi
            fi

            # Taille totale
            SIZE=$(du -sh "$SNAPSHOT_DIR" 2>/dev/null | cut -f1)
            print_status "OK" "Espace utilisé: $SIZE"
        else
            print_status "WARNING" "Aucun snapshot trouvé"
        fi
    else
        print_status "ERROR" "Répertoire de snapshots non trouvé"
    fi
    echo
}

# Vérifier les snapshots LVM
check_lvm_snapshots() {
    echo "=== Snapshots LVM ==="

    if command -v lvs &> /dev/null; then
        # Lister les snapshots
        SNAPSHOTS=$(sudo lvs --noheadings -o lv_name,origin,snap_percent 2>/dev/null | grep -v "^  $")

        if [ ! -z "$SNAPSHOTS" ]; then
            while IFS= read -r line; do
                LV_NAME=$(echo $line | awk '{print $1}')
                ORIGIN=$(echo $line | awk '{print $2}')
                PERCENT=$(echo $line | awk '{print $3}')

                if [ ! -z "$ORIGIN" ]; then
                    if (( $(echo "$PERCENT < 50" | bc -l) )); then
                        print_status "OK" "Snapshot $LV_NAME: ${PERCENT}% utilisé"
                    elif (( $(echo "$PERCENT < 80" | bc -l) )); then
                        print_status "WARNING" "Snapshot $LV_NAME: ${PERCENT}% utilisé"
                    else
                        print_status "ERROR" "Snapshot $LV_NAME: ${PERCENT}% utilisé - CRITIQUE!"
                    fi
                fi
            done <<< "$SNAPSHOTS"
        else
            print_status "WARNING" "Aucun snapshot LVM trouvé"
        fi
    else
        print_status "WARNING" "LVM non installé"
    fi
    echo
}

# Vérifier les snapshots ZFS
check_zfs_snapshots() {
    echo "=== Snapshots ZFS ==="

    if command -v zfs &> /dev/null; then
        # Compter les snapshots
        COUNT=$(zfs list -t snapshot 2>/dev/null | grep -c "^")

        if [ $COUNT -gt 0 ]; then
            print_status "OK" "Nombre de snapshots: $COUNT"

            # Espace utilisé
            USED=$(zfs list -t snapshot -H -o used 2>/dev/null | awk '{sum+=$1} END {print sum}')
            print_status "OK" "Espace utilisé: $USED"

            # Dernier snapshot
            LATEST=$(zfs list -t snapshot -H -o creation 2>/dev/null | tail -1)
            print_status "OK" "Dernier snapshot: $LATEST"
        else
            print_status "WARNING" "Aucun snapshot ZFS trouvé"
        fi
    else
        print_status "WARNING" "ZFS non installé"
    fi
    echo
}

# Vérifier les snapshots CSI (Kubernetes)
check_csi_snapshots() {
    echo "=== Snapshots CSI Kubernetes ==="

    if command -v microk8s &> /dev/null; then
        # Compter les VolumeSnapshots
        COUNT=$(microk8s kubectl get volumesnapshot -A --no-headers 2>/dev/null | wc -l)

        if [ $COUNT -gt 0 ]; then
            print_status "OK" "Nombre de VolumeSnapshots: $COUNT"

            # Vérifier le statut
            READY=$(microk8s kubectl get volumesnapshot -A -o json 2>/dev/null | \
                    jq '[.items[].status.readyToUse] | map(select(. == true)) | length')
            NOT_READY=$((COUNT - READY))

            if [ $NOT_READY -eq 0 ]; then
                print_status "OK" "Tous les snapshots sont prêts"
            else
                print_status "WARNING" "$NOT_READY snapshot(s) non prêt(s)"
            fi
        else
            print_status "WARNING" "Aucun VolumeSnapshot trouvé"
        fi
    fi
    echo
}

# Rapport de santé global
generate_health_report() {
    echo "╔══════════════════════════════════════════════╗"
    echo "║     RAPPORT DE SANTÉ DES SNAPSHOTS          ║"
    echo "╚══════════════════════════════════════════════╝"
    echo
    echo "Date: $(date '+%Y-%m-%d %H:%M:%S')"
    echo "Hostname: $(hostname)"
    echo

    check_hostpath_snapshots
    check_lvm_snapshots
    check_zfs_snapshots
    check_csi_snapshots

    echo "══════════════════════════════════════════════"
}

# Exécution principale
case "$1" in
    report)
        generate_health_report
        ;;
    watch)
        while true; do
            clear
            generate_health_report
            echo
            echo "Actualisation toutes les 30 secondes... (Ctrl+C pour quitter)"
            sleep 30
        done
        ;;
    export)
        generate_health_report > /tmp/snapshot-health-$(date +%Y%m%d-%H%M%S).txt
        echo "Rapport exporté vers /tmp/"
        ;;
    *)
        echo "Usage: $0 {report|watch|export}"
        echo "  report - Générer un rapport unique"
        echo "  watch  - Surveiller en continu"
        echo "  export - Exporter le rapport vers un fichier"
        ;;
esac
```

## Stratégies et bonnes pratiques

### Stratégie 3-2-1 pour les snapshots

La règle 3-2-1 adaptée aux snapshots :

```
3 copies de vos données :
  - Données originales (production)
  - Snapshot local (récupération rapide)
  - Backup externe (disaster recovery)

2 types de médias différents :
  - Stockage local (SSD/HDD avec snapshots)
  - Stockage distant (S3, NAS, Cloud)

1 copie hors site :
  - Export des snapshots importants vers le cloud
  - Réplication vers un autre datacenter
```

### Planning de snapshots par environnement

```yaml
# snapshot-strategy.yaml
Production:
  Fréquence:
    - Horaire : Jours ouvrés 8h-20h
    - Toutes les 4h : Nuits et weekends
    - Quotidien : 2h du matin
  Rétention:
    - Horaires : 24 derniers
    - Quotidiens : 7 derniers
    - Hebdomadaires : 4 derniers
    - Mensuels : 12 derniers
  Technologie: ZFS ou LVM (performance)

Staging:
  Fréquence:
    - Quotidien : Avant les déploiements
    - Hebdomadaire : Dimanche soir
  Rétention:
    - Quotidiens : 3 derniers
    - Hebdomadaires : 2 derniers
  Technologie: OpenEBS CSI

Development:
  Fréquence:
    - Manuel : Avant changements majeurs
    - Quotidien : Si actif
  Rétention:
    - Garder 2-3 derniers
  Technologie: Scripts HostPath (simplicité)
```

### Optimisation des performances

```bash
# Bonnes pratiques pour minimiser l'impact

# 1. Snapshots pendant les heures creuses
CRON_SCHEDULE="0 2-5 * * *"  # Entre 2h et 5h du matin

# 2. Utiliser nice et ionice pour les scripts
nice -n 19 ionice -c 3 ./snapshot-script.sh

# 3. Limiter la bande passante pour les transferts
rsync --bwlimit=10000 source/ destination/  # 10MB/s

# 4. Compression intelligente
# - LZ4 pour la vitesse
# - ZSTD pour le ratio
# - Gzip pour la compatibilité

# 5. Déduplication quand approprié
# ZFS : dedup=on (attention à la RAM)
# Restic/Borg : déduplication native
```

### Matrice de décision pour le choix de technologie

| Critère | HostPath Scripts | OpenEBS CSI | LVM | ZFS | Btrfs |
|---------|-----------------|-------------|-----|-----|-------|
| **Complexité setup** | ⭐ Très simple | ⭐⭐ Simple | ⭐⭐⭐ Moyenne | ⭐⭐⭐⭐ Complexe | ⭐⭐⭐ Moyenne |
| **Performance** | ⭐⭐ Moyenne | ⭐⭐⭐ Bonne | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐ Très bonne | ⭐⭐⭐⭐ Très bonne |
| **Espace requis** | ⭐ 100% copie | ⭐⭐⭐ CoW | ⭐⭐⭐⭐⭐ CoW optimal | ⭐⭐⭐⭐ CoW + compression | ⭐⭐⭐⭐ CoW |
| **Intégration K8s** | ⭐⭐ Manuelle | ⭐⭐⭐⭐⭐ Native | ⭐⭐ Scripts | ⭐⭐ Scripts | ⭐⭐ Scripts |
| **Fonctionnalités** | ⭐ Basique | ⭐⭐⭐ Standard | ⭐⭐⭐ Snapshots | ⭐⭐⭐⭐⭐ Complet | ⭐⭐⭐⭐ Avancé |
| **Fiabilité** | ⭐⭐⭐ Bonne | ⭐⭐⭐⭐ Très bonne | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐ Très bonne |

### Recommandations par profil

**Débutant MicroK8s :**
- Commencer avec les scripts HostPath
- Apprendre les concepts
- Migrer vers OpenEBS quand confortable

**Utilisateur intermédiaire :**
- OpenEBS pour l'intégration Kubernetes
- LVM si familier avec Linux
- Automatisation avec CronJobs

**Utilisateur avancé :**
- ZFS pour les fonctionnalités complètes
- Btrfs pour les systèmes modernes
- Combinaison de plusieurs technologies

## Tests et validation

### Script de test de restauration

```bash
#!/bin/bash
# test-snapshot-restore.sh

TEST_NAMESPACE="snapshot-test"
TEST_PVC="test-pvc"
TEST_DATA="/tmp/test-data-$(date +%s)"

# Créer un environnement de test
setup_test_environment() {
    echo "📦 Création de l'environnement de test..."

    # Créer le namespace
    microk8s kubectl create namespace $TEST_NAMESPACE

    # Créer un PVC de test
    cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: $TEST_PVC
  namespace: $TEST_NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

    # Créer un pod qui écrit des données
    cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: $TEST_NAMESPACE
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Test data $(date)" > /data/test.txt && sleep 3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: $TEST_PVC
EOF

    # Attendre que le pod soit prêt
    microk8s kubectl wait --for=condition=ready pod/writer -n $TEST_NAMESPACE --timeout=60s
}

# Tester la création de snapshot
test_snapshot_creation() {
    echo "📸 Test de création de snapshot..."

    # Créer un snapshot
    ./snapshot-hostpath.sh create $TEST_PVC $TEST_NAMESPACE

    if [ $? -eq 0 ]; then
        echo "✅ Snapshot créé avec succès"
        return 0
    else
        echo "❌ Échec de création du snapshot"
        return 1
    fi
}

# Tester la restauration
test_snapshot_restore() {
    echo "♻️ Test de restauration..."

    # Modifier les données originales
    microk8s kubectl exec -n $TEST_NAMESPACE writer -- sh -c 'echo "Modified data" > /data/test.txt'

    # Trouver le dernier snapshot
    LATEST_SNAPSHOT=$(ls -t /backup/snapshots/${TEST_NAMESPACE}-${TEST_PVC}/ | head -1)

    if [ -z "$LATEST_SNAPSHOT" ]; then
        echo "❌ Aucun snapshot trouvé"
        return 1
    fi

    # Restaurer
    ./snapshot-hostpath.sh restore "/backup/snapshots/${TEST_NAMESPACE}-${TEST_PVC}/$LATEST_SNAPSHOT"

    # Vérifier la restauration
    RESTORED_DATA=$(microk8s kubectl exec -n $TEST_NAMESPACE writer -- cat /data/test.txt)

    if [[ $RESTORED_DATA == *"Test data"* ]]; then
        echo "✅ Restauration réussie"
        return 0
    else
        echo "❌ Données non restaurées correctement"
        return 1
    fi
}

# Nettoyer l'environnement de test
cleanup_test_environment() {
    echo "🧹 Nettoyage..."
    microk8s kubectl delete namespace $TEST_NAMESPACE --wait=false
}

# Exécution des tests
run_tests() {
    echo "🧪 Démarrage des tests de snapshot/restore"
    echo "========================================="

    setup_test_environment

    if test_snapshot_creation; then
        echo "✅ Test de création: PASSÉ"
    else
        echo "❌ Test de création: ÉCHOUÉ"
    fi

    if test_snapshot_restore; then
        echo "✅ Test de restauration: PASSÉ"
    else
        echo "❌ Test de restauration: ÉCHOUÉ"
    fi

    cleanup_test_environment

    echo "========================================="
    echo "🏁 Tests terminés"
}

# Lancer les tests
run_tests
```

### Checklist de validation

Avant de considérer votre système de snapshots comme opérationnel :

- [ ] **Configuration de base**
  - [ ] Technologie de snapshot choisie et installée
  - [ ] Scripts de snapshot fonctionnels
  - [ ] Permissions et accès configurés

- [ ] **Automatisation**
  - [ ] Snapshots automatiques programmés
  - [ ] Politique de rétention implémentée
  - [ ] Rotation automatique des anciens snapshots

- [ ] **Tests**
  - [ ] Création de snapshot testée
  - [ ] Restauration complète testée
  - [ ] Restauration partielle testée
  - [ ] Performance acceptable validée

- [ ] **Monitoring**
  - [ ] Métriques de snapshots collectées
  - [ ] Alertes configurées
  - [ ] Dashboard de visualisation

- [ ] **Documentation**
  - [ ] Procédures documentées
  - [ ] Runbook de dépannage
  - [ ] Contacts et escalade définis

- [ ] **Sécurité**
  - [ ] Snapshots chiffrés si nécessaire
  - [ ] Accès restreints
  - [ ] Audit trail en place

## Conclusion

Les snapshots de volumes sont un outil puissant pour protéger vos données dans MicroK8s. Le choix de la technologie dépend de vos besoins spécifiques :

- **Pour débuter** : Scripts simples avec HostPath
- **Pour la standardisation** : OpenEBS avec CSI
- **Pour la performance** : LVM ou ZFS
- **Pour l'intégration complète** : Combinaison de plusieurs approches

L'important est de commencer simple, tester régulièrement, et améliorer progressivement votre stratégie. Avec les snapshots bien configurés, vous pouvez expérimenter en toute confiance dans votre lab MicroK8s, sachant que vous pouvez toujours revenir à un état stable en quelques minutes.

⏭️
