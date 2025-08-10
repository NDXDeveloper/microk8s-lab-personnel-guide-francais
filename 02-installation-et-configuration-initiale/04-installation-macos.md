🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Installation sur macOS

## Introduction

macOS ne supportant pas nativement les conteneurs Linux, MicroK8s s'exécute dans une machine virtuelle légère via Multipass, un gestionnaire de VM développé par Canonical. Cette approche offre une expérience quasi-native tout en maintenant la simplicité d'installation et d'utilisation caractéristique de MicroK8s. Pour les développeurs macOS, c'est une solution idéale qui s'intègre parfaitement avec l'écosystème Apple tout en offrant un environnement Kubernetes complet.

**Pourquoi MicroK8s avec Multipass sur macOS** :
- Installation en une commande avec gestion automatique de la VM
- Performance optimisée grâce à l'Hypervisor.framework natif de macOS
- Intégration transparente avec les outils de développement macOS
- Consommation de ressources réduite comparée à Docker Desktop
- Pas de licence commerciale requise contrairement à certaines alternatives

## Prérequis macOS

### Versions macOS supportées

MicroK8s via Multipass nécessite :
- **macOS 10.15** (Catalina) ou supérieur
- **Architecture** : Intel (x86_64) ou Apple Silicon (M1/M2/M3)
- **Hypervisor** : Hypervisor.framework (inclus dans macOS)

Pour vérifier votre version macOS :
```bash
# Dans le Terminal
sw_vers

# Devrait afficher quelque chose comme :
# ProductName:    macOS
# ProductVersion: 14.0
# BuildVersion:   23A344
```

### Configuration matérielle requise

```bash
# Vérifier les informations système
system_profiler SPHardwareDataType

# Ressources minimales requises :
# - Processeur : 2 cœurs (4+ recommandé)
# - Mémoire : 4 GB disponibles (8 GB+ recommandé)
# - Stockage : 20 GB d'espace libre (40 GB+ recommandé)

# Vérifier l'espace disque disponible
df -h /

# Vérifier la mémoire
vm_stat | grep "Pages free"
# Ou pour une vue plus claire
top -l 1 | grep PhysMem
```

### Architecture du processeur

Identifier si vous avez un Mac Intel ou Apple Silicon :
```bash
# Vérifier l'architecture
uname -m

# Résultats possibles :
# x86_64 : Mac Intel
# arm64  : Mac Apple Silicon (M1/M2/M3)

# Ou via les informations système
sysctl -n machdep.cpu.brand_string
```

## Installation des prérequis

### Installation de Homebrew (si nécessaire)

Homebrew est le gestionnaire de paquets standard pour macOS :

```bash
# Vérifier si Homebrew est installé
which brew

# Si non installé, l'installer
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Pour Apple Silicon, ajouter Homebrew au PATH si nécessaire
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# Mettre à jour Homebrew
brew update
```

### Installation des outils de ligne de commande

```bash
# Installer les outils essentiels
brew install wget curl git

# Installer les outils de développement (optionnel mais recommandé)
brew install --cask visual-studio-code
brew install jq yq
```

## Installation de Multipass

### Méthode 1 : Installation via Homebrew (recommandée)

```bash
# Installer Multipass
brew install --cask multipass

# Vérifier l'installation
multipass version

# Devrait afficher quelque chose comme :
# multipass   1.13.0
# multipassd  1.13.0
```

### Méthode 2 : Installation via le package officiel

Si vous préférez l'installateur officiel :

