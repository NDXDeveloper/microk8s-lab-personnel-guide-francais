🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.8 Cert-Manager (certificats SSL/TLS)

## Introduction aux certificats SSL/TLS dans Kubernetes

Les certificats SSL/TLS constituent la pierre angulaire de la sécurité sur Internet moderne, transformant des communications en texte clair vulnérables en canaux chiffrés protégés. Dans l'écosystème Kubernetes, où les microservices communiquent constamment entre eux et avec le monde extérieur, la gestion des certificats devient rapidement un cauchemar opérationnel. Imaginez devoir manuellement générer, distribuer, et renouveler des certificats pour des dizaines de services, chacun avec ses propres domaines et sous-domaines. Cette complexité explosive est exactement ce que Cert-Manager résout avec une élégance remarquable.

Cert-Manager transforme Kubernetes en une autorité de certification automatisée, capable de provisionner, renouveler, et gérer des certificats SSL/TLS sans intervention humaine. Pour votre lab personnel MicroK8s, cela signifie que vos applications peuvent automatiquement obtenir des certificats valides de Let's Encrypt, transformant votre environnement de développement local en plateforme sécurisée indistinguable d'un service de production. Plus de warnings de navigateur effrayants, plus de certificats expirés causant des pannes, plus de processus manuels fastidieux.

La magie de Cert-Manager réside dans son intégration native avec Kubernetes. Les certificats deviennent des ressources Kubernetes de première classe, gérées déclarativement comme vos deployments ou services. Cette approche "Certificate as Code" garantit que vos certificats sont versionnés, auditables, et reproductibles. Dans un monde où la sécurité est primordiale et où les certificats HTTPS sont devenus obligatoires même pour le développement, Cert-Manager n'est plus un luxe mais une nécessité.

## Architecture et composants de Cert-Manager

Comprendre l'architecture interne de Cert-Manager révèle pourquoi cette solution est devenue le standard de facto pour la gestion des certificats dans Kubernetes.

Le cœur de Cert-Manager est constitué de trois contrôleurs principaux travaillant en synergie. Le Certificate Controller surveille les ressources Certificate dans votre cluster et orchestre le processus complet d'obtention des certificats. Quand vous créez une ressource Certificate, ce contrôleur analyse vos requirements, sélectionne l'Issuer approprié, et déclenche le processus de génération. Il gère également le cycle de vie complet, incluant le renouvellement automatique avant expiration.

L'Issuer et ClusterIssuer Controllers gèrent les différentes méthodes d'obtention de certificats. Un Issuer est namespace-scoped, parfait pour isoler les certificats de différentes équipes ou environnements. Un ClusterIssuer est global au cluster, idéal pour des configurations partagées comme Let's Encrypt. Ces contrôleurs supportent multiple backends : Let's Encrypt pour les certificats publics gratuits, Vault pour l'infrastructure PKI d'entreprise, ou même des CA auto-signées pour le développement. Cette flexibilité permet d'adapter Cert-Manager à pratiquement n'importe quel environnement.

Le Challenge Controller gère spécifiquement les défis ACME utilisés par Let's Encrypt et d'autres autorités de certification compatibles. Pour prouver que vous contrôlez un domaine, Let's Encrypt propose deux types de défis : HTTP-01 où vous devez servir un fichier spécifique sur votre domaine, et DNS-01 où vous devez créer un enregistrement TXT DNS spécifique. Le Challenge Controller automatise complètement ces processus, créant temporairement les ressources nécessaires, attendant la validation, puis nettoyant après succès.

Les Webhook Extensions permettent d'étendre Cert-Manager pour supporter des providers DNS exotiques ou des workflows de validation personnalisés. Cette architecture extensible garantit que Cert-Manager peut s'adapter à votre infrastructure existante plutôt que de vous forcer à changer votre architecture pour accommoder l'outil. Des webhooks existent pour pratiquement tous les providers DNS majeurs, depuis AWS Route53 jusqu'à Cloudflare, en passant par les solutions self-hosted comme PowerDNS.

## Installation et configuration dans MicroK8s

L'installation de Cert-Manager dans MicroK8s peut s'effectuer de plusieurs manières, chacune avec ses avantages selon votre niveau de confort et vos besoins de personnalisation.

L'installation via Helm représente la méthode recommandée et la plus flexible. Après avoir activé l'addon Helm dans MicroK8s avec `microk8s enable helm3`, l'ajout du repository Jetstack et l'installation de Cert-Manager s'effectuent en quelques commandes. Helm permet de personnaliser l'installation via des values files, ajustant les ressources allouées, activant des fonctionnalités spécifiques, ou configurant des paramètres de sécurité. La version Helm facilite également les upgrades futurs, maintenant votre installation à jour avec les derniers correctifs de sécurité.

