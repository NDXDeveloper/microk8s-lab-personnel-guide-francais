🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.6 Mise à jour de MicroK8s

## Introduction aux mises à jour dans MicroK8s

Maintenir MicroK8s à jour est essentiel pour bénéficier des dernières fonctionnalités, des correctifs de sécurité et des améliorations de performance. Cependant, les mises à jour peuvent aussi introduire des changements incompatibles ou des régressions. Cette section vous guidera à travers les stratégies de mise à jour, les précautions à prendre, et les procédures de rollback en cas de problème. Nous couvrirons les différents types de mises à jour, depuis les patches mineurs jusqu'aux changements de version majeure de Kubernetes.

## Comprendre les versions et les canaux

### Structure des versions

MicroK8s suit un système de versionnement qui reflète la version de Kubernetes qu'il embarque, avec quelques nuances importantes :

**Version Kubernetes** : MicroK8s inclut une version spécifique de Kubernetes (par exemple, 1.28, 1.29, 1.30). Chaque version majeure/mineure de Kubernetes apporte de nouvelles fonctionnalités et peut déprécier ou supprimer certaines APIs.

**Version MicroK8s** : En plus de la version Kubernetes, MicroK8s a ses propres révisions qui incluent des corrections de bugs, des améliorations spécifiques à MicroK8s, et des mises à jour des addons.

**Format de version** : Une version complète ressemble à `1.29/stable` ou `1.28.3` où :
- Le premier nombre est la version majeure de Kubernetes
- Le deuxième est la version mineure de Kubernetes
- Le troisième (si présent) est la version patch
- Le canal (stable, candidate, beta, edge) indique la maturité

### Les canaux de distribution

MicroK8s utilise le système de canaux Snap pour gérer les releases :

**Stable** : Canal recommandé pour la production et les labs personnels. Contient des versions bien testées et stables. Les mises à jour sont moins fréquentes mais plus fiables.

**Candidate** : Versions en phase finale de test avant d'être promues en stable. Utile pour tester les prochaines versions dans un environnement non-critique.

**Beta** : Versions avec de nouvelles fonctionnalités en cours de stabilisation. Peut contenir des bugs mais permet d'accéder aux dernières innovations.

**Edge** : Versions de développement construites automatiquement. Très instables, réservées aux développeurs et contributeurs.

### Cycle de vie des versions

Kubernetes suit un cycle de release prévisible :
- Une nouvelle version mineure tous les 3-4 mois
- Support des trois dernières versions mineures
- Les versions plus anciennes ne reçoivent plus de mises à jour de sécurité

MicroK8s maintient généralement :
- Les versions correspondant aux trois dernières versions de Kubernetes
- Des canaux spécifiques pour chaque version majeure/mineure
- Des mises à jour automatiques dans le même canal

## Vérification de l'état actuel

### Identifier votre version actuelle

Avant toute mise à jour, documentez l'état actuel de votre système :

```bash
# Version de MicroK8s et Kubernetes
microk8s version

# Version détaillée avec le numéro de révision Snap
snap info microk8s | grep installed

# Version des composants Kubernetes
microk8s kubectl version --short

# Lister les addons activés et leurs versions
microk8s status --format short

# Vérifier le canal actuel
snap info microk8s | grep tracking

# Informations complètes sur l'installation
microk8s inspect | grep -A 5 "Version"
```

### Audit de compatibilité

Vérifiez la compatibilité de vos applications :

