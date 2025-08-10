🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.6 Configuration des alias kubectl

## Introduction

Travailler avec Kubernetes implique de taper fréquemment des commandes longues et répétitives. La commande `microk8s kubectl` peut rapidement devenir fastidieuse, surtout lors de sessions de travail intensives. Les alias sont des raccourcis qui permettent de simplifier considérablement votre workflow. Cette section vous guidera dans la configuration d'alias pratiques qui amélioreront votre productivité et rendront votre expérience avec MicroK8s plus agréable.

**Pourquoi les alias sont essentiels** :
- Réduction de la frappe : `k` au lieu de `microk8s kubectl`
- Moins d'erreurs de syntaxe
- Commands plus rapides et fluides
- Workflow plus naturel et intuitif
- Standardisation avec la communauté Kubernetes

## Configuration de base de kubectl

### Option 1 : Alias simple pour microk8s kubectl

La méthode la plus directe consiste à créer un alias pour la commande complète :

```bash
# Ajouter l'alias à votre fichier de configuration shell
# Pour bash (~/.bashrc)
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc

# Pour zsh (~/.zshrc) - utilisé par défaut sur macOS
echo "alias kubectl='microk8s kubectl'" >> ~/.zshrc

# Appliquer immédiatement les changements
source ~/.bashrc  # ou source ~/.zshrc pour zsh

# Tester l'alias
kubectl get nodes
```

### Option 2 : Installation de kubectl standalone

Pour une expérience plus flexible, installez kubectl séparément :

#### Sur Linux (Ubuntu/Debian)
```bash
# Télécharger la dernière version
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Rendre exécutable
chmod +x kubectl

# Déplacer vers le PATH
sudo mv kubectl /usr/local/bin/

# Vérifier l'installation
kubectl version --client
```

#### Sur CentOS/RHEL
```bash
# Ajouter le repository Kubernetes
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
EOF

# Installer kubectl
sudo yum install -y kubectl
```

#### Sur Windows (PowerShell)
```powershell
# Télécharger kubectl
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"

# Créer un dossier pour les binaires
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\bin"

# Déplacer kubectl
Move-Item .\kubectl.exe "$env:USERPROFILE\bin\"

# Ajouter au PATH
$oldPath = [Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::SetEnvironmentVariable("Path", "$oldPath;$env:USERPROFILE\bin", "User")
```

#### Sur macOS
```bash
# Via Homebrew
brew install kubectl

# Vérifier l'installation
kubectl version --client
```

### Configuration de kubeconfig

Après avoir installé kubectl standalone, configurez-le pour MicroK8s :

```bash
# Créer le répertoire de configuration
mkdir -p ~/.kube

# Exporter la configuration MicroK8s
microk8s config > ~/.kube/config

# Définir les permissions appropriées
chmod 600 ~/.kube/config

# Vérifier la connexion
kubectl get nodes
```

## Alias essentiels pour la productivité

### Alias de base recommandés

Créez un fichier dédié aux alias Kubernetes :

```bash
# Créer un fichier pour les alias Kubernetes
cat << 'EOF' > ~/.kubectl_aliases

# Alias principal - le plus utilisé
alias k='kubectl'

# Commandes get fréquentes
alias kg='kubectl get'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kgaa='kubectl get all --all-namespaces'

# Commandes describe
alias kd='kubectl describe'
alias kdp='kubectl describe pod'
alias kds='kubectl describe service'
alias kdd='kubectl describe deployment'
alias kdn='kubectl describe node'

# Commandes logs et exec
alias kl='kubectl logs'
alias klf='kubectl logs -f'  # Follow logs
alias ke='kubectl exec'
alias kei='kubectl exec -it'  # Interactive exec

# Commandes apply et delete
alias ka='kubectl apply'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias kdelf='kubectl delete -f'

# Commandes de création
alias kc='kubectl create'
alias kcf='kubectl create -f'

# Namespace
alias kns='kubectl config set-context --current --namespace'
alias kgns='kubectl get namespaces'

# Contexte
alias kctx='kubectl config current-context'
alias kgc='kubectl config get-contexts'

# Port forwarding
alias kpf='kubectl port-forward'

# Top (monitoring)
alias ktop='kubectl top'
alias ktopp='kubectl top pods'
alias ktopn='kubectl top nodes'

EOF

# Ajouter le fichier d'alias à votre configuration shell
echo "source ~/.kubectl_aliases" >> ~/.bashrc  # ou ~/.zshrc
source ~/.kubectl_aliases
```

