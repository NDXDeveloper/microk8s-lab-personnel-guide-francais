🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1 Stratégies de sauvegarde

## Introduction aux stratégies

Une stratégie de sauvegarde est votre plan d'action pour protéger les données et configurations de votre cluster MicroK8s. Comme pour choisir une assurance, il n'existe pas de solution unique : votre stratégie dépendra de vos besoins, ressources et tolérance au risque. Cette section vous aidera à concevoir une approche adaptée à votre lab personnel.

## Les trois piliers d'une stratégie de sauvegarde

### 1. Complétude (Que sauvegarder ?)

La première décision concerne l'étendue de vos sauvegardes. Pensez à votre cluster comme une maison : voulez-vous sauvegarder toute la maison, juste les meubles, ou seulement les objets de valeur ?

**Approche minimaliste**
Sauvegarde uniquement des éléments critiques :
- Manifestes YAML personnalisés
- Données des bases de données
- Configurations spécifiques
- **Avantage** : Rapide, peu d'espace disque
- **Inconvénient** : Restauration plus complexe

**Approche sélective**
Sauvegarde des ressources Kubernetes importantes :
- Deployments et Services
- ConfigMaps et Secrets
- PersistentVolumes avec données
- Ingress et certificats
- **Avantage** : Bon équilibre efficacité/protection
- **Inconvénient** : Nécessite de bien identifier les dépendances

**Approche exhaustive**
Sauvegarde complète du cluster :
- État complet d'etcd
- Tous les volumes persistants
- Configuration MicroK8s
- Images du registry local
- **Avantage** : Restauration simple et complète
- **Inconvénient** : Consomme beaucoup d'espace

### 2. Fréquence (Quand sauvegarder ?)

La fréquence détermine votre RPO (Recovery Point Objective) - combien de données vous acceptez de perdre.

**Sauvegarde continue**
Pour les données très dynamiques :
```yaml
# Exemple de concept - sauvegarde toutes les heures
Schedule: "0 * * * *"
Éléments: ConfigMaps, Secrets critiques
Méthode: Synchronisation incrémentale
```

**Sauvegarde quotidienne**
Standard pour la plupart des labs :
```yaml
# Sauvegarde chaque nuit à 2h
Schedule: "0 2 * * *"
Éléments: État du cluster, volumes actifs
Méthode: Snapshot complet ou incrémental
```

**Sauvegarde hebdomadaire**
Pour les environnements stables :
```yaml
# Sauvegarde chaque dimanche
Schedule: "0 3 * * 0"
Éléments: Backup complet
Méthode: Full backup avec rotation
```

**Sauvegarde événementielle**
Déclenchée par des actions spécifiques :
- Avant une mise à jour majeure
- Après un déploiement réussi
- Avant des tests destructifs
- Suite à des modifications importantes

### 3. Rétention (Combien de temps garder ?)

La politique de rétention équilibre l'historique disponible avec l'espace de stockage.

**Schéma GFS (Grandfather-Father-Son)**
Approche classique et éprouvée :
```
Quotidiennes : 7 derniers jours
Hebdomadaires : 4 dernières semaines
Mensuelles : 12 derniers mois
```

**Rétention progressive**
Plus simple pour un lab :
```
Récentes : Toutes les sauvegardes des 7 derniers jours
Moyennes : Une par semaine pour le dernier mois
Anciennes : Une par mois au-delà
```

## Stratégies par type de données

### Configuration as Code (GitOps)

Pour les manifestes et configurations, Git est votre meilleur ami :

**Structure recommandée pour votre repository :**
```
my-k8s-lab/
├── manifests/
│   ├── apps/
│   │   ├── wordpress/
│   │   ├── nextcloud/
│   │   └── prometheus/
│   ├── infrastructure/
│   │   ├── ingress/
│   │   ├── storage/
│   │   └── networking/
│   └── secrets/  # Chiffrés avec Sealed Secrets ou SOPS
├── helm/
│   ├── charts/
│   └── values/
├── scripts/
│   ├── backup/
│   └── restore/
└── docs/
    └── runbooks/
```

**Workflow Git pour les configurations :**
1. Toute modification passe par Git
2. Commit avec messages descriptifs
3. Tags pour les versions stables
4. Branches pour les expérimentations

