🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4 Export/Import de configurations (Suite)

## Migration entre environnements

### Script de migration complet

```bash
#!/bin/bash
# migrate-environment.sh

SOURCE_CONTEXT=${1:-microk8s}
TARGET_CONTEXT=${2:-production}
NAMESPACE=${3:-default}
MIGRATION_DIR="./migration-$(date +%Y%m%d-%H%M%S)"

# Configuration
EXCLUDE_RESOURCES="events,pods,replicasets"
EXCLUDE_NAMESPACES="kube-system,kube-public,kube-node-lease"

# Couleurs
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
BLUE='\033[0;34m'
NC='\033[0m'

# Fonction pour afficher l'état
print_status() {
    local status=$1
    local message=$2
    case $status in
        "INFO") echo -e "${BLUE}ℹ${NC} $message" ;;
        "SUCCESS") echo -e "${GREEN}✓${NC} $message" ;;
        "WARNING") echo -e "${YELLOW}⚠${NC} $message" ;;
        "ERROR") echo -e "${RED}✗${NC} $message" ;;
    esac
}

# Vérifier les prérequis
check_prerequisites() {
    print_status "INFO" "Vérification des prérequis..."

    # Vérifier kubectl
    if ! command -v kubectl &> /dev/null; then
        print_status "ERROR" "kubectl n'est pas installé"
        exit 1
    fi

    # Vérifier yq
    if ! command -v yq &> /dev/null; then
        print_status "WARNING" "yq n'est pas installé. Installation..."
        sudo snap install yq
    fi

    # Vérifier les contextes
    if ! kubectl config get-contexts "$SOURCE_CONTEXT" &> /dev/null; then
        print_status "ERROR" "Contexte source '$SOURCE_CONTEXT' introuvable"
        exit 1
    fi

    if ! kubectl config get-contexts "$TARGET_CONTEXT" &> /dev/null; then
        print_status "ERROR" "Contexte cible '$TARGET_CONTEXT' introuvable"
        exit 1
    fi

    print_status "SUCCESS" "Prérequis validés"
}

# Analyser l'environnement source
analyze_source() {
    print_status "INFO" "Analyse de l'environnement source..."

    kubectl config use-context "$SOURCE_CONTEXT"

    # Créer le rapport d'analyse
    mkdir -p "$MIGRATION_DIR/analysis"

    # Lister toutes les ressources
    kubectl api-resources --verbs=list --namespaced -o name | \
    while read -r resource; do
        count=$(kubectl get "$resource" -n "$NAMESPACE" 2>/dev/null | wc -l)
        if [ "$count" -gt 1 ]; then
            echo "$resource: $((count-1))" >> "$MIGRATION_DIR/analysis/resources.txt"
        fi
    done

    # Analyser les dépendances
    print_status "INFO" "Analyse des dépendances..."

    # Services externes
    kubectl get services -n "$NAMESPACE" -o json | \
    jq -r '.items[].spec.externalName // empty' > "$MIGRATION_DIR/analysis/external-services.txt"

    # ConfigMaps et Secrets référencés
    kubectl get deployments,statefulsets,daemonsets -n "$NAMESPACE" -o json | \
    jq -r '.. | .configMapRef?.name // empty, .secretRef?.name // empty' | \
    sort -u > "$MIGRATION_DIR/analysis/references.txt"

    print_status "SUCCESS" "Analyse terminée"
}

# Exporter les ressources
export_resources() {
    print_status "INFO" "Export des ressources depuis $SOURCE_CONTEXT..."

    kubectl config use-context "$SOURCE_CONTEXT"

    mkdir -p "$MIGRATION_DIR/exports"

    # Liste des types de ressources à exporter dans l'ordre
    local resource_types=(
        "namespace"
        "serviceaccount"
        "role"
        "rolebinding"
        "configmap"
        "secret"
        "persistentvolumeclaim"
        "service"
        "deployment"
        "statefulset"
        "daemonset"
        "job"
        "cronjob"
        "ingress"
        "networkpolicy"
    )

    for resource_type in "${resource_types[@]}"; do
        print_status "INFO" "Export des $resource_type..."

        # Obtenir la liste des ressources
        resources=$(kubectl get "$resource_type" -n "$NAMESPACE" -o name 2>/dev/null)

        if [ -n "$resources" ]; then
            while IFS= read -r resource; do
                resource_name=$(echo "$resource" | cut -d'/' -f2)
                output_file="$MIGRATION_DIR/exports/${resource_type}-${resource_name}.yaml"

                # Export avec nettoyage
                kubectl get "$resource" -n "$NAMESPACE" -o yaml | \
                yq eval '
                    del(.metadata.uid) |
                    del(.metadata.resourceVersion) |
                    del(.metadata.generation) |
                    del(.metadata.creationTimestamp) |
                    del(.metadata.selfLink) |
                    del(.metadata.managedFields) |
                    del(.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration") |
                    del(.status)
                ' - > "$output_file"

                echo "  ✓ $resource_name"
            done <<< "$resources"
        fi
    done

    print_status "SUCCESS" "Export terminé"
}

# Transformer les ressources pour le nouvel environnement
transform_resources() {
    print_status "INFO" "Transformation des ressources pour $TARGET_CONTEXT..."

    mkdir -p "$MIGRATION_DIR/transformed"

    # Fichier de configuration de transformation
    cat > "$MIGRATION_DIR/transform-config.yaml" <<EOF
transformations:
  # Changement de namespace si nécessaire
  namespace:
    from: "$NAMESPACE"
    to: "$NAMESPACE"

  # Mise à jour des images
  images:
    registry:
      from: "docker.io"
      to: "registry.company.com"

  # Mise à jour des StorageClass
  storageClass:
    from: "microk8s-hostpath"
    to: "fast-ssd"

  # Mise à jour des domaines Ingress
  ingress:
    domain:
      from: ".local"
      to: ".company.com"

  # Ajustement des ressources
  resources:
    multiply_factor: 2  # Doubler les ressources en production
EOF

    # Appliquer les transformations
    for file in "$MIGRATION_DIR/exports"/*.yaml; do
        [ -f "$file" ] || continue

        filename=$(basename "$file")
        output_file="$MIGRATION_DIR/transformed/$filename"

        # Appliquer les transformations avec yq
        cp "$file" "$output_file"

        # Transformer le namespace
        yq eval -i '.metadata.namespace = "'$NAMESPACE'"' "$output_file"

        # Transformer les images (si c'est un deployment/statefulset/daemonset)
        if [[ "$filename" =~ ^(deployment|statefulset|daemonset) ]]; then
            yq eval -i '.spec.template.spec.containers[].image |= sub("docker.io", "registry.company.com")' "$output_file"

            # Doubler les ressources
            yq eval -i '.spec.template.spec.containers[].resources.requests.memory |= (. | split("Mi")[0] | tonumber * 2 | tostring + "Mi")' "$output_file" 2>/dev/null
            yq eval -i '.spec.template.spec.containers[].resources.limits.memory |= (. | split("Mi")[0] | tonumber * 2 | tostring + "Mi")' "$output_file" 2>/dev/null
        fi

        # Transformer les StorageClass (si c'est un PVC)
        if [[ "$filename" =~ ^persistentvolumeclaim ]]; then
            yq eval -i '.spec.storageClassName = "fast-ssd"' "$output_file"
        fi

        # Transformer les domaines (si c'est un Ingress)
        if [[ "$filename" =~ ^ingress ]]; then
            yq eval -i '.spec.rules[].host |= sub(".local", ".company.com")' "$output_file"
        fi
    done

    print_status "SUCCESS" "Transformation terminée"
}

# Valider les ressources transformées
validate_migration() {
    print_status "INFO" "Validation des ressources pour $TARGET_CONTEXT..."

    kubectl config use-context "$TARGET_CONTEXT"

    local validation_errors=0
    mkdir -p "$MIGRATION_DIR/validation"

    # Dry-run sur chaque fichier
    for file in "$MIGRATION_DIR/transformed"/*.yaml; do
        [ -f "$file" ] || continue

        filename=$(basename "$file")

        if kubectl apply -f "$file" --dry-run=server &> "$MIGRATION_DIR/validation/${filename}.log"; then
            echo "  ✓ $filename"
        else
            print_status "ERROR" "$filename - Voir $MIGRATION_DIR/validation/${filename}.log"
            ((validation_errors++))
        fi
    done

    if [ $validation_errors -eq 0 ]; then
        print_status "SUCCESS" "Toutes les ressources sont valides"
        return 0
    else
        print_status "ERROR" "$validation_errors erreurs de validation détectées"
        return 1
    fi
}

# Créer un plan de migration
create_migration_plan() {
    print_status "INFO" "Création du plan de migration..."

    cat > "$MIGRATION_DIR/migration-plan.md" <<EOF
# Plan de Migration

## Informations
- **Date:** $(date '+%Y-%m-%d %H:%M:%S')
- **Source:** $SOURCE_CONTEXT / $NAMESPACE
- **Cible:** $TARGET_CONTEXT / $NAMESPACE
- **Opérateur:** $(whoami)

## Étapes de migration

### 1. Pré-migration (T-1 jour)
- [ ] Notification aux utilisateurs
- [ ] Backup complet de l'environnement source
- [ ] Vérification des ressources cibles disponibles
- [ ] Test de connectivité réseau

### 2. Préparation (T-2 heures)
- [ ] Mise en mode maintenance de l'application source
- [ ] Export final des données
- [ ] Synchronisation des volumes si nécessaire

### 3. Migration (T0)
\`\`\`bash
# Appliquer les ressources de base
kubectl apply -f transformed/namespace*.yaml
kubectl apply -f transformed/serviceaccount*.yaml
kubectl apply -f transformed/role*.yaml
kubectl apply -f transformed/rolebinding*.yaml

# Appliquer les configurations
kubectl apply -f transformed/configmap*.yaml
kubectl apply -f transformed/secret*.yaml

# Appliquer le stockage
kubectl apply -f transformed/persistentvolumeclaim*.yaml

# Attendre que les PVCs soient bound
kubectl wait --for=condition=Bound pvc --all -n $NAMESPACE --timeout=300s

# Appliquer les services
kubectl apply -f transformed/service*.yaml

# Appliquer les workloads
kubectl apply -f transformed/deployment*.yaml
kubectl apply -f transformed/statefulset*.yaml
kubectl apply -f transformed/daemonset*.yaml

# Attendre que les pods soient ready
kubectl wait --for=condition=ready pods --all -n $NAMESPACE --timeout=600s

# Appliquer le networking
kubectl apply -f transformed/ingress*.yaml
kubectl apply -f transformed/networkpolicy*.yaml
\`\`\`

### 4. Validation post-migration
- [ ] Vérifier que tous les pods sont Running
- [ ] Tester les endpoints de santé
- [ ] Vérifier les logs pour les erreurs
- [ ] Tester les fonctionnalités critiques

### 5. Rollback (si nécessaire)
\`\`\`bash
# Supprimer les ressources migrées
kubectl delete -f transformed/ --ignore-not-found=true

# Réactiver l'environnement source
kubectl config use-context $SOURCE_CONTEXT
kubectl scale deployment --all --replicas=1 -n $NAMESPACE
\`\`\`

## Ressources migrées
$(ls -1 "$MIGRATION_DIR/transformed/" | wc -l) fichiers

## Contacts
- Responsable technique: [À compléter]
- Responsable application: [À compléter]
- Support: [À compléter]
EOF

    print_status "SUCCESS" "Plan de migration créé: $MIGRATION_DIR/migration-plan.md"
}

# Exécuter la migration
execute_migration() {
    print_status "INFO" "Exécution de la migration..."

    kubectl config use-context "$TARGET_CONTEXT"

    # Créer le namespace si nécessaire
    kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

    # Ordre d'application important
    local order=(
        "serviceaccount"
        "role"
        "rolebinding"
        "configmap"
        "secret"
        "persistentvolumeclaim"
        "service"
        "deployment"
        "statefulset"
        "daemonset"
        "job"
        "cronjob"
        "ingress"
        "networkpolicy"
    )

    for resource_type in "${order[@]}"; do
        files=$(ls "$MIGRATION_DIR/transformed/${resource_type}"*.yaml 2>/dev/null)

        if [ -n "$files" ]; then
            print_status "INFO" "Application des $resource_type..."

            for file in $files; do
                if kubectl apply -f "$file"; then
                    echo "  ✓ $(basename "$file")"
                else
                    print_status "ERROR" "Échec pour $(basename "$file")"
                    return 1
                fi

                # Pause pour éviter de surcharger l'API
                sleep 1
            done
        fi
    done

    print_status "SUCCESS" "Migration exécutée"
}

# Vérification post-migration
post_migration_check() {
    print_status "INFO" "Vérification post-migration..."

    kubectl config use-context "$TARGET_CONTEXT"

    # Vérifier les pods
    print_status "INFO" "État des pods:"
    kubectl get pods -n "$NAMESPACE"

    # Vérifier les services
    print_status "INFO" "État des services:"
    kubectl get services -n "$NAMESPACE"

    # Vérifier les ingress
    print_status "INFO" "État des ingress:"
    kubectl get ingress -n "$NAMESPACE"

    # Collecter les métriques
    cat > "$MIGRATION_DIR/post-migration-status.txt" <<EOF
=== Post-Migration Status ===
Date: $(date '+%Y-%m-%d %H:%M:%S')

Pods:
$(kubectl get pods -n "$NAMESPACE" --no-headers | awk '{print "  - " $1 ": " $3}')

Services:
$(kubectl get services -n "$NAMESPACE" --no-headers | awk '{print "  - " $1 ": " $2}')

Ingress:
$(kubectl get ingress -n "$NAMESPACE" --no-headers | awk '{print "  - " $1 ": " $3}')

Events (last 10):
$(kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | tail -10)
EOF

    print_status "SUCCESS" "Vérification terminée - Voir $MIGRATION_DIR/post-migration-status.txt"
}

# Menu principal
main() {
    echo "╔══════════════════════════════════════════════════════════╗"
    echo "║           OUTIL DE MIGRATION KUBERNETES                   ║"
    echo "╚══════════════════════════════════════════════════════════╝"
    echo ""
    echo "Configuration:"
    echo "  Source: $SOURCE_CONTEXT / $NAMESPACE"
    echo "  Cible: $TARGET_CONTEXT / $NAMESPACE"
    echo ""

    check_prerequisites

    # Créer le répertoire de migration
    mkdir -p "$MIGRATION_DIR"

    # Menu interactif
    PS3="Choisir une action: "
    options=(
        "Migration complète automatique"
        "Export seulement"
        "Analyse et planification"
        "Validation seulement"
        "Exécution manuelle"
        "Quitter"
    )

    select opt in "${options[@]}"; do
        case $REPLY in
            1)
                analyze_source
                export_resources
                transform_resources
                if validate_migration; then
                    create_migration_plan
                    read -p "Exécuter la migration maintenant? (y/N) " -n 1 -r
                    echo
                    if [[ $REPLY =~ ^[Yy]$ ]]; then
                        execute_migration
                        post_migration_check
                    fi
                fi
                break
                ;;
            2)
                export_resources
                break
                ;;
            3)
                analyze_source
                create_migration_plan
                break
                ;;
            4)
                transform_resources
                validate_migration
                break
                ;;
            5)
                execute_migration
                post_migration_check
                break
                ;;
            6)
                echo "Migration annulée"
                exit 0
                ;;
            *)
                echo "Option invalide"
                ;;
        esac
    done

    echo ""
    echo "📁 Tous les fichiers sont dans: $MIGRATION_DIR"
}

# Exécution
main
```

