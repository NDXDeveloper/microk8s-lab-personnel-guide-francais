🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Présentation des addons MicroK8s

## Vue d'ensemble du système d'addons

MicroK8s propose un écosystème riche d'addons qui transforment un cluster Kubernetes basique en une plateforme complète et opérationnelle. Ces addons sont des modules prépackagés, testés et optimisés qui s'installent en une seule commande, éliminant la complexité traditionnelle de configuration manuelle des services Kubernetes. Pour un débutant, c'est comme avoir une bibliothèque d'applications prêtes à l'emploi qui s'intègrent parfaitement à votre cluster.

Le système d'addons de MicroK8s se distingue par sa simplicité d'utilisation tout en offrant une flexibilité professionnelle. Chaque addon est maintenu et mis à jour régulièrement par Canonical et la communauté, garantissant compatibilité et sécurité. Cette approche modulaire permet de construire progressivement votre environnement, en ajoutant des fonctionnalités au fur et à mesure de vos besoins.

## Commandes essentielles de gestion

La gestion des addons s'effectue principalement via trois commandes simples qui constituent la base de votre interaction avec le système d'addons.

La commande `microk8s status` affiche l'état général de votre cluster et liste tous les addons actuellement activés. Cette commande est votre point de départ pour comprendre la configuration actuelle de votre environnement. Elle indique également si des addons sont disponibles mais non encore installés.

Pour consulter la liste complète des addons disponibles, utilisez `microk8s status --wait-ready`. Cette commande affiche non seulement les addons core maintenus officiellement, mais aussi les addons communautaires qui peuvent enrichir votre installation. Chaque addon est accompagné d'une brève description de sa fonction.

L'activation d'un addon s'effectue avec `microk8s enable <nom-addon>`. Cette commande déclenche le téléchargement, l'installation et la configuration automatique de tous les composants nécessaires. Certains addons acceptent des paramètres de configuration qui peuvent être spécifiés directement lors de l'activation, par exemple `microk8s enable storage:capacity=50Gi`.

La désactivation suit la même logique avec `microk8s disable <nom-addon>`. Cette opération supprime les ressources Kubernetes associées à l'addon tout en préservant, dans certains cas, les données pour une réactivation ultérieure.

## Catégorisation détaillée des addons

Les addons MicroK8s se répartissent en plusieurs catégories fonctionnelles, chacune répondant à des besoins spécifiques de votre infrastructure Kubernetes.

**Addons Core (Essentiels)** forment la base de tout cluster fonctionnel. Ces addons sont considérés comme fondamentaux et sont souvent interdépendants. Le DNS (CoreDNS) permet la résolution de noms à l'intérieur du cluster, rendant possible la communication entre services par leur nom plutôt que par adresse IP. Le Dashboard offre une interface graphique web pour visualiser et gérer votre cluster sans ligne de commande. L'addon Storage fournit un provisionnement dynamique de volumes persistants, essentiel pour les applications nécessitant de stocker des données. RBAC (Role-Based Access Control) gère les permissions et la sécurité d'accès aux ressources du cluster.

**Addons Réseau et Connectivité** gèrent l'exposition et l'accès à vos applications. L'Ingress Controller, généralement basé sur NGINX, permet de router le trafic HTTP/HTTPS externe vers vos services internes en fonction de règles sophistiquées. MetalLB résout le problème des LoadBalancers dans un environnement bare-metal en attribuant des adresses IP externes à vos services. L'addon DNS peut être étendu avec des configurations personnalisées pour intégrer votre cluster à votre infrastructure DNS existante. Istio ou Linkerd apportent des capacités de service mesh pour une gestion avancée du trafic inter-services.

**Addons Observabilité** vous donnent une visibilité complète sur le fonctionnement de votre cluster. L'addon Metrics-Server collecte les métriques de base CPU et mémoire, prérequis pour l'autoscaling. L'observabilité complète peut être obtenue via l'addon Observability qui déploie une stack complète avec Prometheus pour les métriques, Grafana pour la visualisation, et Loki pour l'agrégation des logs. Ces outils sont préconfigurés avec des dashboards pertinents pour Kubernetes.

**Addons Stockage et Données** enrichissent les capacités de persistance. L'addon Registry fournit un registre Docker privé intégré au cluster, idéal pour stocker vos images personnalisées sans dépendre de services externes. OpenEBS offre des options de stockage distribué avancées. MinIO apporte un stockage objet compatible S3 directement dans votre cluster. L'addon Backup facilite les sauvegardes automatisées de vos volumes persistants.