### Alias avancés avec paramètres

Créez des fonctions pour des opérations plus complexes :

```bash
# Ajouter ces fonctions à ~/.bashrc ou ~/.zshrc

# Fonction pour obtenir rapidement les pods d'un namespace
kgpn() {
    if [ -z "$1" ]; then
        echo "Usage: kgpn <namespace>"
        return 1
    fi
    kubectl get pods -n "$1"
}

# Fonction pour obtenir les logs du premier pod d'un deployment
klogs() {
    if [ -z "$1" ]; then
        echo "Usage: klogs <deployment-name> [namespace]"
        return 1
    fi
    local namespace=${2:-default}
    local pod=$(kubectl get pods -n $namespace -l app=$1 -o jsonpath='{.items[0].metadata.name}')
    if [ -z "$pod" ]; then
        echo "No pod found for deployment $1 in namespace $namespace"
        return 1
    fi
    kubectl logs -n $namespace $pod
}

# Fonction pour redémarrer un deployment
krestart() {
    if [ -z "$1" ]; then
        echo "Usage: krestart <deployment-name> [namespace]"
        return 1
    fi
    local namespace=${2:-default}
    kubectl rollout restart deployment/$1 -n $namespace
}

# Fonction pour obtenir les événements triés
kevents() {
    kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
}

# Fonction pour nettoyer les pods terminés
kclean() {
    kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces
    kubectl delete pods --field-selector=status.phase=Failed --all-namespaces
}

# Fonction pour obtenir l'IP d'un pod
kip() {
    if [ -z "$1" ]; then
        echo "Usage: kip <pod-name> [namespace]"
        return 1
    fi
    local namespace=${2:-default}
    kubectl get pod $1 -n $namespace -o jsonpath='{.status.podIP}'
}

# Fonction pour voir l'utilisation des ressources
kres() {
    echo "=== Node Resources ==="
    kubectl top nodes
    echo ""
    echo "=== Pod Resources (Top 10 by CPU) ==="
    kubectl top pods --all-namespaces | head -11
}
```

## Configuration de l'auto-complétion

### Bash

```bash
# Installer bash-completion si nécessaire
# Ubuntu/Debian
sudo apt-get install bash-completion

# CentOS/RHEL
sudo yum install bash-completion

# Activer l'auto-complétion pour kubectl
kubectl completion bash > ~/.kubectl_completion
echo "source ~/.kubectl_completion" >> ~/.bashrc

# Activer aussi pour l'alias 'k'
echo "complete -F __start_kubectl k" >> ~/.bashrc

# Recharger la configuration
source ~/.bashrc
```

### Zsh

```bash
# Activer l'auto-complétion pour kubectl
kubectl completion zsh > ~/.kubectl_completion
echo "source ~/.kubectl_completion" >> ~/.zshrc

# Pour l'alias 'k'
echo "compdef k=kubectl" >> ~/.zshrc

# Recharger la configuration
source ~/.zshrc
```

### PowerShell (Windows)

```powershell
# Installer PSKubectlCompletion
Install-Module -Name PSKubectlCompletion -Scope CurrentUser

# Ajouter au profil PowerShell
Add-Content $PROFILE "Import-Module PSKubectlCompletion"

# Recharger le profil
. $PROFILE
```

## Alias spécifiques à MicroK8s

### Gestion des addons

```bash
# Alias pour les addons MicroK8s
cat << 'EOF' >> ~/.kubectl_aliases

# MicroK8s specific
alias mks='microk8s status'
alias mke='microk8s enable'
alias mkd='microk8s disable'
alias mki='microk8s inspect'
alias mkr='microk8s reset'
alias mkstart='microk8s start'
alias mkstop='microk8s stop'

# Fonction pour activer plusieurs addons
mkenable() {
    for addon in "$@"; do
        echo "Enabling $addon..."
        microk8s enable $addon
    done
}

# Fonction pour voir les addons activés
mkaddons() {
    microk8s status | grep -A 100 "addons:" | grep "enabled:" -A 20
}

EOF

source ~/.kubectl_aliases
```

### Alias pour différents environnements

Si vous gérez plusieurs clusters ou namespaces :