## Outils et utilitaires

### Comparaison de configurations

```bash
#!/bin/bash
# diff-configs.sh

# Comparer deux environnements
compare_environments() {
    local env1=$1
    local env2=$2
    local namespace=${3:-default}

    echo "📊 Comparaison des environnements"
    echo "================================"

    # Créer des exports temporaires
    local temp_dir=$(mktemp -d)

    # Export env1
    kubectl config use-context "$env1"
    kubectl get all -n "$namespace" -o yaml > "$temp_dir/env1.yaml"

    # Export env2
    kubectl config use-context "$env2"
    kubectl get all -n "$namespace" -o yaml > "$temp_dir/env2.yaml"

    # Nettoyer les métadonnées pour une comparaison propre
    for file in "$temp_dir"/*.yaml; do
        yq eval -i '
            del(.items[].metadata.uid) |
            del(.items[].metadata.resourceVersion) |
            del(.items[].metadata.generation) |
            del(.items[].metadata.creationTimestamp) |
            del(.items[].metadata.selfLink) |
            del(.items[].metadata.managedFields) |
            del(.items[].status)
        ' "$file"
    done

    # Comparaison
    diff -u "$temp_dir/env1.yaml" "$temp_dir/env2.yaml" | \
    grep -E "^(\+|\-)" | grep -v "^(\+\+\+|\-\-\-)" | \
    while IFS= read -r line; do
        if [[ $line == +* ]]; then
            echo -e "${GREEN}$line${NC}"
        elif [[ $line == -* ]]; then
            echo -e "${RED}$line${NC}"
        fi
    done

    # Rapport de synthèse
    echo ""
    echo "📈 Résumé des différences:"
    echo "  Lignes ajoutées: $(diff "$temp_dir/env1.yaml" "$temp_dir/env2.yaml" | grep "^>" | wc -l)"
    echo "  Lignes supprimées: $(diff "$temp_dir/env1.yaml" "$temp_dir/env2.yaml" | grep "^<" | wc -l)"

    # Nettoyer
    rm -rf "$temp_dir"
}

# Vérifier la dérive de configuration
check_drift() {
    local baseline_file=$1
    local namespace=${2:-default}

    echo "🔍 Vérification de la dérive de configuration"

    # Export actuel
    local current_file="/tmp/current-config.yaml"
    kubectl get all -n "$namespace" -o yaml > "$current_file"

    # Nettoyer pour comparaison
    yq eval -i '
        del(.items[].metadata.uid) |
        del(.items[].metadata.resourceVersion) |
        del(.items[].metadata.generation) |
        del(.items[].metadata.creationTimestamp) |
        del(.items[].metadata.selfLink) |
        del(.items[].metadata.managedFields) |
        del(.items[].status)
    ' "$current_file"

    # Identifier les dérives
    local drift_found=false

    diff -q "$baseline_file" "$current_file" > /dev/null 2>&1
    if [ $? -ne 0 ]; then
        drift_found=true
        echo "⚠️  Dérive détectée!"
        echo ""
        echo "Changements:"
        diff -u "$baseline_file" "$current_file" | head -50
    else
        echo "✅ Aucune dérive détectée"
    fi

    # Générer un rapport
    if [ "$drift_found" = true ]; then
        local report_file="drift-report-$(date +%Y%m%d-%H%M%S).html"
        generate_drift_report "$baseline_file" "$current_file" "$report_file"
        echo ""
        echo "📄 Rapport détaillé: $report_file"
    fi
}

# Générer un rapport HTML de dérive
generate_drift_report() {
    local baseline=$1
    local current=$2
    local output=$3

    cat > "$output" <<'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>Rapport de dérive de configuration</title>
    <style>
        body { font-family: monospace; margin: 20px; }
        .added { background-color: #d4f4dd; }
        .removed { background-color: #f4d4d4; }
        .header { background-color: #f0f0f0; padding: 10px; }
        pre { overflow-x: auto; }
    </style>
</head>
<body>
    <h1>Rapport de dérive de configuration</h1>
    <div class="header">
        <p>Date: $(date '+%Y-%m-%d %H:%M:%S')</p>
        <p>Baseline: $baseline</p>
        <p>Current: $current</p>
    </div>
    <h2>Différences détectées</h2>
    <pre>
HTML

    diff -u "$baseline" "$current" | \
    sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g' | \
    sed 's/^+.*/<span class="added">&<\/span>/' | \
    sed 's/^-.*/<span class="removed">&<\/span>/' >> "$output"

    cat >> "$output" <<'HTML'
    </pre>
</body>
</html>
HTML
}
```

