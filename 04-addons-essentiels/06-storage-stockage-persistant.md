🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6 Storage (stockage persistant)

## Introduction au stockage persistant dans Kubernetes

Le stockage persistant représente l'un des défis fondamentaux dans l'orchestration de conteneurs, transformant des applications stateless éphémères en systèmes capables de maintenir des données critiques au-delà de la vie d'un pod. Dans Kubernetes, les conteneurs sont par nature éphémères : ils peuvent être créés, détruits, et recréés à tout moment. Sans stockage persistant, toutes les données écrites dans le système de fichiers d'un conteneur disparaissent lorsque ce conteneur s'arrête. Pour un lab personnel exécutant des bases de données, des applications stateful, ou simplement conservant des configurations importantes, le stockage persistant n'est pas optionnel mais essentiel.

MicroK8s simplifie considérablement la complexité traditionnelle du stockage Kubernetes en fournissant un addon storage préconfiguré qui fonctionne immédiatement. Cette approche élimine les heures de configuration manuelle typiquement nécessaires pour mettre en place un système de stockage fonctionnel dans Kubernetes. Le stockage dans MicroK8s utilise une implémentation locale optimisée pour les environnements single-node ou small-scale, parfaite pour un lab personnel où la simplicité et la performance locale priment sur la distribution géographique.