```bash
# Alias pour changer rapidement de namespace
cat << 'EOF' >> ~/.kubectl_aliases

# Namespaces shortcuts
alias kdefault='kubectl config set-context --current --namespace=default'
alias ksystem='kubectl config set-context --current --namespace=kube-system'
alias kpublic='kubectl config set-context --current --namespace=kube-public'

# Fonction pour créer un alias de namespace à la volée
mkns() {
    if [ -z "$1" ]; then
        echo "Usage: mkns <namespace>"
        return 1
    fi
    kubectl create namespace $1 2>/dev/null
    kubectl config set-context --current --namespace=$1
    echo "Switched to namespace: $1"
}

# Afficher le namespace actuel dans le prompt (bash)
export PS1='[\u@\h \W $(kubectl config view --minify -o jsonpath="{.contexts[0].context.namespace}" 2>/dev/null || echo "no-context")]\$ '

EOF
```

## Configuration pour différents shells

### Fish Shell

```fish
# Créer le fichier de configuration Fish
mkdir -p ~/.config/fish/functions

# Créer les alias pour Fish
cat << 'EOF' > ~/.config/fish/config.fish

# Kubectl aliases for Fish
abbr -a k kubectl
abbr -a kg kubectl get
abbr -a kgp kubectl get pods
abbr -a kgs kubectl get services
abbr -a kgd kubectl get deployments
abbr -a kd kubectl describe
abbr -a kdp kubectl describe pod
abbr -a kl kubectl logs
abbr -a ke kubectl exec -it
abbr -a ka kubectl apply -f
abbr -a kdel kubectl delete -f

# MicroK8s aliases
abbr -a mks microk8s status
abbr -a mke microk8s enable
abbr -a mkd microk8s disable

EOF

# Auto-complétion pour Fish
kubectl completion fish > ~/.config/fish/completions/kubectl.fish
```

### Configuration multi-shell

Pour gérer plusieurs shells, créez un script universel :

```bash
# Créer un script d'installation des alias
cat << 'EOF' > ~/setup-kubectl-aliases.sh
#!/bin/bash

SHELL_RC=""
SHELL_NAME=""

# Détecter le shell
if [ -n "$BASH_VERSION" ]; then
    SHELL_RC="$HOME/.bashrc"
    SHELL_NAME="bash"
elif [ -n "$ZSH_VERSION" ]; then
    SHELL_RC="$HOME/.zshrc"
    SHELL_NAME="zsh"
else
    echo "Shell non supporté"
    exit 1
fi

echo "Configuration pour $SHELL_NAME..."

# Créer le fichier d'alias s'il n'existe pas
if [ ! -f ~/.kubectl_aliases ]; then
    echo "Création des alias kubectl..."
    # [Copier ici le contenu des alias de base]
fi

# Ajouter au fichier RC si pas déjà présent
if ! grep -q "source ~/.kubectl_aliases" "$SHELL_RC"; then
    echo "source ~/.kubectl_aliases" >> "$SHELL_RC"
    echo "Alias ajoutés à $SHELL_RC"
fi

# Configuration de l'auto-complétion
echo "Configuration de l'auto-complétion..."
kubectl completion $SHELL_NAME > ~/.kubectl_completion
if ! grep -q "source ~/.kubectl_completion" "$SHELL_RC"; then
    echo "source ~/.kubectl_completion" >> "$SHELL_RC"
fi

echo "Configuration terminée ! Rechargez votre shell avec: source $SHELL_RC"
EOF

chmod +x ~/setup-kubectl-aliases.sh
```

## Personnalisation du prompt

### Afficher le contexte Kubernetes dans le prompt

#### Bash avec couleurs

```bash
# Ajouter à ~/.bashrc
cat << 'EOF' >> ~/.bashrc

# Fonction pour obtenir le contexte Kubernetes
kube_context() {
    local context=$(kubectl config current-context 2>/dev/null)
    local namespace=$(kubectl config view --minify -o jsonpath='{.contexts[0].context.namespace}' 2>/dev/null)

    if [ -n "$context" ]; then
        namespace=${namespace:-default}
        echo -e "\033[1;34m[$context:$namespace]\033[0m"
    fi
}

# Prompt personnalisé avec contexte Kubernetes
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\] $(kube_context)\n\$ '

EOF
```

#### Zsh avec Powerlevel10k

```bash
# Si vous utilisez Oh My Zsh avec Powerlevel10k
# Ajouter à ~/.zshrc
plugins=(... kubectl)

# Ou pour une configuration manuelle
# Ajouter à ~/.zshrc
PROMPT='%{$fg[green]%}%n@%m%{$reset_color%}:%{$fg[blue]%}%~%{$reset_color%} $(kubectl config current-context 2>/dev/null || echo "")
$ '
```

