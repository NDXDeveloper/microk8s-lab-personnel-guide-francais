🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.7 MetalLB (load balancer)

## Introduction au concept de Load Balancer

Le Load Balancer représente l'un des concepts les plus puissants mais aussi les plus mal compris de Kubernetes, particulièrement dans les environnements bare-metal comme votre lab MicroK8s. Dans les environnements cloud comme AWS, GCP ou Azure, créer un Service de type LoadBalancer déclenche automatiquement le provisionnement d'un équilibreur de charge cloud avec une adresse IP publique. Cette magie apparente cache une infrastructure complexe et coûteuse que les providers cloud gèrent pour vous. Dans un environnement bare-metal ou on-premises, cette fonctionnalité n'existe pas nativement, laissant vos Services LoadBalancer dans un état perpétuel "Pending" frustrant.

MetalLB comble élégamment ce fossé en apportant les capacités de load balancing aux clusters Kubernetes bare-metal. Son nom, une contraction de "Metal" (bare-metal) et "LB" (Load Balancer), reflète parfaitement sa mission : fournir une implémentation de load balancer pour les environnements sans infrastructure cloud. Pour votre lab personnel MicroK8s, MetalLB transforme votre cluster d'un environnement limité aux NodePorts et Ingress en une plateforme capable d'exposer des services avec de vraies adresses IP, exactement comme dans le cloud.

La beauté de MetalLB réside dans sa simplicité conceptuelle masquant une implémentation sophistiquée. Il prend un pool d'adresses IP que vous lui allouez et les distribue aux Services demandant des LoadBalancers. Ces adresses peuvent être de votre réseau local, permettant à vos services d'être accessibles directement depuis n'importe quel dispositif sur votre réseau domestique ou lab. Cette capacité ouvre des possibilités infinies, depuis l'hébergement de services personnels accessibles dans votre maison jusqu'à la simulation fidèle d'architectures de production cloud.

## Architecture et modes de fonctionnement

MetalLB n'est pas une solution monolithique mais un système modulaire offrant deux modes de fonctionnement distincts, chacun adapté à différents environnements et contraintes réseau.

Le mode Layer 2 (L2) est le plus simple et le plus couramment utilisé dans les labs personnels. Dans ce mode, MetalLB répond aux requêtes ARP (Address Resolution Protocol) pour les adresses IP qu'il gère, faisant croire au réseau que ces adresses appartiennent à l'un des nodes de votre cluster. Quand un client veut accéder à un service, il envoie une requête ARP demandant "qui a cette adresse IP ?", et MetalLB répond "c'est moi !" depuis l'un des nodes. Ce mode ne nécessite aucune configuration réseau spéciale, aucun support de votre routeur ou switch, et fonctionne sur pratiquement n'importe quel réseau Ethernet standard.

Le mode BGP (Border Gateway Protocol) est plus sophistiqué et s'intègre avec l'infrastructure réseau existante en utilisant le protocole de routage standard d'Internet. Dans ce mode, MetalLB établit des sessions BGP avec vos routeurs et annonce les routes vers les adresses IP des services. Cette approche permet une véritable haute disponibilité avec plusieurs chemins réseau et une convergence rapide en cas de défaillance. Bien que plus complexe à configurer et nécessitant des routeurs compatibles BGP, ce mode offre des performances et une résilience supérieures, particulièrement pertinentes si votre lab simule des environnements de production.

L'architecture interne de MetalLB comprend deux composants principaux travaillant en harmonie. Le controller est un deployment qui surveille les Services Kubernetes et assigne des adresses IP depuis les pools configurés. Il gère la comptabilité des adresses, s'assurant qu'aucune adresse n'est assignée deux fois et que les adresses sont récupérées quand les services sont supprimés. Le speaker est un DaemonSet qui s'exécute sur chaque node, responsable d'annoncer les adresses IP assignées selon le mode configuré (L2 ou BGP). Cette séparation des responsabilités garantit la scalabilité et la résilience.

## Installation et configuration dans MicroK8s

L'activation de MetalLB dans MicroK8s illustre parfaitement la philosophie de simplicité de la plateforme tout en offrant la flexibilité nécessaire pour s'adapter à votre réseau spécifique.

