🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.7 - Gestion des volumes persistants

## Introduction

Imaginez que vos applications Kubernetes sont comme des employés dans un bureau. Par défaut, ils travaillent sur des bureaux temporaires (les Pods) : si l'employé part (le Pod redémarre), tout ce qui était sur le bureau disparaît. Les volumes persistants sont comme des armoires de rangement permanentes où les employés peuvent stocker leurs dossiers importants, qui resteront disponibles même s'ils changent de bureau.

Dans cette section, nous allons explorer comment gérer le stockage persistant dans Kubernetes, permettant à vos applications de conserver leurs données au-delà du cycle de vie des Pods.

## Le problème du stockage éphémère

### Pourquoi les Pods perdent leurs données

Par défaut, les données dans un Pod sont éphémères pour plusieurs raisons :

1. **Les Pods sont mortels** : Ils peuvent être tués, recréés, déplacés
2. **Les conteneurs sont immutables** : Principe des conteneurs Docker
3. **Le scaling détruit les données** : Scale down = perte de Pods
4. **Les mises à jour remplacent** : Rolling update = nouveaux Pods

**Exemple du problème :**
```yaml
# ❌ Sans stockage persistant
apiVersion: v1
kind: Pod
metadata:
  name: database-ephemere
spec:
  containers:
  - name: postgres
    image: postgres:14
    # Les données sont stockées dans le conteneur
    # Si le Pod redémarre, TOUTES les données sont perdues !
```

### Applications nécessitant du stockage persistant

- **Bases de données** : PostgreSQL, MySQL, MongoDB
- **Systèmes de fichiers** : WordPress uploads, NextCloud
- **Files de messages** : Kafka, RabbitMQ
- **Caches persistants** : Redis avec persistance
- **Logs et métriques** : Elasticsearch, Prometheus
- **Applications stateful** : Sessions, états, configurations

## Architecture du stockage Kubernetes

### Vue d'ensemble des concepts

```
┌─────────────────────────────────────────────────────────┐
│                   Stockage Kubernetes                    │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  StorageClass    →    PersistentVolume    →    Pod      │
│       ↑                      ↑                   ↑       │
│       │                      │                   │       │
│   (Définit)          (Provisionne)          (Monte)     │
│       │                      │                   │       │
│       └──────  PersistentVolumeClaim  ──────────┘       │
│                     (Demande)                            │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Les composants clés

1. **StorageClass (SC)** : Définit les types de stockage disponibles
2. **PersistentVolume (PV)** : Représente un espace de stockage physique
3. **PersistentVolumeClaim (PVC)** : Demande de stockage par une application
4. **Volume** : Montage du stockage dans le Pod

## Partie 1 : Les Volumes de base

### Types de volumes temporaires

#### EmptyDir : Stockage temporaire partagé

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
spec:
  containers:
  # Premier conteneur : écrit des données
  - name: writer
    image: busybox
    command: ['sh', '-c']
    args:
    - while true; do
        echo "$(date) - Message du writer" >> /shared/log.txt;
        sleep 5;
      done
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  # Deuxième conteneur : lit les données
  - name: reader
    image: busybox
    command: ['sh', '-c']
    args:
    - tail -f /shared/log.txt
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  volumes:
  - name: shared-data
    emptyDir: {}  # Créé vide au démarrage du Pod
```

**Variante en mémoire (RAM) :**
```yaml
volumes:
- name: cache-volume
  emptyDir:
    medium: Memory  # Utilise la RAM au lieu du disque
    sizeLimit: 100Mi  # Limite de taille
```

**Cas d'usage emptyDir :**
- Cache temporaire entre conteneurs
- Données de traitement temporaires
- Espace de travail pour jobs
- Fichiers de lock/coordination

#### HostPath : Accès au système de fichiers du nœud

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-hostpath
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /usr/share/nginx/html

  volumes:
  - name: host-volume
    hostPath:
      path: /data/website  # Chemin sur le nœud hôte
      type: DirectoryOrCreate  # Crée si n'existe pas
```

**Types de hostPath :**
```yaml
# Différents types de validation
hostPath:
  path: /data
  type: DirectoryOrCreate  # Crée le répertoire si absent
  # type: Directory        # Doit exister et être un répertoire
  # type: FileOrCreate     # Crée le fichier si absent
  # type: File             # Doit exister et être un fichier
  # type: Socket           # Doit être un socket UNIX
  # type: CharDevice       # Doit être un character device
  # type: BlockDevice      # Doit être un block device