```bash
# Lister toutes les API utilisées dans le cluster
microk8s kubectl api-resources --verbs=list -o name | \
  xargs -n 1 microk8s kubectl get --all-namespaces -o json 2>/dev/null | \
  jq -r '.items[]?.apiVersion' | sort -u

# Vérifier les API dépréciées
microk8s kubectl get apiservices | grep -E "v1beta|v1alpha"

# Examiner les deployments pour les images utilisées
microk8s kubectl get deployments --all-namespaces -o json | \
  jq -r '.items[].spec.template.spec.containers[].image' | sort -u

# Script pour vérifier la compatibilité des manifests
cat <<'EOF' > check-compatibility.sh
#!/bin/bash
echo "=== Vérification de compatibilité des manifests ==="

# APIs à vérifier (ajustez selon votre version cible)
deprecated_apis=(
  "extensions/v1beta1"
  "apps/v1beta1"
  "apps/v1beta2"
  "networking.k8s.io/v1beta1"
)

for api in "${deprecated_apis[@]}"; do
  echo "Recherche de l'utilisation de $api..."
  microk8s kubectl get all --all-namespaces -o json | \
    jq -r --arg api "$api" '.items[] | select(.apiVersion == $api) | "\(.kind)/\(.metadata.name) in \(.metadata.namespace)"'
done
EOF

chmod +x check-compatibility.sh
./check-compatibility.sh
```

### Sauvegarde pré-mise à jour

Toujours effectuer une sauvegarde avant la mise à jour :

```bash
# Créer un répertoire de sauvegarde daté
BACKUP_DIR="$HOME/microk8s-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

# Sauvegarder tous les manifests du cluster
echo "Sauvegarde des manifests..."
for resource in $(microk8s kubectl api-resources --verbs=list -o name); do
  echo "  Sauvegarde de $resource..."
  microk8s kubectl get $resource --all-namespaces -o yaml > \
    "$BACKUP_DIR/${resource}.yaml" 2>/dev/null
done

# Sauvegarder les configurations spécifiques
echo "Sauvegarde des configurations..."
microk8s kubectl get configmaps --all-namespaces -o yaml > "$BACKUP_DIR/configmaps.yaml"
microk8s kubectl get secrets --all-namespaces -o yaml > "$BACKUP_DIR/secrets.yaml"
microk8s kubectl get pv -o yaml > "$BACKUP_DIR/persistentvolumes.yaml"
microk8s kubectl get pvc --all-namespaces -o yaml > "$BACKUP_DIR/persistentvolumeclaims.yaml"

# Sauvegarder la configuration MicroK8s
echo "Sauvegarde de la configuration MicroK8s..."
microk8s status > "$BACKUP_DIR/microk8s-status.txt"
microk8s.inspect > "$BACKUP_DIR/microk8s-inspect.txt" 2>&1

# Créer une archive
tar -czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"
echo "Sauvegarde complète dans: $BACKUP_DIR.tar.gz"
```

## Types de mises à jour

### Mise à jour dans le même canal (patch)

Les mises à jour les plus simples et les plus sûres :

```bash
# Vérifier les mises à jour disponibles
snap refresh --list microk8s

# Mettre à jour vers la dernière version du canal actuel
sudo snap refresh microk8s

# Suivre le processus
watch -n 2 "snap changes | head -20"

# Vérifier après la mise à jour
microk8s status --wait-ready
microk8s version
```

### Changement de version mineure

Pour passer à une version mineure supérieure (ex: 1.28 → 1.29) :

```bash
# Lister les canaux disponibles
snap info microk8s | grep -E "^\s+1\."

# Voir les détails d'un canal spécifique
snap info microk8s --channel=1.29/stable

# Changer de canal et mettre à jour
sudo snap refresh microk8s --channel=1.29/stable

# Attendre que la mise à jour soit complète
microk8s status --wait-ready

# Vérifier les changements
microk8s kubectl version
microk8s kubectl get nodes
```

### Mise à jour avec changement de canal de stabilité

Pour tester une version candidate ou beta :

```bash
# Passer en candidate pour tester
sudo snap refresh microk8s --channel=1.29/candidate

# Revenir en stable si nécessaire
sudo snap refresh microk8s --channel=1.29/stable

# Forcer une mise à jour même si c'est un downgrade
sudo snap refresh microk8s --channel=1.28/stable --amend
```

