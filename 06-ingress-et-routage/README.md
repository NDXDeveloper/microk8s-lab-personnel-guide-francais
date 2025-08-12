🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 06. Ingress et Routage

## Introduction

Dans un cluster Kubernetes, vos applications s'exécutent dans des pods qui, par défaut, ne sont accessibles que depuis l'intérieur du cluster. Pour qu'elles soient disponibles depuis l'extérieur - que ce soit depuis votre réseau local ou depuis Internet - vous devez mettre en place un mécanisme de routage approprié. C'est là qu'intervient **Ingress**, l'une des ressources les plus puissantes de Kubernetes pour gérer l'accès externe à vos services.

## Pourquoi Ingress est essentiel dans votre lab

Imaginez que vous hébergez plusieurs applications dans votre cluster MicroK8s : un blog WordPress, une instance Nextcloud, un serveur GitLab, et plusieurs applications de développement. Sans Ingress, vous devriez :
- Exposer chaque service sur un port différent (NodePort)
- Mémoriser quel port correspond à quelle application
- Gérer manuellement les certificats SSL pour chaque service
- Configurer individuellement chaque point d'entrée

Avec Ingress, vous pouvez :
- **Utiliser des noms de domaine** : accéder à `blog.monlab.local`, `cloud.monlab.local`, `git.monlab.local`
- **Centraliser la gestion SSL/TLS** : un seul point pour tous vos certificats
- **Router intelligemment** : diriger le trafic selon l'URL, les en-têtes, ou d'autres critères
- **Implémenter des fonctionnalités avancées** : limitation de débit, authentification, redirections

## Ce que vous allez apprendre

Cette section vous guidera à travers tous les aspects du routage dans MicroK8s, depuis les concepts fondamentaux jusqu'aux configurations avancées. Vous découvrirez comment transformer votre cluster en une plateforme capable d'héberger et de router efficacement plusieurs applications web.

### Objectifs de la section

À la fin de cette section, vous serez capable de :

✅ **Comprendre l'architecture** d'Ingress et son rôle dans l'écosystème Kubernetes
✅ **Installer et configurer** NGINX Ingress Controller dans MicroK8s
✅ **Créer des règles de routage** basées sur les noms d'hôte et les chemins URL
✅ **Implémenter le HTTPS** automatiquement avec redirection du HTTP
✅ **Utiliser les annotations** pour personnaliser le comportement du routage
✅ **Déboguer** les problèmes de routage courants
✅ **Optimiser** les performances et la sécurité de vos points d'entrée

## Prérequis nécessaires

Avant de commencer cette section, assurez-vous d'avoir :

### ✔️ Environnement technique
- MicroK8s installé et fonctionnel (sections 01-02)
- Addon DNS activé : `microk8s enable dns`
- Au moins une application de test déployée (section 07)
- Accès administrateur au cluster

### ✔️ Configuration réseau
- Compréhension de base du réseau Kubernetes (section 03)
- Un nom de domaine configuré ou la possibilité d'éditer `/etc/hosts`
- Ports 80 et 443 disponibles sur votre machine hôte

### ✔️ Connaissances préalables
- Familiarité avec les concepts de Service dans Kubernetes
- Compréhension basique du protocole HTTP/HTTPS
- Capacité à lire et modifier des fichiers YAML

## Architecture générale d'Ingress

Avant de plonger dans les détails techniques, visualisons comment Ingress s'intègre dans votre cluster :

```
Internet/Réseau Local
         |
         ↓
    [Routeur/Box]
         |
         ↓ (Ports 80/443)
    [Machine Hôte]
         |
         ↓
    [MicroK8s Cluster]
         |
    ┌────────────┐
    │  Ingress   │ ← Point d'entrée unique
    │ Controller │    (NGINX)
    └────────────┘
         |
    ┌────┴────┬──────┬──────┐
    ↓         ↓      ↓      ↓
[Service A] [Service B] [Service C] [Service D]
    ↓         ↓      ↓      ↓
[Pods A]   [Pods B] [Pods C] [Pods D]
```

## Cas d'usage typiques dans un lab personnel

