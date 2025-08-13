🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.1 Documentation officielle

## Vue d'ensemble

La documentation officielle est votre source de vérité pour tout ce qui concerne MicroK8s et Kubernetes. Elle est maintenue par les équipes de développement et mise à jour à chaque nouvelle version. C'est toujours votre premier point de référence pour obtenir des informations fiables et actualisées.

## Documentation MicroK8s

### Site principal de MicroK8s
**URL** : https://microk8s.io/

Le site officiel de MicroK8s est votre point d'entrée principal. Il contient :
- Les guides de démarrage rapide
- Les actualités sur les nouvelles versions
- Les cas d'usage recommandés
- Les comparaisons avec d'autres distributions Kubernetes

### Documentation technique MicroK8s
**URL** : https://microk8s.io/docs

C'est ici que vous trouverez toute la documentation technique détaillée :

#### Sections essentielles pour débuter
- **Getting Started** : Installation pas à pas pour chaque système d'exploitation
- **How-to Guides** : Guides pratiques pour les tâches courantes
- **Tutorials** : Tutoriels complets pour des scénarios spécifiques
- **Reference** : Documentation de référence pour toutes les commandes

#### Navigation dans la documentation
La documentation MicroK8s est organisée selon le principe Diátaxis :
1. **Tutoriels** : Pour apprendre en faisant (idéal pour débuter)
2. **Guides pratiques** : Pour accomplir des tâches spécifiques
3. **Explications** : Pour comprendre les concepts en profondeur
4. **Référence** : Pour consulter les détails techniques

### Changelog et notes de version
**URL** : https://github.com/canonical/microk8s/releases

Consultez cette page pour :
- Connaître les nouvelles fonctionnalités de chaque version
- Identifier les bugs corrigés
- Vérifier les breaking changes avant une mise à jour
- Comprendre l'évolution du projet

**Conseil pour débutants** : Restez sur les versions stables (marquées "stable") plutôt que les versions edge ou beta pour votre lab de production.

## Documentation Kubernetes

### Documentation officielle Kubernetes
**URL** : https://kubernetes.io/docs/

MicroK8s étant une distribution de Kubernetes, la documentation Kubernetes officielle reste essentielle :

#### Sections clés pour les utilisateurs MicroK8s
1. **Concepts** : Pour comprendre les fondamentaux (Pods, Services, Deployments, etc.)
2. **Tasks** : Guides pratiques applicables à MicroK8s
3. **Reference** : API complète et documentation kubectl
4. **Tutorials** : Apprentissage interactif

#### Points d'attention
Certaines sections de la documentation Kubernetes ne s'appliquent pas directement à MicroK8s :
- **Installation** : MicroK8s a sa propre méthode d'installation simplifiée
- **Cluster Administration** : Certaines tâches sont automatisées dans MicroK8s
- **Cloud Providers** : Les intégrations cloud natives ne sont pas toujours pertinentes pour un lab local

### API Reference
**URL** : https://kubernetes.io/docs/reference/kubernetes-api/

Documentation complète de l'API Kubernetes, essentielle pour :
- Comprendre la structure des manifestes YAML
- Connaître tous les champs disponibles pour chaque ressource
- Déboguer les erreurs liées aux définitions de ressources

**Astuce** : Utilisez la version d'API correspondant à votre version de MicroK8s. Vérifiez avec :
```bash
microk8s kubectl version
```

## Documentation des Addons

### Addons MicroK8s officiels
**URL** : https://microk8s.io/docs/addons

Chaque addon MicroK8s a sa propre documentation :
- Configuration et activation
- Options disponibles
- Cas d'usage recommandés
- Limitations connues

### Documentation des projets upstream

Les addons MicroK8s sont souvent des intégrations de projets open source. Voici les documentations originales des addons les plus utilisés :

#### Ingress NGINX
**URL** : https://kubernetes.github.io/ingress-nginx/
- Configuration avancée des règles d'ingress
- Annotations disponibles
- Exemples de configurations