L'installation via manifestes YAML directs offre un contrôle total sur ce qui est déployé. Les manifestes officiels de Cert-Manager peuvent être téléchargés et appliqués directement, parfait pour les environnements où Helm n'est pas désiré ou pour comprendre exactement ce qui est installé. Cette méthode nécessite plus d'attention lors des mises à jour mais offre une transparence totale sur les ressources créées.

La configuration des Custom Resource Definitions (CRDs) est cruciale et s'effectue automatiquement avec les deux méthodes. Ces CRDs définissent les nouveaux types de ressources comme Certificate, Issuer, et ClusterIssuer que Cert-Manager gère. Les CRDs transforment Kubernetes en système conscient des certificats, permettant de déclarer vos besoins en certificats aussi simplement que vous déclarez des deployments.

La vérification post-installation confirme que tous les composants fonctionnent correctement. Les pods de Cert-Manager dans le namespace cert-manager doivent être en état Running. Les webhooks de validation doivent être accessibles, vérifiables en créant une ressource Certificate de test. Les logs des différents contrôleurs révèlent le processus d'initialisation et confirment que Cert-Manager est prêt à gérer vos certificats.

## Configuration des Issuers

Les Issuers sont la porte d'entrée vers les autorités de certification, définissant comment et où obtenir vos certificats.

Let's Encrypt Issuer est de loin le plus populaire pour les certificats publics gratuits. La configuration nécessite de spécifier l'endpoint ACME (production ou staging), une adresse email pour les notifications importantes, et la méthode de validation (HTTP-01 ou DNS-01). L'endpoint staging est crucial pour le testing, offrant des limites de taux plus généreuses mais des certificats non trusted. Une fois votre configuration validée en staging, basculer vers production est trivial. La clé privée du compte ACME est automatiquement générée et stockée dans un Secret Kubernetes.

Les Self-Signed Issuers restent précieux pour le développement local et les tests. Bien que les navigateurs affichent des warnings, ces certificats offrent le chiffrement complet nécessaire pour tester le comportement HTTPS de vos applications. La configuration est minimale, nécessitant seulement de spécifier le type self-signed. Ces certificats peuvent être générés instantanément sans dépendance externe, parfait pour les environnements isolés ou les tests automatisés.

Les CA Issuers permettent d'utiliser votre propre autorité de certification, commune dans les environnements d'entreprise. Vous fournissez le certificat et la clé privée de votre CA, et Cert-Manager peut alors signer des certificats pour votre organisation. Cette approche offre un contrôle total sur la chaîne de confiance et permet des configurations sophistiquées comme des certificats client pour l'authentification mutuelle TLS.

Les Vault Issuers intègrent HashiCorp Vault pour une gestion PKI d'entreprise. Vault gère la CA racine et les politiques tandis que Cert-Manager automatise les demandes de certificats. Cette intégration combine le meilleur des deux mondes : la puissance PKI de Vault avec l'automation Kubernetes-native de Cert-Manager. La configuration nécessite l'endpoint Vault, l'authentification (token, Kubernetes auth, ou AppRole), et le path PKI à utiliser.

## Méthodes de validation de domaine

La validation de domaine prouve à l'autorité de certification que vous contrôlez le domaine pour lequel vous demandez un certificat. Cert-Manager automatise complètement ce processus traditionnellement manuel.

La validation HTTP-01 est la plus simple conceptuellement et souvent la plus facile à implémenter. Let's Encrypt demande que vous serviez un fichier spécifique à une URL bien connue sur votre domaine. Cert-Manager crée automatiquement un pod temporaire et un service qui servent ce fichier. Votre Ingress doit router le path `/.well-known/acme-challenge/` vers ce service temporaire. Cette méthode fonctionne parfaitement pour les domaines publiquement accessibles mais échoue derrière des firewalls ou pour les wildcards.

La validation DNS-01 offre plus de flexibilité en créant un enregistrement TXT dans votre zone DNS. Cette méthode fonctionne même si votre cluster n'est pas publiquement accessible et est la seule option pour les certificats wildcard. Cert-Manager supporte des dizaines de providers DNS via des solvers intégrés ou des webhooks. La configuration nécessite les credentials API de votre provider DNS, stockés de manière sécurisée dans un Secret. Le processus automatique crée l'enregistrement, attend la propagation DNS, valide avec Let's Encrypt, puis nettoie l'enregistrement.