### Backup et restore automatisés

```bash
#!/bin/bash
# auto-backup-restore.sh

# Configuration
BACKUP_DIR="/backup/k8s-configs"
RETENTION_DAYS=30

# Backup automatique avec rotation
automated_backup() {
    local namespace=${1:-all}
    local timestamp=$(date +%Y%m%d-%H%M%S)
    local backup_path="$BACKUP_DIR/$timestamp"

    echo "🔄 Backup automatique démarré: $timestamp"

    mkdir -p "$backup_path"

    if [ "$namespace" = "all" ]; then
        # Backup de tous les namespaces
        for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
            # Ignorer les namespaces système
            if [[ "$ns" =~ ^(kube-|openshift-|default$) ]]; then
                continue
            fi

            echo "  📦 Backup du namespace: $ns"
            mkdir -p "$backup_path/$ns"

            # Export toutes les ressources
            kubectl get all,cm,secret,pvc,ingress -n "$ns" -o yaml > "$backup_path/$ns/all-resources.yaml"
        done
    else
        # Backup d'un namespace spécifique
        echo "  📦 Backup du namespace: $namespace"
        mkdir -p "$backup_path/$namespace"
        kubectl get all,cm,secret,pvc,ingress -n "$namespace" -o yaml > "$backup_path/$namespace/all-resources.yaml"
    fi

    # Créer une archive compressée
    tar czf "$backup_path.tar.gz" -C "$BACKUP_DIR" "$timestamp"
    rm -rf "$backup_path"

    # Rotation des backups
    echo "  🔄 Rotation des anciens backups..."
    find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

    echo "✅ Backup terminé: $backup_path.tar.gz"

    # Créer un lien vers le dernier backup
    ln -sf "$timestamp.tar.gz" "$BACKUP_DIR/latest.tar.gz"
}

# Restore automatique
automated_restore() {
    local backup_file=${1:-"$BACKUP_DIR/latest.tar.gz"}
    local target_namespace=$2

    if [ ! -f "$backup_file" ]; then
        echo "❌ Fichier de backup introuvable: $backup_file"
        return 1
    fi

    echo "♻️  Restauration depuis: $backup_file"

    # Extraire le backup
    local temp_dir=$(mktemp -d)
    tar xzf "$backup_file" -C "$temp_dir"

    # Trouver le répertoire extrait
    local backup_dir=$(ls -d "$temp_dir"/*/ | head -1)

    if [ -z "$target_namespace" ]; then
        # Restaurer tous les namespaces
        for ns_dir in "$backup_dir"*/; do
            local ns=$(basename "$ns_dir")
            echo "  ♻️  Restauration du namespace: $ns"

            # Créer le namespace s'il n'existe pas
            kubectl create namespace "$ns" --dry-run=client -o yaml | kubectl apply -f -

            # Restaurer les ressources
            kubectl apply -f "$ns_dir/all-resources.yaml"
        done
    else
        # Restaurer un namespace spécifique
        if [ -d "$backup_dir/$target_namespace" ]; then
            echo "  ♻️  Restauration du namespace: $target_namespace"
            kubectl create namespace "$target_namespace" --dry-run=client -o yaml | kubectl apply -f -
            kubectl apply -f "$backup_dir/$target_namespace/all-resources.yaml"
        else
            echo "❌ Namespace non trouvé dans le backup: $target_namespace"
            return 1
        fi
    fi

    # Nettoyer
    rm -rf "$temp_dir"

    echo "✅ Restauration terminée"
}

# Configuration de cron pour backup automatique
setup_cron_backup() {
    local schedule=${1:-"0 2 * * *"}  # Par défaut : 2h du matin

    # Créer le script cron
    cat > /etc/cron.d/k8s-backup <<EOF
# Backup automatique Kubernetes
$schedule root /usr/local/bin/auto-backup-restore.sh backup all >> /var/log/k8s-backup.log 2>&1
EOF

    echo "✅ Cron configuré pour backup automatique: $schedule"
}
```

