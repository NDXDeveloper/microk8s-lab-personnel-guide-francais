🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 DNS (CoreDNS)

## Introduction au DNS dans Kubernetes

Le DNS constitue l'épine dorsale invisible mais essentielle de tout cluster Kubernetes, permettant aux services et aux pods de se découvrir et de communiquer entre eux par des noms plutôt que par des adresses IP volatiles. Dans Kubernetes, chaque fois qu'un pod souhaite contacter un service, une base de données, ou un autre pod, il utilise le DNS pour résoudre le nom en adresse IP. Sans un système DNS fonctionnel, votre cluster serait comme une ville sans noms de rues, où chaque destination devrait être mémorisée par ses coordonnées GPS exactes.

CoreDNS est devenu le serveur DNS standard de Kubernetes, remplaçant kube-dns depuis la version 1.11. Cette évolution n'est pas anodine : CoreDNS offre une architecture plus simple, plus flexible et plus performante, particulièrement adaptée aux environnements conteneurisés modernes. Dans MicroK8s, CoreDNS est disponible comme addon, préconfiguré et optimisé pour fonctionner immédiatement avec des paramètres adaptés à un lab personnel.

La beauté du DNS dans Kubernetes réside dans sa transparence. Les développeurs peuvent écrire leurs applications en utilisant des noms de services logiques comme `database` ou `api-backend`, sans se soucier des adresses IP réelles ou de l'infrastructure sous-jacente. Cette abstraction facilite le développement, le déploiement et la portabilité des applications entre différents environnements.

## Architecture et fonctionnement de CoreDNS

CoreDNS n'est pas simplement un serveur DNS traditionnel adapté à Kubernetes, mais une solution conçue dès le départ pour les architectures cloud-native. Son architecture modulaire basée sur des plugins permet une extensibilité et une personnalisation remarquables, tout en maintenant un cœur léger et performant.

Au cœur de CoreDNS se trouve un serveur écrit en Go, langage privilégié de l'écosystème Kubernetes pour ses performances et sa gestion efficace de la concurrence. Le serveur traite les requêtes DNS entrantes et les route à travers une chaîne de plugins configurables. Chaque plugin peut examiner, modifier ou répondre à la requête, créant un pipeline de traitement flexible et puissant.

Dans le contexte Kubernetes, le plugin kubernetes est le plus important. Ce plugin maintient une vue en temps réel de tous les services et endpoints dans votre cluster en surveillant l'API Kubernetes. Lorsqu'une requête DNS arrive pour un service Kubernetes, ce plugin génère dynamiquement la réponse appropriée basée sur l'état actuel du cluster. Cette approche dynamique signifie que les changements dans votre cluster, comme le scaling d'un service ou le déploiement d'une nouvelle application, sont immédiatement reflétés dans les réponses DNS.

Le système de cache intégré améliore considérablement les performances en mémorisant les réponses récentes. Ce cache est intelligent et respecte les TTL (Time To Live) configurés, garantissant que les données obsolètes ne sont jamais servies. Pour un lab personnel où les ressources sont limitées, ce cache réduit significativement la charge sur l'API Kubernetes et améliore les temps de réponse.

## Installation et activation dans MicroK8s

L'activation de CoreDNS dans MicroK8s illustre parfaitement la philosophie de simplicité de la plateforme. La commande `microk8s enable dns` déclenche une séquence d'installation sophistiquée qui déploie CoreDNS avec une configuration optimale pour votre environnement.

Lors de l'activation, MicroK8s crée d'abord un Deployment CoreDNS dans le namespace `kube-system`. Ce deployment garantit que le nombre souhaité de replicas CoreDNS est toujours en cours d'exécution, offrant résilience et haute disponibilité même dans un environnement single-node. La configuration par défaut démarre généralement un ou deux pods CoreDNS, suffisant pour la plupart des labs personnels.

Un Service de type ClusterIP est créé pour exposer CoreDNS au sein du cluster. Ce service obtient traditionnellement l'adresse IP `10.152.183.10` dans MicroK8s, bien que cela puisse varier selon votre configuration réseau. Cette adresse IP est cruciale car elle sera configurée comme serveur DNS dans tous les pods de votre cluster.