La commande d'activation `microk8s enable metallb` déclenche une installation complète mais nécessite immédiatement une décision importante : quel range d'adresses IP allouer à MetalLB. MicroK8s vous demandera interactivement de spécifier ce range, typiquement sous la forme `192.168.1.200-192.168.1.220` ou similaire. Ce choix nécessite une compréhension de votre réseau local pour éviter les conflits avec des adresses existantes. Une approche prudente consiste à réserver une petite plage d'adresses en dehors de la portée DHCP de votre routeur mais dans le même subnet que vos machines.

La configuration automatique créée par MicroK8s déploie MetalLB en mode Layer 2 par défaut, optimal pour la majorité des labs personnels. Le namespace `metallb-system` est créé pour isoler tous les composants MetalLB. Les ConfigMaps contenant la configuration sont générés avec votre range d'IP spécifié. Les déploiements du controller et du speaker sont créés avec les bonnes permissions RBAC et les contraintes de sécurité appropriées. Cette configuration par défaut est production-ready tout en restant simple à comprendre et modifier.

La personnalisation post-installation permet d'adapter MetalLB à des besoins spécifiques. Modifier le ConfigMap `config` dans le namespace `metallb-system` permet d'ajouter des pools d'adresses supplémentaires, de configurer le mode BGP si nécessaire, ou d'implémenter des politiques d'allocation avancées. Par exemple, vous pouvez créer différents pools pour différents types de services : un pool pour les services de développement, un autre pour les bases de données, et un troisième pour les services exposés publiquement.

La vérification de l'installation s'effectue en plusieurs étapes. D'abord, confirmer que les pods MetalLB sont en état Running avec `microk8s kubectl get pods -n metallb-system`. Ensuite, créer un Service de type LoadBalancer simple et vérifier qu'il obtient une External IP depuis votre range configuré. Finalement, tester que cette IP est accessible depuis d'autres machines de votre réseau. Cette validation complète garantit que MetalLB fonctionne correctement dans votre environnement.

## Gestion des pools d'adresses

La gestion efficace des pools d'adresses IP est cruciale pour maximiser l'utilité de MetalLB tout en évitant les conflits réseau qui pourraient déstabiliser votre lab.

La planification de l'allocation d'adresses nécessite une vue d'ensemble de votre réseau. Documenter les plages utilisées par votre DHCP, les adresses statiques existantes, et les réservations futures évite les conflits. Dans un réseau domestique typique 192.168.1.0/24, le routeur est souvent à .1, le DHCP alloue .100-.199, laissant .200-.254 disponibles pour MetalLB. Cette organisation claire facilite le troubleshooting et l'expansion future.

Les stratégies de segmentation des pools permettent d'organiser logiquement vos services. Créer des pools séparés pour différents environnements (dev, staging, prod) même dans un lab personnel inculque de bonnes pratiques. Les pools peuvent également être segmentés par criticité : services essentiels avec des IPs stables versus services expérimentaux avec des IPs temporaires. Cette organisation devient particulièrement précieuse quand votre lab grandit et héberge de multiples projets.

L'allocation automatique versus manuelle offre différents niveaux de contrôle. Par défaut, MetalLB assigne automatiquement la prochaine adresse disponible du pool, simplifiant la gestion. Pour les services critiques nécessitant des adresses spécifiques, l'annotation `metallb.universe.tf/loadBalancerIPs` permet de demander une adresse précise. Cette flexibilité accommode à la fois la simplicité du développement rapide et les besoins de production avec des DNS et firewall rules configurés.

La gestion du cycle de vie des adresses inclut la récupération des adresses quand les services sont supprimés. MetalLB libère immédiatement les adresses pour réutilisation, mais les caches ARP sur le réseau peuvent causer des délais avant que la nouvelle allocation soit pleinement fonctionnelle. Comprendre ces délais évite la confusion lors du redéploiement rapide de services. Dans des cas extrêmes, forcer un flush du cache ARP sur les clients peut accélérer la convergence.

## Mode Layer 2 en détail

Le mode Layer 2 est le pain quotidien de MetalLB dans les labs personnels, méritant une compréhension approfondie de son fonctionnement et de ses implications.

Le mécanisme ARP sous-jacent est élégamment simple mais puissant. Quand MetalLB assigne une adresse IP à un service, le speaker sur l'un des nodes devient responsable de répondre aux requêtes ARP pour cette adresse. Du point de vue du réseau, cette adresse IP appartient maintenant à ce node. Le trafic destiné au service arrive sur ce node, où kube-proxy le route vers les pods appropriés, potentiellement sur d'autres nodes. Cette approche ne nécessite aucune modification de l'infrastructure réseau existante.

