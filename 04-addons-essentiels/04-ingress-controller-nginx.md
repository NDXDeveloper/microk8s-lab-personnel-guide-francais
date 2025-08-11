🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 Ingress Controller (NGINX)

## Introduction à l'Ingress et son rôle fondamental

L'Ingress Controller représente la porte d'entrée intelligente de votre cluster Kubernetes, transformant le trafic HTTP et HTTPS externe en routes précises vers vos services internes. Imaginez-le comme un concierge sophistiqué d'un grand hôtel qui non seulement accueille les visiteurs, mais les dirige exactement vers la bonne chambre basée sur leur demande, tout en gérant la sécurité, les certificats d'identité, et même la répartition de charge quand plusieurs chemins sont possibles.

Dans l'écosystème Kubernetes, exposer des applications au monde extérieur peut rapidement devenir complexe. Sans Ingress, chaque service nécessiterait soit un LoadBalancer coûteux, soit un NodePort avec ses limitations de ports et de gestion. L'Ingress Controller résout élégamment ce problème en centralisant la gestion du trafic entrant à travers un point d'entrée unique et configurable. Pour un lab personnel MicroK8s, c'est la différence entre jongler avec des dizaines de ports et avoir une architecture propre et professionnelle.

NGINX Ingress Controller est devenu le standard de facto dans l'industrie, combinant la robustesse éprouvée de NGINX avec une intégration native Kubernetes. Sa popularité n'est pas accidentelle : NGINX gère aujourd'hui plus de 30% du trafic web mondial, et cette expertise se retrouve dans son implémentation Kubernetes. Dans MicroK8s, l'addon Ingress déploie automatiquement NGINX Ingress Controller avec une configuration optimisée pour votre environnement de lab.

## Architecture et composants de NGINX Ingress

Comprendre l'architecture de NGINX Ingress Controller vous permettra de mieux diagnostiquer les problèmes et d'optimiser vos configurations. Cette architecture n'est pas monolithique mais composée de plusieurs éléments travaillant en synergie.

Le cœur du système est le NGINX Ingress Controller Pod, qui contient deux composants principaux. Le premier est le contrôleur lui-même, écrit en Go, qui surveille continuellement l'API Kubernetes pour détecter les changements dans les ressources Ingress. Lorsqu'un changement est détecté, le contrôleur génère dynamiquement une nouvelle configuration NGINX et recharge le serveur web sans interruption de service. Le second composant est le serveur NGINX proprement dit, qui traite le trafic HTTP/HTTPS selon la configuration générée.

Le Controller Service expose l'Ingress Controller au monde extérieur. Dans MicroK8s, ce service est généralement configuré comme NodePort ou LoadBalancer selon votre configuration réseau. Ce service écoute sur les ports 80 et 443 par défaut, recevant tout le trafic web destiné à votre cluster. La configuration du service détermine comment le trafic externe atteint votre Ingress Controller, un aspect crucial pour l'accessibilité de vos applications.

Les ConfigMaps jouent un rôle central dans la personnalisation du comportement de NGINX. Le ConfigMap principal contient les paramètres globaux de NGINX comme les timeouts, les tailles de buffer, et les options de sécurité. Des ConfigMaps additionnels peuvent contenir des snippets de configuration personnalisés, des en-têtes HTTP à ajouter, ou des règles de réécriture complexes. Cette approche déclarative permet de versionner et de gérer la configuration comme du code.

Le système de validation admission webhook est une fonctionnalité de sécurité cruciale mais souvent méconnue. Avant qu'une ressource Ingress ne soit acceptée dans le cluster, le webhook valide sa syntaxe et sa cohérence. Cette validation préventive évite les configurations invalides qui pourraient crasher ou compromettre l'Ingress Controller. Les erreurs sont reportées immédiatement, facilitant le debugging et réduisant les temps d'arrêt.

## Installation et configuration initiale

L'activation de l'Ingress Controller dans MicroK8s illustre la philosophie de simplicité de la plateforme tout en offrant des options de personnalisation pour les utilisateurs avancés.

La commande `microk8s enable ingress` déclenche une séquence d'installation complète. MicroK8s déploie d'abord le namespace `ingress` qui contiendra tous les composants. Le Deployment de l'Ingress Controller est créé avec une configuration par défaut optimisée pour un lab personnel, incluant des limites de ressources appropriées et des health checks. Le Service est configuré pour exposer les ports 80 et 443, avec une stratégie d'exposition adaptée à votre environnement.