Les stratégies de sélection de méthode dépendent de votre environnement. Pour un lab personnel avec un domaine public et un cluster accessible depuis Internet, HTTP-01 est généralement plus simple. Pour les environnements d'entreprise derrière des firewalls ou nécessitant des wildcards, DNS-01 est incontournable. Cert-Manager permet de configurer plusieurs solvers et de les sélectionner basé sur des règles, accommodant des environnements complexes avec différents domaines utilisant différentes méthodes.

Le troubleshooting des échecs de validation suit des patterns reconnaissables. Pour HTTP-01, vérifier que l'Ingress route correctement vers le solver pod, que le firewall permet le trafic HTTP, et que le domaine pointe vers la bonne IP. Pour DNS-01, confirmer que les credentials API sont corrects, que les permissions sont suffisantes, et que la propagation DNS fonctionne. Les logs de Cert-Manager détaillent chaque étape, facilitant l'identification du point de défaillance.

## Gestion automatique des certificats

L'automation est le superpouvoir de Cert-Manager, éliminant les tâches manuelles répétitives et error-prone de la gestion des certificats.

Le cycle de vie automatisé commence quand vous créez une ressource Certificate. Cert-Manager détecte immédiatement cette nouvelle ressource et initie le processus d'obtention. La clé privée est générée selon vos spécifications (RSA ou ECDSA, taille de clé configurable). La Certificate Signing Request (CSR) est créée et soumise à l'Issuer configuré. Le processus de validation s'exécute automatiquement. Une fois validé, le certificat signé est récupéré et stocké avec la clé privée dans un Secret Kubernetes. Ce Secret est alors prêt à être monté par vos pods ou référencé par votre Ingress.

Le renouvellement automatique élimine le risque d'expiration accidentelle. Cert-Manager surveille continuellement tous les certificats et initie le renouvellement avant expiration. Par défaut, le renouvellement commence 30 jours avant expiration, configurable selon vos besoins. Le processus de renouvellement est identique à l'obtention initiale mais généralement plus rapide car les validations sont cached. Le nouveau certificat remplace atomiquement l'ancien dans le Secret, et les pods utilisant ce Secret voient automatiquement le nouveau certificat.

La rotation des clés peut être configurée pour renouveler non seulement le certificat mais aussi la clé privée. Cette pratique de sécurité limite l'exposition en cas de compromission de clé. La rotation peut être déclenchée manuellement en supprimant le Secret (Cert-Manager le régénère immédiatement) ou automatiquement via des annotations. Les applications doivent être conçues pour gérer gracieusement les changements de certificats, généralement via des file watchers ou des sidecars de reloading.

Les stratégies de retry et backoff gèrent élégamment les échecs temporaires. Si l'obtention d'un certificat échoue (provider DNS temporairement indisponible, limite de taux atteinte), Cert-Manager retry automatiquement avec un backoff exponentiel. Les limites de retry et les délais sont configurables, équilibrant la persistence avec la prévention de spam. Les événements Kubernetes documentent chaque tentative, facilitant le debugging des échecs persistants.

## Intégration avec Ingress

L'intégration transparente entre Cert-Manager et les Ingress Controllers transforme l'obtention de certificats en processus totalement automatique.

Les annotations Ingress déclenchent automatiquement la création de certificats. L'annotation `cert-manager.io/cluster-issuer` spécifie quel ClusterIssuer utiliser. Quand Cert-Manager détecte cette annotation sur un Ingress, il crée automatiquement une ressource Certificate correspondante. Les hostnames de l'Ingress deviennent les SANs (Subject Alternative Names) du certificat. Cette magie signifie qu'ajouter HTTPS à votre service nécessite seulement une ligne d'annotation.

La gestion des secrets TLS est complètement automatisée. Cert-Manager crée le Secret dans le même namespace que l'Ingress, avec le nom spécifié dans la section TLS de l'Ingress. L'Ingress Controller détecte automatiquement ce Secret et commence à servir le trafic HTTPS avec le certificat. Quand Cert-Manager renouvelle le certificat, il met à jour le Secret en place, et l'Ingress Controller recharge automatiquement le nouveau certificat sans interruption de service.

Les configurations multi-domaines sont élégamment gérées. Un seul Ingress peut avoir plusieurs hosts, et Cert-Manager génère un certificat couvrant tous ces domaines via SANs. Pour des besoins plus complexes, plusieurs sections TLS peuvent référencer différents certificats, permettant différents domaines avec différents Issuers ou configurations. Cette flexibilité accommode des architectures sophistiquées tout en gardant la configuration simple.

