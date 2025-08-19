🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12. Sauvegarde et Restauration

## Vue d'ensemble

Dans un environnement Kubernetes, même pour un lab personnel, la sauvegarde et la restauration sont des aspects critiques souvent négligés jusqu'au premier incident. Cette section vous guidera à travers les concepts essentiels et les outils pratiques pour protéger votre travail et vos configurations dans MicroK8s.

## Pourquoi sauvegarder un lab personnel ?

Bien qu'il s'agisse d'un environnement de test, votre lab MicroK8s représente probablement des heures de configuration, d'apprentissage et d'expérimentation. Les raisons principales de mettre en place une stratégie de sauvegarde incluent :

- **Protection de l'investissement en temps** : Vos configurations personnalisées, manifestes YAML, et ajustements représentent un travail considérable
- **Environnement de référence** : Votre lab peut servir de modèle pour des déploiements futurs
- **Continuité d'apprentissage** : Éviter de perdre des configurations complexes qui servent à votre formation
- **Tests de disaster recovery** : Pratiquer les procédures de restauration dans un environnement sans risque
- **Documentation vivante** : Vos configurations sont souvent votre meilleure documentation

## Que faut-il sauvegarder dans Kubernetes ?

Contrairement aux sauvegardes traditionnelles, Kubernetes présente des défis uniques car l'état du cluster est distribué entre plusieurs composants :

### État du cluster (etcd)
Le composant etcd stocke l'état complet du cluster Kubernetes, incluant :
- Tous les objets Kubernetes (Deployments, Services, ConfigMaps, Secrets)
- Configuration du cluster
- État des ressources
- Métadonnées

### Volumes persistants
Les données des applications stateful :
- Bases de données
- Fichiers uploadés
- État des applications
- Logs persistants

### Configuration et manifestes
Les éléments déclaratifs de votre infrastructure :
- Fichiers YAML de déploiement
- Helm charts et values
- Scripts de configuration
- Certificats SSL personnalisés

### Images Docker
Bien que reconstructibles, sauvegarder vos images personnalisées peut économiser du temps :
- Images construites localement
- Images modifiées du registry privé
- Versions spécifiques non disponibles publiquement

## Défis spécifiques à MicroK8s

MicroK8s, en tant que distribution Kubernetes légère, présente quelques particularités pour les sauvegardes :

### Architecture snap
- MicroK8s s'exécute comme un snap, avec ses données dans `/var/snap/microk8s/`
- Les snapshots snap peuvent capturer l'état mais ne sont pas granulaires
- Nécessite une approche hybride pour une protection complète

### Stockage local par défaut
- Le stockage hostpath est simple mais lié au nœud
- Pas de réplication native des données
- Backup des volumes nécessite un accès direct au filesystem

### Addons et leur état
- Chaque addon MicroK8s peut avoir son propre état
- Dashboard, registry, et autres addons stockent des configurations
- Coordination nécessaire entre les sauvegardes d'addons

## Types de sauvegardes dans Kubernetes

### Sauvegardes au niveau de l'infrastructure
Capture de l'ensemble de la VM ou du serveur physique :
- **Avantages** : Simple, capture tout, restauration rapide
- **Inconvénients** : Gourmand en espace, pas granulaire, downtime potentiel

### Sauvegardes au niveau Kubernetes
Utilisation d'outils spécialisés Kubernetes :
- **Avantages** : Granulaire, portable entre clusters, cohérent avec Kubernetes
- **Inconvénients** : Complexité supplémentaire, courbe d'apprentissage

### Sauvegardes au niveau application
Backup des données applicatives directement :
- **Avantages** : Contrôle précis, optimisé pour l'application
- **Inconvénients** : Spécifique à chaque application, maintenance multiple

## Outils de sauvegarde pour MicroK8s

### Velero
L'outil de référence pour les sauvegardes Kubernetes :
- Supporte les sauvegardes programmées
- Snapshot des volumes persistants
- Migration entre clusters
- Restauration sélective

### Solutions natives
Utilisation des commandes kubectl et scripts :
- Export YAML avec `kubectl get -o yaml`
- Backup etcd manuel
- Scripts bash personnalisés
- Cron jobs pour l'automatisation