```

**⚠️ Attention avec hostPath :**
- Lie le Pod à un nœud spécifique
- Problèmes de sécurité potentiels
- Pas portable entre environnements
- À éviter en production

### ConfigMap et Secret comme volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-configs
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    # ConfigMap monté comme fichiers
    - name: config-volume
      mountPath: /etc/config

    # Secret monté comme fichiers
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  # Volume depuis ConfigMap
  - name: config-volume
    configMap:
      name: app-config
      defaultMode: 0644
      items:
      - key: app.properties
        path: application.properties

  # Volume depuis Secret
  - name: secret-volume
    secret:
      secretName: app-secrets
      defaultMode: 0400  # Permissions restrictives
```

## Partie 2 : Stockage persistant avec PV et PVC

### PersistentVolume (PV)

Un PersistentVolume représente un morceau de stockage provisionné dans le cluster.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
  labels:
    type: local
    environment: production
spec:
  # Capacité du volume
  capacity:
    storage: 10Gi

  # Modes d'accès
  accessModes:
    - ReadWriteOnce  # RWO : Un seul nœud en lecture/écriture
    # - ReadOnlyMany   # ROX : Plusieurs nœuds en lecture seule
    # - ReadWriteMany  # RWX : Plusieurs nœuds en lecture/écriture

  # Politique de rétention
  persistentVolumeReclaimPolicy: Retain  # Que faire quand le PVC est supprimé
  # Retain : Garder les données
  # Recycle : Effacer les données (deprecated)
  # Delete : Supprimer le volume

  # Classe de stockage
  storageClassName: manual

  # Type de volume (ici local)
  hostPath:
    path: /mnt/data/pv-example
```

### PersistentVolumeClaim (PVC)

Un PVC est une demande de stockage par un utilisateur.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
  namespace: default
spec:
  # Mode d'accès demandé
  accessModes:
    - ReadWriteOnce

  # Ressources demandées
  resources:
    requests:
      storage: 5Gi  # Demande 5Gi (sera satisfait par un PV de 10Gi)

  # Classe de stockage (doit correspondre au PV)
  storageClassName: manual

  # Sélecteur de PV (optionnel)
  selector:
    matchLabels:
      type: local
      environment: production
```

### Utilisation dans un Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-storage
spec:
  replicas: 1  # Important : RWO = 1 replica seulement
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        volumeMounts:
        - name: persistent-storage
          mountPath: /data
          subPath: myapp  # Sous-répertoire dans le volume

      volumes:
      - name: persistent-storage
        persistentVolumeClaim:
          claimName: pvc-example  # Référence au PVC
```

### Cycle de vie PV/PVC

```
1. Provisioning
   ├── Static : Admin crée manuellement les PV
   └── Dynamic : PV créé automatiquement via StorageClass

2. Binding
   └── PVC trouve un PV compatible et s'y lie (1:1)

3. Using
   └── Pod monte le PVC comme volume

4. Reclaiming
   ├── Retain : Garde les données pour récupération manuelle
   ├── Delete : Supprime le PV et les données
   └── Recycle : Nettoie pour réutilisation (deprecated)
```

## Partie 3 : StorageClass et provisioning dynamique

### Qu'est-ce qu'un StorageClass ?

Un StorageClass définit une "classe" de stockage avec des caractéristiques spécifiques.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Classe par défaut
provisioner: kubernetes.io/aws-ebs  # Qui crée les volumes
parameters:
  type: gp3  # Type de volume AWS
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Delete  # Politique par défaut pour les PV créés
allowVolumeExpansion: true  # Permet l'extension des volumes
mountOptions:
  - debug
  - noatime
volumeBindingMode: WaitForFirstConsumer  # Attendre qu'un Pod ait besoin
# volumeBindingMode: Immediate  # Créer immédiatement
```

### StorageClass pour MicroK8s

```yaml
# StorageClass par défaut de MicroK8s
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: microk8s-hostpath
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

### Provisioning dynamique

Avec un StorageClass, plus besoin de créer manuellement les PV :

```yaml
# PVC avec provisioning dynamique
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: microk8s-hostpath  # Le PV sera créé automatiquement
```

```bash
# Vérifier la création automatique
kubectl get pvc dynamic-pvc
# NAME          STATUS   VOLUME                                     CAPACITY
# dynamic-pvc   Bound    pvc-abc123-def456-789012                  10Gi