L'élection du leader détermine quel node speaker répond aux requêtes ARP pour chaque service. MetalLB utilise un algorithme d'élection basé sur Kubernetes pour choisir un leader par service. Si le node leader devient indisponible, un nouveau leader est automatiquement élu et commence à répondre aux requêtes ARP. Cette transition prend typiquement quelques secondes, pendant lesquelles le service peut être inaccessible. Pour un lab personnel single-node, cette complexité est invisible, mais devient importante en multi-node.

Les limitations du mode L2 doivent être comprises pour des attentes réalistes. Tout le trafic pour un service passe par un seul node, créant un potentiel bottleneck et single point of failure. La bande passante est limitée à celle d'un seul node. Le failover n'est pas instantané, prenant quelques secondes pour la convergence ARP. Malgré ces limitations, le mode L2 reste excellent pour les labs personnels où la simplicité prime sur la haute disponibilité parfaite.

L'optimisation des performances en mode L2 implique de considérer la topologie réseau. Placer les services high-traffic sur des nodes avec la meilleure connectivité réseau améliore les performances. Utiliser l'anti-affinité pour distribuer différents services LoadBalancer sur différents nodes équilibre la charge réseau. Pour les services critiques, considérer l'utilisation de multiples Services LoadBalancer avec DNS round-robin pour une forme basique de load distribution.

## Mode BGP pour les cas avancés

Le mode BGP de MetalLB ouvre des possibilités avancées pour les labs sophistiqués simulant des environnements de production réalistes.

La configuration BGP nécessite une infrastructure réseau compatible, typiquement un routeur supportant BGP comme pfSense, VyOS, ou même un routeur entreprise Cisco/Juniper. La configuration implique l'établissement de peering sessions entre MetalLB et votre routeur, définissant les AS numbers, les neighbor relationships, et les politiques de routage. Cette complexité initiale est récompensée par des capacités de routage sophistiquées impossibles en mode L2.

Les avantages du mode BGP incluent le true load balancing avec ECMP (Equal Cost Multi-Path), où le trafic est distribué entre plusieurs nodes au niveau réseau. La convergence est quasi-instantanée lors des failovers, maintenant la disponibilité du service. L'intégration avec l'infrastructure réseau existante permet des topologies complexes avec multiple hops et routage policy-based. Ces capacités transforment votre lab en environnement quasi-production.

Les topologies multi-homed deviennent possibles avec BGP, permettant d'annoncer les mêmes services sur plusieurs réseaux ou via plusieurs ISPs. Cette capacité est particulièrement précieuse pour simuler des architectures géo-distribuées ou multi-datacenter dans votre lab. Les route maps et communities BGP permettent un contrôle fin du routage, implémentant des stratégies sophistiquées comme le routage préférentiel ou le traffic engineering.

Le debugging BGP nécessite des outils et connaissances spécifiques. Les commandes comme `birdc show protocol` dans le pod speaker révèlent l'état des sessions BGP. Les logs détaillés montrent l'établissement des sessions et les annonces de routes. Wireshark peut capturer et analyser le trafic BGP pour un debugging approfondi. Cette complexité est le prix de la sophistication, mais les compétences acquises sont directement transférables aux environnements de production.

## Intégration avec les Services Kubernetes

L'intégration transparente de MetalLB avec les Services Kubernetes est ce qui rend la solution si élégante et puissante.

La création de Services LoadBalancer devient triviale une fois MetalLB installé. Un simple manifeste Service avec `type: LoadBalancer` suffit pour obtenir une adresse IP externe. MetalLB surveille ces services via l'API Kubernetes, assignant automatiquement des adresses depuis les pools configurés. L'adresse assignée apparaît dans le champ `status.loadBalancer.ingress` du Service, utilisable par d'autres ressources ou pour la configuration DNS.

Les annotations de service permettent un contrôle fin du comportement de MetalLB. L'annotation `metallb.universe.tf/address-pool` spécifie quel pool utiliser pour un service particulier. L'annotation `metallb.universe.tf/allow-shared-ip` permet à plusieurs services de partager la même IP sur différents ports, utile pour grouper des services liés. Ces annotations offrent la flexibilité nécessaire pour des architectures complexes tout en gardant les cas simples simples.

La coexistence avec NodePort et ClusterIP services reste inchangée. MetalLB ne modifie que les services explicitement configurés comme LoadBalancer. Cette isolation permet une migration graduelle, testant MetalLB avec quelques services avant de l'adopter complètement. Les services peuvent être convertis entre types sans perte de données, offrant la flexibilité de changer d'approche selon l'évolution des besoins.