### État du cluster (etcd)

L'état d'etcd contient toute la configuration runtime de Kubernetes.

**Stratégie snapshot etcd :**
```bash
# Localisation dans MicroK8s
/var/snap/microk8s/current/var/kubernetes/backend/

# Stratégie recommandée
- Snapshot quotidien automatique
- Conservation 7 jours en local
- Archive hebdomadaire sur stockage externe
- Test de restauration mensuel
```

**Points d'attention pour etcd :**
- Les snapshots etcd contiennent les Secrets en clair
- Chiffrer les backups avant stockage externe
- Taille généralement petite (quelques MB)
- Critique pour la restauration complète

### Volumes persistants

Les données stateful nécessitent une approche différenciée :

**Classification des volumes :**
```yaml
Critiques:
  - Bases de données de production
  - Données utilisateur
  - État des applications stateful
  Stratégie: Backup quotidien, réplication si possible

Importants:
  - Logs applicatifs
  - Métriques Prometheus
  - Caches reconstruisibles
  Stratégie: Backup hebdomadaire

Éphémères:
  - Données de test
  - Caches temporaires
  - Volumes de build
  Stratégie: Pas de backup
```

**Méthodes de sauvegarde des volumes :**

1. **Snapshot au niveau filesystem**
   ```bash
   # Pour LVM
   lvcreate -L 1G -s -n snapshot_pv /dev/vg/original_pv

   # Pour ZFS
   zfs snapshot pool/volume@backup_$(date +%Y%m%d)
   ```

2. **Backup au niveau application**
   ```bash
   # Pour PostgreSQL
   pg_dump database > backup.sql

   # Pour MySQL
   mysqldump --all-databases > backup.sql

   # Pour MongoDB
   mongodump --out /backup/
   ```

3. **Copie directe des fichiers**
   ```bash
   # Simple mais nécessite l'arrêt de l'application
   kubectl scale deployment/app --replicas=0
   tar czf backup.tar.gz /var/snap/microk8s/common/default-storage/
   kubectl scale deployment/app --replicas=1
   ```

### Images Docker

**Stratégie pour le registry local :**

```yaml
Registry privé MicroK8s:
  Emplacement: localhost:32000

  Images custom:
    - Backup: Export vers tar ou push vers registry externe
    - Fréquence: À chaque build réussi
    - Rétention: 3 dernières versions

  Images publiques modifiées:
    - Backup: Dockerfile + layers modifiés
    - Fréquence: À la modification
    - Rétention: Permanent dans Git
```

## Stratégies selon le profil d'usage

### Lab de développement

**Caractéristiques :**
- Changements fréquents
- Données peu critiques
- Besoin de rollback rapide

**Stratégie recommandée :**
```yaml
Configuration:
  - Git pour tous les manifestes
  - Commit à chaque changement

État:
  - Snapshot etcd quotidien
  - Rétention 7 jours

Données:
  - Backup des BDD de dev hebdomadaire
  - Scripts de réinitialisation disponibles

Méthode:
  - Scripts bash simples
  - Stockage local suffisant
```

### Lab de formation/certification

**Caractéristiques :**
- Environnement stable
- Scénarios reproductibles
- Besoin de reset rapide

**Stratégie recommandée :**
```yaml
Configuration:
  - Templates de labs dans Git
  - Tags pour chaque exercice

État:
  - Snapshot avant chaque session
  - Points de restauration par chapitre

Données:
  - Datasets d'exemple versionnés
  - Pas de backup des données d'exercice

Méthode:
  - Velero pour snapshots rapides
  - Scripts de reset automatisés
```

### Services personnels

**Caractéristiques :**
- Données personnelles importantes
- Disponibilité souhaitée
- Évolution lente

**Stratégie recommandée :**
```yaml
Configuration:
  - Git avec branches stable/test
  - Backup avant chaque modification

État:
  - Snapshot etcd quotidien
  - Réplication vers backup externe

Données:
  - Backup quotidien des données utilisateur
  - Sync vers cloud (chiffré)
  - Test de restauration mensuel

Méthode:
  - Velero + Restic pour volumes
  - Notification en cas d'échec
```

## Matrice de décision

Pour choisir votre stratégie, évaluez chaque composant :