kubectl get pv
# Le PV a été créé automatiquement avec un nom généré
```

### Exemples de StorageClass par provider

#### AWS EBS
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "10000"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/12345678"
```

#### Google Cloud
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
```

#### Azure
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
provisioner: kubernetes.io/azure-disk
parameters:
  skuName: Premium_LRS
  location: westeurope
```

#### NFS
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs.io/provisioner
parameters:
  server: nfs-server.example.com
  path: /exported/path
  readOnly: "false"
```

## Partie 4 : Patterns d'utilisation avancés

### Pattern 1 : StatefulSet avec volumes

Les StatefulSets sont parfaits pour les applications avec état :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data

  # Template de PVC pour chaque Pod
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```

**Résultat :**
- `postgres-0` → `postgres-storage-postgres-0` (PVC) → PV dédié
- `postgres-1` → `postgres-storage-postgres-1` (PVC) → PV dédié
- `postgres-2` → `postgres-storage-postgres-2` (PVC) → PV dédié

### Pattern 2 : Volumes partagés ReadWriteMany

Pour partager des données entre plusieurs Pods :

```yaml
# PVC avec accès RWX
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany  # Plusieurs Pods peuvent écrire
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs-storage  # NFS supporte RWX
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-farm
spec:
  replicas: 5  # 5 Pods partageant le même stockage
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: shared-content
          mountPath: /usr/share/nginx/html

      volumes:
      - name: shared-content
        persistentVolumeClaim:
          claimName: shared-storage
```