## Processus de mise à jour sécurisé

### Procédure étape par étape

Un processus complet et sûr pour les mises à jour importantes :

```bash
#!/bin/bash
# safe-update.sh

set -e  # Arrêter en cas d'erreur

echo "=== Processus de mise à jour sécurisé de MicroK8s ==="

# 1. Variables
TARGET_CHANNEL=${1:-""}
if [ -z "$TARGET_CHANNEL" ]; then
  echo "Usage: $0 <channel> (ex: 1.29/stable)"
  exit 1
fi

BACKUP_DIR="$HOME/microk8s-backup-$(date +%Y%m%d-%H%M%S)"

# 2. Vérifications préliminaires
echo "Étape 1: Vérifications préliminaires"
echo "Version actuelle:"
microk8s version
echo "Canal cible: $TARGET_CHANNEL"
read -p "Continuer? (y/n) " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
  exit 1
fi

# 3. Sauvegarde
echo -e "\nÉtape 2: Sauvegarde"
mkdir -p $BACKUP_DIR
microk8s kubectl get all --all-namespaces -o yaml > "$BACKUP_DIR/all-resources.yaml"
microk8s status > "$BACKUP_DIR/status.txt"
echo "Sauvegarde créée dans $BACKUP_DIR"

# 4. Test de santé pré-mise à jour
echo -e "\nÉtape 3: Test de santé"
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces | grep -v Running | grep -v Completed || true

# 5. Mise à jour
echo -e "\nÉtape 4: Mise à jour vers $TARGET_CHANNEL"
sudo snap refresh microk8s --channel=$TARGET_CHANNEL

# 6. Attente de stabilisation
echo -e "\nÉtape 5: Attente de stabilisation"
sleep 10
microk8s status --wait-ready --timeout 300

# 7. Vérifications post-mise à jour
echo -e "\nÉtape 6: Vérifications post-mise à jour"
microk8s version
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces | grep -v Running | grep -v Completed || true

echo -e "\n✅ Mise à jour terminée avec succès!"
echo "En cas de problème, restaurez depuis: $BACKUP_DIR"
```

### Mise à jour des addons

Après la mise à jour de MicroK8s, les addons peuvent nécessiter une attention :

```bash
# Lister les addons et leur état
microk8s status

# Mettre à jour un addon spécifique
# Désactiver puis réactiver pour forcer la mise à jour
microk8s disable dns
microk8s enable dns

# Mettre à jour cert-manager
microk8s disable cert-manager
microk8s enable cert-manager

# Script pour mettre à jour tous les addons
cat <<'EOF' > update-addons.sh
#!/bin/bash
echo "=== Mise à jour des addons MicroK8s ==="

# Obtenir la liste des addons activés
enabled_addons=$(microk8s status | grep enabled | awk '{print $1}' | tr -d ':')

for addon in $enabled_addons; do
  echo "Mise à jour de $addon..."

  # Certains addons ne doivent pas être désactivés
  if [[ "$addon" == "ha-cluster" ]] || [[ "$addon" == "storage" ]]; then
    echo "  ⚠️  Skipping $addon (critique)"
    continue
  fi

  echo "  Désactivation..."
  microk8s disable $addon

  echo "  Réactivation..."
  microk8s enable $addon

  echo "  ✅ $addon mis à jour"
  sleep 5
done

echo "Mise à jour des addons terminée!"
EOF

chmod +x update-addons.sh
```

## Gestion des problèmes de mise à jour

### Problèmes courants et solutions

**Pods qui ne redémarrent pas après la mise à jour** :

```bash
# Identifier les pods problématiques
microk8s kubectl get pods --all-namespaces | grep -v Running

# Forcer le redémarrage d'un deployment
microk8s kubectl rollout restart deployment/<name> -n <namespace>

# Supprimer les pods stuck
microk8s kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0

# Redémarrer tous les deployments d'un namespace
microk8s kubectl get deployments -n <namespace> -o name | \
  xargs -I {} microk8s kubectl rollout restart {} -n <namespace>
```