La configuration par défaut de MicroK8s inclut des optimisations importantes pour un environnement de développement. Les timeouts sont ajustés pour être plus tolérants aux applications en développement qui peuvent répondre lentement. La taille des buffers est configurée pour gérer efficacement les uploads de fichiers sans consommer trop de mémoire. Les logs sont configurés en mode verbose pour faciliter le debugging, tout en évitant de saturer le stockage.

L'intégration avec MetalLB, si activé, permet à l'Ingress Controller d'obtenir une véritable adresse IP externe même dans un environnement bare-metal. Cette combinaison transforme votre lab personnel en environnement quasi-production, permettant de tester des configurations réalistes. Sans MetalLB, l'Ingress utilise NodePort, accessible via l'adresse IP de votre node et les ports configurés.

La vérification post-installation est cruciale pour s'assurer que tout fonctionne correctement. La commande `microk8s kubectl get all -n ingress` montre tous les composants déployés et leur statut. Le pod de l'Ingress Controller doit être en état Running, et le service doit montrer les ports exposés. Les logs du controller, accessibles via `microk8s kubectl logs -n ingress deployment/nginx-ingress-microk8s-controller`, révèlent le processus d'initialisation et confirment que NGINX est prêt à recevoir du trafic.

## Création et gestion des ressources Ingress

La création de ressources Ingress est l'interface principale pour configurer le routage de votre trafic. Chaque ressource Ingress définit des règles qui mappent des requêtes HTTP/HTTPS vers des services backend spécifiques.

Une ressource Ingress basique définit des règles de routage basées sur le hostname et le path. Le hostname détermine quel domaine ou sous-domaine déclenchera cette règle, permettant d'héberger plusieurs applications sur la même adresse IP. Le path permet un routage plus granulaire, dirigeant différentes parties d'un site vers différents services. Cette combinaison offre une flexibilité immense pour structurer vos applications.

Les annotations sont le mécanisme principal pour personnaliser le comportement de NGINX pour chaque Ingress. Ces métadonnées key-value contrôlent des aspects spécifiques comme les méthodes d'authentification, les redirections, les limites de taux, ou les configurations SSL. Par exemple, l'annotation `nginx.ingress.kubernetes.io/rewrite-target` permet de réécrire les URLs avant de les transmettre au service backend, essentielle pour les applications qui n'écoutent pas sur le même path que leur URL publique.

La gestion des backends multiples permet de créer des architectures sophistiquées. Un seul Ingress peut router vers plusieurs services basés sur des règles complexes. Cette capacité permet d'implémenter des patterns comme les microservices, où différentes parties de votre application sont gérées par différents services. Le load balancing entre plusieurs replicas d'un service est géré automatiquement, distribuant le trafic selon les algorithmes configurés.

Les règles de fallback et les default backends garantissent une expérience utilisateur cohérente même quand les routes ne correspondent pas. Le default backend, généralement une page 404 personnalisée, est servi quand aucune règle ne matche. Cette configuration évite les erreurs NGINX brutes et permet de maintenir votre branding même dans les cas d'erreur.

## Configuration SSL/TLS et gestion des certificats

La sécurisation du trafic HTTPS est devenue non négociable dans le web moderne, et NGINX Ingress Controller offre plusieurs approches pour gérer les certificats SSL/TLS.

L'intégration avec Cert-Manager représente la solution la plus élégante pour la gestion automatique des certificats. Lorsque Cert-Manager est installé, une simple annotation sur votre Ingress déclenche la génération automatique d'un certificat Let's Encrypt. Le processus inclut la validation du domaine, la génération du certificat, et son renouvellement automatique avant expiration. Cette automation élimine la complexité traditionnelle de la gestion des certificats SSL.

Les certificats auto-signés restent utiles pour le développement et les environnements de test. NGINX Ingress peut générer automatiquement des certificats auto-signés pour tout Ingress configuré pour HTTPS. Bien que les navigateurs affichent des avertissements, cette approche permet de tester le comportement HTTPS de vos applications sans configuration complexe. La commande pour créer manuellement un certificat et l'importer comme Secret Kubernetes offre plus de contrôle sur les paramètres du certificat.