## Bonnes pratiques

### Organisation des exports

```
k8s-configs/
├── environments/
│   ├── development/
│   │   ├── base/
│   │   ├── overlays/
│   │   └── secrets/
│   ├── staging/
│   │   ├── base/
│   │   ├── overlays/
│   │   └── secrets/
│   └── production/
│       ├── base/
│       ├── overlays/
│       └── secrets/
├── templates/
│   ├── apps/
│   ├── databases/
│   └── monitoring/
├── scripts/
│   ├── export/
│   ├── import/
│   └── migrate/
└── docs/
    ├── README.md
    ├── MIGRATION.md
    └── DISASTER-RECOVERY.md
```

### Checklist de migration

Avant toute migration importante :

- [ ] **Préparation**
  - [ ] Inventaire complet des ressources
  - [ ] Identification des dépendances externes
  - [ ] Vérification des quotas sur la cible
  - [ ] Plan de communication aux utilisateurs

- [ ] **Export**
  - [ ] Export de toutes les ressources
  - [ ] Nettoyage des métadonnées
  - [ ] Validation de la syntaxe YAML
  - [ ] Versioning dans Git

- [ ] **Transformation**
  - [ ] Mise à jour des références d'images
  - [ ] Adaptation des StorageClass
  - [ ] Modification des domaines Ingress
  - [ ] Ajustement des ressources (CPU/RAM)