**Services inaccessibles après mise à jour** :

```bash
# Vérifier les services et endpoints
microk8s kubectl get svc --all-namespaces
microk8s kubectl get endpoints --all-namespaces

# Recréer les services si nécessaire
microk8s kubectl get service <service-name> -n <namespace> -o yaml > service-backup.yaml
microk8s kubectl delete service <service-name> -n <namespace>
microk8s kubectl apply -f service-backup.yaml

# Vérifier CoreDNS
microk8s kubectl rollout restart deployment/coredns -n kube-system
```

**Addons qui ne fonctionnent plus** :

```bash
# Réinitialiser un addon problématique
microk8s reset addon <addon-name>

# Pour l'ingress
microk8s disable ingress
microk8s kubectl delete namespace ingress
microk8s enable ingress

# Pour le dashboard
microk8s disable dashboard
microk8s kubectl delete namespace kubernetes-dashboard
microk8s enable dashboard
```

### Diagnostic des échecs de mise à jour

Script de diagnostic complet :

```bash
#!/bin/bash
# diagnose-update-issues.sh

echo "=== Diagnostic post-mise à jour ==="

# 1. État général
echo -e "\n1. État général du cluster"
microk8s status
microk8s kubectl get nodes

# 2. Pods problématiques
echo -e "\n2. Pods avec problèmes"
microk8s kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# 3. Événements récents
echo -e "\n3. Événements système (dernière heure)"
microk8s kubectl get events --all-namespaces \
  --sort-by='.lastTimestamp' | head -20

# 4. Logs des composants système
echo -e "\n4. Erreurs dans les logs système"
journalctl -u snap.microk8s.daemon-kubelite --since "1 hour ago" | \
  grep -i error | tail -20

# 5. Vérification des API
echo -e "\n5. APIs disponibles"
microk8s kubectl api-versions

# 6. État des addons
echo -e "\n6. État des addons"
for addon in dns ingress dashboard storage; do
  echo -n "  $addon: "
  if microk8s status | grep -q "$addon: enabled"; then
    echo "✅ enabled"
    # Vérifier les pods de l'addon
    case $addon in
      dns)
        microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns --no-headers
        ;;
      ingress)
        microk8s kubectl get pods -n ingress --no-headers
        ;;
      dashboard)
        microk8s kubectl get pods -n kubernetes-dashboard --no-headers
        ;;
    esac
  else
    echo "❌ disabled"
  fi
done

# 7. Ressources disponibles
echo -e "\n7. Ressources disponibles"
microk8s kubectl top nodes 2>/dev/null || echo "Metrics-server non disponible"

echo -e "\n=== Fin du diagnostic ==="
```

## Rollback (retour en arrière)

### Rollback automatique de Snap

Snap conserve automatiquement la version précédente :

```bash
# Voir les révisions disponibles
snap list microk8s --all

# Revenir à la révision précédente
sudo snap revert microk8s

# Revenir à une révision spécifique
sudo snap revert microk8s --revision=<number>

# Désactiver les mises à jour automatiques temporairement
sudo snap refresh --hold=24h microk8s
```

### Rollback manuel avec sauvegarde

Si le rollback automatique échoue :