La configuration SSL avancée permet d'ajuster finement la sécurité de vos connexions. Les cipher suites peuvent être personnalisées pour respecter des standards de sécurité spécifiques ou pour maintenir la compatibilité avec des clients anciens. Les protocoles TLS supportés peuvent être restreints pour éliminer les versions obsolètes et vulnérables. L'activation de HTTP/2 améliore significativement les performances pour les clients modernes.

Le SSL passthrough est une fonctionnalité spéciale permettant de transmettre le trafic SSL directement au service backend sans décryption. Cette configuration est nécessaire pour certaines applications qui gèrent leur propre terminaison SSL ou pour des protocoles non-HTTP sur TLS. L'annotation `nginx.ingress.kubernetes.io/ssl-passthrough: "true"` active ce mode, bypassant complètement le traitement NGINX pour ces connexions.

## Routage avancé et règles de trafic

Au-delà du routage basique par hostname et path, NGINX Ingress Controller offre des capacités de routage sophistiquées pour implémenter des architectures complexes.

Le routage basé sur les headers HTTP permet de diriger le trafic selon des critères personnalisés. Cette fonctionnalité est précieuse pour implémenter des déploiements canary, où un petit pourcentage d'utilisateurs est dirigé vers une nouvelle version. Les headers comme User-Agent, Accept-Language, ou des headers personnalisés peuvent influencer le routage, permettant des tests A/B sophistiqués ou des expériences personnalisées par client.

Les redirections et réécritures d'URL sont essentielles pour maintenir la compatibilité lors de migrations ou pour simplifier les URLs publiques. NGINX Ingress supporte les redirections permanentes (301) et temporaires (302), avec des règles basées sur regex pour des transformations complexes. La réécriture d'URL permet de présenter une structure d'URL simple aux utilisateurs tout en maintenant une organisation interne différente.

Le rate limiting protège vos services contre les abus et garantit une distribution équitable des ressources. Configurable globalement ou par Ingress, le rate limiting peut être basé sur l'adresse IP client, les headers, ou d'autres critères. Les limites peuvent être définies en requêtes par seconde, minute, ou heure, avec des burst allowances pour gérer les pics légitimes de trafic.

L'affinité de session, ou sticky sessions, garantit qu'un utilisateur est toujours dirigé vers le même pod backend. Cette fonctionnalité est cruciale pour les applications stateful qui stockent des données de session en mémoire. NGINX Ingress supporte plusieurs méthodes d'affinité, incluant les cookies, l'IP source, ou des headers personnalisés, chacune avec ses avantages selon le cas d'usage.

## Monitoring et observabilité

La visibilité sur le comportement de votre Ingress Controller est essentielle pour maintenir la performance et diagnostiquer les problèmes.

Les logs NGINX fournissent une mine d'informations sur chaque requête traitée. Le format de log par défaut inclut l'IP client, la timestamp, la méthode HTTP, l'URL, le code de réponse, la taille de la réponse, et le temps de traitement. Ces logs peuvent être enrichis avec des informations additionnelles comme les headers, les backends utilisés, ou les raisons de rejet. L'intégration avec des systèmes de log centralisés comme ELK ou Loki permet une analyse sophistiquée du trafic.

Les métriques Prometheus exposées par NGINX Ingress Controller offrent une vue quantitative de la performance. Les métriques incluent le nombre de requêtes par seconde, les latences par percentile, les taux d'erreur par code HTTP, et l'utilisation des connexions. Ces métriques peuvent être visualisées dans Grafana avec des dashboards préconfigurés disponibles dans la communauté, offrant une visibilité immédiate sur la santé de votre ingress.

Le health checking actif surveille continuellement la disponibilité des backends. NGINX Ingress effectue des health checks périodiques vers les services backend, marquant automatiquement les pods non disponibles comme unhealthy et les excluant du pool de load balancing. Cette détection proactive évite d'envoyer du trafic vers des services défaillants, améliorant la fiabilité globale.

Les traces distribuées avec OpenTelemetry ou Jaeger permettent de suivre les requêtes à travers votre stack applicatif. NGINX Ingress peut injecter des headers de tracing qui sont propagés aux services backend, permettant de visualiser le chemin complet d'une requête et d'identifier les bottlenecks. Cette visibilité end-to-end est inestimable pour optimiser les performances des applications distribuées.

