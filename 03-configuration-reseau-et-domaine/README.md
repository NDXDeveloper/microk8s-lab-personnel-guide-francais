🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 03. Configuration Réseau et Domaine

## Vue d'ensemble

La configuration réseau est l'épine dorsale de tout cluster Kubernetes, et particulièrement cruciale pour un lab personnel que vous souhaitez rendre accessible depuis l'extérieur. Cette section vous guidera à travers tous les aspects réseau nécessaires pour transformer votre installation MicroK8s locale en un environnement accessible et sécurisé, capable d'héberger des applications avec des noms de domaine professionnels et des certificats SSL valides.

## Objectifs de cette section

À la fin de cette section, vous serez capable de :

- Comprendre l'architecture réseau de Kubernetes et son implémentation dans MicroK8s
- Configurer un système DNS local et public pour vos applications
- Acquérir et configurer un nom de domaine pour votre lab
- Mettre en place le routage nécessaire pour rendre vos services accessibles depuis Internet
- Sécuriser votre installation avec des règles firewall appropriées
- Diagnostiquer et résoudre les problèmes de connectivité réseau

## Pourquoi la configuration réseau est-elle si importante ?

Dans un environnement de production cloud, la plupart des aspects réseau sont gérés automatiquement par le provider (AWS, GCP, Azure). Dans votre lab personnel, vous devez comprendre et configurer manuellement chaque couche du réseau. Cette connaissance approfondie vous donnera :

1. **Une meilleure compréhension de Kubernetes** : En configurant manuellement le réseau, vous comprendrez vraiment comment les services communiquent entre eux et avec l'extérieur.

2. **Un environnement réaliste** : Votre lab pourra simuler des scénarios de production avec des domaines réels, des certificats SSL valides et une architecture réseau proche de celle d'un vrai cluster.

3. **Des compétences de dépannage** : Les problèmes réseau sont parmi les plus courants dans Kubernetes. Cette expérience pratique vous préparera à les résoudre efficacement.

4. **Une flexibilité maximale** : Vous pourrez adapter votre configuration à vos besoins spécifiques, que ce soit pour du développement, des démonstrations ou de l'hébergement personnel.

## Architecture réseau de votre lab

Voici comment s'articulent les différentes couches réseau dans votre futur lab MicroK8s :

```
Internet
    ↓
[Nom de domaine public] (ex: monlab.example.com)
    ↓
[Box Internet / Routeur]
    ↓
[Redirection de ports] (80, 443)
    ↓
[Machine hôte MicroK8s]
    ↓
[Firewall local]
    ↓
[MicroK8s Cluster Network]
    ├── [Pod Network] (10.1.0.0/16)
    ├── [Service Network] (10.152.183.0/24)
    └── [Ingress Controller]
            ↓
        [Vos Applications]
```

## Prérequis pour cette section

Avant de commencer la configuration réseau, assurez-vous d'avoir :

### Prérequis techniques
- MicroK8s installé et fonctionnel (voir Section 02)
- Accès administrateur à votre machine hôte
- Accès à l'interface d'administration de votre box Internet/routeur
- Un éditeur de texte pour modifier les fichiers de configuration

### Prérequis réseau
- Une connexion Internet stable
- La possibilité d'ouvrir des ports sur votre routeur (pas de double NAT)
- Idéalement, une IP publique fixe ou un service de DNS dynamique

### Connaissances recommandées
- Notions de base sur TCP/IP (ports, protocoles)
- Compréhension basique du DNS
- Familiarité avec la ligne de commande Linux

## Ce que nous allons configurer

Cette section est divisée en sept parties progressives, chacune construisant sur la précédente :

1. **Concepts de base** : Nous commencerons par comprendre comment Kubernetes gère le réseau, les différents types d'objets réseau (Services, Endpoints, Ingress) et le modèle de réseau de MicroK8s.

2. **DNS local** : Configuration du DNS sur votre machine pour faciliter l'accès local à vos services sans avoir à mémoriser des adresses IP.

3. **Domaine public** : Guide pour choisir et acquérir un nom de domaine, avec des recommandations de registrars et des considérations de coût.

4. **Configuration DNS externe** : Mise en place des enregistrements DNS nécessaires (A, CNAME) pour pointer vers votre lab.

5. **Routage et NAT** : Configuration de la redirection de ports sur votre routeur pour permettre l'accès externe à vos services.