Le support des wildcards simplifie grandement la gestion des sous-domaines dynamiques. Un certificat pour `*.example.com` couvre tous les sous-domaines, parfait pour les environnements multi-tenant ou les preview deployments. La validation DNS-01 est requise pour les wildcards, mais une fois configurée, n'importe quel nouveau sous-domaine est automatiquement couvert sans nouvelle validation.

## Cas d'usage avancés

Au-delà de la simple obtention de certificats pour HTTPS, Cert-Manager enable des architectures de sécurité sophistiquées.

Le mTLS (mutual TLS) sécurise la communication service-to-service dans votre cluster. Cert-Manager peut gérer non seulement les certificats serveur mais aussi les certificats client pour l'authentification mutuelle. Créer une CA interne avec Cert-Manager, puis générer des certificats client et serveur depuis cette CA. Cette infrastructure PKI complète transforme votre cluster en zero-trust network où chaque connexion est authentifiée et chiffrée.

Les certificats pour services non-HTTP étendent l'utilité de Cert-Manager. Les bases de données PostgreSQL ou MySQL peuvent utiliser des certificats gérés par Cert-Manager pour le chiffrement des connexions. Les brokers de messages comme Kafka ou RabbitMQ bénéficient de certificats automatiquement renouvelés. Même les connexions gRPC peuvent être sécurisées avec des certificats Cert-Manager. Cette universalité fait de Cert-Manager le gestionnaire central pour tous les besoins cryptographiques.

La rotation coordonnée de certificats pour les applications stateful nécessite une orchestration soigneuse. Cert-Manager peut notifier les applications via des webhooks ou des events quand les certificats sont renouvelés. Les applications peuvent alors gracieusement recharger les certificats sans perdre de connexions. Pour les systèmes critiques, implémenter une période de chevauchement où ancien et nouveau certificats sont valides simultanément facilite la transition.

Les architectures multi-cluster bénéficient de la portabilité de Cert-Manager. Les mêmes configurations peuvent être déployées across clusters, garantissant une gestion cohérente des certificats. Pour les certificats partagés entre clusters, des solutions comme Sealed Secrets ou External Secrets peuvent synchroniser les Secrets de manière sécurisée. Cette approche maintient l'isolation des clusters tout en partageant les certificats nécessaires.

## Monitoring et observabilité

La surveillance de Cert-Manager est cruciale pour maintenir la sécurité continue de votre infrastructure.

Les métriques Prometheus exposées par Cert-Manager offrent une visibilité profonde sur l'état des certificats. Les métriques incluent le nombre de certificats par état, le temps avant expiration pour chaque certificat, le taux de succès/échec des renouvellements, et les latences des opérations ACME. Ces métriques peuvent être visualisées dans Grafana avec des dashboards préconfigurés, offrant un aperçu instantané de votre infrastructure de certificats.

Les alertes proactives préviennent les problèmes avant qu'ils n'impactent les services. Alerter quand un certificat approche l'expiration sans renouvellement réussi. Notifier les échecs répétés de validation. Surveiller les limites de taux Let's Encrypt approchantes. Ces alertes transforment la gestion des certificats de réactive en proactive, éliminant les surprises désagréables.

Les événements Kubernetes générés par Cert-Manager documentent chaque action. La création, le renouvellement, et les échecs génèrent des événements attachés aux ressources Certificate. Ces événements créent un audit trail complet, essentiel pour le debugging et la conformité. Les événements peuvent être streamés vers des systèmes de logging centralisés pour une analyse long-terme.

Le debugging avancé utilise les conditions sur les ressources Certificate. Chaque Certificate a des conditions comme Ready, Issuing, et Valid qui indiquent son état actuel. Les messages associés détaillent les problèmes spécifiques. Cette information structurée facilite l'automation, permettant des scripts qui réagissent aux états de certificats.

## Sécurité et bonnes pratiques

La sécurité de votre infrastructure de certificats nécessite attention aux détails et adhérence aux bonnes pratiques.

La protection des clés privées est primordiale. Les Secrets Kubernetes contenant les clés doivent être chiffrés au repos si votre cluster le supporte. Limiter l'accès aux Secrets via RBAC, donnant seulement les permissions minimales nécessaires. Considérer l'utilisation de solutions comme Sealed Secrets ou SOPS pour le stockage git-safe des configurations. Ne jamais logger ou exposer les clés privées dans les logs ou les interfaces.