1. Téléchargez depuis [multipass.run](https://multipass.run/install)
2. Ouvrez le fichier `.pkg` téléchargé
3. Suivez l'assistant d'installation
4. Accordez les permissions nécessaires dans Préférences Système > Sécurité et confidentialité

### Configuration de Multipass

#### Premier lancement et permissions

Lors du premier lancement, macOS demandera des permissions :

```bash
# Lancer Multipass
multipass launch --name test-vm

# Si demandé, autorisez Multipass dans :
# Préférences Système > Sécurité et confidentialité > Confidentialité
```

#### Configuration du driver (Intel vs Apple Silicon)

Pour les Mac Intel :
```bash
# Vérifier le driver utilisé (devrait être hyperkit ou qemu)
multipass get local.driver

# Si nécessaire, configurer hyperkit pour de meilleures performances
multipass set local.driver=hyperkit
```

Pour les Mac Apple Silicon :
```bash
# Le driver qemu est utilisé automatiquement
multipass get local.driver

# Vérifier que la virtualisation est activée
multipass set local.driver=qemu
```

#### Configuration des ressources par défaut

```bash
# Définir les ressources par défaut pour les nouvelles VM
multipass set local.microk8s-vm.cpus=2
multipass set local.microk8s-vm.memory=4G
multipass set local.microk8s-vm.disk=20G

# Voir toutes les configurations
multipass get --keys
```

## Installation de MicroK8s via Multipass

### Méthode 1 : Installation automatique (recommandée)

Multipass offre une commande dédiée pour MicroK8s :

```bash
# Installer MicroK8s en une commande
# Cette commande crée une VM, installe Ubuntu et MicroK8s
microk8s install

# Processus automatique :
# - Crée une VM nommée "microk8s-vm"
# - Installe Ubuntu LTS
# - Installe MicroK8s
# - Configure les alias de commandes

# Le processus peut prendre 5-10 minutes selon votre connexion
```

### Méthode 2 : Installation manuelle (plus de contrôle)

Pour plus de contrôle sur la configuration :

```bash
# Créer une VM Ubuntu avec des ressources spécifiques
multipass launch --name microk8s-vm \
  --cpus 2 \
  --memory 4G \
  --disk 20G \
  22.04

# Se connecter à la VM
multipass shell microk8s-vm

# Une fois dans la VM, installer MicroK8s
sudo snap install microk8s --classic --channel=1.31/stable

# Configurer les permissions
sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
sudo chown -f -R $USER ~/.kube

# Sortir et se reconnecter
exit
multipass shell microk8s-vm
```

### Configuration des alias sur macOS

Pour utiliser MicroK8s directement depuis macOS sans entrer dans la VM :

```bash
# Après installation automatique, les alias sont créés
# Vérifier leur présence
which microk8s

# Si les alias ne sont pas configurés, les créer manuellement
echo "alias microk8s='multipass exec microk8s-vm -- sudo microk8s'" >> ~/.zshrc
echo "alias kubectl='multipass exec microk8s-vm -- sudo microk8s kubectl'" >> ~/.zshrc

# Recharger le shell
source ~/.zshrc

# Tester les alias
microk8s status
kubectl get nodes
```

## Configuration post-installation

### Vérification de l'installation

```bash
# Vérifier le statut de la VM
multipass list

# Devrait afficher :
# Name           State       IPv4            Image
# microk8s-vm    Running     192.168.64.x    Ubuntu 22.04 LTS

# Vérifier MicroK8s
microk8s status --wait-ready

# Vérifier les nodes
microk8s kubectl get nodes

# Informations sur la VM
multipass info microk8s-vm
```

### Configuration kubectl sur macOS

#### Installation de kubectl natif

```bash
# Installer kubectl via Homebrew
brew install kubectl

# Vérifier l'installation
kubectl version --client

# Créer le répertoire de configuration
mkdir -p ~/.kube
```

#### Export de la configuration MicroK8s

```bash
# Obtenir la configuration depuis MicroK8s
microk8s config > ~/.kube/config

# Ou si vous avez plusieurs configs, sauvegarder séparément
microk8s config > ~/.kube/microk8s-config

# Utiliser avec la variable d'environnement
export KUBECONFIG=~/.kube/microk8s-config

# Tester la connexion
kubectl get nodes
```

### Configuration réseau

#### Accès aux services depuis macOS

La VM Multipass utilise un réseau NAT. Pour accéder aux services :

```bash
# Obtenir l'IP de la VM
VMIP=$(multipass info microk8s-vm | grep IPv4 | awk '{print $2}')
echo "VM IP: $VMIP"

# Port forwarding pour un service
multipass exec microk8s-vm -- sudo microk8s kubectl port-forward \
  --address 0.0.0.0 \
  -n kubernetes-dashboard \
  service/kubernetes-dashboard 8443:443
```

Accédez depuis Safari/Chrome : `https://<VM-IP>:8443`

#### Configuration d'un bridge network (avancé)

Pour un accès réseau plus direct :

```bash
# Créer un bridge (nécessite des privilèges admin)
# Note : Cette configuration est avancée et optionnelle

# Voir les interfaces réseau
networksetup -listallhardwareports

# Configuration manuelle du bridge (exemple)
# Consultez la documentation Multipass pour votre configuration spécifique
```

## Gestion de la VM MicroK8s

### Commandes de gestion essentielles

```bash
# Démarrer la VM
multipass start microk8s-vm

# Arrêter la VM
multipass stop microk8s-vm

# Redémarrer la VM
multipass restart microk8s-vm

# Se connecter à la VM
multipass shell microk8s-vm

# Voir les logs de la VM
multipass exec microk8s-vm -- sudo journalctl -f

# Transférer des fichiers vers/depuis la VM
multipass transfer local-file.txt microk8s-vm:/home/ubuntu/
multipass transfer microk8s-vm:/home/ubuntu/file.txt ./
```

### Modification des ressources de la VM

Pour ajuster les ressources après création :

```bash
# Arrêter la VM
multipass stop microk8s-vm

# Malheureusement, Multipass ne permet pas de modifier directement
# les ressources d'une VM existante. Solutions :

# Option 1 : Créer une nouvelle VM avec plus de ressources
multipass launch --name microk8s-vm-large \
  --cpus 4 \
  --memory 8G \
  --disk 40G

# Option 2 : Sauvegarder et recréer
# Exporter la configuration Kubernetes
microk8s config > ~/microk8s-backup.yaml

# Supprimer l'ancienne VM
multipass delete microk8s-vm
multipass purge

# Recréer avec nouvelles ressources
microk8s install --cpu 4 --mem 8 --disk 40
```

### Snapshots et sauvegardes

```bash
# Multipass ne supporte pas nativement les snapshots
# Mais vous pouvez sauvegarder l'état

# Créer un script de sauvegarde
cat << 'EOF' > ~/backup-microk8s.sh
#!/bin/bash
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR=~/microk8s-backups/$DATE

mkdir -p $BACKUP_DIR

# Sauvegarder la config Kubernetes
microk8s config > $BACKUP_DIR/kubeconfig.yaml

# Sauvegarder les manifests
microk8s kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/all-resources.yaml

# Sauvegarder les données persistantes
multipass exec microk8s-vm -- sudo tar czf - /var/snap/microk8s/common/var/lib/kubelet/pods > $BACKUP_DIR/persistent-data.tar.gz

echo "Backup completed in $BACKUP_DIR"
EOF

chmod +x ~/backup-microk8s.sh
```

## Intégration avec l'écosystème macOS

### Intégration avec VS Code

```bash
# Installer l'extension Kubernetes
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools

# Installer l'extension Remote SSH (pour éditer dans la VM)
code --install-extension ms-vscode-remote.remote-ssh

# Configurer SSH pour la VM
multipass exec microk8s-vm -- sudo sh -c "echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config"
multipass exec microk8s-vm -- sudo systemctl restart sshd
```

### Intégration avec Docker Desktop

Si vous avez Docker Desktop installé :

```bash
# Utiliser les images Docker locales dans MicroK8s
# Option 1 : Pousser vers le registry MicroK8s
microk8s enable registry

# Tagger et pousser une image
docker tag myapp:latest localhost:32000/myapp:latest
docker push localhost:32000/myapp:latest

# Option 2 : Exporter/Importer
docker save myapp:latest -o myapp.tar
multipass transfer myapp.tar microk8s-vm:/home/ubuntu/
multipass exec microk8s-vm -- sudo microk8s ctr image import /home/ubuntu/myapp.tar
```

### Scripts d'automatisation

Créez des scripts pratiques pour votre workflow :

```bash
# Créer un répertoire pour les scripts
mkdir -p ~/.local/bin

# Script de démarrage quotidien
cat << 'EOF' > ~/.local/bin/start-lab
#!/bin/bash
echo "Starting MicroK8s lab environment..."
multipass start microk8s-vm
multipass exec microk8s-vm -- sudo microk8s status --wait-ready
echo "MicroK8s is ready!"
echo "VM IP: $(multipass info microk8s-vm | grep IPv4 | awk '{print $2}')"
EOF

chmod +x ~/.local/bin/start-lab

# Ajouter au PATH
echo 'export PATH=$PATH:~/.local/bin' >> ~/.zshrc
source ~/.zshrc
```

## Résolution des problèmes courants

### Problème : "Cannot connect to the multipass socket"

**Cause** : Le daemon Multipass n'est pas démarré

**Solution** :
```bash
# Redémarrer Multipass
brew services restart multipass

# Ou manuellement
sudo launchctl stop com.canonical.multipassd
sudo launchctl start com.canonical.multipassd

# Vérifier le statut
multipass version
```

### Problème : Performance lente sur Apple Silicon

**Cause** : QEMU peut être moins optimisé

**Solutions** :
```bash
# Vérifier l'utilisation des ressources
multipass exec microk8s-vm -- top

# Optimiser QEMU (Apple Silicon)
# Assurez-vous d'avoir la dernière version
brew upgrade multipass

# Allouer plus de ressources si possible
# (nécessite recréation de la VM)
```

### Problème : "VM is taking too long to start"

**Cause** : Ressources insuffisantes ou problème réseau

**Solutions** :
```bash
# Vérifier les logs
multipass exec microk8s-vm -- sudo journalctl -xe

# Augmenter le timeout
export MULTIPASS_TIMEOUT=600

# Nettoyer et réessayer
multipass delete microk8s-vm
multipass purge
microk8s install
```

### Problème : Impossible d'accéder aux services

**Cause** : Configuration réseau ou firewall

**Solution** :
```bash
# Vérifier l'IP de la VM
multipass info microk8s-vm

# Tester la connectivité
ping $(multipass info microk8s-vm | grep IPv4 | awk '{print $2}')

# Vérifier le firewall macOS
sudo pfctl -s info

# Port forwarding alternatif via SSH
ssh -L 8080:localhost:8080 ubuntu@$(multipass info microk8s-vm | grep IPv4 | awk '{print $2}')
```

### Problème : Espace disque insuffisant

**Symptôme** : Erreurs "no space left on device"

**Solutions** :
```bash
# Vérifier l'espace dans la VM
multipass exec microk8s-vm -- df -h

# Nettoyer les images non utilisées
multipass exec microk8s-vm -- sudo microk8s ctr images prune

# Nettoyer les logs
multipass exec microk8s-vm -- sudo journalctl --vacuum-size=100M

# Si nécessaire, agrandir le disque (nécessite recréation)
```

## Optimisations pour macOS

### Performance I/O

```bash
# Désactiver Spotlight pour le répertoire Multipass
sudo mdutil -i off /var/root/Library/Application\ Support/multipassd

# Exclure de Time Machine
sudo tmutil addexclusion /var/root/Library/Application\ Support/multipassd
```

### Gestion de la mémoire

```bash
# Script pour monitorer l'utilisation mémoire
cat << 'EOF' > ~/monitor-microk8s.sh
#!/bin/bash
while true; do
    clear
    echo "=== MicroK8s VM Resources ==="
    multipass info microk8s-vm | grep -E "Load|Memory|Disk"
    echo ""
    echo "=== macOS Memory Pressure ==="
    memory_pressure | head -n 5
    sleep 5
done
EOF

chmod +x ~/monitor-microk8s.sh
```

### Configuration de démarrage automatique

Pour démarrer MicroK8s automatiquement :

1. Créez un fichier plist :
```bash
cat << EOF > ~/Library/LaunchAgents/com.microk8s.autostart.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.microk8s.autostart</string>
    <key>Program</key>
    <string>/usr/local/bin/multipass</string>
    <key>ProgramArguments</key>
    <array>
        <string>multipass</string>
        <string>start</string>
        <string>microk8s-vm</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
EOF

# Charger l'agent
launchctl load ~/Library/LaunchAgents/com.microk8s.autostart.plist
```

## Utilisation avancée

### Accès depuis iPad/iPhone (Tailscale)

Pour accéder à votre cluster depuis vos appareils iOS :

```bash
# Installer Tailscale dans la VM
multipass exec microk8s-vm -- bash -c "curl -fsSL https://tailscale.com/install.sh | sh"

# Configurer Tailscale
multipass exec microk8s-vm -- sudo tailscale up

# Suivre les instructions pour l'authentification
# Votre cluster sera accessible via le réseau Tailscale
```

### Intégration avec Rancher Desktop

Si vous préférez une interface graphique :

```bash
# Installer Rancher Desktop
brew install --cask rancher

# Configurer pour utiliser le cluster MicroK8s existant
# Dans Rancher Desktop, importer la kubeconfig
```

## Monitoring et maintenance

### Surveillance des ressources

```bash
# Dashboard simple
cat << 'EOF' > ~/.local/bin/microk8s-dashboard
#!/bin/bash
echo "MicroK8s Resource Dashboard"
echo "=========================="
echo ""
echo "VM Status:"
multipass list | grep microk8s-vm
echo ""
echo "Resource Usage:"
multipass exec microk8s-vm -- free -h
echo ""
echo "Kubernetes Nodes:"
microk8s kubectl top nodes 2>/dev/null || echo "Metrics server not enabled"
echo ""
echo "Running Pods:"
microk8s kubectl get pods --all-namespaces | grep Running | wc -l
EOF

chmod +x ~/.local/bin/microk8s-dashboard
```

### Mises à jour

```bash
# Mettre à jour Multipass
brew upgrade multipass

# Mettre à jour MicroK8s dans la VM
multipass exec microk8s-vm -- sudo snap refresh microk8s

# Vérifier les versions
multipass version
microk8s version
```

## Points de vérification finale

Avant de continuer, assurez-vous que :

- ✅ Multipass est installé et fonctionne
- ✅ La VM microk8s-vm est en état "Running"
- ✅ MicroK8s répond avec `microk8s status`
- ✅ `kubectl get nodes` fonctionne depuis macOS
- ✅ Vous pouvez accéder aux services via l'IP de la VM
- ✅ Les ressources allouées sont suffisantes
- ✅ Les alias de commandes sont configurés

## Avantages et limitations sur macOS

### Avantages

- **Simplicité** : Installation en une commande
- **Isolation** : VM complètement isolée du système
- **Compatibilité** : Support Intel et Apple Silicon
- **Intégration** : Fonctionne avec l'écosystème macOS

### Limitations

- **Performance** : Overhead de la virtualisation
- **Réseau** : NAT par défaut, configuration additionnelle pour bridge
- **Ressources** : Consomme RAM et CPU continuellement
- **Stockage** : Les volumes sont dans la VM, pas directement accessibles

## Prochaines étapes

Avec MicroK8s installé sur macOS, vous pouvez maintenant :

1. Configurer kubectl avec des alias pratiques (section 2.6)
2. Explorer les addons disponibles
3. Intégrer avec vos outils de développement macOS
4. Déployer vos premières applications

L'environnement MicroK8s sur macOS offre une excellente plateforme pour le développement et les tests Kubernetes, avec la flexibilité de macOS et la puissance de Kubernetes.

⏭️
