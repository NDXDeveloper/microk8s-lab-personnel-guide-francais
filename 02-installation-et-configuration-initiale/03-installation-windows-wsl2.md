🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Installation sur Windows (WSL2)

## Introduction

Windows Subsystem for Linux 2 (WSL2) représente une révolution dans l'intégration Linux sur Windows, offrant une véritable machine virtuelle Linux légère avec un kernel Linux complet. Pour MicroK8s, WSL2 fournit un environnement idéal qui combine la familiarité de Windows avec la puissance de Kubernetes. Cette approche est particulièrement adaptée aux développeurs Windows souhaitant travailler avec des technologies cloud-native sans dual-boot ni machine virtuelle traditionnelle.

**Avantages de MicroK8s sur WSL2** :
- Performance quasi-native grâce à l'hyperviseur Hyper-V optimisé
- Intégration transparente avec les outils Windows (VS Code, browsers, etc.)
- Accès bidirectionnel aux fichiers Windows/Linux
- Consommation de ressources optimisée comparé à une VM classique
- Support complet de Docker et Kubernetes

## Prérequis Windows

### Versions de Windows supportées

MicroK8s via WSL2 nécessite :
- **Windows 10** : Version 2004 (Build 19041) ou supérieure
- **Windows 11** : Toutes les versions
- **Windows Server** : 2019 ou 2022

Pour vérifier votre version Windows :
1. Appuyez sur `Windows + R`
2. Tapez `winver` et appuyez sur Entrée
3. Vérifiez que votre version et build correspondent aux prérequis

### Configuration matérielle requise

```powershell
# Ouvrir PowerShell en tant qu'administrateur et vérifier
# Commande pour voir les informations système
systeminfo

# Vérifications minimales requises :
# - CPU : 2 cœurs ou plus (4+ recommandé)
# - RAM : 4 GB minimum (8 GB+ recommandé)
# - Espace disque : 30 GB libre minimum
# - Architecture : x64 (AMD64 ou Intel 64-bit)
```

### Activation de la virtualisation

WSL2 nécessite que la virtualisation soit activée dans le BIOS/UEFI et Windows.

**Vérifier l'état de la virtualisation** :
1. Ouvrez le Gestionnaire des tâches (`Ctrl + Shift + Esc`)
2. Allez dans l'onglet "Performance"
3. Sélectionnez "CPU"
4. Vérifiez que "Virtualisation" indique "Activé"

Si la virtualisation est désactivée, vous devrez l'activer dans le BIOS/UEFI :
- Redémarrez et accédez au BIOS (généralement F2, F10, F12 ou Del au démarrage)
- Cherchez "Intel VT-x" ou "AMD-V" ou "Virtualization Technology"
- Activez l'option et sauvegardez

## Installation de WSL2

### Méthode 1 : Installation simplifiée (Windows 10 version 2004+)

Ouvrez PowerShell en tant qu'administrateur et exécutez :

```powershell
# Installation complète de WSL2 avec Ubuntu par défaut
wsl --install

# Cette commande unique :
# - Active WSL
# - Installe WSL2
# - Télécharge et installe Ubuntu
# - Configure le kernel Linux
```

Après l'installation, redémarrez votre ordinateur.

### Méthode 2 : Installation manuelle (pour plus de contrôle)

Si vous préférez une installation étape par étape :

```powershell
# Étape 1 : Activer WSL
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Étape 2 : Activer la plateforme de machine virtuelle
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Étape 3 : Redémarrer Windows
Restart-Computer

# Après redémarrage, continuer dans PowerShell admin :

# Étape 4 : Télécharger et installer la mise à jour du kernel WSL2
# Télécharger depuis : https://aka.ms/wsl2kernel
# Ou via PowerShell :
Invoke-WebRequest -Uri https://aka.ms/wsl2kernel -OutFile wsl_update_x64.msi
Start-Process msiexec.exe -ArgumentList '/i', 'wsl_update_x64.msi', '/quiet' -Wait

# Étape 5 : Définir WSL2 comme version par défaut
wsl --set-default-version 2
```

### Installation d'une distribution Linux

Pour MicroK8s, Ubuntu est recommandé :

```powershell
# Lister les distributions disponibles
wsl --list --online

# Installer Ubuntu 22.04 LTS (recommandé)
wsl --install -d Ubuntu-22.04

# Ou installer Ubuntu 24.04 LTS
wsl --install -d Ubuntu-24.04
```

Lors du premier lancement, Ubuntu vous demandera de créer un utilisateur et un mot de passe. Ces identifiants sont uniquement pour votre environnement WSL2.