### Utilisation de kube-ps1

Pour un prompt plus sophistiqué :

```bash
# Cloner kube-ps1
git clone https://github.com/jonmosco/kube-ps1.git ~/.kube-ps1

# Pour Bash
echo "source ~/.kube-ps1/kube-ps1.sh" >> ~/.bashrc
echo "PS1='[\u@\h \W \$(kube_ps1)]\$ '" >> ~/.bashrc

# Pour Zsh
echo "source ~/.kube-ps1/kube-ps1.sh" >> ~/.zshrc
echo "PROMPT='$(kube_ps1)'$PROMPT" >> ~/.zshrc

# Personnaliser l'affichage
export KUBE_PS1_PREFIX="["
export KUBE_PS1_SUFFIX="]"
export KUBE_PS1_SYMBOL_ENABLE=true
export KUBE_PS1_SYMBOL_DEFAULT="⎈"
```

## Alias pour les opérations courantes

### Débogage et troubleshooting

```bash
cat << 'EOF' >> ~/.kubectl_aliases

# Alias de débogage
alias kdbg='kubectl run debug-pod --rm -i --tty --image=busybox -- sh'
alias kdbgubuntu='kubectl run debug-ubuntu --rm -i --tty --image=ubuntu -- bash'
alias kdbgcurl='kubectl run debug-curl --rm -i --tty --image=curlimages/curl -- sh'

# Fonction pour debug un pod existant
kdebug() {
    if [ -z "$1" ]; then
        echo "Usage: kdebug <pod-name> [namespace]"
        return 1
    fi
    local namespace=${2:-default}
    kubectl debug -it $1 -n $namespace --image=busybox -- sh
}

# Voir les derniers événements d'erreur
kerrors() {
    kubectl get events --all-namespaces --field-selector type=Warning --sort-by='.lastTimestamp' | tail -20
}

# Fonction pour obtenir le YAML d'une ressource
kyaml() {
    if [ -z "$1" ] || [ -z "$2" ]; then
        echo "Usage: kyaml <resource-type> <resource-name> [namespace]"
        return 1
    fi
    local namespace=${3:-default}
    kubectl get $1 $2 -n $namespace -o yaml
}

# Fonction pour éditer rapidement une ressource
kedit() {
    if [ -z "$1" ] || [ -z "$2" ]; then
        echo "Usage: kedit <resource-type> <resource-name> [namespace]"
        return 1
    fi
    local namespace=${3:-default}
    kubectl edit $1 $2 -n $namespace
}

EOF
```

### Gestion des ressources

```bash
cat << 'EOF' >> ~/.kubectl_aliases

# Alias pour la gestion des ressources
alias kscale='kubectl scale'
alias kautoscale='kubectl autoscale'

# Fonction pour scaler rapidement
scale() {
    if [ -z "$1" ] || [ -z "$2" ]; then
        echo "Usage: scale <deployment-name> <replicas> [namespace]"
        return 1
    fi
    local namespace=${3:-default}
    kubectl scale deployment/$1 --replicas=$2 -n $namespace
}

# Fonction pour voir l'utilisation des ressources par namespace
kresns() {
    for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
        echo "=== Namespace: $ns ==="
        kubectl top pods -n $ns --no-headers 2>/dev/null | head -5
        echo ""
    done
}

# Fonction pour trouver les pods gourmands en ressources
kheavy() {
    echo "=== Pods using most CPU ==="
    kubectl top pods --all-namespaces | sort -k3 -rn | head -10
    echo ""
    echo "=== Pods using most Memory ==="
    kubectl top pods --all-namespaces | sort -k4 -rn | head -10
}

EOF
```

## Création d'un fichier de configuration complet

Pour simplifier la configuration, créez un script d'installation complet :

