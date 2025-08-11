🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Registry (registre d'images privé)

## Introduction au concept de registre d'images

Le registre d'images constitue le cœur de la distribution des applications conteneurisées dans l'écosystème Docker et Kubernetes. Pensez à un registre comme une bibliothèque sophistiquée où chaque livre est une image de conteneur, soigneusement cataloguée, versionnée, et prête à être déployée instantanément. Dans un environnement Kubernetes, le registre devient encore plus crucial car c'est depuis lui que tous les nodes téléchargent les images nécessaires pour faire tourner vos applications.

Dans le contexte d'un lab personnel MicroK8s, disposer de son propre registre privé transforme radicalement votre workflow de développement. Au lieu de pousser constamment vos images vers Docker Hub ou d'autres registres publics, exposant potentiellement du code propriétaire et consommant de la bande passante, vous gardez tout en local. Cette proximité accélère dramatiquement les cycles de développement, permettant des itérations rapides sans les latences du réseau Internet.

Le registre privé MicroK8s n'est pas simplement un stockage d'images mais un composant intégré qui comprend l'écosystème Kubernetes. Il s'interface naturellement avec les mécanismes d'authentification de Kubernetes, respecte les politiques de sécurité du cluster, et peut être exposé de manière sécurisée via l'Ingress Controller. Cette intégration native élimine les frictions communes lors de l'utilisation de registres externes, où la configuration des secrets d'authentification et des certificats SSL peut devenir complexe.

## Architecture d'un registre Docker

Comprendre l'architecture interne d'un registre Docker vous permettra de mieux exploiter ses capacités et de diagnostiquer efficacement les problèmes éventuels. Le registre n'est pas une application monolithique mais un ensemble de composants travaillant de concert pour gérer le cycle de vie complet des images.

Au cœur du système se trouve l'API Registry, une interface RESTful qui suit la spécification Docker Registry HTTP API V2. Cette API gère toutes les opérations sur les images, depuis le push initial jusqu'au pull par les nodes Kubernetes. Chaque opération est décomposée en transactions atomiques, garantissant l'intégrité des données même en cas d'interruption. L'API supporte le chunked upload, permettant de reprendre les téléchargements interrompus, crucial pour les images volumineuses dans des environnements avec une connectivité instable.

Le système de stockage backend constitue la fondation physique du registre. Par défaut, le registre MicroK8s utilise le système de fichiers local, mais l'architecture supporte diverses backends incluant les stockages objets compatibles S3, Azure Blob Storage, ou Google Cloud Storage. Cette flexibilité permet d'adapter le registre à vos contraintes de stockage, depuis un simple dossier local jusqu'à des solutions cloud scalables. Le stockage utilise une structure content-addressable, où chaque layer d'image est identifié par son hash SHA256, garantissant l'immutabilité et permettant la déduplication automatique des layers partagés entre images.

Le mécanisme de manifestes gère les métadonnées des images et leurs relations. Un manifeste décrit la composition d'une image, listant tous les layers nécessaires et leur ordre d'application. Le support des manifest lists permet de gérer des images multi-architectures, essentielles dans un monde où ARM devient aussi commun que x86. Cette capacité est particulièrement pertinente pour un lab personnel qui pourrait inclure des Raspberry Pi ou d'autres dispositifs ARM.

La couche de sécurité implémente l'authentification et l'autorisation pour protéger vos images. Le registre supporte plusieurs mécanismes d'authentification, depuis le basic auth simple jusqu'à l'intégration avec des systèmes d'identité d'entreprise via OAuth2. L'autorisation granulaire permet de définir qui peut push, pull, ou delete des images spécifiques, crucial pour maintenir l'intégrité de votre pipeline de déploiement.

## Installation et activation dans MicroK8s

L'activation du registre dans MicroK8s démontre la puissance du système d'addons en transformant une opération complexe en une simple commande, tout en offrant des options de personnalisation pour les besoins avancés.

La commande `microk8s enable registry` déclenche une orchestration sophistiquée qui déploie tous les composants nécessaires. MicroK8s crée d'abord un PersistentVolume pour stocker les images, garantissant leur persistance même si le pod du registre est recréé. La taille par défaut de 20GB convient pour un lab personnel typique, mais peut être ajustée avec le paramètre size, par exemple `microk8s enable registry:size=40Gi` pour un stockage de 40GB.

Le déploiement du registre utilise l'image officielle Docker Registry, maintenue et régulièrement mise à jour pour inclure les derniers correctifs de sécurité. Le pod est configuré avec des resource limits appropriées pour un lab personnel, évitant qu'un registre mal configuré ne consomme toutes les ressources de votre système. Les liveness et readiness probes garantissent que Kubernetes peut détecter et récupérer automatiquement des défaillances du registre.

Le service exposant le registre est configuré comme ClusterIP par défaut, accessible uniquement depuis l'intérieur du cluster sur le port 5000. Cette configuration sécurisée par défaut évite l'exposition accidentelle de votre registre privé. L'adresse typique `localhost:32000` est configurée via un NodePort pour permettre l'accès depuis l'hôte, facilitant les opérations de push depuis votre machine de développement.

L'intégration avec containerd, le runtime de conteneur de MicroK8s, est automatiquement configurée. Le fichier de configuration de containerd est modifié pour reconnaître le registre local comme source fiable, éliminant les erreurs de certificat SSL lors du pull d'images. Cette configuration transparente signifie que vos deployments Kubernetes peuvent référencer des images du registre local sans configuration supplémentaire des secrets d'authentification.

## Gestion des images et tags

La gestion efficace des images dans votre registre privé nécessite de comprendre les conventions de nommage, les stratégies de tagging, et les meilleures pratiques pour maintenir un registre organisé et performant.

La convention de nommage des images suit le format `[REGISTRY_HOST[:PORT]/]NAMESPACE/IMAGE[:TAG]`. Dans le contexte MicroK8s, vos images locales suivront typiquement le pattern `localhost:32000/myproject/myapp:v1.0.0`. Cette structure hiérarchique permet d'organiser logiquement vos images par projet, environnement, ou équipe. Les namespaces ne sont pas automatiquement liés aux namespaces Kubernetes, offrant une flexibilité dans l'organisation.

Les stratégies de tagging déterminent comment versionner et identifier vos images. Le semantic versioning (v1.2.3) offre une clarté sur la compatibilité et les changements. Les tags environnementaux (dev, staging, prod) simplifient les déploiements multi-environnements. Les tags de commit Git (git-abc123) permettent une traçabilité parfaite vers le code source. La combinaison de plusieurs stratégies, comme `v1.2.3-git-abc123-dev`, offre maximum de contexte au prix d'une complexité accrue.

Le tag `latest` mérite une attention particulière car il est souvent source de confusion. Contrairement à l'intuition, `latest` n'est pas automatiquement assigné à l'image la plus récente mais doit être explicitement poussé. Dans un environnement de production, éviter `latest` en faveur de tags explicites garantit la reproductibilité des déploiements. Pour le développement rapide, `latest` peut accélérer les itérations mais doit être utilisé consciemment.

La gestion du cycle de vie des images inclut la suppression régulière des images obsolètes pour éviter que le registre ne consomme tout l'espace disque disponible. Le registre n'offre pas de garbage collection automatique par défaut, nécessitant une maintenance manuelle ou scriptée. L'API REST du registre permet de lister, inspecter, et supprimer des images programmatiquement, facilitant l'automatisation de cette maintenance.

## Workflow de développement avec registre local

L'intégration d'un registre privé dans votre workflow de développement transforme la façon dont vous construisez, testez, et déployez vos applications dans MicroK8s.

Le cycle build-push-deploy devient remarquablement fluide avec un registre local. Après avoir construit votre image avec `docker build` ou `podman build`, un simple `docker push localhost:32000/myapp:dev` pousse l'image vers votre registre. Les deployments Kubernetes peuvent immédiatement utiliser cette image sans configuration de secrets ou d'authentification. Cette simplicité accélère dramatiquement les itérations de développement, permettant de tester des changements en quelques secondes plutôt qu'en minutes.

L'intégration avec les pipelines CI/CD locaux devient naturelle avec un registre privé. Des outils comme Jenkins, GitLab Runner, ou Tekton peuvent pousser directement vers votre registre après une build réussie. Le registre devient le point central de votre pipeline, stockant les artefacts de build et servant de source de vérité pour les déploiements. Cette architecture mirror celle des environnements de production, offrant une expérience de développement réaliste.

Le développement multi-architecture est simplifié grâce au support des manifest lists. Vous pouvez construire et pousser des images pour différentes architectures (amd64, arm64) vers le même tag, et Kubernetes sélectionnera automatiquement l'image appropriée selon l'architecture du node. Cette capacité est particulièrement précieuse si votre lab inclut des machines hétérogènes ou si vous développez pour des déploiements edge computing.

La collaboration en équipe, même dans un lab personnel, bénéficie d'un registre centralisé. Plusieurs développeurs peuvent partager des images sans passer par des registres publics, maintenant la confidentialité du code. Les branches de développement peuvent avoir leurs propres tags, permettant des tests isolés sans interférence. Le registre devient un hub de collaboration, facilitant le partage de travail en cours.

## Sécurisation du registre

La sécurité de votre registre privé est cruciale car il contient potentiellement tout votre code applicatif et peut devenir un vecteur d'attaque si mal configuré.

L'authentification basique représente le niveau minimum de sécurité pour un registre exposé. Le registre MicroK8s peut être configuré avec htpasswd pour exiger un username et password pour toute opération. Cette configuration s'effectue via un Secret Kubernetes contenant les credentials hashés, intégré transparemment avec le pod du registre. Bien que simple, cette méthode offre une protection efficace contre les accès non autorisés dans un environnement de lab.

L'exposition sécurisée via HTTPS est essentielle si le registre est accessible depuis l'extérieur du cluster. L'intégration avec Cert-Manager permet d'obtenir automatiquement des certificats Let's Encrypt valides pour votre registre. La configuration d'un Ingress dédié pour le registre, avec les annotations appropriées pour gérer les large uploads, permet un accès sécurisé depuis Internet tout en maintenant les performances.

Le scanning de vulnérabilités des images stockées ajoute une couche de sécurité proactive. Des outils comme Trivy ou Clair peuvent être intégrés pour scanner automatiquement les nouvelles images poussées vers le registre. Les vulnérabilités détectées peuvent déclencher des alertes ou même bloquer le déploiement d'images compromises. Cette pratique, standard dans les environnements de production, est excellente à implémenter même dans un lab personnel.

Les politiques de rétention et d'accès définissent qui peut faire quoi avec quelles images. Bien que le registre Docker basique n'offre pas de contrôle d'accès granulaire natif, des solutions comme Harbor peuvent être déployées dans MicroK8s pour offrir des fonctionnalités entreprise. Ces politiques peuvent limiter qui peut pousser vers certains namespaces, implémenter des quotas par utilisateur, ou automatiquement supprimer les images non utilisées.

## Intégration avec l'écosystème Kubernetes

Le registre privé ne fonctionne pas en isolation mais s'intègre profondément avec les autres composants de votre cluster MicroK8s.

Les ImagePullSecrets permettent de configurer l'authentification pour des registres nécessitant des credentials. Bien que le registre MicroK8s local n'en nécessite pas par défaut, comprendre leur fonctionnement est essentiel pour intégrer des registres externes. Ces secrets peuvent être attachés aux ServiceAccounts, permettant une gestion centralisée des accès aux registres pour différentes applications ou équipes.

L'intégration avec les admission webhooks permet d'implémenter des politiques sophistiquées sur les images autorisées dans votre cluster. Des outils comme Open Policy Agent (OPA) peuvent valider que les images proviennent de registres approuvés, ont été scannées pour les vulnérabilités, ou respectent des conventions de nommage. Cette gouvernance automatisée garantit que seules des images conformes sont déployées.

Le mirroring de registres externes améliore la résilience et les performances. Votre registre local peut être configuré comme proxy cache pour Docker Hub ou d'autres registres, stockant localement les images fréquemment utilisées. Cette configuration réduit la bande passante, accélère les déploiements, et offre une continuité de service même si les registres externes sont indisponibles.

La réplication entre registres permet de synchroniser des images entre différents environnements. Des outils comme Skopeo peuvent automatiser la copie d'images entre votre registre de développement et celui de production, maintenant la cohérence tout en respectant les boundaries de sécurité. Cette capacité est particulièrement utile pour promouvoir des images à travers les stages de votre pipeline.

## Optimisation des performances et du stockage

L'optimisation de votre registre privé garantit des performances optimales tout en minimisant l'utilisation des ressources, crucial dans un environnement de lab aux ressources limitées.

La déduplication des layers est automatique dans le registre Docker, mais comprendre son fonctionnement permet de l'optimiser. Chaque layer d'image n'est stocké qu'une seule fois, même s'il est partagé par plusieurs images. Construire vos images avec des base images communes et ordonnancer correctement les instructions Dockerfile maximise le partage de layers, réduisant significativement l'espace de stockage nécessaire.

Le garbage collection nettoie les layers orphelins et les manifestes non référencés. Cette opération n'est pas automatique et doit être déclenchée manuellement ou via un cron job. La commande `registry garbage-collect` nécessite que le registre soit en mode read-only pendant l'opération, nécessitant une planification appropriée. Dans un lab personnel, exécuter le garbage collection hebdomadairement maintient généralement un bon équilibre.

La configuration du cache HTTP améliore les performances de pull, particulièrement pour les images volumineuses. Les headers de cache appropriés permettent aux clients de valider si leur copie locale est à jour sans télécharger l'image complète. Le support du HTTP/2 dans les versions récentes du registre améliore encore les performances avec le multiplexing des connexions.

Le monitoring des métriques du registre révèle les patterns d'utilisation et les opportunités d'optimisation. Le registre expose des métriques Prometheus incluant le nombre de pulls/pushes, les tailles de réponse, et les latences. Ces métriques peuvent révéler des images problématiques (trop volumineuses), des patterns d'accès inefficaces, ou des problèmes de performance réseau.

## Troubleshooting et problèmes courants

Le diagnostic des problèmes avec le registre suit des patterns récurrents qu'il est utile de connaître pour résoudre rapidement les dysfonctionnements.

Les erreurs de push sont souvent liées à la configuration réseau ou aux permissions. L'erreur "connection refused" indique généralement que le registre n'est pas accessible sur le port attendu. Vérifier que le NodePort est correctement configuré et que les règles firewall permettent l'accès. Les erreurs de certificat SSL lors du push vers localhost:32000 peuvent nécessiter la configuration de Docker pour accepter les registres insecure, bien que MicroK8s configure normalement cela automatiquement.

Les problèmes de pull depuis les pods Kubernetes ont souvent des causes différentes des problèmes de push. L'erreur "ErrImagePull" dans les événements de pod peut indiquer que l'image n'existe pas dans le registre, que le tag est incorrect, ou que le pod n'a pas accès au registre. Vérifier que l'image a été correctement poussée avec `curl http://localhost:32000/v2/_catalog` pour lister les repositories disponibles.

Les problèmes d'espace disque se manifestent par des erreurs de push intermittentes ou des performances dégradées. Le registre ne gère pas gracieusement le manque d'espace, pouvant corrompre des images en cours d'upload. Surveiller régulièrement l'utilisation du PersistentVolume et implémenter des alertes avant d'atteindre la saturation. Le garbage collection régulier et la suppression des images obsolètes préviennent la plupart de ces problèmes.

Les corruptions d'images, bien que rares, peuvent survenir suite à des interruptions pendant le push ou des problèmes de stockage. Le registre vérifie les checksums mais ne peut pas toujours détecter toutes les corruptions. Si une image semble corrompue, la supprimer complètement du registre et la repousser. Dans les cas extrêmes, purger le stockage du registre et recommencer peut être nécessaire.

## Alternatives et solutions avancées

Bien que le registre Docker basique inclus dans MicroK8s soit suffisant pour la plupart des labs personnels, connaître les alternatives permet de choisir la solution optimale pour vos besoins spécifiques.

Harbor représente l'évolution entreprise du registre Docker, offrant une interface web riche, un scanning de vulnérabilités intégré, la réplication entre registres, et un contrôle d'accès granulaire basé sur les rôles. Harbor peut être déployé dans MicroK8s via Helm, transformant votre lab en plateforme de gestion d'images professionnelle. La complexité accrue et les ressources nécessaires doivent être pesées contre les fonctionnalités additionnelles.

GitLab Container Registry intégré avec GitLab CI/CD offre une expérience unifiée du code source au déploiement. Si vous utilisez déjà GitLab pour votre gestion de code, son registre intégré simplifie le workflow en éliminant un composant séparé. L'intégration native avec les pipelines CI/CD et les fonctionnalités de sécurité font de GitLab une option attractive pour les labs orientés DevOps.

Nexus Repository Manager ou JFrog Artifactory sont des gestionnaires de repositories universels supportant Docker images mais aussi Maven, npm, PyPI, et d'autres formats. Pour un lab gérant plusieurs technologies, ces solutions offrent un point central pour tous les artefacts. Leur complexité et leurs requirements en ressources les rendent plus appropriés pour des labs d'équipe que personnels.

Les registres cloud comme Amazon ECR, Google Container Registry, ou Azure Container Registry offrent une alternative managed éliminant la maintenance. Pour un lab hybride avec des composants cloud, ces services peuvent simplifier l'architecture. Les coûts, la latence réseau, et la dépendance externe doivent être considérés.

## Cas d'usage et patterns

Le registre privé dans MicroK8s enable plusieurs patterns architecturaux et workflows qui enrichissent votre environnement de développement.

Le pattern de promotion d'images implémente un workflow où les images progressent à travers différents stages (dev, test, staging, prod) avec des validations à chaque étape. Le registre stocke toutes les versions, permettant des rollbacks rapides si nécessaire. Les tags reflètent le stage actuel, et des scripts automatisent la promotion après validation.

Le caching de base images accélère significativement les builds en stockant localement les images de base fréquemment utilisées. Au lieu de télécharger ubuntu:22.04 ou node:18 depuis Docker Hub à chaque build, votre registre local les sert instantanément. Cette approche est particulièrement bénéfique pour les images volumineuses ou dans des environnements avec une connectivité Internet limitée.

Le développement de microservices bénéficie enormément d'un registre local où chaque service a son propre repository. Les dépendances entre services peuvent être gérées via des tags spécifiques, permettant de tester différentes combinaisons de versions. Le registre devient le hub central coordonnant les versions de tous les composants de votre architecture.

L'archivage d'images pour la conformité ou l'audit utilise le registre comme système de record immutable. Chaque image déployée en production est préservée avec ses métadonnées, permettant de recréer exactement n'importe quel état passé. Cette capacité est cruciale pour le debugging post-mortem ou les requirements de conformité.

## Maintenance et meilleures pratiques

La maintenance régulière de votre registre garantit sa fiabilité et ses performances optimales au fil du temps.

La stratégie de backup doit couvrir à la fois les images et les métadonnées du registre. Le PersistentVolume contenant les données du registre devrait être sauvegardé régulièrement, idéalement avec des snapshots si votre système de stockage le supporte. Les métadonnées de configuration stockées dans les ConfigMaps et Secrets doivent également être incluses dans les backups. Tester régulièrement la restauration garantit que vos backups sont fonctionnels.

Le monitoring continu via des health checks et des métriques permet de détecter les problèmes avant qu'ils n'impactent les utilisateurs. Un simple script vérifiant la disponibilité de l'API du registre et la capacité de pull/push peut alerter sur les dysfonctionnements. Les métriques de performance révèlent les dégradations graduelles nécessitant attention.

La documentation des conventions et procédures assure la cohérence et facilite l'onboarding de nouveaux utilisateurs. Documenter les conventions de nommage, les stratégies de tagging, les procédures de promotion d'images, et les contacts pour le support crée un environnement prévisible et efficace. Cette documentation devrait vivre proche du code, idéalement dans le même repository.

Les audits réguliers de sécurité vérifient que le registre reste sécurisé. Examiner les logs d'accès pour détecter des patterns anormaux, vérifier que les images obsolètes avec des vulnérabilités connues sont supprimées, et s'assurer que les certificats SSL sont valides et à jour. Ces audits peuvent révéler des failles de sécurité ou des mauvaises pratiques avant qu'elles ne soient exploitées.

⏭️
