🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 7 : Déploiement d'Applications

## Vue d'ensemble

Le déploiement d'applications est le cœur de l'utilisation de Kubernetes. Après avoir configuré votre cluster MicroK8s avec le réseau, les certificats SSL et l'ingress controller, vous êtes maintenant prêt à déployer vos premières applications. Ce chapitre vous guidera à travers les concepts fondamentaux et les meilleures pratiques pour déployer, exposer et gérer des applications dans votre lab personnel.

## Objectifs d'apprentissage

À la fin de ce chapitre, vous serez capable de :

- Comprendre la structure complète d'un déploiement Kubernetes et ses composants
- Créer et gérer des manifestes YAML pour vos applications
- Déployer différents types d'applications (stateless et stateful)
- Exposer vos applications à l'intérieur et à l'extérieur du cluster
- Gérer la configuration et les secrets de manière sécurisée
- Implémenter du stockage persistant pour vos applications
- Appliquer les bonnes pratiques de déploiement dans un environnement de lab

## Prérequis

Avant de commencer ce chapitre, assurez-vous d'avoir :

### Infrastructure requise
- MicroK8s installé et fonctionnel (chapitres 1-2)
- Réseau configuré avec un domaine accessible (chapitre 3)
- Ingress controller NGINX activé et configuré (chapitres 4 et 6)
- Cert-Manager opérationnel pour les certificats SSL (chapitre 5)
- Stockage persistant activé via l'addon storage de MicroK8s

### Connaissances préalables
- Compréhension basique des concepts de conteneurisation (Docker/OCI)
- Familiarité avec la syntaxe YAML
- Notions de base sur les architectures d'applications web
- Compréhension du modèle client-serveur et des APIs REST

### Outils nécessaires
```bash
# Vérifier que kubectl est configuré
kubectl version --short

# Vérifier l'accès au cluster
kubectl cluster-info

# Vérifier les addons activés
microk8s status

# S'assurer que les namespaces système sont opérationnels
kubectl get pods --all-namespaces
```

## Ce que nous allons construire

Dans ce chapitre, nous allons progressivement construire un environnement d'applications complet comprenant :

### Applications de démonstration
1. **Application web simple** : Un serveur web nginx servant du contenu statique
2. **Application avec base de données** : Une application WordPress avec MySQL
3. **Microservices** : Un ensemble de services communiquant entre eux
4. **Application stateful** : Un service nécessitant du stockage persistant

### Infrastructure d'application
- Services pour l'exposition interne et externe
- Ingress rules pour le routage HTTP/HTTPS
- ConfigMaps pour la configuration externalisée
- Secrets pour les données sensibles
- PersistentVolumes pour le stockage de données

## Structure du chapitre

Ce chapitre suit une approche progressive, partant des concepts les plus simples vers les plus complexes :

**Fondamentaux (7.1-7.3)** : Nous commencerons par comprendre l'anatomie d'un déploiement Kubernetes, la structure des manifestes YAML, et le déploiement d'une première application simple.

**Exposition et routage (7.4)** : Nous apprendrons à rendre nos applications accessibles via Services et Ingress, en utilisant les configurations établies dans les chapitres précédents.

**Configuration (7.5-7.6)** : Nous explorerons la gestion de la configuration avec ConfigMaps et la sécurisation des données sensibles avec Secrets.

**Stockage (7.7)** : Nous terminerons par la gestion du stockage persistant, essentiel pour les applications stateful.

## Approche pédagogique

### Philosophie "Lab First"
Ce chapitre adopte une approche orientée lab personnel, ce qui signifie :
- Focus sur l'apprentissage et l'expérimentation plutôt que la production
- Configurations simplifiées mais représentatives de cas réels
- Possibilité de tout casser et reconstruire facilement
- Exemples reproductibles et modifiables

### Progression incrémentale
Chaque section s'appuie sur la précédente :
1. Commencer simple avec un pod unique
2. Ajouter la réplication avec un Deployment
3. Exposer avec un Service
4. Ajouter le routage externe avec Ingress
5. Enrichir avec configuration et secrets
6. Finaliser avec le stockage persistant