La configuration de CoreDNS est stockée dans un ConfigMap nommé `coredns`. Ce ConfigMap contient le Corefile, le fichier de configuration principal de CoreDNS, définissant les zones DNS gérées et les plugins actifs. MicroK8s préconfigure ce fichier avec des paramètres optimaux, incluant la gestion de la zone `cluster.local` pour les services internes et le forwarding des requêtes externes vers les serveurs DNS de votre système.

L'intégration avec kubelet est automatiquement configurée pour que tous les nouveaux pods utilisent CoreDNS comme résolveur DNS. Le fichier `/etc/resolv.conf` dans chaque pod est configuré pour pointer vers le service CoreDNS, avec les options de recherche appropriées pour permettre la résolution courte des noms de services.

## Convention de nommage et résolution

Kubernetes suit des conventions de nommage strictes et prévisibles qui permettent une résolution DNS cohérente et intuitive. Comprendre ces conventions est essentiel pour développer et déployer des applications dans votre cluster.

Chaque service dans Kubernetes obtient automatiquement une entrée DNS suivant le format `<service-name>.<namespace>.svc.cluster.local`. Par exemple, un service nommé `web-api` dans le namespace `production` sera accessible via `web-api.production.svc.cluster.local`. Cette convention garantit l'unicité des noms même lorsque des services identiques existent dans différents namespaces.

La résolution courte des noms est une fonctionnalité pratique qui simplifie l'utilisation quotidienne. Depuis un pod dans le même namespace, vous pouvez simplement utiliser `web-api` au lieu du nom complet. Depuis un pod dans un namespace différent, `web-api.production` suffit. Cette flexibilité permet d'écrire des configurations plus concises tout en maintenant la précision quand nécessaire.

Les pods peuvent également obtenir des entrées DNS, bien que ce ne soit pas le comportement par défaut. Lorsqu'activée, la convention de nommage pour les pods suit le format `<pod-ip-with-dashes>.<namespace>.pod.cluster.local`. Par exemple, un pod avec l'IP `10.1.2.3` dans le namespace `default` serait accessible via `10-1-2-3.default.pod.cluster.local`. Cette fonctionnalité est particulièrement utile pour les StatefulSets où chaque pod nécessite une identité stable.

Les StatefulSets bénéficient d'un traitement DNS spécial avec des noms prévisibles et stables. Chaque pod d'un StatefulSet obtient un nom DNS suivant le format `<statefulset-name>-<ordinal>.<service-name>.<namespace>.svc.cluster.local`. Cette convention permet aux applications stateful comme les bases de données de maintenir des identités réseau stables même après des redémarrages.

## Configuration et personnalisation

La configuration de CoreDNS dans MicroK8s peut être adaptée à vos besoins spécifiques en modifiant le ConfigMap `coredns`. Cette flexibilité permet d'ajuster le comportement DNS pour des cas d'usage particuliers tout en conservant la simplicité de gestion.

Le Corefile, contenu dans le ConfigMap, utilise une syntaxe déclarative simple mais puissante. Chaque bloc de configuration définit une zone DNS et les plugins qui la gèrent. La zone principale `cluster.local` gère la résolution interne Kubernetes, tandis que le bloc `.` (point) gère toutes les autres requêtes, typiquement en les forwardant vers des serveurs DNS externes.

La personnalisation du forwarding DNS est l'une des modifications les plus courantes. Par défaut, CoreDNS forward les requêtes non-Kubernetes vers les serveurs DNS configurés sur l'hôte. Vous pouvez modifier ce comportement pour utiliser des serveurs DNS spécifiques comme ceux de Cloudflare (1.1.1.1) ou Google (8.8.8.8), ou pour router certains domaines vers des serveurs DNS internes d'entreprise.

L'ajout de zones DNS personnalisées permet d'intégrer votre cluster avec votre infrastructure existante. Par exemple, vous pouvez configurer CoreDNS pour résoudre les noms de votre domaine local directement, évitant ainsi les allers-retours vers des serveurs DNS externes. Cette configuration est particulièrement utile dans un lab où vous voulez que vos services Kubernetes interagissent avec des ressources sur votre réseau local.

