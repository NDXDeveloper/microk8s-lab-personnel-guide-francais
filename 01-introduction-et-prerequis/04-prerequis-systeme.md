🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.4 - Prérequis système (CPU, RAM, stockage)

## Introduction

Avant d'installer MicroK8s, il est essentiel de vérifier que votre système dispose des ressources nécessaires. Cette section détaille les prérequis matériels, vous aide à évaluer votre configuration actuelle, et propose des recommandations selon différents scénarios d'usage. La bonne nouvelle : MicroK8s est l'une des distributions Kubernetes les plus légères, permettant de démarrer avec du matériel modeste.

## Configuration minimale absolue

### Les chiffres essentiels

MicroK8s peut techniquement fonctionner avec :

| Composant | Minimum absolu | Utilisation réelle |
|-----------|----------------|-------------------|
| **CPU** | 1 cœur | Pour le control plane uniquement |
| **RAM** | 540 MB | Kubernetes seul, sans applications |
| **Stockage** | 700 MB | Installation de base sans données |
| **Architecture** | x86_64, ARM64, ARMv7 | 64-bit recommandé |

**⚠️ Important** : Ces valeurs permettent de démarrer MicroK8s mais sont insuffisantes pour un usage réel. C'est comme avoir une voiture qui démarre mais ne peut pas rouler.

## Configuration recommandée pour un lab

### Pour un usage confortable

Voici ce que nous recommandons pour un laboratoire personnel fonctionnel :

| Composant | Recommandé | Justification |
|-----------|------------|---------------|
| **CPU** | 2-4 cœurs | Permet de faire tourner plusieurs pods simultanément |
| **RAM** | 4-8 GB | 2 GB pour MicroK8s + 2-6 GB pour vos applications |
| **Stockage** | 20-50 GB | OS + MicroK8s + images Docker + logs + données |
| **Réseau** | 100 Mbps | Pour télécharger les images et accéder aux services |

### Explication détaillée des besoins

**CPU (Processeur) :**
- **1 cœur** : Control plane uniquement
- **2 cœurs** : + quelques applications légères
- **4 cœurs** : + monitoring, CI/CD, multiples services
- **6+ cœurs** : Simulation d'environnement production

**RAM (Mémoire) :**
- **2 GB** : MicroK8s + 2-3 pods simples
- **4 GB** : + base de données, monitoring basique
- **8 GB** : + stack complète (Prometheus, Grafana, apps)
- **16+ GB** : Environnement de développement complet

**Stockage :**
- **10 GB** : Installation minimale + quelques images
- **20 GB** : + logs, métriques sur quelques jours
- **50 GB** : + données persistantes, backups locaux
- **100+ GB** : Lab complet avec historique long terme

## Comprendre la consommation des ressources

### Répartition typique de l'utilisation

```
RAM Totale (exemple 8 GB)
├── Système d'exploitation (1-2 GB)
├── MicroK8s Core
│   ├── API Server (200-400 MB)
│   ├── etcd (100-200 MB)
│   ├── Scheduler (50-100 MB)
│   ├── Controller Manager (100-200 MB)
│   └── Kubelet + Containerd (200-400 MB)
├── Addons MicroK8s
│   ├── CoreDNS (50-100 MB)
│   ├── Dashboard (100-200 MB si activé)
│   ├── Ingress Controller (100-200 MB si activé)
│   └── Autres addons (variable)
└── Vos Applications (3-4 GB restants)
    ├── Pods applicatifs
    ├── Bases de données
    └── Services auxiliaires
```

### Impact des addons sur les ressources

| Addon | RAM additionnelle | CPU additionnel | Stockage additionnel |
|-------|------------------|-----------------|---------------------|
| **dashboard** | 100-200 MB | Minimal | 50 MB |
| **dns** | 50-100 MB | Minimal | 20 MB |
| **ingress** | 100-200 MB | 0.1-0.2 cœur | 100 MB |
| **prometheus** | 500-1000 MB | 0.2-0.5 cœur | 5-10 GB (métriques) |
| **registry** | 200-500 MB | 0.1-0.3 cœur | Variable (images) |
| **metallb** | 50-100 MB | Minimal | 20 MB |
| **cert-manager** | 100-200 MB | 0.1 cœur | 50 MB |

## Configurations par type de machine

### Machines de bureau/Laptop moderne

**Configuration typique :**
```
Processeur : Intel Core i5/i7 ou AMD Ryzen 5/7 (4-8 cœurs)
RAM : 8-32 GB
Stockage : SSD 256 GB - 1 TB
OS : Ubuntu 22.04 LTS
```