6. **Sécurité** : Mise en place de règles firewall pour protéger votre installation tout en permettant le trafic légitime.

7. **Tests et validation** : Méthodologie pour vérifier que tout fonctionne correctement, du local à l'externe.

## Approche recommandée

Pour tirer le meilleur parti de cette section, nous recommandons l'approche suivante :

### Phase 1 : Configuration locale
Commencez par configurer et tester tout en local. Assurez-vous que vos services sont accessibles depuis votre machine hôte avant de passer à l'étape suivante. Cette phase vous permet de valider votre configuration sans complexité réseau supplémentaire.

### Phase 2 : Accès réseau local
Étendez l'accès à votre réseau local (LAN). Configurez le DNS local et testez l'accès depuis d'autres machines de votre réseau. Cette étape valide que votre configuration réseau de base est correcte.

### Phase 3 : Exposition Internet
Seulement après avoir validé les phases 1 et 2, configurez l'accès externe. Cette approche progressive facilite le dépannage en cas de problème.

## Points d'attention particuliers

### Sécurité
Exposer des services sur Internet comporte des risques. Nous aborderons les bonnes pratiques de sécurité, mais gardez à l'esprit que votre lab sera potentiellement accessible au monde entier. Ne stockez jamais de données sensibles dans un environnement de lab.

### Limitations des FAI
Certains fournisseurs d'accès Internet (FAI) bloquent les ports 80 et 443 entrants, ou fournissent des IP dynamiques qui changent régulièrement. Nous verrons comment contourner ces limitations avec des solutions alternatives.

### Coûts
Bien que MicroK8s soit gratuit, certains éléments de cette configuration peuvent engendrer des coûts :
- Nom de domaine (10-50€/an selon l'extension)
- Service de DNS dynamique si nécessaire (0-10€/mois)
- IP fixe auprès de votre FAI (optionnel, 5-20€/mois)

### Performance
Héberger des services depuis votre connexion domestique implique des limitations de bande passante. Votre débit upload sera le facteur limitant pour les utilisateurs externes accédant à vos services.

## Outils que nous utiliserons

Tout au long de cette section, nous utiliserons plusieurs outils essentiels :

- **kubectl** : Pour interagir avec le cluster Kubernetes
- **nslookup/dig** : Pour tester la résolution DNS
- **curl/wget** : Pour tester la connectivité HTTP/HTTPS
- **netstat/ss** : Pour vérifier les ports ouverts
- **iptables/ufw** : Pour la configuration du firewall
- **tcpdump** : Pour le débogage réseau avancé (optionnel)

## Résultats attendus

À la fin de cette section, vous disposerez d'un lab MicroK8s avec :

- ✅ Un réseau cluster fonctionnel et bien configuré
- ✅ Un système DNS permettant l'accès par nom plutôt que par IP
- ✅ Un ou plusieurs noms de domaine pointant vers votre lab
- ✅ Des services accessibles depuis Internet de manière sécurisée
- ✅ Une configuration firewall robuste
- ✅ Les connaissances pour diagnostiquer et résoudre les problèmes réseau

## Temps estimé

La configuration complète de cette section nécessite généralement :
- **2-3 heures** pour la configuration de base et les tests locaux
- **1-2 heures** pour l'acquisition et la configuration du domaine
- **1 heure** pour la configuration du routeur et du firewall
- **30 minutes à 1 heure** pour les tests et ajustements finaux

Prévoyez donc une demi-journée pour implémenter confortablement l'ensemble de cette section.

## Dépannage préventif

Les problèmes les plus courants que vous pourriez rencontrer :

1. **Services inaccessibles depuis l'extérieur** : Généralement dû à une mauvaise configuration de la redirection de ports ou au firewall
2. **Résolution DNS qui échoue** : Souvent causé par la propagation DNS (jusqu'à 48h) ou des erreurs de configuration
3. **Certificats SSL non valides** : Problème de DNS ou de configuration de l'Ingress Controller
4. **Performance dégradée** : Limitation de bande passante ou mauvaise configuration réseau

Nous aborderons la résolution de chacun de ces problèmes dans les sections appropriées.

## Commençons !

Maintenant que vous comprenez l'importance et la portée de la configuration réseau, passons à la première étape : comprendre les concepts de base du réseau Kubernetes. Cette fondation théorique vous aidera à mieux appréhender les configurations pratiques qui suivront.

---

⏭️