```bash
#!/bin/bash
# manual-rollback.sh

BACKUP_DIR=$1

if [ -z "$BACKUP_DIR" ]; then
  echo "Usage: $0 <backup-directory>"
  exit 1
fi

echo "=== Rollback manuel depuis $BACKUP_DIR ==="

# 1. Arrêter MicroK8s
echo "Arrêt de MicroK8s..."
microk8s stop

# 2. Revenir à la version précédente
echo "Revert de Snap..."
sudo snap revert microk8s

# 3. Démarrer MicroK8s
echo "Démarrage de MicroK8s..."
microk8s start
microk8s status --wait-ready

# 4. Restaurer les configurations si nécessaire
echo "Restauration des configurations..."
if [ -f "$BACKUP_DIR/configmaps.yaml" ]; then
  microk8s kubectl apply -f "$BACKUP_DIR/configmaps.yaml"
fi

if [ -f "$BACKUP_DIR/secrets.yaml" ]; then
  # Attention avec les secrets système
  grep -v "namespace: kube-system" "$BACKUP_DIR/secrets.yaml" | \
    microk8s kubectl apply -f -
fi

echo "✅ Rollback terminé"
echo "Vérifiez l'état du cluster avec: microk8s status"
```

## Stratégies de mise à jour pour différents environnements

### Lab de développement

Pour un environnement de développement, vous pouvez être plus agressif :

```bash
# Mise à jour automatique vers la dernière version stable
sudo snap refresh microk8s --channel=latest/stable

# Tester les versions beta
sudo snap refresh microk8s --channel=latest/beta

# Script de mise à jour rapide pour dev
cat <<'EOF' > quick-update-dev.sh
#!/bin/bash
echo "Mise à jour rapide pour environnement de dev"
microk8s stop
sudo snap refresh microk8s
microk8s start
microk8s status --wait-ready
echo "Nouvelle version: $(microk8s version)"
EOF
```

### Lab de test/staging

Pour un environnement de test, suivez une approche plus structurée :

```bash
# Script de mise à jour pour staging
cat <<'EOF' > update-staging.sh
#!/bin/bash

# 1. Notification
echo "=== Mise à jour planifiée de l'environnement de staging ==="
echo "Début: $(date)"

# 2. Sauvegarde légère
BACKUP_DIR="/tmp/staging-backup-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR
microk8s kubectl get all --all-namespaces -o yaml > "$BACKUP_DIR/all.yaml"

# 3. Test de smoke avant
echo "Tests pré-mise à jour..."
microk8s kubectl get nodes >/dev/null 2>&1 || exit 1
microk8s kubectl get pods --all-namespaces >/dev/null 2>&1 || exit 1

# 4. Mise à jour
sudo snap refresh microk8s --channel=latest/candidate

# 5. Attente
sleep 30
microk8s status --wait-ready --timeout 300

# 6. Tests de smoke après
echo "Tests post-mise à jour..."
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces | grep -c Running

echo "Fin: $(date)"
EOF
```

### Production personnelle

Pour des services personnels importants :