**Capacité MicroK8s :**
- ✅ Installation complète avec tous les addons
- ✅ 10-20 applications simultanées
- ✅ Stack monitoring complète
- ✅ Développement intensif
- ✅ Multi-node possible (avec VMs)

### Raspberry Pi

**Raspberry Pi 4 (recommandé) :**
```
Processeur : ARM Cortex-A72 (4 cœurs)
RAM : 4 ou 8 GB
Stockage : MicroSD 32-64 GB (SSD USB recommandé)
OS : Ubuntu Server 22.04 ARM64
```

**Capacité MicroK8s :**
- ✅ Installation fonctionnelle
- ✅ 3-5 applications légères
- ⚠️ Monitoring limité
- ✅ Parfait pour IoT/Edge
- ✅ Cluster multi-Pi possible

**Raspberry Pi 3 (minimal) :**
```
Processeur : ARM Cortex-A53 (4 cœurs)
RAM : 1 GB
Stockage : MicroSD 16-32 GB
```

**Capacité MicroK8s :**
- ⚠️ Très limité
- ✅ 1-2 applications simples
- ❌ Pas de monitoring
- ⚠️ Considérez K3s à la place

### Machine virtuelle

**VM de développement :**
```
vCPU : 2-4
RAM : 4-8 GB
Stockage : 20-40 GB
Type : Ubuntu Server, pas de GUI
```

**Recommandations VM :**
- Désactiver le swap (performance Kubernetes)
- Utiliser le stockage "thin provisioning"
- Bridge networking pour l'accès externe
- Snapshots avant les changements majeurs

### Vieux matériel recyclé

**PC de 5-10 ans :**
```
Processeur : Intel Core i3/i5 3ème-6ème gen
RAM : 4-8 GB
Stockage : HDD 250-500 GB (SSD fortement recommandé)
```

**Optimisations nécessaires :**
- Installation minimale de l'OS (sans desktop)
- Désactiver les services inutiles
- Ajouter un SSD même petit (120 GB)
- Maximiser la RAM si possible

## Évaluer votre système actuel

### Commandes de diagnostic Linux

**Vérifier le CPU :**
```bash
# Nombre de cœurs et modèle
lscpu | grep -E "^CPU\(s\)|Model name|Architecture"

# Utilisation actuelle
top -bn1 | head -5

# Charge moyenne
uptime
```

**Vérifier la RAM :**
```bash
# Mémoire totale et disponible
free -h

# Utilisation détaillée
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable"

# Processus consommateurs
ps aux --sort=-%mem | head -10
```

**Vérifier le stockage :**
```bash
# Espace disque disponible
df -h

# Type de disque (SSD/HDD)
lsblk -d -o name,rota  # ROTA=0 pour SSD, 1 pour HDD

# Vitesse I/O (test rapide)
dd if=/dev/zero of=/tmp/test bs=1M count=1024 conv=fdatasync
```

**Vérifier l'architecture :**
```bash
# Architecture système
uname -m  # x86_64, aarch64, armv7l, etc.

# Version du kernel
uname -r
```

### Calculateur de capacité

Utilisez cette formule pour estimer combien d'applications vous pouvez faire tourner :

```
RAM disponible pour apps = RAM totale - 2 GB (OS + MicroK8s)
Nombre d'apps = RAM disponible / RAM moyenne par app

Exemple :
8 GB total - 2 GB = 6 GB disponibles
6 GB / 300 MB par app = ~20 petites applications
```

## Optimisations système recommandées

### Avant l'installation

**1. Libérer des ressources :**
```bash
# Désactiver le swap (recommandé pour Kubernetes)
sudo swapoff -a
# Commenter les lignes swap dans /etc/fstab

# Arrêter les services inutiles
sudo systemctl disable bluetooth
sudo systemctl disable cups
```

**2. Optimiser l'OS :**
```bash
# Utiliser une version server (sans GUI) si possible
# Ubuntu Server utilise ~500 MB RAM vs ~2 GB pour Desktop

# Nettoyer le système
sudo apt autoremove
sudo apt clean
```

**3. Paramètres kernel :**
```bash
# Vérifier les modules requis
lsmod | grep -E "br_netfilter|overlay"

# Les activer si nécessaire
sudo modprobe br_netfilter
sudo modprobe overlay
```

### Configuration du stockage

**Types de stockage et performance :**

