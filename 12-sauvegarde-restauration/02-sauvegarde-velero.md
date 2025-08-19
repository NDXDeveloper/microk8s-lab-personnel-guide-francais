🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2 Sauvegarde avec Velero

## Qu'est-ce que Velero ?

Velero (anciennement Heptio Ark) est l'outil de référence pour la sauvegarde et la restauration dans Kubernetes. Imaginez Velero comme un "Time Machine" pour votre cluster : il peut prendre des instantanés de l'état complet de votre cluster et les restaurer plus tard, même sur un cluster différent.

### Pourquoi Velero pour MicroK8s ?

**Avantages spécifiques :**
- **Conçu pour Kubernetes** : Comprend nativement tous les objets Kubernetes
- **Portable** : Les sauvegardes peuvent être restaurées sur n'importe quel cluster
- **Sélectif** : Sauvegarde par namespace, label, ou type de ressource
- **Programmable** : Sauvegardes automatiques avec des schedules
- **Extensible** : Plugins pour différents types de stockage
- **Open Source** : Gratuit et maintenu par VMware

**Ce que Velero peut sauvegarder :**
- Tous les objets Kubernetes (Deployments, Services, ConfigMaps, etc.)
- Volumes persistants (avec plugins de snapshot)
- État complet ou partiel du cluster
- Métadonnées et annotations

**Ce que Velero ne sauvegarde pas :**
- État d'etcd directement (mais capture les objets stockés dedans)
- Configuration de MicroK8s elle-même
- Données en dehors de Kubernetes

## Architecture de Velero

### Composants principaux

```
┌─────────────────────────────────────────────────────┐
│                   Cluster MicroK8s                   │
│                                                      │
│  ┌─────────────────┐         ┌──────────────────┐  │
│  │  Velero Server  │────────▶│  Backup Storage  │  │
│  │   (Pod dans     │         │  (S3, Minio,    │  │
│  │    namespace    │         │   fichiers...)   │  │
│  │     velero)     │         └──────────────────┘  │
│  └────────┬────────┘                                │
│           │                                         │
│           ▼                                         │
│  ┌─────────────────┐         ┌──────────────────┐  │
│  │   Vos Apps      │         │  Volume          │  │
│  │  (Deployments,  │────────▶│  Snapshots       │  │
│  │   Services...)  │         │                  │  │
│  └─────────────────┘         └──────────────────┘  │
└─────────────────────────────────────────────────────┘
           │
           ▼
    ┌─────────────┐
    │ Velero CLI  │
    │ (ligne de   │
    │  commande)  │
    └─────────────┘
```

### Concepts clés

**Backup (Sauvegarde)**
- Instantané d'objets Kubernetes à un moment donné
- Stocké dans un emplacement externe (object storage)
- Peut inclure ou exclure des ressources spécifiques

**Restore (Restauration)**
- Recréation des objets depuis une sauvegarde
- Peut être complète ou sélective
- Gère les conflits avec objets existants

**Schedule (Programmation)**
- Sauvegardes automatiques périodiques
- Utilise la syntaxe cron
- Politique de rétention configurable

**Backup Storage Location**
- Où les sauvegardes sont stockées
- Supporte S3, Azure Blob, GCS, système de fichiers
- Peut avoir plusieurs emplacements

**Volume Snapshot Location**
- Où les snapshots de volumes sont stockés
- Dépend du provider de stockage
- Optionnel si pas de volumes persistants

## Installation de Velero dans MicroK8s

### Prérequis

Avant d'installer Velero, vérifiez que vous avez :

```bash
# MicroK8s installé et fonctionnel
microk8s status

# kubectl configuré
microk8s kubectl version

# Au moins 1GB d'espace disque libre
df -h

# Accès à un stockage pour les backups (nous utiliserons Minio en local)
```

### Étape 1 : Installation de Minio (stockage local S3)

Pour un lab, Minio est parfait comme stockage S3 local :

```bash
# Créer le namespace
microk8s kubectl create namespace minio

# Déployer Minio
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
spec:
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
          value: "minioadmin"
        - name: MINIO_ROOT_PASSWORD
          value: "minioadmin123"
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio
spec:
  selector:
    app: minio
  ports:
  - name: api
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
  type: NodePort
EOF
```

### Étape 2 : Configuration du bucket Minio

```bash
# Obtenir le port NodePort de Minio
MINIO_PORT=$(microk8s kubectl get svc minio -n minio -o jsonpath='{.spec.ports[?(@.name=="api")].nodePort}')
echo "Minio API disponible sur : http://localhost:$MINIO_PORT"

# Installer le client Minio (mc)
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configurer l'alias Minio
mc alias set myminio http://localhost:$MINIO_PORT minioadmin minioadmin123

# Créer le bucket pour Velero
mc mb myminio/velero-backups
```