```bash
# Script de mise à jour production avec validation
cat <<'EOF' > update-production.sh
#!/bin/bash

set -e

# Configuration
CHANNEL="1.29/stable"  # Toujours spécifier une version exacte
BACKUP_RETENTION_DAYS=30
BACKUP_BASE_DIR="$HOME/microk8s-backups"

# Fonctions
backup_cluster() {
  local backup_dir="$BACKUP_BASE_DIR/$(date +%Y%m%d-%H%M%S)"
  mkdir -p "$backup_dir"

  echo "Création de la sauvegarde complète..."

  # Sauvegarde de tous les objets Kubernetes
  for ns in $(microk8s kubectl get ns -o name | cut -d/ -f2); do
    mkdir -p "$backup_dir/$ns"
    for resource in $(microk8s kubectl api-resources --namespaced -o name); do
      microk8s kubectl get "$resource" -n "$ns" -o yaml > \
        "$backup_dir/$ns/${resource}.yaml" 2>/dev/null || true
    done
  done

  # Sauvegarde des objets cluster-wide
  for resource in $(microk8s kubectl api-resources --namespaced=false -o name); do
    microk8s kubectl get "$resource" -o yaml > \
      "$backup_dir/cluster-${resource}.yaml" 2>/dev/null || true
  done

  echo "Sauvegarde créée: $backup_dir"

  # Nettoyer les vieilles sauvegardes
  find "$BACKUP_BASE_DIR" -maxdepth 1 -type d -mtime +$BACKUP_RETENTION_DAYS -exec rm -rf {} \;
}

health_check() {
  echo "Vérification de santé..."

  # Nodes ready
  [ $(microk8s kubectl get nodes -o json | jq '.items[].status.conditions[] | select(.type=="Ready") | .status' | grep -c True) -gt 0 ]

  # Pas de pods en erreur critique
  [ $(microk8s kubectl get pods --all-namespaces | grep -c CrashLoopBackOff) -eq 0 ]

  # Services essentiels running
  microk8s kubectl get pods -n kube-system | grep -q Running

  echo "✅ Santé OK"
}

# Processus principal
echo "=== Mise à jour Production MicroK8s vers $CHANNEL ==="

# 1. Vérification initiale
health_check

# 2. Sauvegarde
backup_cluster

# 3. Mise à jour
echo "Mise à jour vers $CHANNEL..."
sudo snap refresh microk8s --channel=$CHANNEL

# 4. Attente de stabilisation
echo "Attente de stabilisation (2 minutes)..."
sleep 120
microk8s status --wait-ready --timeout 300

# 5. Vérification finale
health_check

# 6. Test des services critiques
echo "Test des services critiques..."
# Ajoutez vos tests spécifiques ici
# curl http://your-service.local/health

echo "=== ✅ Mise à jour terminée avec succès ==="
EOF
```

## Automatisation et planification

### Mises à jour automatiques contrôlées

Configuration pour des mises à jour automatiques mais contrôlées :

```bash
# Créer un script de mise à jour automatique
cat <<'EOF' > /usr/local/bin/auto-update-microk8s.sh
#!/bin/bash

LOG_FILE="/var/log/microk8s-updates.log"
NOTIFY_EMAIL="admin@example.com"

log_message() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

send_notification() {
  local subject="$1"
  local body="$2"
  # echo "$body" | mail -s "$subject" "$NOTIFY_EMAIL"
  echo "$subject: $body" >> "$LOG_FILE"
}

# Vérifier s'il y a des mises à jour
updates_available=$(snap refresh --list 2>/dev/null | grep microk8s)

if [ -z "$updates_available" ]; then
  log_message "Pas de mise à jour disponible"
  exit 0
fi

log_message "Mise à jour disponible: $updates_available"

# Sauvegarde automatique
BACKUP_DIR="/var/backups/microk8s/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"
microk8s kubectl get all --all-namespaces -o yaml > "$BACKUP_DIR/all-resources.yaml"

# Mise à jour
log_message "Début de la mise à jour"
if sudo snap refresh microk8s 2>&1 | tee -a "$LOG_FILE"; then
  log_message "Mise à jour réussie"
  send_notification "MicroK8s Updated" "Successfully updated to $(microk8s version)"
else
  log_message "Échec de la mise à jour"
  send_notification "MicroK8s Update Failed" "Check logs at $LOG_FILE"
  exit 1
fi

# Vérification post-mise à jour
sleep 30
if microk8s status --wait-ready --timeout 300; then
  log_message "Cluster opérationnel"
else
  log_message "Problème avec le cluster après mise à jour"
  send_notification "MicroK8s Post-Update Issue" "Cluster not ready after update"
fi
EOF

chmod +x /usr/local/bin/auto-update-microk8s.sh

# Créer une tâche cron pour les mises à jour hebdomadaires
(crontab -l 2>/dev/null; echo "0 3 * * 0 /usr/local/bin/auto-update-microk8s.sh") | crontab -
```

### Monitoring des versions

Script pour surveiller les versions disponibles :