#### Cert-Manager
**URL** : https://cert-manager.io/docs/
- Gestion automatique des certificats SSL/TLS
- Intégration avec Let's Encrypt
- Dépannage des problèmes de certificats

#### Prometheus
**URL** : https://prometheus.io/docs/
- Configuration des métriques
- Langage de requête PromQL
- Bonnes pratiques de monitoring

#### Grafana
**URL** : https://grafana.com/docs/grafana/latest/
- Création de dashboards
- Configuration des datasources
- Alerting

#### MetalLB
**URL** : https://metallb.universe.tf/
- Configuration du load balancer pour environnement bare-metal
- Modes Layer 2 et BGP

## GitHub et ressources de développement

### Repository MicroK8s
**URL** : https://github.com/canonical/microk8s

Le repository GitHub est précieux pour :
- **Issues** : Rechercher des problèmes connus et leurs solutions
- **Pull Requests** : Voir les changements à venir
- **Discussions** : Participer aux décisions sur l'évolution du projet
- **Code source** : Comprendre le fonctionnement interne

#### Utilisation efficace des Issues
1. **Recherchez d'abord** : Votre problème a peut-être déjà été signalé
2. **Filtrez par labels** : "bug", "enhancement", "documentation", etc.
3. **Vérifiez les issues fermées** : Les solutions peuvent s'y trouver
4. **Créez une issue structurée** : Si votre problème est nouveau, suivez le template

### Canonical Documentation
**URL** : https://ubuntu.com/kubernetes/docs

Documentation de Canonical (éditeur de MicroK8s) sur leurs solutions Kubernetes :
- Intégration avec l'écosystème Ubuntu
- Solutions entreprise
- Support commercial

## Commandes de documentation intégrées

### Aide en ligne de commande

MicroK8s inclut une aide intégrée accessible directement :

```bash
# Aide générale MicroK8s
microk8s --help

# Aide pour une commande spécifique
microk8s status --help
microk8s enable --help

# Documentation kubectl
microk8s kubectl --help
microk8s kubectl explain pod
microk8s kubectl explain deployment.spec
```

### Documentation des ressources Kubernetes

La commande `kubectl explain` est particulièrement utile pour comprendre les manifestes :

```bash
# Structure complète d'une ressource
microk8s kubectl explain service --recursive

# Champ spécifique
microk8s kubectl explain service.spec.type

# Avec exemples
microk8s kubectl explain deployment.spec.replicas
```

## Stratégies de recherche dans la documentation

### Pour résoudre un problème

1. **Commencez par le message d'erreur exact** : Copiez-le dans le moteur de recherche de la documentation
2. **Consultez les logs** : `microk8s kubectl logs` et `microk8s inspect`
3. **Vérifiez les Issues GitHub** : Quelqu'un a peut-être rencontré le même problème
4. **Lisez la documentation de l'addon concerné** : Si le problème touche un addon spécifique

### Pour apprendre un nouveau concept

1. **Documentation Kubernetes Concepts** : Pour la théorie
2. **MicroK8s Tutorials** : Pour la pratique spécifique à MicroK8s
3. **Exemples de manifestes** : Dans la documentation de référence
4. **Guides How-to** : Pour les implémentations concrètes

### Pour optimiser ou améliorer

1. **Best Practices** dans la documentation Kubernetes
2. **Performance tuning** dans les guides avancés
3. **Security** dans la section dédiée
4. **Production readiness** dans les références

## Versions et compatibilité

### Tableau de correspondance des versions

Il est crucial de consulter la documentation correspondant à votre version :

| MicroK8s | Kubernetes | Documentation |
|----------|------------|---------------|
| 1.29     | 1.29.x     | https://microk8s.io/docs |
| 1.28     | 1.28.x     | Sélecteur de version sur le site |
| 1.27     | 1.27.x     | Archives disponibles |