```bash
cat << 'EOF' > ~/install-kubectl-aliases.sh
#!/bin/bash

echo "Installation des alias kubectl pour MicroK8s"
echo "============================================"

# Détection du shell
if [ -n "$BASH_VERSION" ]; then
    RC_FILE="$HOME/.bashrc"
    SHELL_TYPE="bash"
elif [ -n "$ZSH_VERSION" ]; then
    RC_FILE="$HOME/.zshrc"
    SHELL_TYPE="zsh"
else
    echo "Erreur: Shell non supporté"
    exit 1
fi

echo "Shell détecté: $SHELL_TYPE"
echo "Fichier de configuration: $RC_FILE"

# Backup du fichier RC existant
cp $RC_FILE ${RC_FILE}.backup.$(date +%Y%m%d_%H%M%S)
echo "Backup créé: ${RC_FILE}.backup.*"

# Installation des alias
echo "Installation des alias..."

# [Insérer ici tout le contenu des alias]

echo ""
echo "Installation terminée !"
echo ""
echo "Pour activer les alias immédiatement, exécutez:"
echo "  source $RC_FILE"
echo ""
echo "Alias disponibles:"
echo "  k       - kubectl"
echo "  kgp     - kubectl get pods"
echo "  kgs     - kubectl get services"
echo "  kl      - kubectl logs"
echo "  ke      - kubectl exec -it"
echo "  Et bien d'autres..."
echo ""
echo "Pour voir tous les alias: alias | grep kubectl"

EOF

chmod +x ~/install-kubectl-aliases.sh
```

## Vérification et test des alias

### Script de test

```bash
cat << 'EOF' > ~/test-kubectl-aliases.sh
#!/bin/bash

echo "Test des alias kubectl"
echo "====================="

# Test des alias de base
echo -n "Test de 'k': "
if command -v k &> /dev/null; then
    echo "✓ OK"
else
    echo "✗ Échec"
fi

echo -n "Test de 'kgp': "
if alias kgp &> /dev/null; then
    echo "✓ OK"
else
    echo "✗ Échec"
fi

# Test de l'auto-complétion
echo -n "Test de l'auto-complétion: "
if complete -p kubectl &> /dev/null; then
    echo "✓ OK"
else
    echo "✗ Non configuré"
fi

echo ""
echo "Liste des alias configurés:"
alias | grep -E "^(k|mk)" | head -10
echo "..."

EOF

chmod +x ~/test-kubectl-aliases.sh
```

## Bonnes pratiques

### Organisation des alias

1. **Cohérence** : Utilisez une convention de nommage cohérente
2. **Simplicité** : Les alias doivent être courts et mémorables
3. **Documentation** : Commentez vos alias personnalisés
4. **Groupement** : Organisez par catégorie (get, describe, logs, etc.)

### Sécurité

```bash
# Ajouter des alias de confirmation pour les opérations dangereuses
cat << 'EOF' >> ~/.kubectl_aliases

# Alias avec confirmation pour les opérations critiques
alias kdelforce='kubectl delete --force --grace-period=0'

# Fonction pour supprimer avec confirmation
ksafedelete() {
    echo "Vous allez supprimer: $@"
    read -p "Êtes-vous sûr? (yes/no): " confirm
    if [ "$confirm" = "yes" ]; then
        kubectl delete "$@"
    else
        echo "Suppression annulée"
    fi
}

# Dry-run par défaut pour apply
alias kapply='kubectl apply --dry-run=client -f'
alias kapplyreal='kubectl apply -f'

EOF
```

## Dépannage des alias

### Problèmes courants

| Problème | Solution |
|----------|----------|
| Alias non reconnu | Recharger le shell avec `source ~/.bashrc` |
| Auto-complétion ne fonctionne pas | Installer bash-completion et recharger |
| Conflit avec des alias existants | Utiliser `unalias <nom>` avant de redéfinir |
| Alias non persistant | Vérifier qu'il est dans le bon fichier RC |

### Debug des alias

```bash
# Voir tous les alias définis
alias

# Voir la définition d'un alias spécifique
alias k

# Tracer l'exécution d'un alias
set -x
k get pods
set +x

# Vérifier quel fichier définit un alias
grep -r "alias k=" ~/.*
```

## Points de vérification finale

Avant de continuer, assurez-vous que :

- ✅ L'alias `k` fonctionne pour `kubectl`
- ✅ Les alias de base sont configurés (kgp, kgs, kl, etc.)
- ✅ L'auto-complétion fonctionne
- ✅ Les alias sont persistants après redémarrage du shell
- ✅ Vous connaissez vos alias les plus utilisés
- ✅ Le prompt affiche le contexte Kubernetes (optionnel)

## Prochaines étapes

Avec vos alias configurés, vous êtes prêt à :

1. Commencer le diagnostic rapide du cluster (section 2.7)
2. Explorer les addons essentiels
3. Travailler efficacement avec Kubernetes
4. Personnaliser davantage selon vos besoins

Les alias que vous venez de configurer vous feront gagner un temps précieux dans votre utilisation quotidienne de MicroK8s.

⏭️