```bash
#!/bin/bash
# monitor-versions.sh

echo "=== Monitoring des versions MicroK8s ==="

# Version actuelle
current=$(snap info microk8s | grep installed | awk '{print $2}')
echo "Version actuelle: $current"

# Versions disponibles
echo -e "\nVersions disponibles:"
snap info microk8s | grep -E "^\s+1\." | head -10

# Vérifier les CVE connues
echo -e "\nVérification de sécurité:"
current_k8s=$(microk8s kubectl version --short | grep Server | awk '{print $3}')
echo "Kubernetes version: $current_k8s"

# Comparer avec la dernière version stable
latest_stable=$(snap info microk8s | grep "latest/stable:" | awk '{print $2}')
if [ "$current" != "$latest_stable" ]; then
  echo "⚠️  Une version plus récente est disponible: $latest_stable"
else
  echo "✅ Vous utilisez la dernière version stable"
fi

# Vérifier l'âge de la version
install_date=$(snap info microk8s | grep installed | grep -oP '\d{4}-\d{2}-\d{2}')
if [ ! -z "$install_date" ]; then
  days_old=$(( ($(date +%s) - $(date -d "$install_date" +%s)) / 86400 ))
  echo "Version installée depuis: $days_old jours"

  if [ $days_old -gt 90 ]; then
    echo "⚠️  Considérez une mise à jour (>90 jours)"
  fi
fi
```

## Bonnes pratiques et recommandations

### Checklist pré-mise à jour

Avant chaque mise à jour importante :

1. **Lire les release notes** de la nouvelle version
2. **Vérifier la compatibilité** des APIs utilisées
3. **Effectuer une sauvegarde complète**
4. **Tester dans un environnement similaire** si possible
5. **Planifier une fenêtre de maintenance**
6. **Préparer un plan de rollback**
7. **Documenter la procédure**

### Template de documentation de mise à jour

```markdown
# Mise à jour MicroK8s - [DATE]

## Informations de version
- Version source:
- Version cible:
- Canal utilisé:
- Raison de la mise à jour:

## Pré-requis vérifiés
- [ ] Release notes lues
- [ ] Compatibilité API vérifiée
- [ ] Sauvegarde effectuée
- [ ] Test en environnement de dev

## Procédure
1. Heure de début:
2. Commandes exécutées:
   ```
   [commandes]
   ```
3. Problèmes rencontrés:
4. Solutions appliquées:
5. Heure de fin:

## Vérifications post-mise à jour
- [ ] Cluster opérationnel
- [ ] Tous les pods running
- [ ] Services accessibles
- [ ] Performances normales

## Notes et observations
[Notes additionnelles]
```

### Configuration de maintenance

Gérer les fenêtres de maintenance :

```bash
# Désactiver temporairement les mises à jour automatiques
sudo snap refresh --hold=48h microk8s

# Réactiver les mises à jour automatiques
sudo snap refresh --unhold microk8s

# Définir une heure spécifique pour les mises à jour
sudo snap set system refresh.timer=sun,02:00-04:00

# Vérifier la configuration
snap refresh --time
```

## Conclusion

La mise à jour de MicroK8s est une opération critique qui nécessite planification et précaution. Bien que MicroK8s simplifie grandement le processus grâce à Snap, il reste important de suivre une méthodologie rigoureuse, surtout pour les environnements contenant des données ou services importants. Les clés du succès sont : toujours sauvegarder avant, tester dans un environnement non-critique si possible, et avoir un plan de rollback prêt. Avec les outils et scripts présentés dans cette section, vous disposez de tout le nécessaire pour maintenir votre cluster à jour en toute sécurité.

N'oubliez pas que rester à jour n'est pas seulement une question de nouvelles fonctionnalités - c'est aussi et surtout une question de sécurité. Les vulnérabilités découvertes dans Kubernetes sont régulièrement corrigées, et retarder les mises à jour expose votre cluster à des risques inutiles. Trouvez le bon équilibre entre stabilité et actualité, établissez une routine de mise à jour régulière, et votre cluster MicroK8s restera performant et sécurisé pour longtemps.

⏭️
