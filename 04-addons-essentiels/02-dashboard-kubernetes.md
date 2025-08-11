🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 Dashboard Kubernetes

## Introduction au Dashboard Kubernetes

Le Dashboard Kubernetes est une interface web universelle qui transforme la gestion de votre cluster d'une expérience en ligne de commande à une interaction visuelle intuitive. Pour les débutants, c'est souvent le premier contact tangible avec les concepts Kubernetes, rendant visible ce qui se passe réellement dans votre cluster. Cette interface graphique officielle, maintenue par le projet Kubernetes lui-même, offre une vue d'ensemble complète de vos ressources, leur état, et permet d'effectuer la plupart des opérations courantes sans toucher à kubectl.

Dans le contexte de MicroK8s, le Dashboard s'installe comme un addon simple, préconfiguré et sécurisé. Il devient rapidement un outil indispensable pour comprendre les interactions entre vos applications, diagnostiquer les problèmes, et apprendre les concepts Kubernetes de manière visuelle. Contrairement à d'autres solutions qui nécessitent une configuration complexe, l'addon Dashboard de MicroK8s est opérationnel en quelques secondes avec des paramètres de sécurité appropriés.

## Architecture et composants du Dashboard

Le Dashboard Kubernetes n'est pas une application monolithique mais un ensemble de composants travaillant en harmonie pour offrir une expérience utilisateur fluide. Comprendre cette architecture vous aidera à mieux diagnostiquer les problèmes éventuels et à personnaliser votre installation.

Le composant principal est le Dashboard UI, une application web moderne développée en Angular qui s'exécute dans votre navigateur. Cette interface communique avec le Dashboard Backend, un service qui s'exécute dans votre cluster et fait office de proxy entre l'interface utilisateur et l'API Kubernetes. Ce backend gère l'authentification, l'autorisation, et traduit les actions de l'interface en appels API Kubernetes appropriés.