### Bonnes pratiques intégrées
Tout au long du chapitre, nous intégrerons les bonnes pratiques :
- Utilisation de labels et selectors cohérents
- Gestion des ressources (requests/limits)
- Health checks (liveness/readiness probes)
- Stratégies de mise à jour
- Principes de sécurité de base

## Points d'attention pour un lab personnel

### Différences avec la production
Dans un lab personnel, certains aspects diffèrent d'un environnement de production :

**Ressources limitées** : Votre lab dispose probablement de ressources limitées. Nous adapterons les configurations pour être économes en CPU et mémoire.

**Haute disponibilité optionnelle** : En lab, la redondance n'est pas critique. Nous utiliserons souvent des replicas=1 pour économiser les ressources.

**Sécurité allégée** : Bien que nous appliquions les bonnes pratiques de sécurité, certaines contraintes de production peuvent être assouplies pour faciliter l'apprentissage.

### Opportunités d'apprentissage
Un lab personnel offre des avantages uniques :

**Liberté d'expérimentation** : Testez des configurations risquées, cassez des choses, apprenez des erreurs.

**Cycles rapides** : Déployez, détruisez, redéployez rapidement sans processus d'approbation.

**Stack complète** : Gérez tous les aspects, de l'infrastructure à l'application.

## Conventions utilisées

### Nommage
Nous utiliserons des conventions de nommage cohérentes :
```yaml
# Format : [type]-[application]-[environnement]
name: deployment-webapp-lab
name: service-webapp-lab
name: ingress-webapp-lab
```

### Namespaces
Organisation par namespaces logiques :
```yaml
# Applications de test
namespace: lab-apps

# Applications de développement
namespace: dev-apps

# Outils et services
namespace: lab-tools
```

### Labels standards
Labels cohérents pour la gestion :
```yaml
labels:
  app: webapp
  environment: lab
  version: v1
  managed-by: kubectl
```

## Ressources et support

### Documentation de référence
- [Documentation officielle Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [MicroK8s Applications Guide](https://microk8s.io/docs/addon-dashboard)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### Outils de debugging
Commandes essentielles pour le troubleshooting :
```bash
# Logs d'une application
kubectl logs -f deployment/mon-app

# Description détaillée d'une ressource
kubectl describe pod/mon-pod

# Accès shell dans un conteneur
kubectl exec -it pod/mon-pod -- /bin/bash

# Port-forward pour test local
kubectl port-forward service/mon-service 8080:80
```

### Environnement de test
Avant de commencer les exercices, créons un namespace dédié :
```bash
# Créer le namespace pour nos tests
kubectl create namespace lab-apps

# Définir comme namespace par défaut
kubectl config set-context --current --namespace=lab-apps

# Vérifier le contexte
kubectl config get-contexts
```

## Résumé des concepts clés

Avant de plonger dans les détails techniques, rappelons les concepts Kubernetes essentiels pour le déploiement d'applications :

**Pod** : L'unité de base, contenant un ou plusieurs conteneurs partageant réseau et stockage.

**Deployment** : Gère la création et mise à jour déclarative des Pods, avec gestion des replicas et stratégies de déploiement.

**Service** : Abstraction réseau stable pour accéder à un ensemble de Pods, avec load balancing intégré.

**Ingress** : Routage HTTP/HTTPS externe vers les services, avec gestion des domaines et certificats SSL.

**ConfigMap** : Stockage de configuration non-sensible, injectable dans les conteneurs.

**Secret** : Stockage sécurisé de données sensibles comme mots de passe et clés API.

**PersistentVolume** : Abstraction du stockage, permettant aux applications de persister des données.

## Prêt à commencer ?

Vous êtes maintenant prêt à explorer le déploiement d'applications sur votre cluster MicroK8s. Nous allons commencer par comprendre en détail l'anatomie d'un déploiement Kubernetes, puis progresser vers des scénarios de plus en plus complexes et réalistes.

N'oubliez pas : l'objectif principal est l'apprentissage. N'hésitez pas à expérimenter, modifier les exemples, et même à casser des choses. C'est en pratiquant dans votre lab que vous développerez une compréhension profonde de Kubernetes.

---

*Note : Les exemples de ce chapitre sont conçus pour MicroK8s en environnement lab. Pour un usage en production, des considérations supplémentaires de sécurité, performance et haute disponibilité seraient nécessaires.*

⏭️
