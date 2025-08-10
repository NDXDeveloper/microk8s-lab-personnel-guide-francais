🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 02. Installation et Configuration Initiale

## Vue d'ensemble

L'installation de MicroK8s représente votre première étape vers la création d'un environnement Kubernetes fonctionnel pour votre lab personnel. Contrairement aux distributions Kubernetes traditionnelles qui nécessitent une configuration complexe et multiple composants, MicroK8s offre une approche simplifiée tout en maintenant la compatibilité avec l'écosystème Kubernetes standard.

Cette section vous guidera à travers les différentes méthodes d'installation selon votre système d'exploitation, ainsi que les configurations initiales essentielles pour démarrer avec un cluster opérationnel.

## Objectifs de cette section

À la fin de cette section, vous serez capable de :

- Installer MicroK8s sur votre système d'exploitation préféré
- Vérifier que votre installation fonctionne correctement
- Configurer kubectl pour interagir efficacement avec votre cluster
- Effectuer les premiers diagnostics pour valider la santé de votre environnement
- Comprendre la structure des fichiers et répertoires créés par MicroK8s

## Pourquoi MicroK8s pour votre lab ?

MicroK8s se distingue par plusieurs caractéristiques qui le rendent idéal pour un environnement de lab personnel :

**Légèreté et efficacité** : MicroK8s utilise snap comme système de packaging, ce qui signifie une installation atomique, des mises à jour automatiques et une isolation complète du système hôte. L'empreinte mémoire minimale (540MB) permet de l'exécuter même sur des machines modestes.

**Production-ready** : Bien qu'optimisé pour le développement et les labs, MicroK8s utilise les mêmes binaires Kubernetes que les déploiements de production. Vos apprentissages et configurations sont directement transférables vers des environnements d'entreprise.

**Gestion simplifiée** : Les certificats, le réseau et le stockage sont préconfigurés avec des valeurs sensées. Vous pouvez avoir un cluster fonctionnel en moins de 60 secondes après l'installation.

## Architecture de l'installation

MicroK8s s'installe comme un snap confiné qui encapsule tous les composants Kubernetes nécessaires :

```
/snap/microk8s/
├── current/           # Lien vers la version active
├── common/           # Données partagées entre versions
│   ├── .kube/       # Configuration kubectl
│   └── var/         # Données du cluster
└── [version]/       # Binaires spécifiques à chaque version
    ├── kubectl
    ├── kubelet
    ├── kube-apiserver
    └── ...
```

Cette structure permet des mises à jour sans interruption et la possibilité de revenir à une version antérieure si nécessaire.

## Prérequis techniques

Avant de procéder à l'installation, assurez-vous que votre système répond aux exigences minimales :

**Ressources matérielles minimales** :
- CPU : 1 cœur (2+ recommandés pour des charges de travail réelles)
- RAM : 2GB minimum (4GB+ recommandé)
- Stockage : 20GB d'espace disque disponible
- Architecture : x86_64, ARM64, ou s390x

**Connectivité réseau** :
- Accès Internet pour télécharger les images de conteneurs
- Ports 16443 (API server) et 10250-10255 (kubelet) disponibles
- Support IPv4 (IPv6 optionnel mais supporté)

**Systèmes d'exploitation supportés** :
- Ubuntu 18.04 LTS ou supérieur
- Debian 10 ou supérieur
- CentOS 7 ou supérieur / RHEL 7+
- macOS 10.13+ (via Multipass)
- Windows 10/11 Pro/Enterprise (via WSL2 ou Multipass)

## Méthodes d'installation disponibles

MicroK8s offre plusieurs voies d'installation adaptées à différents contextes :

**Installation native via Snap** : La méthode recommandée pour les distributions Linux qui supportent snap. Offre les meilleures performances et l'intégration la plus transparente.

**Installation via package manager** : Pour les systèmes qui ne supportent pas snap nativement, des packages .deb et .rpm sont disponibles, bien que cette méthode nécessite une gestion manuelle des mises à jour.

**Installation virtualisée** : Pour Windows et macOS, MicroK8s s'exécute dans une machine virtuelle légère via Multipass ou WSL2, offrant une expérience proche du natif avec une overhead minimal.

**Installation depuis les sources** : Pour les utilisateurs avancés souhaitant personnaliser leur build ou contribuer au développement.

## Considérations de sécurité initiales

Par défaut, MicroK8s s'installe avec des permissions restrictives pour garantir la sécurité :

- Seuls les utilisateurs du groupe `microk8s` peuvent interagir avec le cluster
- Les certificats sont générés automatiquement et stockés de manière sécurisée
- Le firewall local n'est pas modifié automatiquement
- L'API server n'est accessible que localement par défaut

Nous aborderons les configurations de sécurité avancées dans la section dédiée, mais ces defaults offrent un bon équilibre entre sécurité et facilité d'utilisation pour un lab.

## Stratégies de déploiement pour votre lab

Selon vos objectifs, vous pouvez adopter différentes approches :

**Single-node pour l'apprentissage** : Parfait pour découvrir Kubernetes, tester des applications et comprendre les concepts fondamentaux. La majorité de ce guide se concentre sur cette configuration.

**Multi-node pour la simulation** : Si vous disposez de plusieurs machines (physiques ou virtuelles), MicroK8s permet facilement de créer un cluster multi-node pour simuler des scénarios de production.

**Hybride avec cloud** : MicroK8s peut s'intégrer avec des nodes dans le cloud pour créer un cluster hybride, idéal pour tester des scénarios de burst ou de disaster recovery.

## Workflow post-installation

Une fois MicroK8s installé, le workflow typique comprend :

1. **Vérification** : Confirmer que tous les composants sont opérationnels
2. **Configuration kubectl** : Établir l'accès en ligne de commande au cluster
3. **Activation des addons** : Enabled les fonctionnalités dont vous avez besoin
4. **Test de déploiement** : Déployer une application simple pour valider
5. **Configuration réseau** : Ajuster selon vos besoins d'exposition externe

## Points d'attention

Quelques éléments à garder à l'esprit lors de l'installation :

**Gestion des versions** : MicroK8s suit le calendrier de release de Kubernetes upstream. Les canaux (stable, candidate, beta, edge) permettent de choisir votre niveau de stabilité vs nouveautés.

**Persistance des données** : Les données du cluster sont stockées dans `/var/snap/microk8s/common/`. Assurez-vous d'avoir suffisamment d'espace et considérez des sauvegardes régulières.

**Compatibilité des addons** : Tous les addons ne sont pas compatibles entre eux. La documentation de chaque addon précise les éventuels conflits.

**Impact sur le système** : Bien que confiné, MicroK8s peut consommer des ressources significatives selon les workloads. Monitorer l'utilisation CPU/RAM est recommandé.

## Préparation à l'installation

Avant de passer aux instructions spécifiques par OS, voici une checklist de préparation :

- [ ] Vérifier les prérequis système (CPU, RAM, stockage)
- [ ] S'assurer d'avoir les droits administrateur/sudo
- [ ] Vérifier la disponibilité des ports requis
- [ ] Désactiver temporairement les antivirus/pare-feu restrictifs
- [ ] Préparer une connexion Internet stable pour les téléchargements
- [ ] Décider du canal de release à utiliser (stable recommandé)
- [ ] Identifier les addons que vous souhaitez activer

---

Passons maintenant aux instructions d'installation spécifiques pour chaque système d'exploitation, en commençant par Ubuntu/Debian qui offre l'expérience la plus native avec MicroK8s.

⏭️