| Composant | Criticité | Fréquence changement | Taille | Stratégie suggérée |
|-----------|-----------|---------------------|--------|-------------------|
| Manifestes YAML | Haute | Élevée | Petite | Git + commit fréquents |
| État etcd | Haute | Moyenne | Petite | Snapshot quotidien |
| BDD de prod | Haute | Élevée | Variable | Backup applicatif quotidien |
| Logs | Basse | Élevée | Grande | Rotation, pas de backup |
| Images custom | Moyenne | Basse | Moyenne | Registry externe + Dockerfile |
| Certificats | Haute | Basse | Petite | Git (chiffré) + backup |

## Implémentation par phases

### Phase 1 : Protection minimale (Semaine 1)
```bash
# Script backup basique
#!/bin/bash
BACKUP_DIR="/backup/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Export des ressources critiques
kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/all-resources.yaml
kubectl get pv,pvc --all-namespaces -o yaml > $BACKUP_DIR/storage.yaml
kubectl get ingress,secrets --all-namespaces -o yaml > $BACKUP_DIR/ingress-secrets.yaml

# Compression
tar czf $BACKUP_DIR.tar.gz $BACKUP_DIR/
rm -rf $BACKUP_DIR/
```

### Phase 2 : Automatisation (Semaine 2-3)
```bash
# Crontab pour automatisation
# Backup quotidien à 2h du matin
0 2 * * * /home/user/scripts/backup-microk8s.sh

# Rotation automatique
find /backup -name "*.tar.gz" -mtime +7 -delete
```

### Phase 3 : Backup avancé (Mois 2)
- Installation de Velero
- Configuration du stockage S3/Minio
- Snapshots de volumes
- Politiques de rétention

### Phase 4 : Haute disponibilité (Mois 3+)
- Réplication vers site distant
- Monitoring des backups
- Tests de restauration automatisés
- Documentation et runbooks

## Métriques de succès

Pour valider votre stratégie :

**RTO (Recovery Time Objective)**
- Lab de dev : < 1 heure
- Services personnels : < 30 minutes
- Formation : < 15 minutes

**RPO (Recovery Point Objective)**
- Configurations : 0 (avec Git)
- État du cluster : < 24 heures
- Données utilisateur : < 4 heures

**Taux de succès**
- Backup réussis : > 99%
- Tests de restauration : 100% de succès
- Temps de détection d'échec : < 1 heure

## Erreurs courantes à éviter

### 1. Oublier les dépendances
**Problème :** Sauvegarder un Deployment sans ses ConfigMaps
**Solution :** Toujours identifier et documenter les dépendances

### 2. Ne pas tester la restauration
**Problème :** Découvrir que les backups sont corrompus lors d'un incident
**Solution :** Test mensuel obligatoire avec documentation

### 3. Stocker les backups au même endroit
**Problème :** Perte du serveur = perte des backups
**Solution :** Règle 3-2-1 (3 copies, 2 médias différents, 1 hors site)

### 4. Ignorer la sécurité
**Problème :** Secrets en clair dans les backups
**Solution :** Chiffrement systématique des backups sensibles

### 5. Sur-complexifier
**Problème :** Système trop complexe, difficile à maintenir
**Solution :** Commencer simple, évoluer progressivement

## Checklist de validation

Avant de considérer votre stratégie comme complète :

- [ ] Tous les composants critiques sont identifiés
- [ ] Fréquence de backup définie pour chaque composant
- [ ] Politique de rétention documentée
- [ ] Stockage externe configuré
- [ ] Scripts de backup testés
- [ ] Procédure de restauration documentée
- [ ] Test de restauration réussi
- [ ] Monitoring des backups en place
- [ ] Notification d'échec configurée
- [ ] Documentation accessible en cas d'urgence

## Conclusion

Une bonne stratégie de sauvegarde pour votre lab MicroK8s n'est pas celle qui sauvegarde tout, mais celle qui protège ce qui compte vraiment pour vous, de manière fiable et sans complexité excessive. Commencez simple avec les éléments critiques, puis enrichissez progressivement votre stratégie en fonction de vos besoins réels et de votre expérience.

La section suivante détaillera l'implémentation pratique avec Velero, l'outil de référence pour les sauvegardes Kubernetes.

⏭️