- [ ] **Validation**
  - [ ] Dry-run sur l'environnement cible
  - [ ] Vérification des dépendances
  - [ ] Tests de connectivité réseau
  - [ ] Validation des certificats

- [ ] **Migration**
  - [ ] Application dans le bon ordre
  - [ ] Monitoring pendant la migration
  - [ ] Tests de santé après chaque étape
  - [ ] Documentation des problèmes

- [ ] **Post-migration**
  - [ ] Vérification fonctionnelle complète
  - [ ] Tests de performance
  - [ ] Mise à jour de la documentation
  - [ ] Communication de fin de migration

### Sécurité des exports

```bash
#!/bin/bash
# secure-export-practices.sh

# 1. Ne jamais commiter de secrets en clair
check_secrets_before_commit() {
    local files_to_check=$1

    echo "🔍 Vérification des secrets avant commit..."

    # Patterns à détecter
    local patterns=(
        "password:"
        "apikey:"
        "token:"
        "secret:"
        "private_key:"
        "aws_access_key"
        "-----BEGIN"
    )

    local found_secrets=false

    for pattern in "${patterns[@]}"; do
        if grep -r "$pattern" "$files_to_check" > /dev/null 2>&1; then
            echo "⚠️  Pattern suspect trouvé: $pattern"
            found_secrets=true
        fi
    done

    if [ "$found_secrets" = true ]; then
        echo "❌ Secrets potentiels détectés! Ne pas commiter."
        return 1
    fi

    echo "✅ Aucun secret évident détecté"
    return 0
}

# 2. Chiffrer les exports sensibles
encrypt_sensitive_exports() {
    local export_dir=$1
    local gpg_key=$2

    echo "🔐 Chiffrement des exports sensibles..."

    # Chiffrer les secrets
    find "$export_dir" -name "*secret*.yaml" -o -name "*credentials*.yaml" | \
    while read -r file; do
        gpg --encrypt --recipient "$gpg_key" --output "${file}.gpg" "$file"
        shred -u "$file"  # Suppression sécurisée
        echo "  ✓ Chiffré: $(basename "$file")"
    done
}

# 3. Utiliser .gitignore approprié
create_gitignore() {
    cat > .gitignore <<'EOF'
# Secrets et credentials
*secret*.yaml
*credentials*.yaml
*password*
*.key
*.pem
*.crt
*.pfx

# Fichiers temporaires
*.tmp
*.bak
*.swp
.DS_Store

# Répertoires sensibles
/secrets/
/credentials/
/backup/

# Sauf les fichiers chiffrés
!*.gpg
!*.enc
!*-sealed.yaml
EOF

    echo "✅ .gitignore créé avec règles de sécurité"
}

# 4. Audit trail des exports
log_export_activity() {
    local action=$1
    local user=$(whoami)
    local timestamp=$(date -Iseconds)
    local log_file="/var/log/k8s-export-audit.log"

    echo "[$timestamp] USER:$user ACTION:$action" | sudo tee -a "$log_file"
}
```