### Configuration de WSL2

#### Définir la distribution par défaut

```powershell
# Lister les distributions installées
wsl --list --verbose

# Définir Ubuntu comme distribution par défaut
wsl --set-default Ubuntu-22.04

# Vérifier la version WSL utilisée (doit afficher VERSION 2)
wsl --list --verbose
```

#### Configuration des ressources WSL2

Créez ou modifiez le fichier `.wslconfig` dans votre dossier utilisateur Windows :

```powershell
# Créer/éditer le fichier de configuration
notepad.exe $env:USERPROFILE\.wslconfig
```

Ajoutez la configuration suivante :

```ini
[wsl2]
# Mémoire maximale allouée à WSL2 (ajustez selon votre système)
memory=4GB

# Nombre de processeurs virtuels
processors=2

# Taille du swap (optionnel)
swap=2GB

# Emplacement du fichier swap (optionnel)
# swapFile=C:\\temp\\wsl-swap.vhdx

# Support de localhost forwarding
localhostForwarding=true

# Support nested virtualization (requis pour certains workloads)
nestedVirtualization=true
```

Appliquez les changements :

```powershell
# Arrêter WSL2
wsl --shutdown

# Redémarrer pour appliquer la configuration
wsl
```

## Configuration de l'environnement Ubuntu dans WSL2

### Première connexion et mise à jour

Lancez Ubuntu depuis le menu Démarrer ou via PowerShell :

```powershell
# Depuis PowerShell
wsl -d Ubuntu-22.04
```

Une fois dans Ubuntu :

```bash
# Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# Installer les outils essentiels
sudo apt install -y curl wget git vim build-essential

# Vérifier la version du kernel WSL2
uname -r
# Devrait afficher quelque chose comme : 5.15.xxx.x-microsoft-standard-WSL2
```

### Configuration réseau WSL2

WSL2 utilise un réseau NAT interne. Pour comprendre la configuration :

```bash
# Voir l'adresse IP de votre instance WSL2
ip addr show eth0

# Voir l'adresse IP de l'hôte Windows depuis WSL2
cat /etc/resolv.conf | grep nameserver

# Tester la connectivité Internet
ping -c 3 google.com
```

### Optimisations pour MicroK8s

```bash
# Désactiver swap (recommandé pour Kubernetes)
sudo swapoff -a

# Configurer les limites système
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
EOF

sudo sysctl --system
```

## Installation de MicroK8s dans WSL2

### Installation via Snap

```bash
# Installer snapd
sudo apt install -y snapd

# Important : Sur WSL2, systemd n'est pas activé par défaut sur les anciennes versions
# Vérifier si systemd est disponible
if systemctl is-system-running &>/dev/null; then
    echo "SystemD est disponible"
else
    echo "SystemD n'est pas disponible - activation requise"
fi
```

#### Activation de SystemD (si nécessaire)

Pour WSL2 versions récentes (Windows 11 ou Windows 10 avec mises à jour) :

```bash
# Créer ou éditer /etc/wsl.conf
sudo tee /etc/wsl.conf <<EOF
[boot]
systemd=true

[network]
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = true
EOF

# Redémarrer WSL2 depuis PowerShell
exit
```

Dans PowerShell :
```powershell
wsl --shutdown
wsl
```

#### Installation de MicroK8s

De retour dans Ubuntu :

```bash
# Installer MicroK8s
sudo snap install microk8s --classic --channel=1.31/stable

# Vérifier l'installation
snap list microk8s

# Ajouter l'utilisateur au groupe microk8s
sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
sudo chown -f -R $USER ~/.kube

# Se reconnecter pour appliquer les changements de groupe
exit
```

Reconnectez-vous :
```powershell
wsl
```

### Configuration post-installation

```bash
# Vérifier le statut
microk8s status --wait-ready

# Activer les addons essentiels pour un lab
microk8s enable dns storage

# Vérifier les nodes
microk8s kubectl get nodes

# Vérifier les pods système
microk8s kubectl get pods --all-namespaces
```

## Intégration Windows-WSL2

### Accès depuis Windows

#### Configuration de kubectl sur Windows

Installez kubectl sur Windows pour accéder au cluster depuis PowerShell :

```powershell
# Télécharger kubectl pour Windows
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"

# Créer un dossier pour kubectl
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\bin"

# Déplacer kubectl
Move-Item .\kubectl.exe $env:USERPROFILE\bin\

# Ajouter au PATH (nécessite redémarrage PowerShell)
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$env:USERPROFILE\bin", [EnvironmentVariableTarget]::User)
```

#### Copier la configuration Kubernetes

