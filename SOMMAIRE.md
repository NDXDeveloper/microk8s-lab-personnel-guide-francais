# Documentation MicroK8s pour Lab Personnel
## Guide complet pour débutants

### 01. Introduction et Prérequis
- **1.1** Qu'est-ce que MicroK8s ?
- **1.2** Avantages pour un lab personnel
- **1.3** Comparaison avec d'autres solutions Kubernetes
- **1.4** Prérequis système (CPU, RAM, stockage)
- **1.5** Systèmes d'exploitation supportés
- **1.6** Architecture générale du lab

### 02. Installation et Configuration Initiale
- **2.1** Installation sur Ubuntu/Debian
- **2.2** Installation sur CentOS/RHEL
- **2.3** Installation sur Windows (WSL2)
- **2.4** Installation sur macOS
- **2.5** Vérification de l'installation
- **2.6** Configuration des alias kubectl
- **2.7** Premier diagnostic du cluster

### 03. Configuration Réseau et Domaine
- **3.1** Concepts de base du réseau Kubernetes
- **3.2** Configuration DNS locale
- **3.3** Acquisition d'un nom de domaine
- **3.4** Configuration des enregistrements DNS
- **3.5** Redirection de ports et NAT
- **3.6** Firewall et règles de sécurité
- **3.7** Test de connectivité externe

### 04. Addons Essentiels
- **4.1** Présentation des addons MicroK8s
- **4.2** Dashboard Kubernetes
- **4.3** DNS (CoreDNS)
- **4.4** Ingress Controller (NGINX)
- **4.5** Registry (registre d'images privé)
- **4.6** Storage (stockage persistant)
- **4.7** MetalLB (load balancer)
- **4.8** Cert-Manager (certificats SSL/TLS)

### 05. Gestion des Certificats SSL/TLS
- **5.1** Introduction aux certificats
- **5.2** Configuration de Cert-Manager
- **5.3** Certificats Let's Encrypt automatiques
- **5.4** Certificats auto-signés pour le développement
- **5.5** Wildcard certificates
- **5.6** Renouvellement automatique
- **5.7** Dépannage des certificats

### 06. Ingress et Routage
- **6.1** Concepts d'Ingress
- **6.2** Configuration NGINX Ingress Controller
- **6.3** Règles de routage par nom d'hôte
- **6.4** Règles de routage par chemin
- **6.5** Middleware et annotations
- **6.6** Gestion des redirections HTTPS
- **6.7** Exemples pratiques d'Ingress

### 07. Déploiement d'Applications
- **7.1** Anatomie d'un déploiement Kubernetes
- **7.2** Manifestes YAML de base
- **7.3** Déploiement d'une application web simple
- **7.4** Exposition via Service et Ingress
- **7.5** Variables d'environnement et ConfigMaps
- **7.6** Secrets et données sensibles
- **7.7** Gestion des volumes persistants

### 08. Stack Prometheus et Grafana
- **8.01** Introduction à l'écosystème Prometheus
- **8.02** Installation de Prometheus sur MicroK8s
- **8.03** Configuration des targets et service discovery
- **8.04** PromQL : langage de requête Prometheus
- **8.05** Exporters essentiels (node, kube-state, cadvisor)
- **8.06** Installation et configuration de Grafana
- **8.07** Connexion Grafana-Prometheus
- **8.08** Dashboards Kubernetes pré-configurés
- **8.09** Création de dashboards personnalisés
- **8.10** Variables et templating dans Grafana
- **8.11** Alerting avec Prometheus Alertmanager
- **8.12** Notifications (Slack, email, webhook)
- **8.13** Recording rules et optimisation
- **8.14** Haute disponibilité Prometheus

### 09. Observabilité Avancée
- **9.1** Métriques custom dans vos applications
- **9.2** Logs centralisés (ELK/EFK stack)
- **9.3** Correlation logs-métriques avec Loki
- **9.4** Tracing distribué avec Jaeger
- **9.5** Synthetic monitoring avec Blackbox exporter
- **9.6** SLI/SLO et dashboards de fiabilité
- **9.7** Runbooks et documentation des alertes

### 10. DevOps et CI/CD
- **10.1** Intégration avec Git
- **10.2** Registry privé et gestion d'images
- **10.3** Pipeline GitLab CI/Jenkins
- **10.4** ArgoCD pour GitOps
- **10.5** Helm Charts
- **10.6** Automated testing
- **10.7** Blue-Green et Canary deployments

### 11. Sécurité
- **11.1** RBAC (contrôle d'accès basé sur les rôles)
- **11.2** Network Policies
- **11.3** Pod Security Standards
- **11.4** Scan de vulnérabilités
- **11.5** Secrets management avancé
- **11.6** Audit logging
- **11.7** Bonnes pratiques sécurité

### 12. Sauvegarde et Restauration
- **12.1** Stratégies de sauvegarde
- **12.2** Sauvegarde avec Velero
- **12.3** Snapshots de volumes
- **12.4** Export/Import de configurations
- **12.5** Tests de restauration
- **12.6** Plan de reprise d'activité

### 13. Scaling et Performance
- **13.1** Horizontal Pod Autoscaler (HPA)
- **13.2** Vertical Pod Autoscaler (VPA)
- **13.3** Cluster Autoscaler (pour multi-node)
- **13.4** Resource Quotas et LimitRanges
- **13.5** Node affinity et taints
- **13.6** Optimisation des ressources
- **13.7** Tests de charge

### 14. Multi-Node et Haute Disponibilité
- **14.1** Extension vers un cluster multi-node
- **14.2** Ajout de worker nodes
- **14.3** Load balancing entre nodes
- **14.4** Stockage distribué
- **14.5** Backup du control plane
- **14.6** Stratégies de redondance

### 15. Cas d'Usage Pratiques
- **15.1** Lab de développement complet
- **15.2** Environnement de test automatisé
- **15.3** Hébergement de services personnels
- **15.4** Learning playground Kubernetes
- **15.5** Démonstrations et PoC
- **15.6** Formation et certification

### 16. Dépannage et Maintenance
- **16.1** Commandes de diagnostic essentielles
- **16.2** Analyse des logs
- **16.3** Problèmes réseau courants
- **16.4** Problèmes de certificats
- **16.5** Performance et ressources
- **16.6** Mise à jour de MicroK8s
- **16.7** Nettoyage et maintenance

### 17. Ressources et Références
- **17.1** Documentation officielle
- **17.2** Communauté et support
- **17.3** Outils complémentaires
- **17.4** Formations avancées
- **17.5** Certifications Kubernetes
- **17.6** Glossaire des termes

### Annexes
- **A.** Scripts d'installation automatisée
- **B.** Templates de manifestes YAML
- **C.** Configuration DNS complète
- **D.** Checklist de sécurité
- **E.** Commandes kubectl essentielles
- **F.** Troubleshooting rapide