### Pattern 3 : Backup et restore

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:14
            command:
            - /bin/sh
            - -c
            - |
              # Backup de la base de données
              pg_dump -h postgres-service -U postgres mydb > /backup/backup-$(date +%Y%m%d).sql

              # Garder seulement les 7 derniers backups
              ls -t /backup/*.sql | tail -n +8 | xargs rm -f

              echo "Backup completed"
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            volumeMounts:
            - name: source-data
              mountPath: /data
              readOnly: true
            - name: backup-storage
              mountPath: /backup

          restartPolicy: OnFailure
          volumes:
          - name: source-data
            persistentVolumeClaim:
              claimName: postgres-data
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

### Pattern 4 : Migration de volumes

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: volume-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Starting volume migration..."

          # Copier les données de l'ancien vers le nouveau volume
          cp -av /old-volume/* /new-volume/

          # Vérifier l'intégrité
          if [ "$(ls -A /new-volume)" ]; then
            echo "Migration successful"
          else
            echo "Migration failed - new volume is empty"
            exit 1
          fi
        volumeMounts:
        - name: old-volume
          mountPath: /old-volume
          readOnly: true
        - name: new-volume
          mountPath: /new-volume

      restartPolicy: Never
      volumes:
      - name: old-volume
        persistentVolumeClaim:
          claimName: old-pvc
      - name: new-volume
        persistentVolumeClaim:
          claimName: new-pvc
```

## Partie 5 : Gestion avancée

### Expansion de volumes

Augmenter la taille d'un volume existant :

```yaml
# 1. Vérifier que le StorageClass permet l'expansion
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true  # Doit être true
```

```bash
# 2. Éditer le PVC pour augmenter la taille
kubectl edit pvc my-pvc

# Changer :
# spec:
#   resources:
#     requests:
#       storage: 10Gi  # Ancienne taille
# En :
#   resources:
#     requests:
#       storage: 20Gi  # Nouvelle taille

# 3. Vérifier l'expansion
kubectl describe pvc my-pvc
# Conditions:
#   Type                      Status
#   FileSystemResizePending   True  # En cours
#   # Puis :
#   FileSystemResizeSuccessful True  # Terminé
```

### Snapshots de volumes

Créer des snapshots pour backup ou clonage :

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
# Créer un snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: postgres-data
---
# Restaurer depuis un snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  dataSource:
    name: postgres-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Clonage de volumes

Cloner un PVC existant :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: source-pvc  # PVC à cloner
```

### Volume Populator

Pré-remplir un volume avec des données :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: populated-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  dataSourceRef:
    apiGroup: populator.storage.k8s.io
    kind: GitRepo
    name: my-git-repo
---
# Source de données
apiVersion: populator.storage.k8s.io/v1alpha1
kind: GitRepo
metadata:
  name: my-git-repo
spec:
  url: https://github.com/myorg/myrepo.git
  branch: main
```

## Monitoring et maintenance

### Métriques de stockage

```yaml
# ServiceMonitor pour Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: storage-metrics
spec:
  selector:
    matchLabels:
      app: csi-provisioner
  endpoints:
  - port: metrics
    interval: 30s
```

Requêtes Prometheus utiles :
```promql
# Utilisation des PV
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100

# PVC pending
kube_persistentvolumeclaim_status_phase{phase="Pending"}

# Volumes presque pleins (>80%)
(kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.8
```

### Alertes de stockage

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
spec:
  groups:
  - name: storage
    rules:
    - alert: PVCPending
      expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} > 0
      for: 5m
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} is pending"

    - alert: VolumeAlmostFull
      expr: |
        (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.8
      for: 5m
      annotations:
        summary: "Volume {{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"

    - alert: VolumeIOHigh
      expr: |
        rate(kubelet_volume_stats_inodes_used[5m]) > 1000
      for: 5m
      annotations:
        summary: "High IO on volume {{ $labels.persistentvolumeclaim }}"
```

### Nettoyage et maintenance

```bash
#!/bin/bash
# cleanup-volumes.sh

# Lister les PV non utilisés
echo "=== PV non liés ==="
kubectl get pv | grep Released

# Lister les PVC non utilisés
echo "=== PVC non montés ==="
for pvc in $(kubectl get pvc -o name); do
  if ! kubectl get pods --all-namespaces -o json | grep -q "$(basename $pvc)"; then
    echo "$pvc n'est pas utilisé"
  fi
done

# Nettoyer les PV en état Released
echo "=== Nettoyage des PV Released ==="
for pv in $(kubectl get pv | grep Released | awk '{print $1}'); do
  echo "Suppression de $pv"
  kubectl delete pv $pv
done

# Vérifier l'espace disque sur les nœuds
echo "=== Espace disque des nœuds ==="
kubectl get nodes -o json | jq -r '.items[] |
  "\(.metadata.name): \(.status.allocatable.storage)"'
```

## Troubleshooting

### Problèmes courants

#### PVC reste en Pending

```bash
# Diagnostic
kubectl describe pvc my-pvc

# Causes possibles :
# 1. Pas de PV disponible correspondant
kubectl get pv

# 2. StorageClass n'existe pas
kubectl get storageclass

# 3. Provisioner ne fonctionne pas
kubectl get pods -n kube-system | grep provision

# Solutions :
# Créer un PV manuel si provisioning statique
# Vérifier que le provisioner est installé
# Vérifier les quotas de ressources
```

#### Pod ne peut pas monter le volume

```bash
# Message d'erreur typique :
# Unable to attach or mount volumes

# Vérifications :
# 1. Le PVC existe et est Bound
kubectl get pvc

# 2. Le nœud peut accéder au stockage
kubectl describe node <node-name>

# 3. Vérifier les events
kubectl get events --sort-by='.lastTimestamp'

# 4. Logs du kubelet sur le nœud
journalctl -u snap.microk8s.daemon-kubelet -f
```

#### Performance dégradée

```bash
# Vérifier les IOPS et la latence
kubectl exec -it <pod> -- iostat -x 1

# Vérifier l'utilisation du disque
kubectl exec -it <pod> -- df -h

# Métriques du volume
kubectl top pod <pod> --containers
```

## Bonnes pratiques

### 1. Choix du type de stockage

| Type | Cas d'usage | Avantages | Inconvénients |
|------|------------|-----------|---------------|
| **emptyDir** | Cache, temp | Simple, rapide | Éphémère |
| **hostPath** | Dev, test | Accès direct | Non portable |
| **PVC local** | DB single node | Performance | Pas de HA |
| **PVC réseau** | Apps distribuées | HA, flexible | Latence |
| **CSI** | Production | Standards, features | Complexité |

### 2. Dimensionnement

```yaml
# Toujours définir des limites
resources:
  requests:
    storage: 10Gi  # Minimum garanti
  limits:
    storage: 20Gi  # Maximum autorisé (si supporté)

# Prévoir la croissance
# Règle : Demander 120% du besoin actuel
# Si besoin = 8Gi, demander 10Gi
```

### 3. Sécurité

```yaml
# Montage en lecture seule quand possible
volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true

# Permissions restrictives
securityContext:
  fsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
```

### 4. Backup stratégies

```yaml
# Stratégie 3-2-1
# 3 copies des données
# 2 supports différents
# 1 copie hors-site

# Exemple :
# - Production PV (copie 1)
# - Snapshot quotidien (copie 2, même cloud)
# - Backup S3 hebdo (copie 3, différent provider)
```

### 5. Labels et annotations

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  labels:
    app: myapp
    environment: production
    tier: database
  annotations:
    backup-policy: "daily"
    encryption: "aes-256"
    owner: "database-team"
    created-by: "terraform"
spec:
  # ...
```

## Checklist de déploiement

### Avant le déploiement

- [ ] Type de stockage approprié choisi
- [ ] Dimensionnement calculé (+20% marge)
- [ ] Mode d'accès correct (RWO/ROX/RWX)
- [ ] StorageClass défini ou existant
- [ ] Politique de reclaim appropriée
- [ ] Backup stratégie définie

### Configuration

- [ ] PVC créé dans le bon namespace
- [ ] Labels cohérents appliqués
- [ ] Annotations de documentation
- [ ] Limites de ressources définies
- [ ] Monitoring configuré
- [ ] Alertes mises en place

### Sécurité

- [ ] Chiffrement activé si nécessaire
- [ ] Montages ReadOnly où possible
- [ ] Permissions filesystem appropriées
- [ ] Network policies si données sensibles
- [ ] Audit logging activé

### Maintenance

- [ ] Procédure de backup testée
- [ ] Procédure de restore documentée
- [ ] Expansion possible si besoin
- [ ] Monitoring des métriques
- [ ] Alertes configurées
- [ ] Documentation à jour

## Conclusion

La gestion des volumes persistants est essentielle pour toute application stateful dans Kubernetes. Une bonne compréhension et utilisation du stockage permet de :

**Points clés à retenir :**

1. **Éphémère vs Persistant** : Comprendre quand utiliser chaque type
   - emptyDir pour le temporaire
   - PVC pour le persistant

2. **Architecture en couches** : StorageClass → PV → PVC → Pod
   - Abstraction et flexibilité
   - Provisioning dynamique quand possible

3. **Modes d'accès** : Choisir le bon mode
   - RWO : Bases de données single-instance
   - RWX : Stockage partagé (NFS, CephFS)
   - ROX : Contenu statique distribué

4. **StatefulSets** : Pour les applications avec état
   - Un PVC par Pod
   - Identité stable
   - Ordre de démarrage garanti

5. **Backup et récupération** : Toujours avoir un plan
   - Snapshots réguliers
   - Tests de restauration
   - Stratégie 3-2-1

6. **Monitoring** : Surveiller l'utilisation
   - Espace disponible
   - Performance I/O
   - Alertes proactives

7. **Sécurité** : Protéger les données
   - Chiffrement si nécessaire
   - Permissions appropriées
   - Audit des accès

## Exemples complets par cas d'usage

### WordPress avec stockage persistant

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
---
# Secret pour MySQL
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
stringData:
  password: "W0rdPr3ss!2024"
---
# PVC pour MySQL
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: microk8s-hostpath
---
# PVC pour WordPress
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: microk8s-hostpath
---
# Deployment MySQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
# Service MySQL
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
---
# Deployment WordPress
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql:3306
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: WORDPRESS_DB_NAME
          value: wordpress
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc
---
# Service WordPress
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: LoadBalancer
  selector:
    app: wordpress
  ports:
  - port: 80
    targetPort: 80
```

### Elasticsearch cluster avec StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elastic
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data

      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true

      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: cluster.name
          value: es-cluster
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        - name: xpack.security.enabled
          value: "false"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 30Gi
---
# Service Headless pour discovery
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elastic
spec:
  clusterIP: None
  selector:
    app: elasticsearch
  ports:
  - name: http
    port: 9200
  - name: transport
    port: 9300
```

### Kafka avec volumes séparés pour logs et data

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: streaming
spec:
  serviceName: kafka
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        env:
        - name: KAFKA_BROKER_ID_COMMAND
          value: "hostname | awk -F'-' '{print $NF}'"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper:2181"
        - name: KAFKA_LISTENERS
          value: "PLAINTEXT://0.0.0.0:9092"
        - name: KAFKA_DATA_DIRS
          value: "/var/lib/kafka/data,/var/lib/kafka/logs"
        - name: KAFKA_LOG_RETENTION_HOURS
          value: "168"
        - name: KAFKA_LOG_SEGMENT_BYTES
          value: "1073741824"
        - name: KAFKA_NUM_PARTITIONS
          value: "3"
        - name: KAFKA_DEFAULT_REPLICATION_FACTOR
          value: "2"
        ports:
        - containerPort: 9092
        volumeMounts:
        - name: data
          mountPath: /var/lib/kafka/data
        - name: logs
          mountPath: /var/lib/kafka/logs
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
  - metadata:
      name: logs
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 50Gi
```

## Scripts utiles pour la gestion du stockage

### Script de monitoring d'utilisation

```bash
#!/bin/bash
# monitor-storage.sh

echo "=== Rapport d'utilisation du stockage ==="
echo ""

# Utilisation par PVC
echo "Utilisation des PVC:"
echo "--------------------"
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  pvcs=$(kubectl get pvc -n $ns -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)
  if [ ! -z "$pvcs" ]; then
    echo "Namespace: $ns"
    for pvc in $pvcs; do
      # Obtenir le Pod qui utilise ce PVC
      pod=$(kubectl get pods -n $ns -o json | jq -r --arg pvc "$pvc" '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName == $pvc) | .metadata.name' | head -1)

      if [ ! -z "$pod" ]; then
        # Obtenir l'utilisation depuis le Pod
        usage=$(kubectl exec -n $ns $pod -- df -h 2>/dev/null | grep -E "^/dev/" | head -1 | awk '{print $5}')
        size=$(kubectl get pvc $pvc -n $ns -o jsonpath='{.status.capacity.storage}')
        echo "  - $pvc: $usage utilisé sur $size"
      else
        echo "  - $pvc: Non monté"
      fi
    done
    echo ""
  fi
done

# PV en état Released
echo "PV en état Released (à nettoyer):"
echo "----------------------------------"
kubectl get pv | grep Released || echo "Aucun"
echo ""

# StorageClass utilisation
echo "Répartition par StorageClass:"
echo "-----------------------------"
for sc in $(kubectl get storageclass -o jsonpath='{.items[*].metadata.name}'); do
  count=$(kubectl get pvc -A -o json | jq --arg sc "$sc" '[.items[] | select(.spec.storageClassName == $sc)] | length')
  total_size=$(kubectl get pvc -A -o json | jq --arg sc "$sc" '[.items[] | select(.spec.storageClassName == $sc) | .spec.resources.requests.storage // "0" | rtrimstr("Gi") | tonumber] | add')
  echo "$sc: $count PVC, ${total_size:-0}Gi total"
done
```

### Script de backup automatique

```bash
#!/bin/bash
# backup-volumes.sh

BACKUP_DIR="/backup/volumes/$(date +%Y%m%d)"
NAMESPACE="${1:-default}"

mkdir -p $BACKUP_DIR

echo "Démarrage backup des volumes du namespace: $NAMESPACE"

# Pour chaque PVC
for pvc in $(kubectl get pvc -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  echo "Backup de $pvc..."

  # Créer un Job de backup
  cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-$pvc-$(date +%s)
  namespace: $NAMESPACE
spec:
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      containers:
      - name: backup
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          tar czf /backup/$pvc-$(date +%Y%m%d-%H%M%S).tar.gz /data/*
          echo "Backup de $pvc terminé"
        volumeMounts:
        - name: source
          mountPath: /data
          readOnly: true
        - name: backup
          mountPath: /backup
      restartPolicy: Never
      volumes:
      - name: source
        persistentVolumeClaim:
          claimName: $pvc
      - name: backup
        hostPath:
          path: $BACKUP_DIR
          type: DirectoryOrCreate
EOF

  # Attendre que le job se termine
  kubectl wait --for=condition=complete job/backup-$pvc-$(date +%s) -n $NAMESPACE --timeout=300s
done

echo "Backup terminé dans $BACKUP_DIR"
```

### Script de migration de StorageClass

```bash
#!/bin/bash
# migrate-storageclass.sh

OLD_SC="$1"
NEW_SC="$2"
NAMESPACE="${3:-default}"

if [ -z "$OLD_SC" ] || [ -z "$NEW_SC" ]; then
  echo "Usage: $0 <old-storageclass> <new-storageclass> [namespace]"
  exit 1
fi

echo "Migration de $OLD_SC vers $NEW_SC dans le namespace $NAMESPACE"

# Pour chaque PVC utilisant l'ancien StorageClass
for pvc in $(kubectl get pvc -n $NAMESPACE -o json | jq -r --arg sc "$OLD_SC" '.items[] | select(.spec.storageClassName == $sc) | .metadata.name'); do
  echo "Migration de $pvc..."

  # Obtenir la taille
  size=$(kubectl get pvc $pvc -n $NAMESPACE -o jsonpath='{.spec.resources.requests.storage}')

  # Créer un nouveau PVC
  kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${pvc}-new
  namespace: $NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: $size
  storageClassName: $NEW_SC
EOF

  # Créer un Job pour copier les données
  kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-${pvc}
  namespace: $NAMESPACE
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: busybox
        command: ['sh', '-c', 'cp -av /old/* /new/ && echo "Migration complete"']
        volumeMounts:
        - name: old
          mountPath: /old
        - name: new
          mountPath: /new
      restartPolicy: Never
      volumes:
      - name: old
        persistentVolumeClaim:
          claimName: $pvc
      - name: new
        persistentVolumeClaim:
          claimName: ${pvc}-new
EOF

  # Attendre la fin
  kubectl wait --for=condition=complete job/migrate-${pvc} -n $NAMESPACE --timeout=600s

  echo "$pvc migré vers ${pvc}-new"
done

echo "Migration terminée. Vérifiez les données avant de supprimer les anciens PVC."
```

## Ressources et références

### Documentation officielle
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
- [CSI Specification](https://github.com/container-storage-interface/spec)
- [MicroK8s Storage](https://microk8s.io/docs/addon-hostpath-storage)

### Outils recommandés
- **Velero** : Backup et restore de clusters
- **Rook** : Orchestrateur de stockage (Ceph)
- **Longhorn** : Stockage distribué lightweight
- **OpenEBS** : Container Attached Storage
- **MinIO** : Object storage S3-compatible

### Commandes de référence rapide

```bash
# PVC
kubectl get pvc -A
kubectl describe pvc <name> -n <namespace>
kubectl edit pvc <name> -n <namespace>
kubectl delete pvc <name> -n <namespace>

# PV
kubectl get pv
kubectl describe pv <name>
kubectl delete pv <name>

# StorageClass
kubectl get storageclass
kubectl describe storageclass <name>
kubectl set default storageclass <name>

# Utilisation
kubectl exec <pod> -- df -h
kubectl top pod <pod> --containers

# Events liés au stockage
kubectl get events --field-selector reason=FailedMount
kubectl get events --field-selector reason=FailedAttachVolume
```

## Résumé final

La gestion des volumes persistants dans Kubernetes permet de :

✅ **Découpler les données du cycle de vie des Pods**
- Les données survivent aux redémarrages
- Migration possible entre nœuds

✅ **Abstraire la complexité du stockage**
- StorageClass pour différents types
- Provisioning dynamique automatique

✅ **Supporter tous les patterns d'applications**
- Stateless avec emptyDir
- Stateful avec PVC
- Distributed avec RWX

✅ **Garantir la disponibilité des données**
- Snapshots et backups
- Réplication et distribution

✅ **Optimiser les performances**
- Classes de stockage adaptées
- Monitoring et tuning

Le chapitre 7 sur le déploiement d'applications est maintenant complet. Vous avez tous les outils pour :
1. Comprendre l'anatomie d'un déploiement (7.1)
2. Écrire des manifestes YAML efficaces (7.2)
3. Déployer des applications complètes (7.3)
4. Les exposer via Services et Ingress (7.4)
5. Gérer la configuration avec ConfigMaps (7.5)
6. Sécuriser les données sensibles avec Secrets (7.6)
7. Implémenter le stockage persistant (7.7)

Avec ces connaissances, vous êtes prêt à déployer et gérer des applications complexes sur votre cluster MicroK8s !

⏭️