Le plugin hosts permet d'ajouter des entrées statiques similaires au fichier `/etc/hosts`. Cette fonctionnalité est pratique pour définir des alias personnalisés ou pour overrider temporairement certaines résolutions DNS pendant le développement. Les entrées peuvent être définies directement dans le Corefile ou chargées depuis un ConfigMap séparé.

La configuration du cache peut être ajustée pour optimiser les performances selon votre charge de travail. Les paramètres incluent la durée de cache (TTL), la taille maximale du cache, et les stratégies d'éviction. Pour un lab personnel avec des changements fréquents, un TTL court garantit que les modifications sont rapidement propagées. Pour un environnement plus stable, un TTL plus long réduit la charge sur CoreDNS.

## Intégration avec les services externes

CoreDNS dans MicroK8s ne se limite pas à la résolution interne du cluster mais peut servir de pont entre votre cluster Kubernetes et le monde extérieur. Cette capacité d'intégration est cruciale pour créer un lab réaliste où les applications peuvent interagir avec des services externes.

La résolution des domaines externes fonctionne automatiquement grâce au forwarding DNS. Lorsqu'un pod tente d'accéder à un domaine externe comme `api.github.com`, CoreDNS reconnaît que ce n'est pas un service Kubernetes et forward la requête vers les serveurs DNS configurés. Cette transparence permet aux applications de fonctionner sans modification, qu'elles accèdent à des services internes ou externes.

L'intégration avec des services de découverte externes comme Consul ou etcd est possible via des plugins dédiés. Ces intégrations permettent à CoreDNS de résoudre des services enregistrés dans ces systèmes externes, créant un pont entre différentes plateformes d'orchestration. Cette capacité est particulièrement utile pendant les migrations ou dans des environnements hybrides.

Les ExternalName Services offrent une méthode élégante pour créer des alias DNS pour des services externes. Un ExternalName Service crée une entrée CNAME dans CoreDNS pointant vers un domaine externe. Cette abstraction permet de changer facilement la destination sans modifier les applications, utile pour basculer entre environnements ou pour migrer des services.

La configuration de split-horizon DNS permet de servir différentes réponses selon l'origine de la requête. Cette technique est utile lorsque vous voulez que les services internes résolvent vers des adresses privées tandis que les requêtes externes obtiennent des adresses publiques. Bien que plus complexe à configurer, cette approche offre une flexibilité maximale pour les architectures sophistiquées.

## Troubleshooting et diagnostic

Le diagnostic des problèmes DNS est une compétence essentielle car de nombreux problèmes de connectivité dans Kubernetes proviennent de dysfonctionnements DNS. CoreDNS offre plusieurs outils et techniques pour identifier et résoudre rapidement ces problèmes.

La vérification basique de la santé de CoreDNS commence par examiner les pods dans le namespace kube-system. La commande `microk8s kubectl get pods -n kube-system | grep coredns` montre l'état des pods CoreDNS. Des pods en état `Running` avec des restarts fréquents peuvent indiquer des problèmes de configuration ou de ressources.

Les logs de CoreDNS sont la première source d'information pour le diagnostic. La commande `microk8s kubectl logs -n kube-system deployment/coredns` révèle les requêtes traitées et les erreurs éventuelles. L'activation du plugin log dans le Corefile peut augmenter la verbosité pour un debugging approfondi, montrant chaque requête DNS et sa réponse.

Le test de résolution DNS depuis un pod est un diagnostic fondamental. Déployer un pod de test avec des outils réseau comme `nslookup` ou `dig` permet de vérifier la résolution depuis la perspective d'une application. La commande `microk8s kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default` teste la résolution du service Kubernetes par défaut, un bon indicateur de la santé DNS.

Les problèmes de performances DNS se manifestent souvent par des latences applicatives difficiles à diagnostiquer. Le plugin cache de CoreDNS maintient des statistiques sur les hits et miss du cache, accessibles via les métriques Prometheus si configurées. Un taux de cache miss élevé peut indiquer un TTL trop court ou un cache sous-dimensionné.

La vérification de la configuration réseau du cluster est cruciale quand CoreDNS semble fonctionner mais les pods ne peuvent pas résoudre les noms. Vérifier que l'adresse IP du service CoreDNS est correctement configurée dans `/etc/resolv.conf` des pods, et que les règles iptables permettent le trafic vers le port 53 de CoreDNS.

