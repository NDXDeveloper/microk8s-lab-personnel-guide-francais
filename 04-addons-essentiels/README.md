🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 04. Addons Essentiels

## Introduction

Les addons constituent l'un des points forts majeurs de MicroK8s, transformant un cluster Kubernetes minimaliste en une plateforme complète et fonctionnelle pour votre lab personnel. Contrairement à une installation Kubernetes traditionnelle où chaque composant doit être installé et configuré manuellement, MicroK8s propose un système d'addons qui simplifie considérablement le déploiement des fonctionnalités essentielles.

## Qu'est-ce qu'un Addon MicroK8s ?

Un addon MicroK8s est un composant pré-configuré et optimisé qui peut être activé ou désactivé avec une simple commande. Ces modules étendent les capacités de base de votre cluster en ajoutant des fonctionnalités critiques comme le stockage persistant, la gestion du réseau, l'observabilité, ou encore la sécurité. Chaque addon est maintenu par l'équipe Canonical et la communauté, garantissant une intégration harmonieuse et des mises à jour régulières.

## Pourquoi les Addons sont Essentiels pour un Lab Personnel

Dans le contexte d'un laboratoire personnel, les addons MicroK8s offrent plusieurs avantages décisifs. D'abord, ils permettent de gagner un temps considérable en évitant la configuration manuelle complexe de chaque service. Ensuite, ils garantissent une cohérence dans les versions et les configurations, réduisant ainsi les problèmes de compatibilité. Enfin, leur nature modulaire permet d'activer uniquement les fonctionnalités nécessaires, optimisant ainsi l'utilisation des ressources de votre machine.

## Architecture et Fonctionnement

Les addons MicroK8s fonctionnent selon un principe d'encapsulation intelligente. Lorsqu'un addon est activé, MicroK8s déploie automatiquement les ressources Kubernetes nécessaires (deployments, services, configmaps, etc.) dans des namespaces dédiés. Cette approche garantit une isolation claire entre les composants système et vos applications, facilitant la maintenance et le dépannage.

La gestion des addons s'effectue via la commande `microk8s enable` suivie du nom de l'addon. Cette simplicité masque une orchestration sophistiquée qui configure automatiquement les certificats, les permissions RBAC, les volumes de stockage et les interconnexions entre services. Par exemple, activer l'addon DNS configure non seulement CoreDNS mais aussi l'intègre automatiquement avec le réseau du cluster et les autres addons qui en dépendent.

## Catégories d'Addons

Les addons MicroK8s peuvent être regroupés en plusieurs catégories fonctionnelles, chacune répondant à des besoins spécifiques de votre lab :

**Infrastructure de base** : Ces addons forment le socle technique de votre cluster. Ils incluent la gestion DNS pour la résolution de noms internes, le stockage persistant pour les données applicatives, et les contrôleurs réseau pour la communication entre pods.

**Exposition et routage** : Cette catégorie regroupe les addons permettant d'exposer vos applications au monde extérieur. L'Ingress Controller gère le routage HTTP/HTTPS intelligent, MetalLB fournit des adresses IP externes dans un environnement bare-metal, et Cert-Manager automatise la gestion des certificats SSL/TLS.

**Observabilité et monitoring** : Pour maintenir la santé de votre cluster, ces addons offrent des capacités de surveillance, de métriques et de logging. Le dashboard Kubernetes fournit une interface visuelle, tandis que les intégrations Prometheus permettent une observabilité approfondie.

**Développement et CI/CD** : Ces addons facilitent le cycle de développement avec un registry privé pour vos images Docker, des outils de déploiement automatisé, et des environnements de test isolés.

## Stratégie de Sélection des Addons

Le choix des addons à activer dépend directement de vos objectifs et des contraintes de votre environnement. Pour un lab de développement, privilégiez les addons qui accélèrent le cycle de développement comme le registry local et le dashboard. Pour un environnement de production domestique hébergeant des services personnels, concentrez-vous sur la fiabilité avec le stockage persistant, les sauvegardes et le monitoring.

Il est important de considérer l'impact sur les ressources système. Chaque addon consomme de la mémoire et du CPU, même au repos. Sur une machine avec des ressources limitées (moins de 8GB de RAM), soyez sélectif et activez uniquement les addons essentiels. Vous pouvez toujours les activer et désactiver selon vos besoins ponctuels.

## Interdépendances et Ordre d'Installation

Certains addons dépendent d'autres pour fonctionner correctement. Par exemple, de nombreux addons nécessitent que DNS soit activé pour la résolution de noms internes. L'Ingress Controller bénéficie grandement de Cert-Manager pour la gestion automatique des certificats. Comprendre ces dépendances vous aidera à planifier l'ordre d'activation et à éviter les problèmes de configuration.

La séquence d'installation recommandée commence généralement par les addons d'infrastructure (DNS, storage), suivis des addons réseau (Ingress, MetalLB), puis des addons de sécurité (Cert-Manager), et enfin des addons d'observabilité et de développement selon vos besoins.

## Gestion du Cycle de Vie

Les addons MicroK8s suivent un cycle de vie simple mais important à comprendre. L'activation d'un addon déploie ses composants et les configure automatiquement. La désactivation supprime ces composants mais peut, dans certains cas, conserver des données ou configurations pour une réactivation future. Les mises à jour des addons sont généralement gérées lors des mises à jour de MicroK8s lui-même, garantissant la cohérence du système.

Il est crucial de comprendre que désactiver un addon peut impacter les applications qui en dépendent. Par exemple, désactiver l'addon storage alors que des applications utilisent des volumes persistants causera des dysfonctionnements. Planifiez toujours les modifications d'addons pendant des fenêtres de maintenance appropriées.

## Personnalisation et Configuration Avancée

Bien que les addons MicroK8s soient conçus pour fonctionner avec des configurations par défaut optimales, ils offrent souvent des options de personnalisation. Ces options peuvent être spécifiées lors de l'activation ou modifiées après coup via les ConfigMaps et les ressources Kubernetes standard. Par exemple, l'addon Ingress permet de personnaliser les paramètres NGINX, tandis que l'addon storage permet de définir différentes classes de stockage.

La personnalisation doit être effectuée avec prudence, en documentant tous les changements apportés aux configurations par défaut. Cela facilitera le dépannage futur et les migrations éventuelles vers d'autres environnements.

## Monitoring et Troubleshooting des Addons

Surveiller l'état et les performances des addons est essentiel pour maintenir un lab stable. La commande `microk8s status` fournit une vue d'ensemble rapide des addons activés. Pour un diagnostic plus approfondi, utilisez les commandes kubectl standard pour examiner les pods, services et logs dans les namespaces dédiés aux addons.

Les problèmes courants incluent les conflits de ports, les limitations de ressources, et les problèmes de réseau. Chaque addon a ses propres patterns de logs et métriques qu'il est important d'apprendre à interpréter. La documentation officielle de chaque addon constitue une ressource précieuse pour le dépannage.

## Préparation à l'Exploration Détaillée

Dans les sections suivantes, nous examinerons en détail chaque addon essentiel, en commençant par une présentation générale de l'écosystème des addons MicroK8s. Vous apprendrez comment installer, configurer et optimiser chaque composant pour créer un lab Kubernetes professionnel et performant. Que votre objectif soit l'apprentissage, le développement ou l'hébergement de services personnels, la maîtrise des addons MicroK8s sera la clé de votre succès.

⏭️