L'interaction avec l'Ingress Controller crée des architectures puissantes. L'Ingress Controller lui-même peut être exposé via MetalLB LoadBalancer, obtenant une vraie IP externe plutôt qu'un NodePort. Cette configuration permet un vrai routage layer 7 avec une adresse IP stable, simulant fidèlement les configurations de production cloud. Multiple Ingress Controllers peuvent avoir différentes IPs pour isoler différents domaines ou environnements.

## Monitoring et observabilité

La surveillance de MetalLB est essentielle pour maintenir la fiabilité de vos services exposés et diagnostiquer rapidement les problèmes.

Les métriques Prometheus exposées par MetalLB offrent une visibilité profonde sur son fonctionnement. Les métriques incluent le nombre d'adresses allouées par pool, l'état des sessions BGP, les statistiques de trafic par service, et les événements d'allocation/libération. Ces métriques peuvent être visualisées dans Grafana avec des dashboards communautaires disponibles, offrant une vue temps réel de votre infrastructure de load balancing.

Les logs des composants MetalLB révèlent les décisions d'allocation et les problèmes potentiels. Le controller log montre l'assignation et la libération d'adresses. Le speaker log révèle les annonces ARP/BGP et les changements de leadership. Le niveau de log peut être ajusté pour un debugging détaillé sans redémarrage, facilitant le troubleshooting en production.

Les événements Kubernetes générés par MetalLB documentent les actions importantes. L'assignation d'une adresse à un service génère un événement. Les échecs d'allocation dus à l'épuisement du pool créent des événements warning. Ces événements, visibles via `kubectl describe service`, offrent un historique des actions MetalLB facilitant le post-mortem analysis.

Les health checks et readiness probes garantissent que MetalLB fonctionne correctement. Les pods exposent des endpoints de santé surveillés par Kubernetes. Les métriques de liveness détectent les deadlocks ou crashes. Les readiness checks confirment que les speakers sont prêts à annoncer des routes. Cette surveillance automatique garantit que Kubernetes peut récupérer automatiquement des défaillances.

## Troubleshooting et problèmes courants

Les problèmes avec MetalLB suivent généralement des patterns reconnaissables avec des solutions établies.

L'absence d'External IP (stuck en Pending) est le problème le plus commun pour les débutants. Vérifier d'abord que MetalLB est correctement installé et que les pods sont Running. Examiner les logs du controller pour des erreurs d'allocation. Confirmer qu'il reste des adresses disponibles dans les pools configurés. Vérifier que le Service n'a pas d'annotations conflictuelles empêchant l'allocation. Cette approche méthodique résout la majorité des cas.

Les problèmes de connectivité réseau peuvent avoir plusieurs causes. En mode L2, vérifier que les requêtes ARP sont correctement répondues avec `arping`. Confirmer que le firewall du node permet le trafic entrant sur les ports du service. Tester depuis différents points du réseau pour isoler les problèmes de routage. Les caches ARP obsolètes peuvent nécessiter un flush manuel sur les clients ou routeurs.

Les conflits d'adresses IP causent des comportements erratiques difficiles à diagnostiquer. Deux dispositifs revendiquant la même IP créent une "ARP war" avec connectivité intermittente. Scanner le réseau avec nmap ou arp-scan avant de configurer MetalLB évite ces conflits. Si un conflit est suspecté, temporairement désactiver MetalLB et vérifier si l'IP répond toujours. Maintenir une documentation précise des allocations IP prévient ces problèmes.

Les problèmes de performance se manifestent par des latences élevées ou des timeouts. En mode L2, vérifier que le node leader n'est pas surchargé. Examiner les métriques réseau pour saturation de bande passante. Considérer la distribution des services sur différents nodes. En mode BGP, vérifier les métriques de routage et l'ECMP fonctionne correctement. L'optimisation peut nécessiter des ajustements de la topologie réseau ou du placement des services.

## Cas d'usage et architectures

MetalLB enable des architectures sophistiquées dans votre lab MicroK8s qui seraient impossibles ou complexes autrement.

L'hébergement de services domestiques bénéficie enormément de MetalLB. Un NAS software comme NextCloud peut avoir sa propre IP stable. Un serveur média Plex ou Jellyfin devient accessible depuis toutes les smart TVs du réseau. Home Assistant pour la domotique obtient une adresse fixe facilitant l'intégration avec les dispositifs IoT. Ces services deviennent des citoyens de première classe sur votre réseau, indistinguables de dispositifs physiques.