Dans WSL2 Ubuntu :
```bash
# Afficher la configuration
microk8s config

# Ou sauvegarder dans un fichier
microk8s config > ~/kubeconfig
```

Dans PowerShell :
```powershell
# Créer le dossier .kube
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.kube"

# Copier la configuration depuis WSL2
wsl cat ~/kubeconfig | Out-File -Encoding UTF8 "$env:USERPROFILE\.kube\config"

# Tester la connexion
kubectl get nodes
```

### Accès aux services depuis Windows

#### Port Forwarding automatique

WSL2 forward automatiquement certains ports. Pour accéder aux services Kubernetes depuis Windows :

```bash
# Dans WSL2, exposer un service sur un port
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 8443:443 --address=0.0.0.0
```

Accédez depuis le navigateur Windows : `https://localhost:8443`

#### Configuration d'un port forwarding permanent

Créez un script PowerShell pour le port forwarding :

```powershell
# Sauvegarder dans forward-ports.ps1
$wslIp = (wsl hostname -I).Trim()
Write-Host "WSL2 IP: $wslIp"

# Forwarding pour l'API Kubernetes
netsh interface portproxy add v4tov4 `
    listenport=16443 `
    listenaddress=0.0.0.0 `
    connectport=16443 `
    connectaddress=$wslIp

# Afficher les règles
netsh interface portproxy show all
```

### Intégration avec VS Code

Installation de l'extension Remote-WSL :

1. Ouvrez VS Code
2. Installez l'extension "WSL" ou "Remote Development"
3. Cliquez sur l'icône WSL en bas à gauche
4. Sélectionnez "Connect to WSL"

Une fois connecté, installez les extensions Kubernetes :
- Kubernetes
- YAML
- Docker (optionnel)

## Gestion des problèmes courants sur WSL2

### Problème : WSL2 consomme trop de mémoire

**Solution** : Ajustez `.wslconfig` :
```ini
[wsl2]
memory=2GB
processors=2
```

Puis redémarrez :
```powershell
wsl --shutdown
```

### Problème : Erreur "System has not been booted with systemd"

**Solution** : Activez systemd dans `/etc/wsl.conf` :
```bash
sudo tee /etc/wsl.conf <<EOF
[boot]
systemd=true
EOF
```

Redémarrez WSL2.

### Problème : Pas d'accès Internet depuis les pods

**Solution** : Vérifiez la configuration DNS :
```bash
# Dans WSL2
sudo tee /etc/docker/daemon.json <<EOF
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
EOF

# Redémarrer MicroK8s
microk8s stop
microk8s start
```

### Problème : Performance dégradée

**Causes et solutions** :

1. **Fichiers sur le système Windows** :
   - Évitez de stocker les fichiers Kubernetes sur `/mnt/c/`
   - Utilisez le système de fichiers Linux natif (`/home/`)

2. **Antivirus Windows** :
   - Excluez les dossiers WSL2 de l'analyse
   - Path typique : `%USERPROFILE%\AppData\Local\Packages\CanonicalGroupLimited*`

3. **Mémoire insuffisante** :
   - Augmentez la mémoire dans `.wslconfig`
   - Fermez les applications Windows non nécessaires

### Problème : Clock skew (décalage d'horloge)

WSL2 peut avoir des problèmes de synchronisation d'horloge après hibernation :

```bash
# Resynchroniser l'horloge
sudo hwclock -s

# Ou automatiser avec un service
cat <<EOF | sudo tee /etc/systemd/system/time-sync.service
[Unit]
Description=WSL2 time sync
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/hwclock -s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable time-sync.service
```

## Optimisations avancées pour WSL2

### Amélioration des performances I/O

```bash
# Utiliser des volumes Docker natifs plutôt que bind mounts Windows
# Créer un volume dans le filesystem Linux
docker volume create microk8s-data

# Monter dans /var/snap/microk8s pour de meilleures performances
sudo mkdir -p /var/snap/microk8s
sudo mount -t tmpfs -o size=2G tmpfs /var/snap/microk8s/common/var/lib/containerd
```

### Configuration de la mise en réseau avancée

Pour exposer MicroK8s sur votre réseau local :

```powershell
# Script PowerShell pour bridge networking
# À exécuter en tant qu'administrateur

$wslIp = (wsl hostname -I).Trim()
$hostIp = (Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias "Wi-Fi").IPAddress

# Créer les règles de forwarding
@(16443, 80, 443, 8080) | ForEach-Object {
    netsh interface portproxy delete v4tov4 listenport=$_ listenaddress=$hostIp
    netsh interface portproxy add v4tov4 listenport=$_ listenaddress=$hostIp connectport=$_ connectaddress=$wslIp
}

# Autoriser dans le firewall Windows
New-NetFirewallRule -DisplayName "WSL2 MicroK8s" -Direction Inbound -Protocol TCP -LocalPort 16443,80,443,8080 -Action Allow
```