### Vérifier votre version

```bash
# Version de MicroK8s
microk8s version

# Version de Kubernetes
microk8s kubectl version --short

# Addons installés et leurs versions
microk8s status --format short
```

## Documentation hors ligne

### Téléchargement pour consultation hors ligne

Pour travailler sans connexion internet :

1. **PDF de la documentation Kubernetes** : Disponible sur le site officiel
2. **Clone du repository** : `git clone https://github.com/canonical/microk8s`
3. **Cache local avec wget** : Pour aspirer le site de documentation
4. **Dash/Zeal** : Applications de documentation hors ligne avec docsets Kubernetes

### Génération de documentation locale

```bash
# Pour générer la documentation des API de votre cluster
microk8s kubectl proxy &
# Puis accédez à http://localhost:8001/openapi/v2
```

## Bonnes pratiques de consultation

### Organisation personnelle

1. **Bookmarks organisés** : Créez des dossiers par thème
2. **Notes personnelles** : Gardez un fichier de notes avec les solutions qui fonctionnent
3. **Snippets** : Conservez les commandes et configurations utiles
4. **Changelog personnel** : Notez les changements lors des mises à jour

### Apprentissage progressif

1. **Commencez par les tutoriels** : Même s'ils semblent basiques
2. **Lisez les concepts** : Pour comprendre le "pourquoi"
3. **Pratiquez avec les how-to** : Pour maîtriser le "comment"
4. **Consultez les références** : Quand vous avez besoin de détails précis

### Veille documentaire

1. **RSS Feed** : Abonnez-vous aux flux RSS des blogs officiels
2. **Release notes** : Lisez-les à chaque mise à jour
3. **Changelog** : Surveillez les changements importants
4. **Deprecation notices** : Anticipez les fonctionnalités obsolètes

## Contribuer à la documentation

### Signaler des erreurs

Si vous trouvez une erreur dans la documentation :

1. **MicroK8s Docs** : Bouton "Edit this page" sur GitHub
2. **Kubernetes Docs** : Processus de contribution via Pull Request
3. **Traductions** : Aidez à traduire la documentation

### Proposer des améliorations

- **Exemples manquants** : Proposez des cas d'usage concrets
- **Clarifications** : Suggérez des reformulations plus claires
- **Nouveaux guides** : Documentez des scénarios non couverts

## Points d'attention pour les débutants

### Erreurs courantes à éviter

1. **Ne pas vérifier la version** : La documentation évolue avec les versions
2. **Ignorer les prérequis** : Chaque guide a ses conditions préalables
3. **Copier-coller sans comprendre** : Prenez le temps de lire les explications
4. **Négliger les warnings** : Les avertissements sont là pour vous protéger

### Conseils pour bien démarrer

1. **Suivez l'ordre suggéré** : La documentation a une progression logique
2. **Testez chaque étape** : Ne passez pas à la suite si quelque chose ne fonctionne pas
3. **Comprenez les erreurs** : Lisez les messages d'erreur attentivement
4. **Gardez des notes** : Documentez ce qui fonctionne dans votre environnement

## Ressources complémentaires officielles

### Webinars et présentations
- **Canonical Webinars** : Sessions régulières sur MicroK8s
- **KubeCon Talks** : Présentations sur MicroK8s lors des conférences
- **YouTube Canonical** : Tutoriels vidéo officiels

### Formations officielles
- **Canonical Training** : Formations certifiées sur MicroK8s
- **Linux Foundation** : Cours Kubernetes applicables à MicroK8s
- **CNCF Training** : Ressources d'apprentissage cloud native

---

Cette documentation officielle constitue votre fondation. Prenez le temps de vous familiariser avec sa structure et son organisation. Plus vous la consulterez, plus vous deviendrez efficace dans la résolution de problèmes et l'apprentissage de nouvelles fonctionnalités.

⏭️