## Optimisation des performances

L'optimisation de CoreDNS peut significativement améliorer les performances globales de votre cluster, particulièrement dans un environnement de lab où les ressources sont limitées.

Le dimensionnement approprié des replicas CoreDNS dépend de votre charge de travail. Pour un lab personnel avec quelques applications, un seul replica suffit généralement. Pour des environnements plus chargés ou nécessitant une haute disponibilité, deux ou trois replicas offrent redondance et distribution de charge. Le HorizontalPodAutoscaler peut être configuré pour ajuster automatiquement le nombre de replicas selon la charge.

L'optimisation du cache est l'amélioration la plus impactante pour la plupart des environnements. Augmenter la durée de cache pour les domaines stables réduit les requêtes vers l'API Kubernetes et les serveurs DNS externes. Le plugin cache supporte également le prefetch, rechargeant proactivement les entrées populaires avant leur expiration, éliminant la latence pour les requêtes fréquentes.

La configuration de la négative cache améliore la gestion des domaines inexistants. Par défaut, CoreDNS cache les réponses NXDOMAIN pendant une courte période, évitant de répéter des requêtes pour des domaines qui n'existent pas. Ajuster cette durée selon vos patterns d'utilisation peut réduire significativement la charge inutile.

Les resources limits et requests pour les pods CoreDNS doivent être soigneusement calibrées. Des limites trop basses causent des throttling CPU ou des OOM kills, impactant la résolution DNS pour tout le cluster. Des valeurs trop élevées gaspillent des ressources précieuses dans un lab personnel. Commencer avec des valeurs conservatives et ajuster basé sur les métriques réelles est l'approche recommandée.

## Sécurité DNS

La sécurité du DNS est souvent négligée mais reste critique car elle peut être exploitée pour des attaques de type man-in-the-middle ou DNS spoofing. CoreDNS offre plusieurs mécanismes pour sécuriser la résolution DNS dans votre cluster.

Le DNS over TLS (DoT) et DNS over HTTPS (DoH) peuvent être configurés pour les requêtes forwarded vers des serveurs DNS externes. Ces protocoles chiffrent les requêtes DNS, empêchant l'espionnage ou la modification des réponses. La configuration s'effectue via le plugin forward avec les options appropriées, though avec un léger impact sur les performances.

La validation DNSSEC vérifie l'authenticité des réponses DNS en utilisant des signatures cryptographiques. Le plugin dnssec de CoreDNS peut valider les signatures DNSSEC pour les domaines qui les supportent, offrant une protection contre les attaques de DNS spoofing. Cette fonctionnalité est particulièrement importante si votre lab interagit avec des services externes critiques.

Les politiques de firewall DNS permettent de bloquer ou rediriger certaines requêtes DNS basées sur des règles. Le plugin firewall peut bloquer des domaines malveillants connus, implémenter des listes blanches pour restreindre les domaines accessibles, ou logger les tentatives d'accès à des domaines suspects. Cette capacité transforme CoreDNS en première ligne de défense contre les menaces.

L'audit des requêtes DNS fournit une visibilité sur l'utilisation du DNS dans votre cluster. Le plugin log peut être configuré pour enregistrer toutes les requêtes DNS, créant un audit trail pour l'analyse de sécurité ou le debugging. Ces logs peuvent être intégrés avec votre stack de monitoring pour détecter des patterns anormaux ou des tentatives d'exfiltration de données.

## Cas d'usage avancés

Au-delà de la résolution DNS basique, CoreDNS dans MicroK8s peut être exploité pour des scénarios sophistiqués qui enrichissent votre lab personnel.

Le load balancing DNS distribue les requêtes entre plusieurs endpoints d'un service. Bien que Kubernetes gère nativement le load balancing au niveau service, CoreDNS peut implémenter des stratégies avancées comme le geo-routing ou le weighted round-robin. Cette capacité est utile pour simuler des déploiements multi-régions dans votre lab.

L'implémentation de service discovery dynamique permet aux applications de découvrir automatiquement de nouveaux services sans configuration manuelle. En combinant CoreDNS avec des annotations Kubernetes personnalisées, vous pouvez créer des conventions de nommage sophistiquées qui simplifient l'intégration entre microservices.