La beauté du système de stockage Kubernetes réside dans son abstraction. Les développeurs déclarent leurs besoins en stockage (taille, performance, mode d'accès) sans se soucier de l'implémentation sous-jacente. Cette séparation des préoccupations permet de développer des applications portables qui fonctionnent identiquement sur votre lab MicroK8s local et sur un cluster de production dans le cloud, changeant simplement la StorageClass utilisée.

## Concepts fondamentaux du stockage Kubernetes

Comprendre les concepts de base du stockage Kubernetes est crucial pour utiliser efficacement le stockage persistant dans votre lab MicroK8s. Ces abstractions peuvent sembler complexes initialement, mais elles offrent une flexibilité et une puissance remarquables une fois maîtrisées.

Les Volumes constituent le niveau le plus basique du stockage dans Kubernetes. Un Volume est essentiellement un répertoire accessible aux conteneurs dans un pod, avec une durée de vie liée à celle du pod. Les Volumes résolvent le problème du partage de données entre conteneurs dans le même pod et survivent aux redémarrages de conteneurs, mais pas à la suppression du pod. Il existe de nombreux types de Volumes, depuis les simples emptyDir temporaires jusqu'aux volumes cloud sophistiqués.

Les PersistentVolumes (PV) élèvent le concept de stockage au niveau du cluster. Un PV est une ressource de stockage dans le cluster, provisionnée par un administrateur ou dynamiquement créée via une StorageClass. Contrairement aux Volumes simples, les PV existent indépendamment de tout pod et survivent aux cycles de vie des applications. Pensez aux PV comme des disques durs virtuels disponibles dans votre cluster, prêts à être montés par les applications qui en ont besoin.

Les PersistentVolumeClaims (PVC) représentent une demande de stockage par un utilisateur. Un PVC est analogue à un pod : comme un pod consomme des ressources CPU et mémoire, un PVC consomme des ressources de stockage. Les PVC spécifient la taille désirée et le mode d'accès (ReadWriteOnce, ReadOnlyMany, ReadWriteMany), et Kubernetes trouve automatiquement un PV correspondant ou en crée un nouveau si le provisionnement dynamique est configuré. Cette indirection permet aux développeurs de demander du stockage sans connaître les détails de l'infrastructure sous-jacente.

Les StorageClasses définissent différentes "classes" ou "qualités" de stockage disponibles dans votre cluster. Une StorageClass peut représenter du stockage SSD rapide, du stockage HDD économique, ou du stockage répliqué haute disponibilité. Dans MicroK8s, la StorageClass par défaut utilise le stockage local de votre machine, mais vous pouvez définir des classes supplémentaires pour différents besoins. Le provisionnement dynamique via StorageClasses élimine le besoin de créer manuellement des PV, Kubernetes les créant automatiquement lorsqu'un PVC les demande.

## Architecture du storage addon MicroK8s

L'addon storage de MicroK8s implémente une solution de stockage élégante basée sur hostpath-provisioner, spécialement conçue pour les environnements de développement et les labs personnels où la simplicité prime sur la complexité distribuée.

Le hostpath-provisioner est un contrôleur Kubernetes qui surveille les PVC et crée dynamiquement des PV utilisant le stockage local de l'hôte. Lorsqu'un PVC est créé, le provisioner alloue automatiquement un répertoire sur le système de fichiers de l'hôte et crée un PV correspondant. Cette approche élimine complètement la gestion manuelle des volumes, offrant une expérience similaire aux solutions de stockage cloud mais utilisant simplement votre disque local.

Le stockage physique est organisé dans un répertoire dédié sur votre système, typiquement `/var/snap/microk8s/common/default-storage` pour MicroK8s. Chaque PVC obtient son propre sous-répertoire, isolant les données entre applications. Cette structure simple facilite les backups, les inspections manuelles, et la compréhension de l'utilisation du stockage. La performance est excellente car il n'y a pas de couche réseau ou de protocole distribué, les I/O allant directement vers votre disque local.

La StorageClass par défaut configurée par MicroK8s spécifie les paramètres du hostpath-provisioner, incluant le répertoire de base pour le stockage et les options de reclaim policy. La politique par défaut est "Delete", signifiant que les données sont supprimées quand le PVC est supprimé, approprié pour le développement mais à considérer pour les données importantes. Vous pouvez créer des StorageClasses supplémentaires avec différentes politiques ou pointant vers différents répertoires, permettant de séparer les données de développement et de production même dans un lab personnel.

L'intégration avec le système de quotas Kubernetes permet de limiter la consommation de stockage par namespace ou par utilisateur. Bien que souvent ignoré dans les labs personnels, cette capacité devient précieuse quand vous expérimentez avec des applications gourmandes en stockage ou simulez des environnements multi-tenant. Les ResourceQuotas peuvent limiter le nombre total de PVC, la taille totale du stockage demandé, ou même spécifier des limites par StorageClass.

## Installation et configuration

L'activation du storage addon dans MicroK8s est remarquablement simple, mais comprendre les options disponibles permet d'adapter la configuration à vos besoins spécifiques.

La commande basique `microk8s enable storage` active l'addon avec les paramètres par défaut, suffisants pour la plupart des cas d'usage. Cette commande déploie le hostpath-provisioner, crée la StorageClass par défaut, et configure tous les RBAC nécessaires. Le processus prend généralement quelques secondes, après quoi vous pouvez immédiatement commencer à créer des PVC.

La personnalisation du chemin de stockage peut être nécessaire si votre partition système est limitée en espace ou si vous préférez utiliser un disque dédié. Bien que l'addon n'offre pas de paramètre direct pour changer le chemin, vous pouvez modifier la ConfigMap du hostpath-provisioner après installation pour pointer vers un répertoire différent. Cette flexibilité permet d'utiliser des SSD pour les applications nécessitant des performances élevées ou des disques plus grands pour les données volumineuses.

La configuration de StorageClasses supplémentaires enrichit votre environnement avec différents tiers de stockage. Vous pouvez créer une StorageClass "fast" pointant vers un SSD, une classe "archive" sur un disque lent mais volumineux, ou une classe "replicated" utilisant une solution comme GlusterFS ou Ceph si vous évoluez vers un cluster multi-node. Chaque StorageClass peut avoir des paramètres différents, des reclaim policies distinctes, et même utiliser des provisioners complètement différents.

La vérification de l'installation s'effectue en examinant les composants déployés dans le namespace kube-system. La commande `microk8s kubectl get storageclass` devrait montrer au moins la classe "microk8s-hostpath" marquée comme default. Le pod du hostpath-provisioner devrait être en état Running, visible via `microk8s kubectl get pods -n kube-system | grep hostpath`. Ces vérifications confirment que votre système de stockage est opérationnel et prêt à servir les demandes.

## Création et gestion des PersistentVolumeClaims

Les PersistentVolumeClaims sont votre interface principale avec le système de stockage, transformant les besoins abstraits en stockage concret utilisable par vos applications.

La création d'un PVC commence par définir vos besoins en stockage. La spécification inclut la taille désirée (par exemple 5Gi pour 5 gigaoctets), le mode d'accès (ReadWriteOnce pour un accès exclusif, ReadOnlyMany pour lecture partagée, ou ReadWriteMany pour écriture partagée), et optionnellement la StorageClass à utiliser. Dans MicroK8s avec le hostpath-provisioner, seul ReadWriteOnce est supporté car le stockage est local à un node, mais c'est suffisant pour la majorité des applications stateful.

Le cycle de vie d'un PVC traverse plusieurs phases observables. Après création, le PVC est initialement en état "Pending" pendant que Kubernetes cherche ou provisionne un PV approprié. Avec le provisionnement dynamique de MicroK8s, cette phase est généralement très courte. Une fois lié à un PV, le PVC passe en état "Bound" et peut être monté par des pods. Si aucun PV satisfaisant n'est disponible et que le provisionnement dynamique échoue, le PVC reste "Pending" avec des événements expliquant le problème.

Le redimensionnement des PVC est supporté dans les versions récentes de Kubernetes, permettant d'augmenter la taille d'un volume existant sans recréer l'application. Cette capacité est précieuse quand vos besoins en stockage évoluent. Le processus implique simplement d'éditer le PVC pour augmenter la taille demandée, et selon la StorageClass et le système de fichiers, l'expansion peut se faire en ligne ou nécessiter un redémarrage du pod.

La gestion du cycle de vie des données nécessite de comprendre les reclaim policies. Avec la politique "Delete" par défaut, supprimer un PVC supprime également le PV et les données sous-jacentes. Pour les données importantes, créer des StorageClasses avec la politique "Retain" préserve les données même après suppression du PVC, permettant la récupération ou la migration. Cette distinction est cruciale pour éviter la perte accidentelle de données dans votre lab.

## Utilisation du stockage dans les applications

L'intégration du stockage persistant dans vos applications Kubernetes transforme des conteneurs stateless en systèmes capables de maintenir l'état à travers les redémarrages et les mises à jour.

Le montage de volumes dans les pods s'effectue via la section volumes et volumeMounts de la spécification du pod. La section volumes référence le PVC par son nom, créant un volume nommé utilisable dans le pod. Les volumeMounts dans chaque conteneur spécifient où ce volume doit être monté dans le système de fichiers du conteneur. Cette séparation permet à plusieurs conteneurs dans le même pod de partager le même stockage ou de monter différents volumes selon leurs besoins.

Les patterns d'utilisation courants incluent le stockage de données d'application pour les bases de données, les fichiers uploadés par les utilisateurs, ou les caches persistants. Les fichiers de configuration peuvent être stockés de manière persistante, permettant des modifications qui survivent aux redéploiements. Les logs applicatifs peuvent être écrits sur stockage persistant pour analyse ultérieure, bien que des solutions de logging centralisées soient généralement préférables. Les répertoires home pour les environnements de développement ou les notebooks Jupyter bénéficient du stockage persistant pour préserver le travail entre les sessions.

Les considérations de performance deviennent importantes pour les applications I/O intensives. Le stockage local de MicroK8s offre d'excellentes performances car il n'y a pas de latence réseau, mais il partage les I/O avec le système hôte. Pour les applications critiques en performance, considérer l'utilisation de disques SSD dédiés ou la séparation des données sur différents disques physiques. Le tuning du système de fichiers, comme l'utilisation de ext4 avec les bonnes options de montage ou XFS pour les gros fichiers, peut améliorer significativement les performances.

Les stratégies de backup doivent être soigneusement planifiées pour les données importantes. Bien que le stockage local soit pratique, il n'offre aucune protection contre les défaillances matérielles. Des solutions comme Velero peuvent automatiser les backups de PVC vers des stockages externes. Pour un lab personnel, des scripts simples copiant régulièrement le répertoire de stockage vers un NAS ou un service cloud peuvent suffire. L'important est d'avoir une stratégie et de la tester régulièrement.

## StatefulSets et stockage ordonné

Les StatefulSets représentent l'utilisation la plus sophistiquée du stockage dans Kubernetes, gérant des applications nécessitant des identités stables et du stockage persistant ordonné.

Le concept de volumeClaimTemplates dans les StatefulSets automatise la création de PVC pour chaque replica. Contrairement aux Deployments où tous les pods partagent potentiellement le même volume, chaque pod d'un StatefulSet obtient son propre PVC dédié. Cette isolation garantit que les données de chaque instance restent séparées et persistantes, essentielles pour les bases de données distribuées ou les systèmes de messaging.

L'ordre de création et de suppression des volumes suit celui des pods dans un StatefulSet. Le pod-0 et son PVC sont créés en premier, suivis par pod-1, et ainsi de suite. Cette orchestration ordonnée garantit que les systèmes master-slave ou les clusters distribués peuvent s'initialiser correctement. Lors de la suppression, l'ordre inverse est respecté, permettant un shutdown gracieux des systèmes interdépendants.

La migration et le scaling des StatefulSets avec stockage nécessitent une attention particulière. Augmenter le nombre de replicas crée automatiquement de nouveaux PVC pour les nouvelles instances. Réduire les replicas supprime les pods mais, par sécurité, conserve les PVC et leurs données. Cette préservation permet de re-scaler sans perte de données mais nécessite un nettoyage manuel des PVC orphelins si les données ne sont plus nécessaires.

Les patterns de bases de données distribuées illustrent parfaitement l'utilisation des StatefulSets avec stockage. Une base de données PostgreSQL avec réplication peut utiliser un StatefulSet où chaque replica a son propre volume pour les données. Le pod-0 agit comme master avec un volume pour les données primaires, tandis que les replicas ont leurs propres volumes pour les données répliquées. Cette architecture survit aux redémarrages, aux mises à jour, et même aux défaillances de pods individuels.

## Volumes dynamiques vs statiques

La distinction entre provisionnement dynamique et statique influence fondamentalement comment vous gérez le stockage dans votre lab MicroK8s.

Le provisionnement dynamique, utilisé par défaut avec l'addon storage de MicroK8s, crée automatiquement des PV quand des PVC les demandent. Cette approche élimine le travail manuel de pré-création des volumes et garantit que le stockage est toujours disponible à la demande. L'automatisation réduit les erreurs, accélère les déploiements, et simplifie la gestion du stockage. Pour un lab personnel où l'agilité prime, le provisionnement dynamique est généralement optimal.

Le provisionnement statique reste utile pour des cas spécifiques où vous voulez un contrôle précis sur le stockage. Vous pouvez créer manuellement des PV pointant vers des répertoires spécifiques, des disques externes, ou même des partages réseau NFS. Cette approche permet d'intégrer du stockage existant, d'utiliser des dispositifs spéciaux, ou d'implémenter des politiques de stockage personnalisées. Les PV statiques peuvent également être utilisés pour "importer" des données existantes dans Kubernetes.

Les scénarios hybrides combinent les deux approches selon les besoins. Les applications de développement peuvent utiliser le provisionnement dynamique pour la flexibilité, tandis que les bases de données de production utilisent des PV statiques sur des SSD dédiés pour les performances. Cette flexibilité permet d'optimiser chaque workload selon ses caractéristiques sans compromettre la simplicité globale.

La migration entre modes est relativement simple grâce à l'abstraction des PVC. Une application utilisant un PVC ne sait pas si le PV sous-jacent a été provisionné dynamiquement ou statiquement. Cette transparence permet de commencer avec le provisionnement dynamique pour le prototypage puis de migrer vers des volumes statiques optimisés pour la production sans modifier l'application.

## Monitoring et métriques du stockage

La surveillance du stockage est cruciale pour maintenir la santé de votre lab et prévenir les problèmes avant qu'ils n'impactent les applications.

Les métriques au niveau du système de fichiers révèlent l'utilisation réelle du stockage sur votre hôte. Des outils comme `df` et `du` permettent d'examiner l'utilisation du répertoire de stockage MicroK8s. Le monitoring de l'espace disponible prévient les situations où le manque d'espace cause des échecs de pods ou des corruptions de données. Des scripts simples peuvent alerter quand l'utilisation dépasse des seuils définis.

Les métriques Kubernetes offrent une vue logique de l'utilisation du stockage. La commande `kubectl get pv` montre tous les PersistentVolumes avec leur capacité et leur statut. Les PVC peuvent être examinés pour voir leur utilisation demandée versus leur utilisation réelle. Ces métriques aident à identifier les PVC surdimensionnés gaspillant de l'espace ou sous-dimensionnés nécessitant une expansion.

L'intégration avec Prometheus permet un monitoring sophistiqué du stockage. Le kubelet expose des métriques sur l'utilisation des volumes par pod, permettant de créer des dashboards détaillés dans Grafana. Des alertes peuvent être configurées pour l'espace faible, les I/O élevées, ou les erreurs de montage. Cette observabilité proactive transforme la gestion du stockage de réactive à préventive.

Les patterns d'utilisation émergent du monitoring à long terme. Identifier les pics d'utilisation, les tendances de croissance, et les applications gourmandes en I/O informe les décisions d'architecture. Ces insights peuvent révéler des opportunités d'optimisation, comme déplacer les caches vers emptyDir, utiliser des volumes séparés pour les logs, ou implémenter des politiques de rétention des données.

## Troubleshooting des problèmes de stockage

Les problèmes de stockage peuvent être frustrants mais suivent généralement des patterns identifiables avec des solutions standards.

Les erreurs de montage sont parmi les plus courantes, se manifestant par des pods stuck en état "ContainerCreating". Les événements du pod révèlent généralement la cause, qu'il s'agisse de permissions incorrectes, de chemins inexistants, ou de PVC non bound. Vérifier que le PVC est en état Bound, que le PV sous-jacent existe, et que les permissions du répertoire sur l'hôte permettent l'accès par les conteneurs.

Les problèmes de performance se manifestent par des applications lentes ou des timeouts. Examiner les métriques I/O avec `iostat` ou `iotop` sur l'hôte peut révéler si le disque est saturé. Les applications écrivant excessivement peuvent impacter tout le cluster dans un environnement single-node. Considérer l'utilisation de volumes séparés, l'optimisation des patterns d'écriture, ou l'upgrade vers des disques plus rapides.

La corruption de données, bien que rare, peut survenir suite à des arrêts non gracieux ou des problèmes matériels. Les symptômes incluent des applications crashant au démarrage ou des erreurs de lecture/écriture. La vérification du système de fichiers avec `fsck`, la restauration depuis backup, ou dans les cas extrêmes la recréation complète du PVC peuvent être nécessaires. Implémenter des health checks applicatifs détectant la corruption tôt limite les dégâts.

L'épuisement de l'espace disque cause des échecs cascadés difficiles à diagnostiquer. Les pods peuvent crasher, les logs peuvent être perdus, et même le control plane peut devenir instable. Maintenir toujours un buffer d'espace libre, implémenter des quotas, et monitorer proactivement l'utilisation préviennent ces situations critiques. En cas d'urgence, identifier et supprimer les PVC ou les logs volumineux peut restaurer rapidement la stabilité.

## Stratégies de backup et restauration

La protection des données dans votre lab MicroK8s nécessite une stratégie de backup réfléchie adaptée à la valeur et la criticité de vos données.

Les backups au niveau fichier représentent l'approche la plus simple, copiant directement les répertoires de stockage vers une destination sûre. Des outils comme rsync, tar, ou même de simples scripts cp peuvent archiver le répertoire `/var/snap/microk8s/common/default-storage`. Cette méthode est directe et universelle mais nécessite de coordonner les backups avec l'état des applications pour garantir la cohérence.

Les snapshots de volumes, quand supportés par votre système de stockage, offrent des backups instantanés et cohérents. LVM, ZFS, ou Btrfs permettent de créer des snapshots du système de fichiers hébergeant le stockage MicroK8s. Ces snapshots peuvent être créés rapidement sans interrompre les applications, puis exportés ou répliqués vers un stockage de backup. La restauration depuis snapshots est généralement plus rapide que depuis des backups fichier.

Les solutions Kubernetes-native comme Velero comprennent les abstractions Kubernetes et peuvent backuper non seulement les données mais aussi les définitions de PVC, PV, et les métadonnées associées. Cette approche holistique facilite la migration entre clusters ou la restauration après un désastre majeur. Velero peut être configuré pour backuper vers S3, GCS, Azure Blob, ou même MinIO local, offrant flexibilité dans le choix du stockage de backup.

Les tests de restauration sont aussi importants que les backups eux-mêmes. Régulièrement, prendre un backup, le restaurer dans un environnement de test, et vérifier que les applications fonctionnent correctement. Cette pratique révèle les problèmes de backup avant qu'ils ne deviennent critiques et maintient la familiarité avec les procédures de restauration. Documenter le processus de restauration avec des exemples concrets accélère la récupération en cas de vrai problème.

## Évolution vers des solutions avancées

Bien que l'addon storage de MicroK8s soit excellent pour commencer, votre lab peut éventuellement nécessiter des capacités plus avancées.

OpenEBS représente une évolution naturelle, offrant du stockage container-attached distribué avec des fonctionnalités entreprise. OpenEBS peut être déployé sur MicroK8s pour obtenir des capacités comme la réplication synchrone, les snapshots application-aware, et les clones instantanés. La complexité accrue est compensée par la résilience et les fonctionnalités avancées, particulièrement utiles si vous évoluez vers un cluster multi-node.

Longhorn, développé par Rancher, offre une alternative cloud-native avec une interface utilisateur intuitive pour la gestion du stockage. Longhorn automatise les backups, gère la réplication, et offre même disaster recovery cross-cluster. Pour un lab évoluant vers production ou nécessitant haute disponibilité, Longhorn transforme MicroK8s en plateforme storage-entreprise.

Rook orchestrate Ceph dans Kubernetes, apportant le stockage distribué mature de Ceph avec la simplicité de gestion Kubernetes. Bien que probablement overkill pour un lab personnel single-node, Rook devient pertinent si vous expandez vers plusieurs nodes ou nécessitez object storage S3-compatible intégré. La courbe d'apprentissage est raide mais les capacités sont impressionnantes.

Les solutions cloud hybrides permettent d'étendre votre stockage local avec le cloud. Des outils comme Kasten K10 ou CloudCasa peuvent répliquer vos données vers AWS, Azure, ou GCP, offrant disaster recovery sans maintenir une infrastructure de backup complexe. Cette approche combine la performance du stockage local avec la durabilité du cloud, idéale pour les données critiques dans un lab personnel.

⏭️