**Addons Développement** accélèrent votre workflow de développement. L'addon Registry mentionné précédemment est fondamental pour le développement local. Knative permet le déploiement d'applications serverless. L'addon GPU active le support des cartes graphiques pour les workloads d'intelligence artificielle. ArgoCD peut être installé pour implémenter des pratiques GitOps.

**Addons Sécurité** renforcent la protection de votre cluster. Cert-Manager automatise la gestion et le renouvellement des certificats SSL/TLS, s'intégrant parfaitement avec Let's Encrypt pour des certificats gratuits et reconnus. L'addon Fluentd centralise la collecte et le forwarding des logs. Les politiques de sécurité peuvent être renforcées avec des addons comme OPA (Open Policy Agent).

## Architecture technique des addons

Comprendre l'architecture sous-jacente des addons vous aidera à mieux diagnostiquer les problèmes et optimiser votre cluster. Chaque addon est essentiellement un ensemble de manifestes Kubernetes qui définissent des deployments, services, configmaps, et autres ressources nécessaires.

Lorsqu'un addon est activé, MicroK8s exécute une série d'actions orchestrées. D'abord, il vérifie les prérequis et les dépendances. Ensuite, il applique les manifestes Kubernetes correspondants, généralement dans un namespace dédié pour maintenir l'isolation. Les configurations spécifiques sont générées dynamiquement en fonction de votre environnement. Enfin, des health checks vérifient que tous les composants sont opérationnels.

Les addons utilisent souvent des namespaces dédiés pour leurs composants. Par exemple, l'addon Ingress déploie ses ressources dans le namespace `ingress`, le Dashboard dans `kube-system`, et Cert-Manager dans `cert-manager`. Cette séparation facilite la gestion et le troubleshooting en isolant logiquement les différents composants.

## Gestion des dépendances entre addons

Les addons MicroK8s entretiennent souvent des relations de dépendance qu'il est crucial de comprendre pour éviter les problèmes de configuration. Ces dépendances peuvent être explicites, où un addon refuse de s'installer sans ses prérequis, ou implicites, où la fonctionnalité complète nécessite d'autres addons.

L'addon DNS constitue une dépendance fondamentale pour de nombreux autres addons. Sans résolution DNS fonctionnelle, les services ne peuvent pas communiquer efficacement par nom. L'addon Storage est souvent requis pour les applications nécessitant de la persistance, incluant certains addons comme Registry ou les bases de données. L'Ingress Controller bénéficie grandement de Cert-Manager pour une gestion automatisée des certificats HTTPS.

Certaines combinaisons d'addons créent des synergies puissantes. Par exemple, activer ensemble Ingress, Cert-Manager et DNS externe permet d'exposer automatiquement vos applications avec des certificats SSL valides. La stack d'observabilité complète (Prometheus, Grafana, Loki) offre une vue unifiée de vos métriques, logs et traces.

## Considérations de ressources et performances

Chaque addon consomme des ressources système, même au repos. Pour un lab personnel avec des ressources limitées, il est essentiel de comprendre l'impact de chaque addon sur votre système.

Les addons légers comme DNS ou Storage ont une empreinte minimale, généralement moins de 100MB de RAM. Les addons de taille moyenne comme Ingress ou Dashboard consomment entre 100-300MB. Les addons lourds comme la stack d'observabilité complète ou Istio peuvent nécessiter plusieurs gigaoctets de RAM et plusieurs cœurs CPU.

Sur une machine avec 4GB de RAM, limitez-vous aux addons essentiels comme DNS, Storage et peut-être Dashboard. Avec 8GB, vous pouvez confortablement ajouter Ingress, Cert-Manager et quelques services. Pour une expérience complète avec observabilité et multiples services, 16GB ou plus sont recommandés.

La gestion dynamique des addons permet d'optimiser l'utilisation des ressources. Vous pouvez activer temporairement des addons pour des besoins spécifiques puis les désactiver. Par exemple, activer la stack d'observabilité uniquement lors de sessions de debugging ou d'analyse de performance.

## Versioning et mises à jour

Les addons MicroK8s suivent un cycle de vie aligné sur les versions de MicroK8s et Kubernetes. Chaque version de MicroK8s est testée avec des versions spécifiques de chaque addon, garantissant la compatibilité et la stabilité.

Les mises à jour des addons sont généralement effectuées lors de la mise à jour de MicroK8s lui-même via `snap refresh microk8s`. Cette approche garantit que tous les composants restent synchronisés et compatibles. Dans certains cas, des addons peuvent être mis à jour indépendamment, mais cela doit être fait avec précaution pour éviter les incompatibilités.