### Automatisation avec CI/CD

```yaml
# .gitlab-ci.yml - Pipeline GitLab pour export/import
stages:
  - export
  - validate
  - deploy

variables:
  KUBECONFIG: /tmp/kubeconfig
  NAMESPACE: ${CI_COMMIT_REF_NAME}

# Export automatique des configs
export-configs:
  stage: export
  script:
    - kubectl get all,cm,secret -n $NAMESPACE -o yaml > configs.yaml
    - yq eval 'del(.items[].metadata.managedFields)' -i configs.yaml
    - git add configs.yaml
    - git commit -m "Auto-export configs from $NAMESPACE"
    - git push
  only:
    - schedules
  artifacts:
    paths:
      - configs.yaml
    expire_in: 1 week

# Validation des configs
validate-configs:
  stage: validate
  script:
    - yamllint configs.yaml
    - kubectl apply -f configs.yaml --dry-run=server
  dependencies:
    - export-configs

# Import vers environnement cible
deploy-configs:
  stage: deploy
  script:
    - kubectl apply -f configs.yaml
    - kubectl rollout status deployment --all -n $NAMESPACE
  dependencies:
    - validate-configs
  when: manual
  only:
    - main
```

### Script de disaster recovery

```bash
#!/bin/bash
# disaster-recovery.sh

DR_BACKUP_DIR="/backup/disaster-recovery"
DR_REMOTE="s3://company-dr-bucket/k8s-configs"

# Création d'un point de récupération
create_recovery_point() {
    local rp_name="RP-$(date +%Y%m%d-%H%M%S)"
    local rp_dir="$DR_BACKUP_DIR/$rp_name"

    echo "🔄 Création du point de récupération: $rp_name"

    mkdir -p "$rp_dir"

    # 1. Export de l'état complet du cluster
    echo "  📦 Export de l'état du cluster..."
    kubectl get nodes -o yaml > "$rp_dir/nodes.yaml"
    kubectl get namespaces -o yaml > "$rp_dir/namespaces.yaml"
    kubectl get storageclass -o yaml > "$rp_dir/storageclasses.yaml"

    # 2. Export par namespace
    for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
        if [[ ! "$ns" =~ ^kube- ]]; then
            echo "  📦 Export du namespace: $ns"
            mkdir -p "$rp_dir/namespaces/$ns"
            kubectl get all,cm,secret,pvc,ingress -n "$ns" -o yaml > "$rp_dir/namespaces/$ns/resources.yaml"
        fi
    done

    # 3. Export etcd (si accessible)
    if command -v etcdctl &> /dev/null; then
        echo "  📦 Backup etcd..."
        ETCDCTL_API=3 etcdctl snapshot save "$rp_dir/etcd-snapshot.db"
    fi

    # 4. Créer le manifeste de récupération
    cat > "$rp_dir/recovery-manifest.yaml" <<EOF
apiVersion: v1
kind: RecoveryPoint
metadata:
  name: $rp_name
  timestamp: $(date -Iseconds)
spec:
  cluster:
    version: $(kubectl version --short | grep Server | awk '{print $3}')
    nodes: $(kubectl get nodes --no-headers | wc -l)
  namespaces: $(kubectl get ns --no-headers | wc -l)
  resources:
    deployments: $(kubectl get deploy --all-namespaces --no-headers | wc -l)
    services: $(kubectl get svc --all-namespaces --no-headers | wc -l)
    pvcs: $(kubectl get pvc --all-namespaces --no-headers | wc -l)
  backup:
    location: $rp_dir
    size: $(du -sh "$rp_dir" | cut -f1)
EOF

    # 5. Compression et upload
    echo "  📤 Upload vers stockage distant..."
    tar czf "$rp_dir.tar.gz" -C "$DR_BACKUP_DIR" "$rp_name"
    aws s3 cp "$rp_dir.tar.gz" "$DR_REMOTE/"

    echo "✅ Point de récupération créé: $rp_name"
}

# Récupération d'urgence
emergency_recovery() {
    local recovery_point=${1:-latest}

    echo "🚨 RÉCUPÉRATION D'URGENCE"
    echo "========================"

    # Télécharger le point de récupération
    if [ "$recovery_point" = "latest" ]; then
        recovery_point=$(aws s3 ls "$DR_REMOTE/" | sort | tail -1 | awk '{print $4}' | sed 's/.tar.gz//')
    fi

    echo "📥 Téléchargement du point de récupération: $recovery_point"
    aws s3 cp "$DR_REMOTE/$recovery_point.tar.gz" "/tmp/"

    # Extraire
    tar xzf "/tmp/$recovery_point.tar.gz" -C "/tmp/"

    # Ordre de restauration critique
    echo "♻️  Début de la restauration..."

    # 1. Namespaces
    kubectl apply -f "/tmp/$recovery_point/namespaces.yaml"

    # 2. StorageClasses
    kubectl apply -f "/tmp/$recovery_point/storageclasses.yaml"

    # 3. Ressources par namespace
    for ns_dir in /tmp/$recovery_point/namespaces/*/; do
        ns=$(basename "$ns_dir")
        echo "  ♻️  Restauration du namespace: $ns"
        kubectl apply -f "$ns_dir/resources.yaml"
    done

    echo "✅ Récupération d'urgence terminée"
}
```