La séparation des environnements utilise différents Issuers pour différents stages. Utiliser Let's Encrypt staging pour le développement évite d'atteindre les limites de taux production. Les environnements de test peuvent utiliser des CA auto-signées pour l'isolation complète. La production utilise Let's Encrypt production ou votre CA d'entreprise. Cette séparation prévient les interférences et maintient des limites de sécurité claires.

Les stratégies de backup incluent les certificats et les configurations. Bien que Cert-Manager puisse régénérer les certificats, backuper les Secrets permet une restauration rapide. Plus important, backuper les ressources Certificate, Issuer, et ClusterIssuer garantit que vous pouvez recréer votre infrastructure de certificats. Tester régulièrement la restauration valide vos procédures de recovery.

L'audit et la conformité bénéficient de la nature déclarative de Cert-Manager. Toutes les configurations sont versionnées dans Git. Les changements sont tracés via les events Kubernetes. Les métriques documentent l'état historique. Cette transparence facilite les audits de sécurité et démontre la conformité avec les politiques de gestion des certificats.

## Troubleshooting et problèmes courants

Les problèmes avec Cert-Manager suivent généralement des patterns identifiables avec des solutions établies.

Les échecs de validation ACME sont les plus courants, particulièrement lors de la configuration initiale. Pour HTTP-01, vérifier que votre domaine pointe vers la bonne IP et que l'Ingress route correctement le challenge path. Pour DNS-01, confirmer que les credentials API fonctionnent et ont les permissions nécessaires. Les logs de Cert-Manager détaillent exactement où le processus échoue. Tester manuellement les étapes de validation peut isoler le problème.

Les limites de taux Let's Encrypt peuvent bloquer l'obtention de certificats. La limite principale est 50 certificats par domaine par semaine. Utiliser Let's Encrypt staging pour le développement et les tests. Implémenter des stratégies de réutilisation de certificats pour les environnements éphémères. Surveiller votre utilisation via les métriques pour éviter les surprises. En cas de blocage, attendre le reset hebdomadaire ou utiliser un Issuer alternatif temporairement.

Les problèmes de performance se manifestent par des délais dans l'obtention ou le renouvellement. Vérifier les ressources allouées aux pods Cert-Manager, particulièrement si vous gérez beaucoup de certificats. Les webhooks peuvent devenir un bottleneck avec beaucoup d'opérations concurrentes. Augmenter les replicas du webhook peut améliorer le throughput. Pour les très grandes installations, considérer de séparer les Issuers par namespace pour distribuer la charge.

Les conflits de versions surviennent lors des upgrades de Cert-Manager ou Kubernetes. Toujours consulter les notes de release pour les breaking changes. Les CRDs doivent être mis à jour avant le déploiement des nouveaux pods. Certaines migrations nécessitent des étapes manuelles documentées. Maintenir Cert-Manager raisonnablement à jour évite les migrations complexes multi-versions.

## Migration et évolution

L'évolution de votre infrastructure de certificats avec Cert-Manager suit naturellement la croissance de votre lab.

La migration depuis une gestion manuelle vers Cert-Manager peut s'effectuer graduellement. Importer les certificats existants comme Secrets Kubernetes. Créer des ressources Certificate avec l'annotation `cert-manager.io/issuer-kind: Issuer` pointant vers vos certificats existants. Graduellement, migrer vers des certificats gérés par Cert-Manager. Cette approche minimise les risques et permet un rollback facile.

L'évolution vers des architectures complexes est naturellement supportée. Commencer avec un simple ClusterIssuer Let's Encrypt peut évoluer vers multiple Issuers pour différents domaines, intégration avec votre PKI d'entreprise, et même des architectures multi-cluster avec certificats partagés. La nature déclarative de Cert-Manager rend ces évolutions des changements de configuration plutôt que des migrations majeures.

L'intégration avec des outils externes enrichit les capacités. External Secrets Operator peut synchroniser des certificats depuis des vaults externes. Flux ou ArgoCD peuvent gérer les configurations Cert-Manager en GitOps. Les service meshes peuvent utiliser les certificats Cert-Manager pour mTLS. Cette interopérabilité fait de Cert-Manager un composant central de votre infrastructure de sécurité.

Le futur de Cert-Manager continue d'évoluer avec de nouvelles fonctionnalités. Le support pour de nouvelles autorités de certification et méthodes de validation s'étend constamment. L'intégration avec les standards émergents comme SPIFFE/SPIRE pour l'identité des workloads. Les améliorations de performance pour les très grandes installations. Rester informé des développements garantit que votre infrastructure reste moderne et sécurisée.

⏭️