Il est recommandé de suivre les channels de release de MicroK8s. Le channel `stable` offre des versions éprouvées et testées, idéales pour un lab de production personnelle. Le channel `edge` propose les dernières fonctionnalités mais peut présenter des instabilités. Le channel `candidate` offre un compromis entre nouveautés et stabilité.

## Personnalisation et extension

Bien que les addons MicroK8s soient conçus pour fonctionner out-of-the-box, ils offrent de nombreuses possibilités de personnalisation. Cette flexibilité permet d'adapter votre cluster à vos besoins spécifiques sans sacrifier la simplicité d'utilisation.

La personnalisation peut s'effectuer à plusieurs niveaux. Au moment de l'activation, de nombreux addons acceptent des paramètres de configuration. Après l'installation, vous pouvez modifier les ConfigMaps et Deployments créés par les addons en utilisant kubectl. Pour des personnalisations plus avancées, vous pouvez créer vos propres overlays Kustomize ou Helm values.

La création d'addons personnalisés est également possible pour les utilisateurs avancés. Un addon MicroK8s est essentiellement un script d'installation et un ensemble de manifestes Kubernetes. Vous pouvez créer vos propres addons pour automatiser le déploiement de vos stacks applicatives favorites ou de configurations spécifiques à votre organisation.

## Troubleshooting et diagnostic

La résolution des problèmes liés aux addons commence par une compréhension de leur état et de leur fonctionnement. La commande `microk8s kubectl get all -A` offre une vue globale de toutes les ressources dans tous les namespaces, permettant d'identifier rapidement les composants en erreur.

Les logs des addons sont accessibles via kubectl logs, en ciblant les pods dans leurs namespaces respectifs. Par exemple, pour diagnostiquer des problèmes d'Ingress, examinez les logs du pod nginx-ingress-controller dans le namespace ingress. La commande `microk8s inspect` génère un rapport détaillé de l'état du cluster, utile pour le diagnostic approfondi.

Les problèmes courants incluent les conflits de ports, particulièrement pour les addons exposant des services sur les ports standard (80, 443). Les limitations de ressources peuvent causer des redémarrages fréquents des pods. Les problèmes de réseau peuvent empêcher la communication entre les composants des addons. Une approche méthodique, commençant par vérifier l'état des pods, puis les logs, et enfin les événements Kubernetes, résout la majorité des problèmes.

## Meilleures pratiques d'utilisation

L'adoption de bonnes pratiques dès le début vous évitera de nombreux problèmes et optimisera votre expérience avec les addons MicroK8s.

Documentez toujours les addons activés et leurs configurations personnalisées. Maintenez un fichier README ou un script d'initialisation qui liste les commandes d'activation des addons avec leurs paramètres. Cela facilitera la reconstruction de votre environnement si nécessaire.

Testez les addons dans un ordre logique, en commençant par les plus fondamentaux. Activez d'abord DNS et Storage, vérifiez leur bon fonctionnement, puis ajoutez progressivement d'autres addons. Cette approche incrémentale facilite l'identification des problèmes.

Surveillez régulièrement l'utilisation des ressources avec `microk8s kubectl top nodes` et `microk8s kubectl top pods -A`. Cela vous aidera à identifier les addons gourmands en ressources et à ajuster votre configuration en conséquence.

Gardez une stratégie de backup pour les configurations importantes. Bien que les addons soient faciles à réinstaller, vos configurations personnalisées et les données associées doivent être sauvegardées régulièrement.

## Évolution et roadmap

L'écosystème des addons MicroK8s évolue constamment, avec de nouveaux addons ajoutés régulièrement et des améliorations apportées aux existants. La communauté joue un rôle crucial dans cette évolution, proposant de nouveaux addons et améliorant ceux existants.

Les tendances actuelles montrent un focus croissant sur l'observabilité, avec des addons de plus en plus sophistiqués pour le monitoring et le tracing. La sécurité reste une priorité avec l'amélioration continue des addons liés à l'authentification, l'autorisation et le chiffrement. L'intégration avec les outils de CI/CD et les pratiques GitOps continue de s'enrichir.

Pour rester informé des nouveautés, suivez les release notes de MicroK8s, participez aux forums communautaires, et explorez régulièrement les nouveaux addons disponibles. La flexibilité du système d'addons garantit que votre lab MicroK8s restera moderne et adapté à vos besoins évolutifs.

⏭️