## Troubleshooting

### Problèmes courants et solutions

```bash
#!/bin/bash
# troubleshoot-export-import.sh

# Diagnostic des problèmes d'export/import
diagnose_issues() {
    echo "🔍 Diagnostic des problèmes d'export/import"
    echo "==========================================="

    # 1. Vérifier les permissions
    echo "📋 Vérification des permissions..."
    if ! kubectl auth can-i get pods --all-namespaces > /dev/null 2>&1; then
        echo "❌ Permissions insuffisantes pour l'export"
        echo "   Solution: Vérifier votre RBAC"
        kubectl auth can-i --list
    else
        echo "✅ Permissions OK"
    fi

    # 2. Vérifier la connectivité
    echo "📋 Vérification de la connectivité API..."
    if ! kubectl cluster-info > /dev/null 2>&1; then
        echo "❌ Impossible de contacter l'API Kubernetes"
        echo "   Solution: Vérifier KUBECONFIG et la connectivité réseau"
    else
        echo "✅ Connectivité OK"
    fi

    # 3. Vérifier l'espace disque
    echo "📋 Vérification de l'espace disque..."
    available_space=$(df -h . | awk 'NR==2 {print $4}')
    echo "   Espace disponible: $available_space"

    # 4. Vérifier les outils nécessaires
    echo "📋 Vérification des outils..."
    for tool in kubectl yq jq; do
        if command -v $tool &> /dev/null; then
            echo "✅ $tool installé"
        else
            echo "❌ $tool manquant"
        fi
    done
}

# Résolution des erreurs communes
fix_common_errors() {
    local error_type=$1

    case $error_type in
        "immutable-field")
            echo "🔧 Fix: Champs immutables"
            echo "Les champs comme 'spec.clusterIP' ne peuvent pas être modifiés."
            echo "Solution: Supprimer ces champs avant l'import"
            echo ""
            echo "yq eval 'del(.spec.clusterIP)' -i service.yaml"
            ;;

        "already-exists")
            echo "🔧 Fix: Ressource déjà existante"
            echo "Solution 1: Utiliser kubectl apply au lieu de create"
            echo "Solution 2: Supprimer la ressource existante"
            echo "Solution 3: Utiliser --force pour remplacer"
            echo ""
            echo "kubectl apply -f resource.yaml --force"
            ;;

        "invalid-yaml")
            echo "🔧 Fix: YAML invalide"
            echo "Vérifier la syntaxe avec:"
            echo ""
            echo "yq eval '.' file.yaml"
            echo "yamllint file.yaml"
            ;;

        "namespace-not-found")
            echo "🔧 Fix: Namespace introuvable"
            echo "Créer le namespace avant l'import:"
            echo ""
            echo "kubectl create namespace <namespace-name>"
            ;;

        *)
            echo "Type d'erreur non reconnu: $error_type"
            echo "Types disponibles: immutable-field, already-exists, invalid-yaml, namespace-not-found"
            ;;
    esac
}

# Validation avancée des exports
validate_exports() {
    local export_dir=$1

    echo "🔍 Validation approfondie des exports"

    local errors=0
    local warnings=0

    for file in $(find "$export_dir" -name "*.yaml" -o -name "*.yml"); do
        echo "Validation de: $(basename "$file")"

        # Syntaxe YAML
        if ! yq eval '.' "$file" > /dev/null 2>&1; then
            echo "  ❌ Erreur de syntaxe YAML"
            ((errors++))
            continue
        fi

        # Vérifier les champs requis
        kind=$(yq eval '.kind' "$file")
        if [ "$kind" = "null" ]; then
            echo "  ❌ Champ 'kind' manquant"
            ((errors++))
        fi

        # Vérifier les références
        if [[ "$kind" == "Deployment" ]] || [[ "$kind" == "StatefulSet" ]]; then
            # Vérifier les références aux ConfigMaps
            configmaps=$(yq eval '.. | select(has("configMapRef")) | .configMapRef.name' "$file" 2>/dev/null)
            for cm in $configmaps; do
                if [ "$cm" != "null" ]; then
                    echo "  ℹ️  Référence ConfigMap: $cm"
                    ((warnings++))
                fi
            done

            # Vérifier les références aux Secrets
            secrets=$(yq eval '.. | select(has("secretRef")) | .secretRef.name' "$file" 2>/dev/null)
            for secret in $secrets; do
                if [ "$secret" != "null" ]; then
                    echo "  ℹ️  Référence Secret: $secret"
                    ((warnings++))
                fi
            done
        fi
    done

    echo ""
    echo "📊 Résultat de la validation:"
    echo "  Erreurs: $errors"
    echo "  Avertissements: $warnings"

    if [ $errors -gt 0 ]; then
        return 1
    fi
    return 0
}
```