## Sécurité et hardening

La sécurisation de votre Ingress Controller est cruciale car il représente le point d'entrée principal vers votre cluster.

L'authentification et l'autorisation peuvent être implémentées à plusieurs niveaux. L'authentification basique HTTP, bien que simple, offre une protection rapide pour les environnements de développement. L'intégration avec OAuth2/OIDC permet une authentification moderne avec des providers comme Google, GitHub, ou Keycloak. L'autorisation peut être gérée via des annotations définissant des règles d'accès basées sur les headers, les IPs, ou d'autres critères.

La protection contre les attaques communes est intégrée dans NGINX Ingress. La protection DDoS basique via rate limiting évite la saturation de vos services. Les ModSecurity rules peuvent être activées pour une Web Application Firewall (WAF) complète, détectant et bloquant les tentatives d'injection SQL, XSS, et autres attaques OWASP Top 10. Les limites de taille de requête et les timeouts appropriés évitent les attaques par épuisement de ressources.

Les security headers améliorent significativement la posture de sécurité de vos applications. Headers comme Content-Security-Policy, X-Frame-Options, et Strict-Transport-Security peuvent être automatiquement ajoutés à toutes les réponses. Ces headers protègent contre diverses attaques côté client et sont souvent requis pour la conformité avec les standards de sécurité modernes.

L'isolation réseau via Network Policies limite l'exposition de l'Ingress Controller. Définir des politiques qui restreignent le trafic entrant et sortant de l'Ingress Controller réduit la surface d'attaque. Seuls les ports nécessaires devraient être accessibles, et l'Ingress devrait uniquement pouvoir communiquer avec les services qu'il doit router.

## Optimisation des performances

L'optimisation de NGINX Ingress Controller peut dramatiquement améliorer les performances de vos applications, particulièrement important dans un lab personnel aux ressources limitées.

Le tuning des workers NGINX adapte le serveur à votre charge de travail. Le nombre de worker processes devrait correspondre au nombre de CPU cores disponibles. Les worker connections déterminent combien de connexions simultanées chaque worker peut gérer. Pour un lab personnel, des valeurs conservatrices évitent la surconsommation de ressources tout en maintenant des performances acceptables.

La configuration du buffering influence significativement les performances et l'utilisation mémoire. Les proxy buffers stockent temporairement les réponses des backends avant de les transmettre aux clients. Des buffers trop petits causent des allers-retours fréquents, impactant les performances. Des buffers trop grands consomment de la mémoire précieuse. L'équilibre dépend de la taille typique de vos réponses et de la mémoire disponible.

Le caching au niveau de l'Ingress peut réduire drastiquement la charge sur vos backends. NGINX peut cacher les réponses statiques et même certaines réponses dynamiques selon les headers de cache. La configuration du cache inclut la taille maximale, les règles d'éviction, et les conditions de bypass. Pour les assets statiques, un cache agressif améliore les temps de réponse et réduit la consommation de ressources.

La compression GZIP des réponses réduit la bande passante utilisée et améliore les temps de chargement pour les clients. NGINX peut compresser dynamiquement les réponses textuelles (HTML, CSS, JavaScript, JSON) avec des niveaux de compression configurables. Le compromis entre CPU utilisé pour la compression et la réduction de bande passante doit être évalué selon votre environnement.

## Troubleshooting et résolution de problèmes

Le diagnostic des problèmes avec l'Ingress Controller suit une méthodologie structurée qui permet d'identifier rapidement la source des dysfonctionnements.

La vérification de base commence par examiner l'état du pod Ingress Controller. Les événements Kubernetes associés au pod révèlent souvent des problèmes de démarrage ou de ressources. Les logs du container nginx-ingress-controller montrent le processus de configuration et les erreurs de validation. Si le pod redémarre fréquemment, examiner les limites de ressources et les métriques d'utilisation.

Les problèmes de routage sont parmi les plus communs. Vérifier que la ressource Ingress est correctement créée et que ses règles correspondent à vos attentes. L'annotation `nginx.ingress.kubernetes.io/configuration-snippet` permet d'injecter des directives NGINX personnalisées pour le debugging. Les logs d'accès NGINX montrent exactement comment les requêtes sont traitées et vers quels backends elles sont dirigées.