### Sauvegarde et restauration

#### Sauvegarde de l'environnement WSL2

```powershell
# Exporter la distribution WSL2
wsl --export Ubuntu-22.04 "C:\WSL-Backups\Ubuntu-MicroK8s-$(Get-Date -Format 'yyyyMMdd').tar"

# Vérifier la taille
Get-Item "C:\WSL-Backups\*.tar" | Select-Object Name, @{Name="SizeGB";Expression={$_.Length / 1GB}}
```

#### Restauration

```powershell
# Importer une sauvegarde
wsl --import Ubuntu-MicroK8s-Restored "C:\WSL\Restored" "C:\WSL-Backups\Ubuntu-MicroK8s-20240101.tar"

# Définir comme défaut si nécessaire
wsl --set-default Ubuntu-MicroK8s-Restored
```

## Utilisation quotidienne

### Démarrage et arrêt

```powershell
# Démarrer WSL2 et MicroK8s
wsl -d Ubuntu-22.04 -u root -e sh -c "microk8s start"

# Arrêter proprement
wsl -d Ubuntu-22.04 -e sh -c "microk8s stop"
wsl --shutdown
```

### Script de démarrage automatique

Créez un script batch pour démarrer MicroK8s au démarrage de Windows :

```batch
@echo off
REM Sauvegarder dans %USERPROFILE%\start-microk8s.bat
echo Starting WSL2 and MicroK8s...
wsl -d Ubuntu-22.04 -u root -e microk8s start
echo MicroK8s started successfully
pause
```

Ajoutez ce script au démarrage Windows via le Planificateur de tâches.

### Accès depuis d'autres machines

Pour accéder à votre cluster MicroK8s depuis d'autres machines sur votre réseau :

1. Obtenez l'IP de votre machine Windows
2. Configurez le port forwarding (script PowerShell ci-dessus)
3. Configurez le firewall Windows
4. Partagez la configuration kubectl avec l'IP externe

## Monitoring des ressources

### Depuis Windows

```powershell
# Voir l'utilisation des ressources WSL2
wsl --list --verbose
Get-Process "vmmem" | Select-Object Name, @{N="MemoryGB";E={$_.WorkingSet64/1GB}}, @{N="CPU%";E={$_.CPU}}

# Voir les statistiques détaillées
wsl -d Ubuntu-22.04 -e sh -c "free -h && df -h && top -bn1 | head -20"
```

### Depuis WSL2

```bash
# Installation d'outils de monitoring
sudo apt install -y htop iotop

# Monitoring MicroK8s
watch microk8s kubectl top nodes
watch microk8s kubectl top pods --all-namespaces
```

## Points de vérification finale

Avant de continuer, assurez-vous que :

- ✅ WSL2 est installé et fonctionne avec Ubuntu
- ✅ SystemD est activé dans WSL2
- ✅ MicroK8s est installé et `microk8s status` indique "running"
- ✅ Vous pouvez exécuter `microk8s kubectl get nodes`
- ✅ L'accès depuis Windows fonctionne (kubectl ou navigateur)
- ✅ Les ressources sont correctement configurées dans `.wslconfig`
- ✅ Les performances sont acceptables pour votre lab

## Avantages et limitations

### Avantages de MicroK8s sur WSL2

- **Intégration Windows** : Utilisez vos outils Windows favoris
- **Performance** : Proche du natif avec WSL2
- **Isolation** : Environnement Linux complet et isolé
- **Portabilité** : Facilement sauvegardable et transférable

### Limitations à considérer

- **Performance I/O** : Plus lente sur les fichiers Windows (`/mnt/c/`)
- **Réseau** : NAT par défaut, configuration additionnelle pour l'exposition
- **Ressources** : Partage des ressources avec Windows
- **GPU** : Support limité pour les workloads GPU

## Prochaines étapes

Avec MicroK8s opérationnel sur WSL2, vous pouvez maintenant :

1. Configurer kubectl avec des alias pratiques (section 2.6)
2. Installer les addons nécessaires pour votre lab
3. Intégrer avec vos outils de développement Windows
4. Déployer vos premières applications

L'environnement WSL2 offre un excellent compromis entre la simplicité de Windows et la puissance de Kubernetes, idéal pour le développement et l'apprentissage.

⏭️