## Métriques et monitoring

```yaml
# prometheus-export-import-metrics.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: export-import-metrics
  namespace: monitoring
data:
  metrics.rules: |
    groups:
    - name: export_import_metrics
      interval: 30s
      rules:
      - record: k8s_export_duration_seconds
        expr: histogram_quantile(0.95, rate(export_duration_seconds_bucket[5m]))

      - record: k8s_import_success_rate
        expr: rate(import_success_total[5m]) / rate(import_attempts_total[5m])

      - record: k8s_config_drift_detected
        expr: changes(config_hash[1h]) > 0
```

## Conclusion et checklist finale

### Checklist d'export/import réussi

- [ ] **Préparation**
  - [ ] Outils installés (kubectl, yq, jq)
  - [ ] Permissions vérifiées
  - [ ] Espace disque suffisant
  - [ ] Stratégie de backup définie

- [ ] **Export**
  - [ ] Scripts d'export testés
  - [ ] Métadonnées nettoyées
  - [ ] Secrets sécurisés
  - [ ] Validation YAML passée

- [ ] **Stockage**
  - [ ] Versionning Git configuré
  - [ ] Backup externe en place
  - [ ] Rotation configurée
  - [ ] Chiffrement activé pour les secrets

- [ ] **Import**
  - [ ] Ordre de dépendances respecté
  - [ ] Dry-run effectué
  - [ ] Monitoring actif
  - [ ] Plan de rollback prêt

- [ ] **Documentation**
  - [ ] Procédures documentées
  - [ ] Runbooks à jour
  - [ ] Contacts d'urgence listés
  - [ ] Historique des changements maintenu

L'export/import de configurations est un pilier fondamental de la gestion d'un cluster Kubernetes. Avec les outils et scripts présentés dans cette section, vous disposez d'une base solide pour gérer efficacement vos configurations MicroK8s, que ce soit pour la sauvegarde, la migration ou le disaster recovery.

⏭️