Le Metrics Scraper est un composant auxiliaire qui collecte les métriques de ressources pour les afficher dans le Dashboard. Il interroge régulièrement le Metrics Server (s'il est installé) pour obtenir les données de consommation CPU et mémoire, permettant l'affichage de graphiques et de statistiques en temps réel dans l'interface.

L'ensemble de ces composants est déployé dans le namespace `kube-system` par défaut dans MicroK8s, avec des ServiceAccounts et des Roles RBAC appropriés pour garantir un accès sécurisé et contrôlé aux ressources du cluster. Cette isolation garantit que le Dashboard ne peut accéder qu'aux ressources pour lesquelles il a été explicitement autorisé.

## Installation et activation

L'activation du Dashboard dans MicroK8s est remarquablement simple grâce au système d'addons. La commande `microk8s enable dashboard` déclenche l'installation complète, incluant tous les composants nécessaires et leur configuration. Cette simplicité cache une orchestration sophistiquée qui configure automatiquement les certificats SSL, les comptes de service, et les règles de sécurité.

Lors de l'activation, MicroK8s génère automatiquement un certificat auto-signé pour sécuriser les communications HTTPS avec le Dashboard. Ce certificat est stocké comme Secret Kubernetes et automatiquement monté dans le pod du Dashboard. Bien que les navigateurs affichent un avertissement pour les certificats auto-signés, cette configuration garantit que toutes les communications sont chiffrées, essentielles pour la sécurité, surtout si vous accédez au Dashboard depuis un réseau non sécurisé.

Le processus d'installation configure également un ServiceAccount nommé `kubernetes-dashboard` avec les permissions appropriées. Par défaut, ce compte a des permissions limitées pour des raisons de sécurité, mais elles peuvent être étendues selon vos besoins. Un ClusterRoleBinding lie ce ServiceAccount au ClusterRole approprié, définissant précisément ce que le Dashboard peut voir et modifier dans votre cluster.

## Méthodes d'accès au Dashboard

Accéder au Dashboard nécessite de comprendre les différentes options disponibles et leurs implications en termes de sécurité et de praticité. MicroK8s offre plusieurs méthodes, chacune adaptée à différents scénarios d'utilisation.

La méthode la plus sécurisée et recommandée pour un accès local est le proxy kubectl. La commande `microk8s kubectl proxy` crée un tunnel sécurisé entre votre machine locale et le cluster, exposant l'API Kubernetes sur localhost:8001. Le Dashboard devient alors accessible via `http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`. Cette méthode garantit que le Dashboard n'est jamais exposé directement sur le réseau.

Pour un accès plus direct, vous pouvez utiliser le port-forward. La commande `microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443` redirige le port 443 du service Dashboard vers le port 10443 de votre machine locale. Vous pouvez alors accéder au Dashboard via `https://localhost:10443`. Cette méthode est pratique pour un accès temporaire mais doit être relancée à chaque fois.

Dans un environnement de lab où la sécurité n'est pas critique, vous pouvez exposer le Dashboard via un NodePort ou un Ingress. Cependant, cette approche nécessite une configuration supplémentaire de l'authentification et du SSL pour rester sécurisée. Si vous choisissez cette voie, assurez-vous d'implémenter une authentification forte et d'utiliser des certificats SSL valides, idéalement via Cert-Manager et Let's Encrypt.

## Authentification et autorisation

La sécurité du Dashboard repose sur un système d'authentification robuste qui détermine qui peut accéder à l'interface et ce qu'ils peuvent y faire. MicroK8s configure par défaut une authentification basée sur les tokens, mais comprendre les options disponibles vous permettra d'adapter la sécurité à vos besoins.

L'authentification par token est la méthode par défaut et la plus courante. Pour obtenir un token d'accès, vous devez d'abord identifier le Secret associé au ServiceAccount du Dashboard. La commande `microk8s kubectl -n kube-system get secret | grep kubernetes-dashboard-token` liste les secrets disponibles. Une fois le secret identifié, `microk8s kubectl -n kube-system describe secret kubernetes-dashboard-token-xxxxx` révèle le token que vous pouvez copier et coller dans l'interface de connexion du Dashboard.

Pour une expérience plus pratique dans un environnement de développement, vous pouvez créer un ServiceAccount avec des permissions administrateur. Créez un nouveau ServiceAccount, attribuez-lui le ClusterRole `cluster-admin` via un ClusterRoleBinding, puis récupérez son token. Cette approche offre un accès complet au cluster via le Dashboard mais ne devrait jamais être utilisée dans un environnement de production ou exposé sur Internet.

Le Dashboard supporte également l'authentification via kubeconfig, où vous pouvez uploader votre fichier de configuration kubectl directement dans l'interface. Cette méthode est pratique si vous gérez plusieurs clusters et souhaitez utiliser les mêmes credentials. Le fichier kubeconfig de MicroK8s se trouve généralement dans `/var/snap/microk8s/current/credentials/`.

## Navigation et interface utilisateur

L'interface du Dashboard est organisée de manière logique et hiérarchique, reflétant la structure des ressources Kubernetes. La navigation principale sur la gauche groupe les ressources par catégorie, facilitant l'exploration même pour les débutants qui ne connaissent pas encore tous les concepts Kubernetes.

La section Workloads est généralement votre point de départ, affichant vos Deployments, Pods, ReplicaSets, et StatefulSets. Chaque ressource est présentée dans une vue tabulaire avec des informations essentielles comme le statut, l'âge, et les labels. En cliquant sur une ressource, vous accédez à une vue détaillée montrant les spécifications complètes, les événements associés, et les logs pour les pods.

La section Service Discovery and Load Balancing révèle comment vos applications sont exposées, montrant les Services, Ingresses, et EndPoints. Cette vue est particulièrement utile pour comprendre le routage du trafic dans votre cluster et diagnostiquer les problèmes de connectivité. Les relations entre services et pods sont visualisées clairement, aidant à comprendre les dépendances.

Config and Storage présente vos ConfigMaps, Secrets, et PersistentVolumeClaims. Cette section est cruciale pour gérer la configuration de vos applications et comprendre l'utilisation du stockage. Le Dashboard masque intelligemment le contenu des Secrets par défaut pour des raisons de sécurité, mais permet de les révéler si nécessaire.

La vue Cluster offre une perspective administrative, montrant les Namespaces, Nodes, et Roles RBAC. Cette section est particulièrement utile pour comprendre la structure globale de votre cluster et gérer les permissions. Les métriques de ressources au niveau du cluster sont également affichées ici si le Metrics Server est installé.

## Fonctionnalités de monitoring et métriques

Le Dashboard excelle dans la visualisation des métriques et de l'état de santé de votre cluster, transformant des données brutes en informations actionnables. Cette capacité de monitoring intégrée en fait un outil précieux pour le diagnostic et l'optimisation des performances.

Lorsque le Metrics Server est activé dans votre cluster MicroK8s, le Dashboard affiche automatiquement des graphiques de consommation CPU et mémoire pour les pods et les nodes. Ces graphiques temps réel permettent d'identifier rapidement les applications gourmandes en ressources ou les problèmes de performance. Les données historiques sur les dernières minutes donnent un contexte précieux pour comprendre les patterns d'utilisation.

La vue d'ensemble du cluster présente des statistiques agrégées comme le nombre total de pods en cours d'exécution, les deployments en erreur, et l'utilisation globale des ressources. Ces indicateurs de haut niveau offrent un health check rapide de votre environnement. Les alertes visuelles, utilisant des codes couleur intuitifs, attirent l'attention sur les problèmes nécessitant une intervention.

Les événements Kubernetes sont présentés de manière chronologique et filtrables, permettant de retracer l'historique des actions et des erreurs dans votre cluster. Cette fonctionnalité est inestimable pour le troubleshooting, montrant exactement ce qui s'est passé et quand. Les événements sont corrélés avec les ressources concernées, facilitant l'identification de la cause racine des problèmes.

## Gestion des ressources via le Dashboard

Au-delà de la simple visualisation, le Dashboard permet de créer, modifier, et supprimer des ressources Kubernetes directement depuis l'interface web. Cette capacité transforme le Dashboard d'un simple outil de monitoring en une plateforme de gestion complète.

La création de ressources peut s'effectuer de plusieurs manières. L'interface propose des formulaires guidés pour les ressources communes comme les Deployments et Services, avec des champs pré-remplis et une validation en temps réel. Pour les utilisateurs plus avancés, un éditeur YAML intégré permet de coller ou éditer directement les manifestes, avec coloration syntaxique et validation de schéma.

L'édition des ressources existantes s'effectue via le même éditeur YAML, permettant de modifier les configurations en direct. Le Dashboard affiche les différences avant application, réduisant le risque d'erreurs. Les modifications sont appliquées immédiatement, et les effets sont visibles en temps réel dans l'interface, offrant un feedback immédiat.

La suppression de ressources est protégée par des confirmations pour éviter les suppressions accidentelles. Le Dashboard gère intelligemment les cascades de suppression, avertissant si la suppression d'une ressource affectera d'autres objets. Cette protection est particulièrement importante pour les débutants qui pourraient ne pas comprendre toutes les implications d'une suppression.

Le scaling des deployments est simplifié à l'extrême, permettant d'ajuster le nombre de replicas avec un simple slider ou champ numérique. Les effets du scaling sont immédiatement visibles, avec les nouveaux pods apparaissant dans l'interface en temps réel. Cette fonctionnalité est parfaite pour tester le comportement de vos applications sous différentes charges.

## Consultation des logs et debugging

Une des fonctionnalités les plus utilisées du Dashboard est la consultation des logs, transformant le debugging d'une tâche complexe en ligne de commande en une expérience visuelle intuitive. Cette capacité est particulièrement précieuse pour les débutants qui ne maîtrisent pas encore toutes les options de kubectl logs.

L'accès aux logs s'effectue directement depuis la vue détaillée de chaque pod, avec un bouton dédié qui ouvre une fenêtre de logs en temps réel. Les logs sont affichés avec coloration syntaxique quand possible, facilitant la lecture et l'identification des erreurs. La fonction de recherche intégrée permet de filtrer les logs par mots-clés, dates, ou patterns.

Pour les pods avec plusieurs containers, le Dashboard permet de sélectionner le container dont vous souhaitez consulter les logs. Cette fonctionnalité est essentielle pour les applications complexes utilisant des sidecars ou des init containers. Les logs de chaque container sont accessibles indépendamment, avec la possibilité de les télécharger pour une analyse offline.

Le streaming en temps réel des logs permet de suivre l'activité de vos applications en direct, particulièrement utile lors du debugging ou du déploiement de nouvelles versions. L'auto-scroll maintient automatiquement l'affichage sur les dernières lignes, mais peut être désactivé pour examiner des sections spécifiques. La fonctionnalité de tail permet de limiter l'affichage aux N dernières lignes, évitant de surcharger le navigateur avec des logs volumineux.

L'intégration avec les événements Kubernetes enrichit le contexte de debugging. Lorsque vous consultez les logs d'un pod en erreur, le Dashboard affiche également les événements associés, permettant de corréler les messages d'erreur dans les logs avec les événements système. Cette vue unifiée accélère considérablement l'identification des problèmes.

## Personnalisation et configuration avancée

Bien que le Dashboard fonctionne parfaitement avec sa configuration par défaut, de nombreuses options de personnalisation permettent d'adapter l'interface à vos besoins spécifiques et préférences.

Les paramètres d'affichage peuvent être ajustés pour chaque vue, incluant le nombre d'éléments par page, les colonnes visibles, et l'ordre de tri. Ces préférences sont sauvegardées localement dans votre navigateur, offrant une expérience personnalisée persistante. Les filtres avancés permettent de créer des vues personnalisées basées sur les labels, namespaces, ou statuts.

La configuration du timeout de session et de l'auto-refresh peut être ajustée selon vos besoins de sécurité et de performance. Un timeout court améliore la sécurité mais peut être frustrant lors de longues sessions de travail. L'auto-refresh peut être désactivé pour économiser les ressources ou accéléré pour un monitoring plus réactif.

Pour les environnements multi-tenant, le Dashboard peut être configuré avec des restrictions de namespace, limitant ce que chaque utilisateur peut voir et modifier. Cette configuration s'effectue via les RBAC Kubernetes standards, permettant une granularité fine des permissions. Vous pouvez créer des ServiceAccounts dédiés avec des permissions spécifiques pour différents types d'utilisateurs.

L'intégration avec des outils externes est possible via les annotations Kubernetes. Par exemple, vous pouvez ajouter des liens vers votre documentation, votre système de ticketing, ou vos dashboards Grafana directement dans l'interface du Dashboard. Ces personnalisations enrichissent l'expérience utilisateur et centralisent l'accès aux outils de votre écosystème.

## Limitations et considérations

Comprendre les limitations du Dashboard vous aidera à définir des attentes réalistes et à identifier quand d'autres outils pourraient être plus appropriés.

Le Dashboard n'est pas conçu pour gérer des clusters massifs avec des milliers de ressources. Les performances peuvent se dégrader significativement avec un grand nombre d'objets, rendant l'interface lente et peu réactive. Pour les très grands clusters, des outils spécialisés comme Lens ou k9s peuvent offrir de meilleures performances.

Les capacités de monitoring du Dashboard restent basiques comparées à des solutions dédiées comme Prometheus et Grafana. Bien que suffisantes pour un aperçu général, elles ne remplacent pas une stack d'observabilité complète pour une analyse approfondie des performances et des tendances historiques.

La gestion des manifestes complexes ou des déploiements multi-ressources est limitée dans le Dashboard. Pour des déploiements sophistiqués utilisant Helm ou Kustomize, vous devrez toujours recourir à la ligne de commande. Le Dashboard excelle dans la gestion quotidienne mais n'est pas un remplacement complet de kubectl pour les opérations avancées.

## Sécurité et bonnes pratiques

La sécurité du Dashboard doit être une priorité, particulièrement si vous envisagez de l'exposer au-delà de localhost. Plusieurs pratiques essentielles garantissent que votre Dashboard reste un atout plutôt qu'une vulnérabilité.

Ne jamais exposer le Dashboard directement sur Internet sans authentification forte et chiffrement SSL. Même dans un environnement de lab, utilisez toujours HTTPS et des tokens d'authentification. Si vous devez absolument exposer le Dashboard, utilisez un VPN ou un tunnel SSH plutôt qu'une exposition directe.

Créez des ServiceAccounts avec le minimum de privilèges nécessaires pour chaque utilisateur. Évitez d'utiliser le cluster-admin pour l'accès quotidien. Implémentez une rotation régulière des tokens et surveillez les logs d'accès pour détecter toute activité suspecte.

Gardez le Dashboard à jour en mettant régulièrement à jour MicroK8s. Les mises à jour incluent souvent des correctifs de sécurité importants. Surveillez les CVE (Common Vulnerabilities and Exposures) liées au Dashboard Kubernetes et appliquez les patches rapidement.

Utilisez les fonctionnalités d'audit de Kubernetes pour logger toutes les actions effectuées via le Dashboard. Ces logs peuvent être précieux pour le debugging mais aussi pour la conformité et la sécurité. Configurez des alertes pour les actions sensibles comme les suppressions de ressources critiques.

## Alternatives et compléments

Bien que le Dashboard Kubernetes soit un excellent point de départ, connaître les alternatives et compléments enrichira votre boîte à outils de gestion Kubernetes.

Lens est une application desktop populaire offrant une interface plus riche et des performances supérieures pour les grands clusters. Elle propose des fonctionnalités avancées comme la gestion multi-cluster, un terminal intégré, et des extensions. Pour un usage quotidien intensif, Lens peut être plus approprié que le Dashboard web.

Octant, développé par VMware, offre une approche différente avec un focus sur l'extensibilité et les plugins. Il fonctionne localement et offre une vue développeur-centrique de votre cluster. K9s propose une interface terminal interactive pour ceux qui préfèrent rester en ligne de commande tout en bénéficiant d'une navigation simplifiée.

Pour le monitoring avancé, Grafana reste inégalé avec ses capacités de visualisation sophistiquées et ses intégrations étendues. Le Dashboard Kubernetes et Grafana sont complémentaires plutôt que concurrents, le premier pour la gestion quotidienne, le second pour l'analyse approfondie.

## Conclusion et perspectives

Le Dashboard Kubernetes dans MicroK8s représente une porte d'entrée idéale vers l'écosystème Kubernetes, offrant une visualisation immédiate et une gestion simplifiée de votre cluster. Pour les débutants, c'est un outil pédagogique inestimable qui rend concrets les concepts abstraits de Kubernetes. Pour les utilisateurs expérimentés, il reste un outil pratique pour les tâches quotidiennes et le debugging rapide.

L'évolution continue du Dashboard apporte régulièrement de nouvelles fonctionnalités et améliorations. Les versions récentes ont amélioré les performances, enrichi les capacités de monitoring, et renforcé la sécurité. La communauté active continue de contribuer des améliorations, garantissant que le Dashboard reste pertinent et utile.

Dans votre journey avec MicroK8s, le Dashboard sera probablement l'un de vos outils les plus utilisés initialement. Au fur et à mesure que vous gagnerez en expertise, vous le compléterez naturellement avec d'autres outils, mais il restera toujours utile pour obtenir rapidement une vue d'ensemble de votre cluster et effectuer des actions simples. Sa simplicité et son accessibilité en font un composant essentiel de tout lab Kubernetes.

⏭️