### Solutions hybrides
Combinaison d'approches pour une protection maximale :
- Velero pour l'état Kubernetes
- Rsync/Restic pour les volumes
- Git pour les manifestes
- Registry pour les images

## Fréquence et rétention

Pour un lab personnel, l'équilibre entre protection et ressources est crucial :

### Sauvegardes quotidiennes
- Configuration du cluster (légère)
- Changements de manifestes
- État des expérimentations en cours

### Sauvegardes hebdomadaires
- Volumes persistants complets
- Images du registry privé
- Snapshots etcd

### Sauvegardes avant changements majeurs
- Mise à jour de MicroK8s
- Installation de nouveaux addons
- Modifications d'architecture
- Tests destructifs

## Stockage des sauvegardes

### Local
- Disque externe USB
- NAS personnel
- Partition séparée
- **Attention** : Protège uniquement contre les erreurs logiques

### Cloud
- Object storage (S3, GCS, Azure Blob)
- Services de backup cloud
- Repos Git pour les configurations
- **Avantage** : Protection contre les sinistres physiques

### Hybride recommandé
- Sauvegardes fréquentes en local (rapidité)
- Sauvegardes périodiques dans le cloud (sécurité)
- Manifestes dans Git (versioning)

## Tests de restauration

Un aspect souvent négligé mais critique :

### Importance des tests
- "Une sauvegarde non testée n'est pas une sauvegarde"
- Validation de la procédure complète
- Identification des données manquantes
- Formation pratique sans stress

### Environnement de test
Pour un lab, créer un second environnement de test :
- VM temporaire pour les tests
- Namespace dédié aux tests de restauration
- Cluster MicroK8s secondaire minimal

### Métriques de restauration
Mesurer et documenter :
- RTO (Recovery Time Objective) : Temps de restauration
- RPO (Recovery Point Objective) : Perte de données acceptable
- Complexité de la procédure
- Prérequis et dépendances

## Considérations de sécurité

### Chiffrement
- Secrets Kubernetes en clair dans etcd
- Nécessité de chiffrer les backups
- Gestion des clés de chiffrement

### Accès aux sauvegardes
- Permissions restrictives
- Audit trail des restaurations
- Isolation des backups du cluster principal

### Conformité pour un lab
Même en lab, adopter les bonnes pratiques :
- RGPD si données personnelles
- Rétention appropriée
- Suppression sécurisée

## Meilleures pratiques pour un lab personnel

### Automatisation pragmatique
- Scripts simples mais robustes
- Notifications en cas d'échec
- Logs détaillés des opérations
- Documentation inline dans les scripts

### Documentation
Maintenir un runbook simple :
- Procédure de backup
- Procédure de restauration
- Localisation des sauvegardes
- Contacts et mots de passe (sécurisés)

### Approche progressive
Commencer simple et améliorer :
1. Backup manuel des manifestes critiques
2. Script automatisé quotidien
3. Ajout de Velero pour l'état complet
4. Intégration cloud pour l'offsite
5. Tests de restauration réguliers

### Monitoring des sauvegardes
Même basique, surveiller :
- Succès/échec des jobs
- Espace disque utilisé
- Âge de la dernière sauvegarde
- Alerting simple (email, Discord, Slack)

## Cas d'usage réels pour un lab

### Retour arrière après expérimentation
- Test d'une nouvelle application qui casse le cluster
- Mauvaise configuration réseau
- Suppression accidentelle de ressources

### Migration de matériel
- Passage à un nouveau serveur
- Migration VM vers bare metal
- Changement de distribution Linux

### Partage de configuration
- Reproduction de l'environnement
- Aide à la communauté
- Portfolio de configurations

### Apprentissage du disaster recovery
- Simulation de pannes
- Formation aux procédures d'urgence
- Test de résilience

## Prochaines étapes

Les sections suivantes détailleront la mise en œuvre pratique de ces concepts, en commençant par les stratégies de sauvegarde adaptées à différents scénarios d'utilisation d'un lab MicroK8s. Vous apprendrez à choisir la bonne approche selon vos besoins, implémenter des solutions concrètes, et surtout, valider que vos données sont réellement récupérables en cas de besoin.

⏭️
