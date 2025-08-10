# Guide MicroK8s Lab Personnel 🚀

Un guide complet en français pour créer et administrer votre propre laboratoire MicroK8s avec nom de domaine personnalisé.

## 🎯 À propos

Ce guide vous accompagne dans la création d'un environnement MicroK8s professionnel, depuis l'installation de base jusqu'aux concepts avancés comme la surveillance avec Prometheus/Grafana, la gestion des certificats SSL/TLS automatiques, et l'implémentation de pipelines CI/CD.

## 📋 Prérequis

- Connaissance de base de Linux
- Machine virtuelle ou serveur avec minimum 4GB RAM
- Accès à un nom de domaine (optionnel pour débuter)
- Motivation pour apprendre Kubernetes ! 💪

## 🗂️ Contenu

La table des matières complète est disponible dans [SOMMAIRE.md](SOMMAIRE.md).

### Principaux chapitres :

- **Installation et Configuration** - Mise en place sur différents OS
- **Réseau et Domaine** - Configuration DNS et certificats SSL/TLS
- **Addons Essentiels** - Dashboard, Ingress, Storage, MetalLB
- **Stack Prometheus/Grafana** - Monitoring et observabilité complète
- **Sécurité** - RBAC, Network Policies, bonnes pratiques
- **DevOps/CI-CD** - GitOps avec ArgoCD, pipelines automatisés
- **Haute Disponibilité** - Scaling et clustering multi-node

## 🚀 Démarrage rapide

1. **Clonez le repository**
   ```bash
   git clone https://github.com/NDXdev/microk8s-lab-personnel-guide-francais.git
   cd microk8s-lab-personnel-guide-francais
   ```

2. **Consultez le sommaire**
   ```bash
   cat SOMMAIRE.md
   ```

3. **Commencez par l'introduction**
   ```bash
   cd 01-introduction-et-prerequis
   ```

## 💡 Pourquoi ce guide ?

- **🇫🇷 En français** - Documentation claire dans notre langue
- **🏠 Lab personnel** - Adapté aux environnements domestiques et petites structures
- **📚 Pédagogique** - Progression logique du débutant à l'expert
- **🔧 Pratique** - Exemples concrets et configurations réelles
- **🔄 Moderne** - Technologies actuelles (2025) et bonnes pratiques

## 🛠️ Technologies couvertes

| Catégorie | Technologies |
|-----------|-------------|
| **Orchestration** | MicroK8s, Kubernetes |
| **Monitoring** | Prometheus, Grafana, AlertManager |
| **Observabilité** | Loki, Jaeger, Exporters |
| **Sécurité** | Cert-Manager, Let's Encrypt, RBAC |
| **Réseau** | NGINX Ingress, MetalLB, CoreDNS |
| **DevOps** | ArgoCD, Helm, GitOps |
| **Storage** | Volumes persistants, Snapshots |

## 📖 Structure du guide

```
📁 microk8s-lab-personnel-guide-francais/
├── 📄 README.md (vous êtes ici)
├── 📄 SOMMAIRE.md (table des matières détaillée)
├── 📁 01-introduction-et-prerequis/
├── 📁 02-installation-et-configuration-initiale/
├── 📁 08-stack-prometheus-et-grafana/
├── 📁 annexes/
└── 📁 examples/ (configurations et manifestes)
```

## ⚡ Exemples inclus

- Configurations YAML prêtes à l'emploi
- Scripts d'installation automatisée
- Dashboards Grafana préconfigurés
- Manifestes d'applications de démonstration
- Templates de certificats SSL/TLS

## 🎓 Niveau de difficulté

- **🟢 Débutant** - Chapitres 1-4 (Installation, concepts de base)
- **🟡 Intermédiaire** - Chapitres 5-10 (SSL/TLS, monitoring, DevOps)
- **🔴 Avancé** - Chapitres 11+ (Sécurité, HA, optimisation)

## 📝 Licence

Ce projet est sous licence MIT - voir le fichier [LICENSE](LICENSE) pour plus de détails.

## ✍️ Auteur

**Nicolas DEOUX**
📧 NDXdev@gmail.com

---

*Dernière mise à jour : Août 2025*