| Type | IOPS | Latence | Usage MicroK8s |
|------|------|---------|----------------|
| **NVMe SSD** | 100k+ | <0.1ms | Idéal, overkill pour lab |
| **SATA SSD** | 10-50k | 0.1-0.5ms | Recommandé |
| **HDD 7200rpm** | 100-200 | 5-10ms | Fonctionnel mais lent |
| **HDD 5400rpm** | 50-100 | 10-20ms | Déconseillé |
| **SD Card Class 10** | 10-50 | 10-50ms | Minimum pour Pi |

**Partitionnement recommandé :**
```
/ (racine)     : 10-15 GB
/var           : 15-30 GB (données MicroK8s)
/home          : Reste disponible
```

## Scénarios d'usage et leurs prérequis

### Scénario 1 : Apprentissage basique

**Objectif :** Apprendre Kubernetes, déployer des apps simples

**Configuration minimale :**
- CPU : 2 cœurs
- RAM : 4 GB
- Stockage : 20 GB
- Addons : dns, dashboard

**Ce que vous pouvez faire :**
- Déployer des applications web simples
- Apprendre les concepts Kubernetes
- Préparer les certifications
- Tests basiques

### Scénario 2 : Développement d'applications

**Objectif :** Développer et tester des applications conteneurisées

**Configuration recommandée :**
- CPU : 4 cœurs
- RAM : 8 GB
- Stockage : 50 GB SSD
- Addons : dns, ingress, registry

**Ce que vous pouvez faire :**
- Développement full-stack
- Tests d'intégration
- CI/CD local
- Debugging avancé

### Scénario 3 : Lab complet avec monitoring

**Objectif :** Environnement proche production avec observabilité

**Configuration recommandée :**
- CPU : 4-6 cœurs
- RAM : 16 GB
- Stockage : 100 GB SSD
- Addons : dns, ingress, prometheus, cert-manager

**Ce que vous pouvez faire :**
- Stack monitoring complète
- Multiples environnements
- Tests de charge
- Simulation production

### Scénario 4 : Cluster multi-nodes

**Objectif :** Haute disponibilité et répartition de charge

**Configuration par nœud :**
- CPU : 2 cœurs minimum
- RAM : 4 GB minimum
- Stockage : 20 GB
- Réseau : Gigabit LAN

**Configuration totale (3 nœuds) :**
- CPU : 6+ cœurs total
- RAM : 12+ GB total
- Stockage : 60+ GB total

### Scénario 5 : Edge/IoT

**Objectif :** Déploiement sur matériel contraint

**Configuration minimale :**
- CPU : ARM 4 cœurs
- RAM : 2 GB
- Stockage : 16 GB
- Réseau : WiFi/4G

**Limitations acceptées :**
- Pas de monitoring lourd
- Applications optimisées
- Données locales minimales

## Planification de la croissance

### Évolution type d'un lab

```
Phase 1 : Découverte (Mois 1-3)
├── 2 cœurs, 4 GB RAM
├── Apps simples
└── Apprentissage

Phase 2 : Développement (Mois 4-6)
├── 4 cœurs, 8 GB RAM
├── Stack complète
└── CI/CD local

Phase 3 : Production-like (Mois 7-12)
├── 4-6 cœurs, 16 GB RAM
├── Monitoring complet
└── Multi-environnements

Phase 4 : Avancé (Année 2+)
├── Multi-nodes
├── Hybrid cloud
└── Architecture complexe
```

### Signaux d'upgrade nécessaire

**CPU insuffisant quand :**
- Load average > nombre de cœurs
- Pods en état "Pending" fréquent
- Builds/deployments très lents
- Interface dashboard lente

**RAM insuffisante quand :**
- OOM Killer actif (Out Of Memory)
- Pods "Evicted" régulièrement
- Swap utilisé (si activé)
- Cache système minimal

**Stockage insuffisant quand :**
- Espace < 20% disponible
- Images Docker non téléchargeables
- Logs rotation aggressive
- Performances I/O dégradées

## Cloud vs Local : Analyse des coûts

### Comparaison économique

**Option 1 : Hardware local**
```
PC d'occasion (i5, 8GB RAM, 256GB SSD) : 200-400€
Consommation électrique : ~5€/mois
Durée de vie : 3-5 ans
Coût mensuel amorti : ~10€
```

**Option 2 : Cloud (équivalent)**
```
VM 2 vCPU, 8GB RAM : 50-80€/mois
Stockage 50GB : 5-10€/mois
Bande passante : 10-20€/mois
Total mensuel : 65-110€
```

**ROI du hardware local :** 3-6 mois

### Stratégie hybride intelligente

Utilisez vos ressources locales pour :
- Développement quotidien (gratuit)
- Tests et apprentissage (gratuit)
- Services permanents légers (gratuit)