Les environnements de développement multi-services utilisent MetalLB pour simuler des architectures de production. Chaque microservice peut avoir sa propre IP LoadBalancer pendant le développement, facilitant le debugging et les tests d'intégration. Les développeurs peuvent accéder directement aux services sans proxy ou port-forwarding complexe. Cette approche accélère le développement et réduit les différences entre développement et production.

La simulation de multi-datacenter peut être implémentée avec plusieurs pools MetalLB représentant différentes "régions". Les services peuvent être déployés avec des IPs de différents pools, simulant la géo-distribution. Combiner avec des Network Policies pour simuler la latence ou les pannes inter-régions crée un environnement de test réaliste pour les applications distribuées.

L'intégration avec l'infrastructure d'entreprise existante devient possible avec MetalLB. Les services Kubernetes peuvent obtenir des IPs du réseau d'entreprise, s'intégrant transparemment avec les systèmes legacy. Les firewalls et load balancers hardware peuvent router vers les services MetalLB comme vers n'importe quel serveur. Cette capacité facilite la migration graduelle vers Kubernetes sans disruption massive.

## Sécurité et considérations

La sécurité de MetalLB et des services exposés nécessite attention et planification appropriée.

L'isolation réseau des services exposés doit être soigneusement considérée. Les Services LoadBalancer sont directement accessibles depuis le réseau, contournant potentiellement les protections de l'Ingress. Implémenter des Network Policies pour limiter le trafic entrant aux ports nécessaires. Considérer l'utilisation de firewalls externes ou iptables rules sur les nodes pour une défense en profondeur.

La protection contre l'épuisement d'adresses peut être implémentée via des ResourceQuotas limitant le nombre de Services LoadBalancer par namespace. Cette protection évite qu'une application mal configurée ou compromise ne consomme toutes les adresses disponibles. Les alertes sur l'utilisation des pools permettent une intervention proactive avant l'épuisement complet.

L'audit et la conformité bénéficient de la traçabilité de MetalLB. Chaque allocation d'adresse est loggée et associée à un service spécifique. Les événements Kubernetes créent un audit trail des changements. Cette visibilité facilite la conformité avec les politiques de sécurité réseau et l'investigation des incidents.

Les mises à jour de MetalLB doivent être planifiées soigneusement. Bien que MetalLB supporte les rolling updates sans interruption de service en théorie, la pratique peut varier selon votre configuration. Tester les updates dans un environnement de staging, préparer des plans de rollback, et planifier des fenêtres de maintenance pour les services critiques. La stabilité de votre infrastructure de load balancing est cruciale pour la disponibilité globale.

## Évolution et alternatives

Comprendre les alternatives et l'évolution possible de votre stratégie de load balancing prépare votre lab pour le futur.

L'évolution vers des solutions hardware peut devenir nécessaire si votre lab grandit. Les load balancers hardware comme F5 ou Citrix ADC offrent des performances et fonctionnalités supérieures mais à un coût significatif. MetalLB peut coexister avec ces solutions, gérant les services de développement tandis que le hardware gère la production. Cette approche hybride optimise coûts et performances.

Les alternatives software incluent kube-vip, qui offre des fonctionnalités similaires avec une approche architecturale différente. HAProxy ou nginx peuvent être configurés comme load balancers externes pour des besoins spécifiques. Ces alternatives peuvent être plus appropriées selon vos contraintes ou préférences, mais MetalLB reste généralement le meilleur choix pour la simplicité et l'intégration Kubernetes.

L'intégration avec les service meshes comme Istio ou Linkerd crée des architectures sophistiquées. Le mesh gère le trafic interne tandis que MetalLB gère l'exposition externe. Cette séparation des responsabilités permet des stratégies de routage complexes, de l'observabilité avancée, et des politiques de sécurité granulaires. La complexité accrue doit être justifiée par les besoins réels.

Le futur de MetalLB continue d'évoluer avec de nouvelles fonctionnalités et améliorations. Le support IPv6 s'améliore constamment. L'intégration avec les nouvelles APIs Kubernetes comme Gateway API est en développement. Les performances et la résilience continuent d'être optimisées. Rester informé des développements garantit que votre lab reste moderne et aligné avec les meilleures pratiques de l'industrie.

⏭️