### 🏠 **Hébergement multi-applications**
Exposer plusieurs applications web sous différents sous-domaines :
- `app1.lab.local` → Application de développement
- `app2.lab.local` → Application de test
- `monitoring.lab.local` → Grafana/Prometheus

### 🔧 **Environnements de développement**
Router vers différentes versions d'une même application :
- `dev.monapp.local` → Version développement
- `staging.monapp.local` → Version pré-production
- `prod.monapp.local` → Version production

### 🔒 **Gestion centralisée de la sécurité**
- Terminer le SSL/TLS au niveau d'Ingress
- Implémenter l'authentification basique ou OAuth
- Appliquer des politiques de sécurité uniformes

### 📊 **Services internes**
Exposer des outils d'administration et de monitoring :
- Dashboard Kubernetes
- Interfaces de gestion des bases de données
- Outils de CI/CD

## Avantages spécifiques pour MicroK8s

MicroK8s simplifie considérablement la mise en place d'Ingress :

1. **Installation en une commande** : `microk8s enable ingress`
2. **Configuration pré-optimisée** pour un usage lab/développement
3. **Intégration native** avec les autres addons MicroK8s
4. **Ressources limitées** : optimisé pour fonctionner sur une machine modeste
5. **Support MetalLB** : permet d'avoir un vrai LoadBalancer même en bare-metal

## Points d'attention

⚠️ **Considérations importantes** :

- **Consommation mémoire** : L'Ingress Controller consomme environ 100-200MB de RAM
- **Conflit de ports** : Assurez-vous que les ports 80/443 ne sont pas utilisés
- **DNS local** : La résolution DNS doit être correctement configurée
- **Certificats** : Sans configuration SSL, vos connexions ne seront pas chiffrées
- **Performance** : Un seul Ingress Controller peut devenir un goulot d'étranglement

## Structure de cette section

Nous allons progresser de manière logique et pratique :

1. **Concepts fondamentaux** (6.1) : Comprendre ce qu'est Ingress et comment il fonctionne
2. **Installation et configuration** (6.2) : Mettre en place NGINX Ingress Controller
3. **Routage par nom d'hôte** (6.3) : Router différents domaines vers différents services
4. **Routage par chemin** (6.4) : Utiliser les paths URL pour diriger le trafic
5. **Fonctionnalités avancées** (6.5) : Middleware, annotations et personnalisation
6. **HTTPS et redirections** (6.6) : Sécuriser vos connexions
7. **Exemples pratiques** (6.7) : Cas d'usage réels avec code complet

## Préparation de l'environnement

Avant de commencer, vérifions que votre environnement est prêt :

```bash
# Vérifier le statut de MicroK8s
microk8s status

# Vérifier les addons activés
microk8s status | grep enabled

# Si l'addon ingress n'est pas activé, l'activer maintenant
microk8s enable ingress

# Vérifier que l'Ingress Controller est en cours d'exécution
microk8s kubectl get pods -n ingress -l name=nginx-ingress-microk8s

# Vérifier les services exposés
microk8s kubectl get svc -n ingress
```

## Ressources et documentation

Pour approfondir vos connaissances pendant cette section :

- 📚 **Documentation officielle NGINX Ingress** : Configuration détaillée et références
- 🔧 **Kubernetes Ingress API** : Spécifications officielles de la ressource Ingress
- 💡 **MicroK8s Ingress Guide** : Guide spécifique à MicroK8s
- 🎯 **Ingress Patterns** : Patterns et anti-patterns courants

## Prêt à commencer ?

Vous disposez maintenant d'une vue d'ensemble complète de ce qui vous attend dans cette section. L'Ingress est l'une des fonctionnalités les plus puissantes de Kubernetes pour exposer vos services, et maîtriser son utilisation transformera votre lab MicroK8s en une véritable plateforme d'hébergement professionnelle.

Dans la prochaine sous-section (6.1), nous plongerons dans les concepts fondamentaux d'Ingress pour comprendre en détail comment cette ressource fonctionne sous le capot.

---

💡 **Conseil** : Gardez un terminal ouvert avec `kubectl get ingress -A --watch` pendant vos expérimentations pour voir en temps réel les changements dans vos ressources Ingress.

⏭️