Utilisez le cloud pour :
- Tests de charge ponctuels
- Démonstrations clients
- Backup externe
- Pics de charge

## Cas particuliers et solutions

### Contraintes communes et solutions

**"J'ai seulement 4 GB de RAM"**
- Utilisez Ubuntu Server (pas Desktop)
- Activez seulement les addons essentiels
- Limitez les ressources des pods
- Considérez K3s pour encore plus léger

**"Mon CPU est ancien (pre-2015)"**
- Désactivez les features non essentielles
- Utilisez des images Docker Alpine
- Évitez les applications Java/JVM
- Préférez les languages compilés (Go, Rust)

**"J'ai un HDD, pas de SSD"**
- Minimisez les écritures de logs
- Utilisez ramdisk pour etcd
- Cachez les images frequentes
- Patience pour les déploiements

**"Je suis sur Raspberry Pi"**
- Utilisez Ubuntu Server ARM64
- Boot depuis SSD USB si possible
- Overclocking modéré acceptable
- Cluster de plusieurs Pi pour plus de puissance

## Check-list pré-installation

### Vérifications essentielles

- [ ] **CPU** : Au moins 2 cœurs disponibles
- [ ] **RAM** : Au moins 4 GB total, 2 GB libres
- [ ] **Stockage** : Au moins 20 GB disponibles
- [ ] **OS** : Linux 64-bit supporté
- [ ] **Kernel** : Version 4.15 ou supérieure
- [ ] **Réseau** : Connexion internet pour les images
- [ ] **Permissions** : Accès sudo/root
- [ ] **Firewall** : Ports nécessaires ouverts
- [ ] **Swap** : Désactivé (recommandé)
- [ ] **Temps** : 30 minutes pour l'installation

### Script de vérification automatique

```bash
#!/bin/bash
echo "=== Vérification des prérequis MicroK8s ==="

# CPU
cores=$(nproc)
echo "✓ CPU: $cores cœurs"
[[ $cores -ge 2 ]] && echo "  → OK" || echo "  → Warning: 2 cœurs minimum recommandés"

# RAM
ram_gb=$(free -g | awk '/^Mem:/{print $2}')
echo "✓ RAM: ${ram_gb} GB"
[[ $ram_gb -ge 4 ]] && echo "  → OK" || echo "  → Warning: 4 GB minimum recommandés"

# Stockage
disk_available=$(df -BG / | awk 'NR==2 {print $4}' | sed 's/G//')
echo "✓ Stockage disponible: ${disk_available} GB"
[[ $disk_available -ge 20 ]] && echo "  → OK" || echo "  → Warning: 20 GB minimum recommandés"

# Architecture
arch=$(uname -m)
echo "✓ Architecture: $arch"
[[ "$arch" == "x86_64" || "$arch" == "aarch64" ]] && echo "  → OK" || echo "  → Warning: Architecture non testée"

# Kernel
kernel=$(uname -r)
echo "✓ Kernel: $kernel"

# Swap
swap_on=$(swapon --show)
if [[ -z "$swap_on" ]]; then
    echo "✓ Swap: Désactivé (recommandé)"
else
    echo "⚠ Swap: Activé (recommandé de désactiver)"
fi
```

## Conclusion

### Résumé des configurations

| Usage | CPU min | RAM min | Stockage min | Idéal pour |
|-------|---------|---------|--------------|------------|
| **Test rapide** | 1 cœur | 2 GB | 10 GB | Découverte |
| **Apprentissage** | 2 cœurs | 4 GB | 20 GB | Formation |
| **Développement** | 4 cœurs | 8 GB | 50 GB | Projets |
| **Lab complet** | 4-6 cœurs | 16 GB | 100 GB | Production-like |
| **Multi-node** | 2 cœurs/node | 4 GB/node | 20 GB/node | HA & scaling |

### Points clés à retenir

1. **MicroK8s est flexible** : Fonctionne sur du matériel modeste mais scale bien
2. **La RAM est critique** : C'est souvent le facteur limitant
3. **SSD fortement recommandé** : Impact majeur sur les performances
4. **Commencez petit** : Vous pouvez toujours upgrader plus tard
5. **Réutilisez** : Vieux matériel parfait pour un lab

Avec ces informations, vous pouvez maintenant évaluer si votre système est prêt pour MicroK8s et planifier les éventuelles améliorations nécessaires pour votre cas d'usage spécifique.

---

*La section suivante couvrira les systèmes d'exploitation supportés et les meilleures pratiques pour chaque plateforme.*

⏭️