### Étape 3 : Téléchargement de Velero CLI

```bash
# Télécharger la dernière version de Velero
VELERO_VERSION=v1.13.0
wget https://github.com/vmware-tanzu/velero/releases/download/$VELERO_VERSION/velero-$VELERO_VERSION-linux-amd64.tar.gz

# Extraire et installer
tar -xvf velero-$VELERO_VERSION-linux-amd64.tar.gz
sudo mv velero-$VELERO_VERSION-linux-amd64/velero /usr/local/bin/

# Vérifier l'installation
velero version --client-only
```

### Étape 4 : Installation de Velero dans le cluster

```bash
# Créer le fichier de credentials pour Minio
cat <<EOF > credentials-velero
[default]
aws_access_key_id = minioadmin
aws_secret_access_key = minioadmin123
EOF

# Installer Velero avec le plugin AWS (compatible S3/Minio)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio:9000 \
  --kubeconfig /var/snap/microk8s/current/credentials/client.config

# Attendre que Velero soit prêt
microk8s kubectl wait --for=condition=ready pod -l component=velero -n velero --timeout=300s
```

### Étape 5 : Vérification de l'installation

```bash
# Vérifier les pods Velero
microk8s kubectl get pods -n velero

# Vérifier la connexion au stockage
velero backup-location get

# Les logs de Velero
microk8s kubectl logs -n velero deployment/velero
```

## Utilisation basique de Velero

### Créer une sauvegarde manuelle

```bash
# Sauvegarde complète du cluster
velero backup create full-backup-$(date +%Y%m%d-%H%M%S)

# Sauvegarde d'un namespace spécifique
velero backup create my-app-backup --include-namespaces my-app

# Sauvegarde avec exclusions
velero backup create backup-sans-logs \
  --exclude-namespaces kube-system,kube-public,kube-node-lease

# Sauvegarde avec sélecteur de labels
velero backup create prod-backup \
  --selector environment=production
```

### Surveiller les sauvegardes

```bash
# Lister toutes les sauvegardes
velero backup get

# Détails d'une sauvegarde
velero backup describe full-backup-20240315-143022

# Logs d'une sauvegarde
velero backup logs full-backup-20240315-143022

# Progression d'une sauvegarde en cours
velero backup describe full-backup-20240315-143022 --details
```

### Structure d'une sauvegarde Velero

Quand Velero crée une sauvegarde, voici ce qui est stocké :

```
velero-backups/
└── backups/
    └── full-backup-20240315-143022/
        ├── velero-backup.json         # Métadonnées de la sauvegarde
        ├── full-backup-20240315.tar.gz # Objets Kubernetes
        ├── logs/                       # Logs de la sauvegarde
        └── resource-list.json          # Liste des ressources
```

## Sauvegardes programmées

### Créer un schedule

```bash
# Sauvegarde quotidienne à 2h du matin
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h0m0s  # Garde les backups 7 jours

# Sauvegarde hebdomadaire le dimanche
velero schedule create weekly-backup \
  --schedule="0 3 * * 0" \
  --ttl 720h0m0s  # Garde les backups 30 jours

# Sauvegarde d'un namespace toutes les 6 heures
velero schedule create app-backup \
  --schedule="0 */6 * * *" \
  --include-namespaces my-app \
  --ttl 72h0m0s  # Garde les backups 3 jours
```

### Syntaxe cron pour les schedules

```
┌───────────── minute (0 - 59)
│ ┌───────────── heure (0 - 23)
│ │ ┌───────────── jour du mois (1 - 31)
│ │ │ ┌───────────── mois (1 - 12)
│ │ │ │ ┌───────────── jour de la semaine (0 - 6) (dimanche = 0)
│ │ │ │ │
│ │ │ │ │
* * * * *

Exemples :
"0 2 * * *"     = Tous les jours à 2h00
"*/30 * * * *"  = Toutes les 30 minutes
"0 0 * * 0"     = Chaque dimanche à minuit
"0 8-18 * * 1-5" = Toutes les heures de 8h à 18h en semaine
```

### Gérer les schedules

```bash
# Lister les schedules
velero schedule get

# Détails d'un schedule
velero schedule describe daily-backup

# Suspendre un schedule
velero schedule pause daily-backup

# Reprendre un schedule
velero schedule unpause daily-backup

# Supprimer un schedule
velero schedule delete weekly-backup
```

## Restauration avec Velero

### Restauration complète

```bash
# Restaurer depuis la dernière sauvegarde
velero restore create --from-backup full-backup-20240315-143022

# Restaurer avec un nom personnalisé
velero restore create my-restore-$(date +%Y%m%d) \
  --from-backup full-backup-20240315-143022
```

### Restauration sélective