Les erreurs SSL/TLS peuvent avoir plusieurs origines. Vérifier que les certificats sont correctement montés comme Secrets et référencés dans l'Ingress. Les logs NGINX révèlent les erreurs de handshake SSL et les problèmes de certificat. L'outil `openssl s_client` permet de tester la connexion SSL depuis l'extérieur et d'identifier les problèmes de chaîne de certificats ou de compatibilité.

Les problèmes de performance se manifestent souvent par des timeouts ou des réponses lentes. Les métriques NGINX révèlent si le problème vient de l'Ingress lui-même ou des backends. Les logs montrent les temps de réponse pour chaque requête. Augmenter temporairement les timeouts peut aider à identifier si le problème est lié au temps de traitement ou à d'autres facteurs.

## Cas d'usage et patterns architecturaux

L'Ingress Controller permet d'implémenter des architectures sophistiquées qui seraient complexes à réaliser autrement.

Le pattern API Gateway utilise l'Ingress comme point d'entrée unique pour tous les microservices. L'Ingress gère l'authentification, le rate limiting, et le routage vers les services appropriés basé sur les paths. Cette centralisation simplifie la gestion de la sécurité et du monitoring tout en offrant une interface cohérente aux clients.

Les déploiements Blue-Green utilisent l'Ingress pour basculer instantanément entre deux versions d'une application. Deux deployments identiques mais avec des versions différentes coexistent, et l'Ingress dirige le trafic vers l'un ou l'autre. Cette approche permet des rollbacks instantanés et des déploiements sans downtime.

Le multi-tenancy peut être implémenté en utilisant différents Ingress ou règles pour différents tenants. Chaque tenant obtient son propre sous-domaine ou path, avec potentiellement différentes configurations de sécurité et de rate limiting. Cette isolation logique permet de servir plusieurs clients depuis le même cluster.

L'hébergement de sites statiques via l'Ingress est étonnamment efficace. En combinant NGINX Ingress avec un simple serveur web comme nginx ou apache servant des fichiers statiques, vous pouvez héberger des sites web performants avec caching agressif et compression automatique.

## Intégration avec l'écosystème Kubernetes

L'Ingress Controller ne fonctionne pas en isolation mais s'intègre profondément avec d'autres composants Kubernetes.

L'intégration avec Service Mesh comme Istio ou Linkerd étend les capacités de l'Ingress. Le mesh gère le trafic interne tandis que l'Ingress gère l'entrée externe. Cette combinaison offre une observabilité complète, des politiques de sécurité uniformes, et des capacités de traffic management avancées.

La coordination avec Horizontal Pod Autoscaler (HPA) assure que vos backends peuvent gérer la charge entrante. L'Ingress distribue le trafic, HPA scale les backends selon la demande, créant un système auto-adaptatif. Les métriques de l'Ingress peuvent même être utilisées pour déclencher le scaling, créant une boucle de feedback complète.

L'utilisation avec Persistent Volumes pour servir du contenu statique volumineux optimise les performances. Les fichiers peuvent être montés directement dans le pod Ingress, évitant le proxy vers un backend pour le contenu statique. Cette approche est particulièrement efficace pour les assets média ou les downloads volumineux.

## Evolution et tendances futures

L'écosystème Ingress continue d'évoluer rapidement avec de nouvelles fonctionnalités et standards.

La Gateway API représente l'évolution future de l'Ingress, offrant une API plus expressive et extensible. Bien que toujours en développement, elle promet une meilleure séparation des concerns et plus de flexibilité. NGINX Ingress Controller supporte déjà partiellement cette nouvelle API, permettant une migration progressive.

Le support HTTP/3 et QUIC améliora significativement les performances, particulièrement pour les connexions mobiles ou instables. Ces protocoles modernes réduisent la latence et améliorent la fiabilité. L'implémentation dans NGINX Ingress est en cours et sera bientôt disponible.

L'intelligence artificielle pour l'optimisation automatique du routage et de la sécurité est explorée. L'analyse des patterns de trafic pourrait automatiquement ajuster les règles de cache, de rate limiting, et même détecter des anomalies de sécurité. Cette évolution transformerait l'Ingress d'un routeur configurable en système adaptatif intelligent.

⏭️