La création d'environnements isolés avec des vues DNS différentes permet de tester des configurations complexes. En utilisant des instances CoreDNS séparées ou des configurations conditionnelles, différents namespaces peuvent avoir des résolutions DNS complètement différentes, utile pour les tests A/B ou les environnements de staging isolés.

L'intégration avec des systèmes de configuration dynamique comme ConfigMaps ou des APIs externes permet de modifier le comportement DNS sans redémarrer CoreDNS. Le plugin auto détecte automatiquement les changements dans les ConfigMaps et recharge la configuration, permettant des ajustements en temps réel basés sur les conditions opérationnelles.

## Monitoring et métriques

La surveillance de CoreDNS est essentielle pour maintenir la santé de votre cluster et identifier proactivement les problèmes potentiels.

CoreDNS expose nativement des métriques Prometheus via le plugin metrics. Ces métriques incluent le nombre de requêtes par type, les latences de réponse, les taux de cache hit/miss, et les erreurs. L'activation de ce plugin et son intégration avec Prometheus permet de créer des dashboards détaillés et des alertes automatiques.

Les métriques clés à surveiller incluent la latence de résolution DNS, qui impacte directement les performances applicatives. Un pic soudain peut indiquer des problèmes réseau ou une surcharge. Le taux de requêtes NXDOMAIN élevé peut révéler des problèmes de configuration ou des tentatives d'accès à des services non existants. La consommation mémoire de CoreDNS, particulièrement importante avec de gros caches, doit être surveillée pour éviter les OOM kills.

L'intégration avec Grafana permet de visualiser ces métriques dans des dashboards intuitifs. Des dashboards préconfigurés pour CoreDNS sont disponibles dans la communauté, offrant des visualisations pour les métriques communes et facilitant l'identification rapide des anomalies.

Les alertes basées sur ces métriques permettent une réaction proactive aux problèmes. Des alertes pour une latence DNS excessive, un taux d'erreur élevé, ou des pods CoreDNS non disponibles garantissent que les problèmes DNS sont détectés avant qu'ils n'impactent significativement les applications.

## Migration et compatibilité

La migration depuis kube-dns vers CoreDNS, bien qu'automatique dans les versions récentes de Kubernetes, mérite attention pour comprendre les différences et assurer une transition douce.

CoreDNS maintient une compatibilité complète avec l'API DNS de Kubernetes, signifiant que les applications existantes continuent de fonctionner sans modification. Les conventions de nommage, les formats de requête, et les réponses restent identiques, garantissant une migration transparente du point de vue applicatif.

Les différences de configuration entre kube-dns et CoreDNS nécessitent une traduction des anciennes configurations. Le ConfigMap kube-dns utilisait un format JSON, tandis que CoreDNS utilise le Corefile. MicroK8s gère cette conversion automatiquement, mais comprendre les équivalences aide lors de personnalisations avancées.

La migration des customisations DNS existantes requiert une attention particulière. Les stub domains et upstream nameservers configurés dans kube-dns doivent être reconfigurés dans le Corefile de CoreDNS. Les annotations spécifiques à kube-dns sur les pods ou services peuvent nécessiter des ajustements pour fonctionner avec CoreDNS.

## Perspectives futures

L'évolution de CoreDNS continue d'apporter de nouvelles fonctionnalités et améliorations qui enrichiront votre lab MicroK8s.

Le support amélioré pour les architectures multi-cluster permettra une résolution DNS transparente entre clusters, facilitant la création de labs complexes simulant des déploiements distribués. Les améliorations de performance continues, particulièrement pour les environnements à haute charge, bénéficieront même aux labs personnels en réduisant l'utilisation des ressources.

L'intégration plus profonde avec les service meshes comme Istio ou Linkerd offrira des capacités de routage et de résolution encore plus sophistiquées. Les nouvelles fonctionnalités de sécurité, incluant le support étendu pour le chiffrement et la validation, renforceront la protection de votre infrastructure DNS.

La simplicité de gestion continuera d'être une priorité, avec des outils améliorés pour le diagnostic automatique et la résolution des problèmes DNS courants. L'intelligence artificielle pourrait même être intégrée pour optimiser automatiquement les configurations basées sur les patterns d'utilisation observés.

⏭️