```bash
# Restaurer uniquement un namespace
velero restore create restore-app \
  --from-backup full-backup-20240315-143022 \
  --include-namespaces my-app

# Restaurer des types de ressources spécifiques
velero restore create restore-configs \
  --from-backup full-backup-20240315-143022 \
  --include-resources configmap,secret

# Restaurer avec nouveau namespace
velero restore create restore-to-dev \
  --from-backup prod-backup \
  --namespace-mappings prod:dev
```

### Gérer les conflits lors de la restauration

```bash
# Comportement par défaut : ne pas écraser les objets existants
velero restore create careful-restore \
  --from-backup full-backup-20240315-143022

# Forcer le remplacement des objets existants
velero restore create replace-restore \
  --from-backup full-backup-20240315-143022 \
  --existing-resource-policy update
```

### Vérifier une restauration

```bash
# Lister les restaurations
velero restore get

# Détails d'une restauration
velero restore describe my-restore-20240315

# Logs d'une restauration
velero restore logs my-restore-20240315

# Vérifier les objets restaurés
microk8s kubectl get all --all-namespaces
```

## Gestion des volumes avec Velero

### Configuration pour les volumes MicroK8s

MicroK8s utilise par défaut hostPath pour le stockage. Pour sauvegarder les données :

```bash
# Option 1 : File System Backup (Restic/Kopia)
# Installer Velero avec support Restic
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --use-node-agent \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio:9000

# Annoter les pods pour backup Restic
microk8s kubectl annotate pod -n my-app my-pod backup.velero.io/backup-volumes=data-volume
```

### Backup de volumes avec Restic

```yaml
# Exemple de pod avec annotation pour backup Restic
apiVersion: v1
kind: Pod
metadata:
  name: database
  namespace: my-app
  annotations:
    backup.velero.io/backup-volumes: postgres-data,postgres-config
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: postgres-data
      mountPath: /var/lib/postgresql/data
    - name: postgres-config
      mountPath: /etc/postgresql
  volumes:
  - name: postgres-data
    persistentVolumeClaim:
      claimName: postgres-pvc
  - name: postgres-config
    configMap:
      name: postgres-config
```

### Vérifier les backups de volumes

```bash
# Voir les volumes sauvegardés
velero backup describe my-backup --details | grep -A 10 "Restic Backups"

# Progression du backup Restic
microk8s kubectl logs -n velero -l name=node-agent
```

## Configuration avancée

### Personnalisation des ressources à sauvegarder

```yaml
# velero-backup-config.yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: custom-backup
  namespace: velero
spec:
  # Inclure ces namespaces
  includedNamespaces:
  - my-app
  - database

  # Exclure ces namespaces
  excludedNamespaces:
  - temp
  - test

  # Inclure uniquement ces types de ressources
  includedResources:
  - deployments
  - services
  - configmaps
  - secrets
  - persistentvolumeclaims

  # Exclure ces types de ressources
  excludedResources:
  - events
  - pods

  # Sélecteur de labels
  labelSelector:
    matchLabels:
      backup: "true"
      environment: "production"

  # TTL (Time To Live)
  ttl: 720h0m0s  # 30 jours

  # Hooks (commandes avant/après)
  hooks:
    resources:
    - name: database-dump
      includedNamespaces:
      - database
      labelSelector:
        matchLabels:
          app: postgres
      pre:
      - exec:
          container: postgres
          command:
          - /bin/bash
          - -c
          - pg_dumpall > /backup/dump.sql
          onError: Fail
          timeout: 30s
```

### Politiques de rétention

```bash
# Créer un schedule avec politique de rétention
velero schedule create tiered-backup \
  --schedule="0 */6 * * *" \
  --ttl 24h0m0s

# Script pour rétention personnalisée
#!/bin/bash
# Garder : 7 quotidiennes, 4 hebdomadaires, 12 mensuelles

# Supprimer les backups de plus de 7 jours sauf hebdomadaires
velero backup get -o json | jq -r '.items[] |
  select(.metadata.creationTimestamp < (now - 7*86400 | strftime("%Y-%m-%dT%H:%M:%SZ"))) |
  select(.metadata.labels.type != "weekly") |
  .metadata.name' | xargs -I {} velero backup delete {} --confirm
```

### Monitoring de Velero

```yaml
# prometheus-metrics.yaml
apiVersion: v1
kind: Service
metadata:
  name: velero-metrics
  namespace: velero
  labels:
    app.kubernetes.io/name: velero
spec:
  ports:
  - name: metrics
    port: 8085
    targetPort: 8085
  selector:
    component: velero

---
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: velero
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

## Stratégies et bonnes pratiques

### Organisation des sauvegardes

```
Stratégie recommandée pour un lab :

1. Backups automatiques :
   - Quotidien complet : 2h du matin, rétention 7 jours
   - Hebdomadaire : Dimanche 3h, rétention 30 jours
   - Mensuel : 1er du mois 4h, rétention 1 an

2. Backups manuels :
   - Avant chaque modification majeure
   - Après configuration réussie
   - Pour les tests de restauration

3. Naming convention :
   - auto-daily-YYYYMMDD
   - auto-weekly-YYYYWW
   - auto-monthly-YYYYMM
   - manual-[description]-YYYYMMDD-HHMMSS
```

### Tests de restauration

```bash
# Script de test de restauration
#!/bin/bash

# 1. Créer un namespace de test
microk8s kubectl create namespace restore-test

# 2. Restaurer dans le namespace de test
velero restore create test-restore-$(date +%Y%m%d) \
  --from-backup daily-backup-20240315 \
  --namespace-mappings production:restore-test

# 3. Vérifier la restauration
microk8s kubectl get all -n restore-test

# 4. Nettoyer
microk8s kubectl delete namespace restore-test
velero restore delete test-restore-$(date +%Y%m%d)
```

### Optimisation du stockage

```bash
# Nettoyer les vieux backups
velero backup delete --all --confirm \
  --selector 'velero.io/backup-ttl-expired=true'

# Compacter le stockage Minio
mc admin heal -r myminio/velero-backups

# Analyser l'utilisation
mc du myminio/velero-backups
```

## Dépannage courant

### Problèmes fréquents et solutions

**Backup bloqué en "InProgress"**
```bash
# Vérifier les logs
velero backup logs stuck-backup
microk8s kubectl logs -n velero deployment/velero

# Forcer l'annulation si nécessaire
velero backup delete stuck-backup
```

**Erreur de connexion au stockage**
```bash
# Vérifier la configuration
velero backup-location get -o yaml

# Tester la connexion Minio
mc ls myminio/velero-backups

# Recréer les credentials
velero backup-location update default \
  --credential=cloud-credentials=./credentials-velero
```

**Restauration incomplète**
```bash
# Vérifier les erreurs
velero restore describe incomplete-restore --details

# Logs détaillés
velero restore logs incomplete-restore

# Ressources partiellement restaurées
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

**Volumes non sauvegardés**
```bash
# Vérifier que Restic est actif
microk8s kubectl get pods -n velero -l name=node-agent

# Vérifier les annotations
microk8s kubectl get pods -A -o json | \
  jq '.items[] | select(.metadata.annotations."backup.velero.io/backup-volumes" != null) |
  {namespace: .metadata.namespace, name: .metadata.name, volumes: .metadata.annotations."backup.velero.io/backup-volumes"}'
```

## Migration entre clusters

Velero excelle pour migrer des applications entre clusters :

```bash
# Sur le cluster source
velero backup create migration-backup \
  --include-namespaces app1,app2,app3

# Copier vers le nouveau cluster (si stockage différent)
mc mirror myminio/velero-backups s3/new-cluster-backups

# Sur le cluster destination
velero backup-location create new-location \
  --provider aws \
  --bucket new-cluster-backups \
  --config region=us-east-1

# Restaurer
velero restore create migration-restore \
  --from-backup migration-backup
```

## Intégration avec CI/CD

### GitOps avec Velero

```yaml
# .gitlab-ci.yml exemple
backup-before-deploy:
  stage: pre-deploy
  script:
    - velero backup create pre-deploy-$(date +%Y%m%d-%H%M%S)
        --include-namespaces production
        --wait
  only:
    - main

rollback-on-failure:
  stage: rollback
  when: on_failure
  script:
    - LAST_BACKUP=$(velero backup get -o json | jq -r '.items[0].metadata.name')
    - velero restore create rollback-$(date +%Y%m%d-%H%M%S)
        --from-backup $LAST_BACKUP
  only:
    - main
```

## Checklist de mise en production

Avant de considérer Velero comme opérationnel :

- [ ] Installation validée (pods running, connexion stockage OK)
- [ ] Premier backup manuel réussi
- [ ] Première restauration testée
- [ ] Schedules configurés (daily, weekly, monthly)
- [ ] Politique de rétention définie
- [ ] Monitoring des métriques Velero
- [ ] Alerting sur échec de backup
- [ ] Documentation des procédures
- [ ] Test de restauration mensuel planifié
- [ ] Stockage externe configuré (pas seulement local)

## Conclusion

Velero transforme la gestion des sauvegardes Kubernetes d'une tâche complexe en processus simple et automatisé. Pour votre lab MicroK8s, commencez avec une configuration basique (backup quotidien, stockage Minio local), puis évoluez progressivement vers des stratégies plus sophistiquées (multiples schedules, stockage cloud, hooks personnalisés).

La clé du succès avec Velero est la régularité : des sauvegardes automatiques fréquentes et des tests de restauration réguliers. Avec cette approche, votre lab MicroK8s sera protégé contre les incidents tout en vous permettant d'expérimenter en toute confiance.

⏭️
