🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5 Tests de restauration

## Introduction aux tests de restauration

Un test de restauration est comme une simulation d'incendie : vous pratiquez la procédure avant qu'une vraie urgence ne survienne. Dans le contexte de MicroK8s, cela signifie vérifier régulièrement que vos sauvegardes peuvent réellement restaurer votre système. Une sauvegarde non testée est comme une assurance dont vous n'êtes pas sûr qu'elle fonctionne - elle vous donne une fausse sécurité.

### Pourquoi tester la restauration ?

**La règle d'or :** "Une sauvegarde n'est valide que si elle a été restaurée avec succès."

**Raisons critiques pour tester :**
- **Détection précoce des problèmes** : Les sauvegardes peuvent être corrompues ou incomplètes
- **Validation des procédures** : S'assurer que la documentation est à jour et correcte
- **Formation de l'équipe** : Pratiquer réduit le stress lors d'une vraie crise
- **Mesure du temps de récupération** : Savoir combien de temps prend une restauration
- **Vérification de la complétude** : S'assurer que toutes les données nécessaires sont sauvegardées
- **Test des dépendances** : Identifier les ressources manquantes ou les configurations oubliées

### Types de tests de restauration

```
┌─────────────────────────────────────────────────────────┐
│                Types de Tests de Restauration           │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ╔═══════════════════╗    ╔═══════════════════╗       │
│  ║   Test Partiel     ║    ║   Test Complet    ║       │
│  ╠═══════════════════╣    ╠═══════════════════╣       │
│  ║ • Un namespace     ║    ║ • Cluster entier  ║       │
│  ║ • Une application  ║    ║ • Tous services   ║       │
│  ║ • Durée: Minutes   ║    ║ • Durée: Heures   ║       │
│  ║ • Risque: Faible   ║    ║ • Risque: Moyen   ║       │
│  ╚═══════════════════╝    ╚═══════════════════╝       │
│                                                          │
│  ╔═══════════════════╗    ╔═══════════════════╗       │
│  ║   Test Isolé       ║    ║   Test en Prod    ║       │
│  ╠═══════════════════╣    ╠═══════════════════╣       │
│  ║ • Env. de test     ║    ║ • Env. réel       ║       │
│  ║ • Sans impact      ║    ║ • Maintenance     ║       │
│  ║ • Validation       ║    ║ • Validation      ║       │
│  ║   technique        ║    ║   complète        ║       │
│  ╚═══════════════════╝    ╚═══════════════════╝       │
└─────────────────────────────────────────────────────────┘
```

## Environnement de test isolé

### Création d'un environnement de test

```bash
#!/bin/bash
# setup-test-environment.sh

TEST_NAMESPACE="restore-test-$(date +%Y%m%d-%H%M%S)"
TEST_CLUSTER_CONTEXT="microk8s-test"

# Créer un namespace isolé pour les tests
create_test_namespace() {
    echo "🔧 Création du namespace de test: $TEST_NAMESPACE"

    cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: $TEST_NAMESPACE
  labels:
    purpose: restore-test
    created: $(date +%Y-%m-%d)
    temporary: "true"
  annotations:
    description: "Namespace temporaire pour test de restauration"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: $TEST_NAMESPACE
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    persistentvolumeclaims: "5"
EOF

    echo "✅ Namespace de test créé avec quotas de ressources"
}

# Déployer une application de test
deploy_test_application() {
    echo "📦 Déploiement de l'application de test..."

    cat <<EOF | microk8s kubectl apply -n $TEST_NAMESPACE -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  app.properties: |
    environment=test
    version=1.0
    timestamp=$(date -Iseconds)
---
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
type: Opaque
data:
  username: $(echo -n "testuser" | base64)
  password: $(echo -n "testpass123" | base64)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/config
        - name: data
          mountPath: /usr/share/nginx/html
        env:
        - name: TEST_USER
          valueFrom:
            secretKeyRef:
              name: test-secret
              key: username
      volumes:
      - name: config
        configMap:
          name: test-config
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: test-service
spec:
  selector:
    app: test-app
  ports:
  - port: 80
    targetPort: 80
EOF

    # Attendre que les pods soient prêts
    echo "⏳ Attente du démarrage des pods..."
    microk8s kubectl wait --for=condition=ready pod -l app=test-app -n $TEST_NAMESPACE --timeout=60s

    echo "✅ Application de test déployée"
}

# Injecter des données de test
inject_test_data() {
    echo "💉 Injection de données de test..."

    # Créer des données dans les pods
    for pod in $(microk8s kubectl get pods -n $TEST_NAMESPACE -l app=test-app -o jsonpath='{.items[*].metadata.name}'); do
        echo "  Injection dans pod: $pod"

        # Créer des fichiers de test
        microk8s kubectl exec -n $TEST_NAMESPACE $pod -- sh -c "
            echo '<h1>Test Data</h1>' > /usr/share/nginx/html/index.html
            echo 'Timestamp: $(date)' >> /usr/share/nginx/html/index.html
            echo 'Pod: $pod' >> /usr/share/nginx/html/index.html
            echo 'Test file content' > /usr/share/nginx/html/test.txt
        "
    done

    # Créer des objets Kubernetes supplémentaires
    microk8s kubectl create configmap test-data \
        --from-literal=key1=value1 \
        --from-literal=key2=value2 \
        -n $TEST_NAMESPACE

    echo "✅ Données de test injectées"
}

# Créer un snapshot de l'état actuel
create_test_snapshot() {
    echo "📸 Création d'un snapshot de l'état de test..."

    local snapshot_dir="./test-snapshots/$(date +%Y%m%d-%H%M%S)"
    mkdir -p "$snapshot_dir"

    # Exporter l'état complet
    microk8s kubectl get all,cm,secret,pvc -n $TEST_NAMESPACE -o yaml > "$snapshot_dir/namespace-state.yaml"

    # Créer un hash pour validation
    sha256sum "$snapshot_dir/namespace-state.yaml" > "$snapshot_dir/checksum.sha256"

    echo "✅ Snapshot créé: $snapshot_dir"
    echo "$snapshot_dir" > last-test-snapshot.txt
}

# Configuration complète de l'environnement de test
setup_complete_test_env() {
    echo "🚀 Configuration de l'environnement de test complet"
    echo "===================================================="

    create_test_namespace
    deploy_test_application
    inject_test_data
    create_test_snapshot

    echo ""
    echo "📊 Résumé de l'environnement de test:"
    echo "  Namespace: $TEST_NAMESPACE"
    echo "  Pods: $(microk8s kubectl get pods -n $TEST_NAMESPACE --no-headers | wc -l)"
    echo "  Services: $(microk8s kubectl get svc -n $TEST_NAMESPACE --no-headers | wc -l)"
    echo "  ConfigMaps: $(microk8s kubectl get cm -n $TEST_NAMESPACE --no-headers | wc -l)"
    echo "  Secrets: $(microk8s kubectl get secret -n $TEST_NAMESPACE --no-headers | wc -l)"
    echo ""
    echo "✅ Environnement de test prêt pour les tests de restauration"
}

# Nettoyage de l'environnement de test
cleanup_test_env() {
    echo "🧹 Nettoyage de l'environnement de test..."

    if [ -z "$1" ]; then
        echo "Usage: cleanup_test_env <namespace>"
        return 1
    fi

    microk8s kubectl delete namespace "$1" --wait=false
    echo "✅ Namespace $1 marqué pour suppression"
}

# Exécution
setup_complete_test_env
```

## Tests de restauration basiques

### Test de restauration simple

```bash
#!/bin/bash
# basic-restore-test.sh

# Configuration
BACKUP_TOOL=${BACKUP_TOOL:-velero}  # velero, scripts, ou manual
TEST_TYPE=${1:-partial}  # partial, full, ou disaster

# Couleurs pour l'output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

# Fonction de logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Test de restauration partielle
test_partial_restore() {
    log "🔬 Test de restauration partielle"

    # 1. Créer un namespace de test
    local test_ns="partial-restore-test"
    microk8s kubectl create namespace $test_ns

    # 2. Déployer une application simple
    cat <<EOF | microk8s kubectl apply -n $test_ns -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

    # 3. Attendre le déploiement
    microk8s kubectl wait --for=condition=available deployment/test-deployment -n $test_ns --timeout=60s

    # 4. Faire un backup
    log "📸 Création du backup..."
    case $BACKUP_TOOL in
        velero)
            velero backup create test-backup-$(date +%s) --include-namespaces=$test_ns --wait
            ;;
        *)
            microk8s kubectl get all -n $test_ns -o yaml > /tmp/test-backup.yaml
            ;;
    esac

    # 5. Supprimer l'application
    log "🗑️ Suppression de l'application..."
    microk8s kubectl delete deployment test-deployment -n $test_ns

    # 6. Vérifier la suppression
    if microk8s kubectl get deployment test-deployment -n $test_ns 2>/dev/null; then
        echo -e "${RED}❌ Échec de la suppression${NC}"
        return 1
    fi

    # 7. Restaurer
    log "♻️ Restauration..."
    case $BACKUP_TOOL in
        velero)
            velero restore create test-restore-$(date +%s) --from-backup test-backup-* --wait
            ;;
        *)
            microk8s kubectl apply -f /tmp/test-backup.yaml
            ;;
    esac

    # 8. Vérifier la restauration
    if microk8s kubectl wait --for=condition=available deployment/test-deployment -n $test_ns --timeout=60s 2>/dev/null; then
        echo -e "${GREEN}✅ Restauration réussie!${NC}"

        # Nettoyer
        microk8s kubectl delete namespace $test_ns --wait=false
        return 0
    else
        echo -e "${RED}❌ Échec de la restauration${NC}"
        return 1
    fi
}

# Test de restauration complète
test_full_restore() {
    log "🔬 Test de restauration complète"

    # 1. Créer plusieurs namespaces avec applications
    local namespaces=("app-frontend" "app-backend" "app-database")

    for ns in "${namespaces[@]}"; do
        log "📦 Création de $ns..."
        microk8s kubectl create namespace $ns

        # Déployer une application dans chaque namespace
        cat <<EOF | microk8s kubectl apply -n $ns -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${ns}-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: $ns
  template:
    metadata:
      labels:
        app: $ns
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: ${ns}-service
spec:
  selector:
    app: $ns
  ports:
  - port: 80
EOF
    done

    # 2. Attendre que tout soit prêt
    log "⏳ Attente du déploiement complet..."
    for ns in "${namespaces[@]}"; do
        microk8s kubectl wait --for=condition=available deployment/${ns}-deployment -n $ns --timeout=60s
    done

    # 3. Créer un backup complet
    log "📸 Backup complet..."
    local backup_name="full-backup-$(date +%Y%m%d-%H%M%S)"

    case $BACKUP_TOOL in
        velero)
            velero backup create $backup_name --wait
            ;;
        *)
            for ns in "${namespaces[@]}"; do
                microk8s kubectl get all -n $ns -o yaml >> /tmp/${backup_name}.yaml
                echo "---" >> /tmp/${backup_name}.yaml
            done
            ;;
    esac

    # 4. Simuler un désastre
    log "💥 Simulation de désastre..."
    for ns in "${namespaces[@]}"; do
        microk8s kubectl delete namespace $ns --wait=false
    done

    sleep 10  # Attendre la suppression

    # 5. Restaurer
    log "♻️ Restauration complète..."
    case $BACKUP_TOOL in
        velero)
            velero restore create restore-$(date +%s) --from-backup $backup_name --wait
            ;;
        *)
            microk8s kubectl apply -f /tmp/${backup_name}.yaml
            ;;
    esac

    # 6. Validation
    log "🔍 Validation de la restauration..."
    local success=true

    for ns in "${namespaces[@]}"; do
        if ! microk8s kubectl get namespace $ns 2>/dev/null; then
            echo -e "${RED}❌ Namespace $ns non restauré${NC}"
            success=false
        elif ! microk8s kubectl get deployment ${ns}-deployment -n $ns 2>/dev/null; then
            echo -e "${RED}❌ Deployment dans $ns non restauré${NC}"
            success=false
        else
            echo -e "${GREEN}✓ $ns restauré${NC}"
        fi
    done

    # 7. Nettoyer
    for ns in "${namespaces[@]}"; do
        microk8s kubectl delete namespace $ns --wait=false
    done

    if [ "$success" = true ]; then
        echo -e "${GREEN}✅ Test de restauration complète réussi!${NC}"
        return 0
    else
        echo -e "${RED}❌ Test de restauration complète échoué${NC}"
        return 1
    fi
}

# Test de disaster recovery
test_disaster_recovery() {
    log "🔬 Test de disaster recovery"

    # Ce test simule une perte totale et une restauration
    echo "⚠️  Ce test va simuler une perte totale de données"
    echo "Continuer? (y/N)"
    read -r response

    if [[ ! "$response" =~ ^[Yy]$ ]]; then
        echo "Test annulé"
        return 0
    fi

    # Implémentation du test DR...
    # (Code similaire mais avec des scénarios plus complexes)
}

# Menu principal
main() {
    echo "╔════════════════════════════════════════════╗"
    echo "║     TESTS DE RESTAURATION MICROK8S        ║"
    echo "╚════════════════════════════════════════════╝"
    echo ""
    echo "Outil de backup: $BACKUP_TOOL"
    echo ""

    case $TEST_TYPE in
        partial)
            test_partial_restore
            ;;
        full)
            test_full_restore
            ;;
        disaster)
            test_disaster_recovery
            ;;
        *)
            echo "Type de test non reconnu: $TEST_TYPE"
            echo "Options: partial, full, disaster"
            exit 1
            ;;
    esac
}

# Exécution
main
```

## Tests automatisés et programmés

### Configuration de tests automatiques

```bash
#!/bin/bash
# automated-restore-tests.sh

# Configuration
TEST_SCHEDULE=${TEST_SCHEDULE:-"weekly"}  # daily, weekly, monthly
NOTIFICATION_EMAIL=${NOTIFICATION_EMAIL:-"admin@example.com"}
SLACK_WEBHOOK=${SLACK_WEBHOOK:-""}
TEST_LOG_DIR="/var/log/restore-tests"

# Créer la structure de logs
mkdir -p "$TEST_LOG_DIR"

# Fonction pour envoyer des notifications
send_notification() {
    local status=$1
    local message=$2
    local details=$3

    # Email
    if [ -n "$NOTIFICATION_EMAIL" ]; then
        echo "$details" | mail -s "Test de restauration: $status - $message" "$NOTIFICATION_EMAIL"
    fi

    # Slack
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"Test de restauration: $status\\n$message\\n\`\`\`$details\`\`\`\"}" \
            "$SLACK_WEBHOOK"
    fi

    # Log local
    echo "[$(date -Iseconds)] $status: $message" >> "$TEST_LOG_DIR/notifications.log"
}

# Test automatisé avec rapport
run_automated_test() {
    local test_name=$1
    local test_function=$2
    local start_time=$(date +%s)
    local log_file="$TEST_LOG_DIR/test-$(date +%Y%m%d-%H%M%S).log"

    echo "🤖 Exécution du test automatisé: $test_name" | tee -a "$log_file"
    echo "Démarré à: $(date)" | tee -a "$log_file"
    echo "==========================================" | tee -a "$log_file"

    # Exécuter le test avec capture des logs
    if $test_function >> "$log_file" 2>&1; then
        local status="SUCCESS"
        local emoji="✅"
    else
        local status="FAILURE"
        local emoji="❌"
    fi

    local end_time=$(date +%s)
    local duration=$((end_time - start_time))

    # Générer le rapport
    local report=$(cat <<EOF
$emoji Test: $test_name
Status: $status
Durée: ${duration}s
Date: $(date)
Log: $log_file

Détails:
$(tail -20 "$log_file")
EOF
    )

    echo "$report" | tee -a "$log_file"

    # Envoyer les notifications
    send_notification "$status" "$test_name" "$report"

    # Retourner le status
    [ "$status" = "SUCCESS" ]
}

# Suite de tests programmés
run_test_suite() {
    echo "🔄 Lancement de la suite de tests de restauration"
    echo "=================================================="

    local suite_log="$TEST_LOG_DIR/suite-$(date +%Y%m%d-%H%M%S).log"
    local tests_passed=0
    local tests_failed=0

    # Liste des tests à exécuter
    local tests=(
        "test_backup_validity:Validation des backups"
        "test_partial_restore:Restauration partielle"
        "test_full_restore:Restauration complète"
        "test_data_integrity:Intégrité des données"
        "test_performance_metrics:Métriques de performance"
    )

    for test_spec in "${tests[@]}"; do
        IFS=':' read -r test_function test_description <<< "$test_spec"

        echo "" | tee -a "$suite_log"
        echo "▶️ Test: $test_description" | tee -a "$suite_log"

        if run_automated_test "$test_description" "$test_function"; then
            ((tests_passed++))
            echo "✅ Réussi" | tee -a "$suite_log"
        else
            ((tests_failed++))
            echo "❌ Échoué" | tee -a "$suite_log"
        fi
    done

    # Rapport final
    local suite_report=$(cat <<EOF

╔════════════════════════════════════════╗
║        RAPPORT DE SUITE DE TESTS       ║
╚════════════════════════════════════════╝

Date: $(date)
Tests réussis: $tests_passed
Tests échoués: $tests_failed
Taux de réussite: $(echo "scale=2; $tests_passed * 100 / ($tests_passed + $tests_failed)" | bc)%

Log complet: $suite_log
EOF
    )

    echo "$suite_report" | tee -a "$suite_log"

    # Notification finale
    if [ $tests_failed -eq 0 ]; then
        send_notification "✅ SUCCESS" "Tous les tests ont réussi" "$suite_report"
    else
        send_notification "⚠️ WARNING" "$tests_failed tests ont échoué" "$suite_report"
    fi
}

# Tests spécifiques
test_backup_validity() {
    echo "🔍 Test de validité des backups"

    # Vérifier l'existence des backups
    case $BACKUP_TOOL in
        velero)
            local backup_count=$(velero backup get --output json | jq '.items | length')
            if [ "$backup_count" -eq 0 ]; then
                echo "❌ Aucun backup trouvé"
                return 1
            fi

            # Vérifier l'âge du dernier backup
            local latest_backup=$(velero backup get --output json | jq -r '.items[0].metadata.creationTimestamp')
            local backup_age=$(( ($(date +%s) - $(date -d "$latest_backup" +%s)) / 3600 ))

            if [ $backup_age -gt 24 ]; then
                echo "⚠️  Le dernier backup date de plus de 24h"
                return 1
            fi
            ;;
        *)
            # Vérification pour d'autres outils
            if [ ! -d "/backup" ] || [ -z "$(ls -A /backup)" ]; then
                echo "❌ Répertoire de backup vide"
                return 1
            fi
            ;;
    esac

    echo "✅ Backups valides trouvés"
    return 0
}

test_data_integrity() {
    echo "🔍 Test d'intégrité des données"

    # Créer des données de test avec checksum
    local test_data="test-data-$(date +%s)"
    echo "$test_data" | sha256sum > /tmp/test-checksum.txt

    # Créer un ConfigMap avec les données
    microk8s kubectl create configmap integrity-test \
        --from-literal=data="$test_data" \
        -n default

    # Backup
    microk8s kubectl get configmap integrity-test -n default -o yaml > /tmp/integrity-backup.yaml

    # Supprimer
    microk8s kubectl delete configmap integrity-test -n default

    # Restaurer
    microk8s kubectl apply -f /tmp/integrity-backup.yaml

    # Vérifier l'intégrité
    local restored_data=$(microk8s kubectl get configmap integrity-test -n default -o jsonpath='{.data.data}')
    local restored_checksum=$(echo "$restored_data" | sha256sum)
    local original_checksum=$(cat /tmp/test-checksum.txt)

    # Nettoyer
    microk8s kubectl delete configmap integrity-test -n default

    if [ "$restored_checksum" = "$original_checksum" ]; then
        echo "✅ Intégrité des données vérifiée"
        return 0
    else
        echo "❌ Corruption de données détectée"
        return 1
    fi
}

test_performance_metrics() {
    echo "📊 Test des métriques de performance"

    local start_time=$(date +%s%N)

    # Test de backup
    case $BACKUP_TOOL in
        velero)
            velero backup create perf-test-$(date +%s) --include-namespaces=default --wait
            ;;
        *)
            microk8s kubectl get all -n default -o yaml > /tmp/perf-test.yaml
            ;;
    esac

    local backup_time=$(date +%s%N)
    local backup_duration=$(( ($backup_time - $start_time) / 1000000 ))

    # Test de restauration
    case $BACKUP_TOOL in
        velero)
            velero restore create perf-restore-$(date +%s) --from-backup perf-test-* --wait
            ;;
        *)
            microk8s kubectl apply -f /tmp/perf-test.yaml --dry-run=server
            ;;
    esac

    local restore_time=$(date +%s%N)
    local restore_duration=$(( ($restore_time - $backup_time) / 1000000 ))

    echo "⏱️ Temps de backup: ${backup_duration}ms"
    echo "⏱️ Temps de restauration: ${restore_duration}ms"

    # Seuils de performance (en millisecondes)
    local backup_threshold=30000   # 30 secondes
    local restore_threshold=60000  # 60 secondes

    if [ $backup_duration -lt $backup_threshold ] && [ $restore_duration -lt $restore_threshold ]; then
        echo "✅ Performance acceptable"
        return 0
    else
        echo "⚠️  Performance dégradée"
        return 1
    fi
}

# Configuration du scheduler
setup_test_scheduler() {
    echo "📅 Configuration du scheduler de tests"

    local cron_schedule=""

    case $TEST_SCHEDULE in
        daily)
            cron_schedule="0 3 * * *"  # 3h du matin
            ;;
        weekly)
            cron_schedule="0 3 * * 0"  # Dimanche 3h
            ;;
        monthly)
            cron_schedule="0 3 1 * *"  # 1er du mois 3h
            ;;
        *)
            echo "❌ Schedule non reconnu: $TEST_SCHEDULE"
            return 1
            ;;
    esac

    # Créer le cron job
    cat > /etc/cron.d/restore-tests <<EOF
# Tests automatisés de restauration MicroK8s
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

$cron_schedule root /usr/local/bin/automated-restore-tests.sh run_test_suite >> $TEST_LOG_DIR/cron.log 2>&1
EOF

    echo "✅ Scheduler configuré: $TEST_SCHEDULE ($cron_schedule)"
}

# Menu principal
case "$1" in
    test)
        run_automated_test "$2" "$3"
        ;;
    suite)
        run_test_suite
        ;;
    schedule)
        setup_test_scheduler
        ;;
    *)
        echo "Usage: $0 {test|suite|schedule}"
        echo "  test <name> <function> - Exécuter un test spécifique"
        echo "  suite                  - Exécuter la suite complète"
        echo "  schedule               - Configurer le scheduler"
        ;;
esac
```

## Validation et métriques

### Métriques de test

```bash
#!/bin/bash
# test-metrics.sh

# Collecter et analyser les métriques de test
collect_test_metrics() {
    echo "📊 Collecte des métriques de test"

    local metrics_file="/var/log/restore-tests/metrics-$(date +%Y%m%d).json"
    local test_name=$1
    local test_status=$2
    local test_duration=$3
    local test_timestamp=$(date -Iseconds)

    # Créer l'entrée de métrique
    local metric_entry=$(cat <<EOF
{
    "timestamp": "$test_timestamp",
    "test_name": "$test_name",
    "status": "$test_status",
    "duration_seconds": $test_duration,
    "backup_tool": "${BACKUP_TOOL:-unknown}",
    "cluster_nodes": $(microk8s kubectl get nodes --no-headers | wc -l),
    "total_pods": $(microk8s kubectl get pods --all-namespaces --no-headers | wc -l),
    "total_namespaces": $(microk8s kubectl get namespaces --no-headers | wc -l)
}
EOF
    )

    # Ajouter au fichier de métriques
    echo "$metric_entry" >> "$metrics_file"

    # Calculer les statistiques
    calculate_test_statistics
}

# Calculer les statistiques de test
calculate_test_statistics() {
    echo "📈 Calcul des statistiques de test"

    local metrics_dir="/var/log/restore-tests"
    local stats_file="$metrics_dir/statistics.txt"

    # Analyser les 30 derniers jours
    local total_tests=0
    local successful_tests=0
    local failed_tests=0
    local total_duration=0

    # Parcourir les fichiers de métriques
    for metric_file in "$metrics_dir"/metrics-*.json; do
        [ -f "$metric_file" ] || continue

        while IFS= read -r line; do
            ((total_tests++))

            local status=$(echo "$line" | jq -r '.status')
            local duration=$(echo "$line" | jq -r '.duration_seconds')

            if [ "$status" = "SUCCESS" ]; then
                ((successful_tests++))
            else
                ((failed_tests++))
            fi

            total_duration=$(echo "$total_duration + $duration" | bc)
        done < "$metric_file"
    done

    # Calculer les moyennes
    local success_rate=0
    local avg_duration=0

    if [ $total_tests -gt 0 ]; then
        success_rate=$(echo "scale=2; $successful_tests * 100 / $total_tests" | bc)
        avg_duration=$(echo "scale=2; $total_duration / $total_tests" | bc)
    fi

    # Générer le rapport
    cat > "$stats_file" <<EOF
=== STATISTIQUES DES TESTS DE RESTAURATION ===
Date de génération: $(date)

Tests totaux: $total_tests
Tests réussis: $successful_tests
Tests échoués: $failed_tests
Taux de réussite: ${success_rate}%
Durée moyenne: ${avg_duration}s

Tendances (30 derniers jours):
$(generate_trend_analysis)

Prochains tests programmés:
$(list_scheduled_tests)
EOF

    echo "✅ Statistiques générées: $stats_file"
}

# Analyser les tendances
generate_trend_analysis() {
    local metrics_dir="/var/log/restore-tests"

    # Analyser les tendances sur 30 jours
    for i in {0..29}; do
        local date=$(date -d "$i days ago" +%Y%m%d)
        local file="$metrics_dir/metrics-$date.json"

        if [ -f "$file" ]; then
            local day_tests=$(wc -l < "$file")
            local day_success=$(grep '"status": "SUCCESS"' "$file" | wc -l)
            local day_rate=0

            if [ $day_tests -gt 0 ]; then
                day_rate=$(echo "scale=0; $day_success * 100 / $day_tests" | bc)
            fi

            echo "$(date -d "$i days ago" +%Y-%m-%d): $day_tests tests, ${day_rate}% réussite"
        fi
    done | head -7  # Afficher seulement la dernière semaine
}

# Lister les tests programmés
list_scheduled_tests() {
    if [ -f /etc/cron.d/restore-tests ]; then
        grep -v "^#" /etc/cron.d/restore-tests | awk '{print $1,$2,$3,$4,$5,"- Test automatique"}'
    else
        echo "Aucun test programmé"
    fi
}

# Exporter les métriques pour Prometheus
export_prometheus_metrics() {
    echo "📤 Export des métriques Prometheus"

    local metrics_file="/var/lib/prometheus/node-exporter/restore_tests.prom"
    local metrics_dir="/var/log/restore-tests"

    # Créer le répertoire si nécessaire
    mkdir -p "$(dirname "$metrics_file")"

    # Calculer les métriques actuelles
    local total_tests=$(find "$metrics_dir" -name "metrics-*.json" -exec wc -l {} \; | awk '{sum+=$1} END {print sum}')
    local last_test_time=$(find "$metrics_dir" -name "metrics-*.json" -exec stat -c %Y {} \; | sort -n | tail -1)
    local last_success=$(grep -h '"status": "SUCCESS"' "$metrics_dir"/metrics-*.json | tail -1 | jq -r '.timestamp' | xargs -I {} date -d {} +%s)
    local last_failure=$(grep -h '"status": "FAILURE"' "$metrics_dir"/metrics-*.json | tail -1 | jq -r '.timestamp' | xargs -I {} date -d {} +%s)

    # Générer le fichier de métriques Prometheus
    cat > "$metrics_file" <<EOF
# HELP restore_tests_total Total number of restore tests executed
# TYPE restore_tests_total counter
restore_tests_total $total_tests

# HELP restore_tests_last_execution_timestamp Timestamp of last test execution
# TYPE restore_tests_last_execution_timestamp gauge
restore_tests_last_execution_timestamp $last_test_time

# HELP restore_tests_last_success_timestamp Timestamp of last successful test
# TYPE restore_tests_last_success_timestamp gauge
restore_tests_last_success_timestamp ${last_success:-0}

# HELP restore_tests_last_failure_timestamp Timestamp of last failed test
# TYPE restore_tests_last_failure_timestamp gauge
restore_tests_last_failure_timestamp ${last_failure:-0}
EOF

    echo "✅ Métriques exportées: $metrics_file"
}
```

### Dashboard de monitoring

```yaml
# grafana-restore-tests-dashboard.json
{
  "dashboard": {
    "title": "Tests de Restauration MicroK8s",
    "panels": [
      {
        "id": 1,
        "title": "Taux de Réussite",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(restore_tests_success[24h]) / rate(restore_tests_total[24h]) * 100"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"value": 0, "color": "red"},
                {"value": 80, "color": "yellow"},
                {"value": 95, "color": "green"}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Temps depuis Dernier Test",
        "type": "stat",
        "targets": [
          {
            "expr": "(time() - restore_tests_last_execution_timestamp) / 3600"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "h",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 24, "color": "yellow"},
                {"value": 168, "color": "red"}
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Historique des Tests",
        "type": "graph",
        "targets": [
          {
            "expr": "increase(restore_tests_total[1h])",
            "legendFormat": "Tests totaux"
          },
          {
            "expr": "increase(restore_tests_success[1h])",
            "legendFormat": "Tests réussis"
          },
          {
            "expr": "increase(restore_tests_failure[1h])",
            "legendFormat": "Tests échoués"
          }
        ]
      },
      {
        "id": 4,
        "title": "Durée des Tests",
        "type": "graph",
        "targets": [
          {
            "expr": "restore_test_duration_seconds",
            "legendFormat": "{{test_type}}"
          }
        ],
        "yaxes": [
          {
            "format": "s",
            "label": "Durée"
          }
        ]
      }
    ]
  }
}
```

## Scénarios de test avancés

### Test de corruption de données

```bash
#!/bin/bash
# corruption-test.sh

test_data_corruption_recovery() {
    echo "🔬 Test de récupération après corruption de données"

    local test_namespace="corruption-test"
    local test_data="Important data that must be preserved"

    # 1. Créer l'environnement de test
    echo "📦 Création de l'environnement..."
    microk8s kubectl create namespace $test_namespace

    # Créer une application avec données
    cat <<EOF | microk8s kubectl apply -n $test_namespace -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: critical-data
data:
  data.txt: |
    $test_data
  checksum: $(echo -n "$test_data" | sha256sum | cut -d' ' -f1)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-app
  template:
    metadata:
      labels:
        app: data-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'while true; do cat /data/data.txt; sleep 30; done']
        volumeMounts:
        - name: config
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: critical-data
EOF

    # 2. Créer un backup
    echo "📸 Création du backup..."
    microk8s kubectl get all,cm -n $test_namespace -o yaml > /tmp/corruption-backup.yaml
    local original_checksum=$(microk8s kubectl get cm critical-data -n $test_namespace -o jsonpath='{.data.checksum}')

    # 3. Simuler une corruption
    echo "💥 Simulation de corruption..."
    microk8s kubectl patch configmap critical-data -n $test_namespace \
        --type='json' -p='[{"op": "replace", "path": "/data/data.txt", "value": "CORRUPTED DATA!!!"}]'

    # Vérifier la corruption
    local corrupted_data=$(microk8s kubectl get cm critical-data -n $test_namespace -o jsonpath='{.data.data\.txt}')
    if [[ "$corrupted_data" == "CORRUPTED DATA!!!" ]]; then
        echo "✅ Corruption simulée avec succès"
    else
        echo "❌ Échec de la simulation de corruption"
        return 1
    fi

    # 4. Détecter la corruption
    echo "🔍 Détection de la corruption..."
    local current_data=$(microk8s kubectl get cm critical-data -n $test_namespace -o jsonpath='{.data.data\.txt}')
    local current_checksum=$(echo -n "$current_data" | sha256sum | cut -d' ' -f1)

    if [ "$current_checksum" != "$original_checksum" ]; then
        echo "⚠️  Corruption détectée! Checksum ne correspond pas."
        echo "  Original: $original_checksum"
        echo "  Actuel: $current_checksum"
    fi

    # 5. Restaurer depuis le backup
    echo "♻️  Restauration des données non corrompues..."
    microk8s kubectl delete cm critical-data -n $test_namespace
    microk8s kubectl apply -f /tmp/corruption-backup.yaml

    # 6. Valider la restauration
    echo "✅ Validation de la restauration..."
    local restored_data=$(microk8s kubectl get cm critical-data -n $test_namespace -o jsonpath='{.data.data\.txt}')
    local restored_checksum=$(echo -n "$restored_data" | sha256sum | cut -d' ' -f1)

    if [ "$restored_checksum" == "$original_checksum" ]; then
        echo "✅ Données restaurées avec succès!"
        echo "  Données: $restored_data"

        # Nettoyer
        microk8s kubectl delete namespace $test_namespace --wait=false
        return 0
    else
        echo "❌ Échec de la restauration - les données sont toujours corrompues"
        return 1
    fi
}
```

### Test de montée en charge

```bash
#!/bin/bash
# load-test-restore.sh

test_restore_under_load() {
    echo "🔬 Test de restauration sous charge"

    local test_namespace="load-test"
    local pod_count=50

    # 1. Créer un namespace avec beaucoup de ressources
    echo "📦 Création de $pod_count pods..."
    microk8s kubectl create namespace $test_namespace

    for i in $(seq 1 $pod_count); do
        cat <<EOF | microk8s kubectl apply -n $test_namespace -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-app-$i
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-app-$i
  template:
    metadata:
      labels:
        app: load-app-$i
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            memory: "10Mi"
            cpu: "10m"
          limits:
            memory: "50Mi"
            cpu: "50m"
EOF
    done

    # 2. Attendre que tous les pods soient prêts
    echo "⏳ Attente du démarrage des pods..."
    local ready_pods=0
    local max_wait=120
    local waited=0

    while [ $ready_pods -lt $pod_count ] && [ $waited -lt $max_wait ]; do
        ready_pods=$(microk8s kubectl get pods -n $test_namespace --field-selector=status.phase=Running --no-headers | wc -l)
        echo "  Pods prêts: $ready_pods/$pod_count"
        sleep 5
        ((waited+=5))
    done

    # 3. Mesurer les ressources utilisées
    echo "📊 Ressources utilisées avant backup:"
    microk8s kubectl top nodes
    microk8s kubectl top pods -n $test_namespace | head -10

    # 4. Créer un backup avec mesure du temps
    echo "📸 Création du backup sous charge..."
    local backup_start=$(date +%s)

    if command -v velero &> /dev/null; then
        velero backup create load-backup-$(date +%s) --include-namespaces=$test_namespace --wait
    else
        microk8s kubectl get all -n $test_namespace -o yaml > /tmp/load-backup.yaml
    fi

    local backup_end=$(date +%s)
    local backup_duration=$((backup_end - backup_start))
    echo "⏱️  Durée du backup: ${backup_duration}s"

    # 5. Supprimer le namespace
    echo "🗑️  Suppression du namespace..."
    microk8s kubectl delete namespace $test_namespace --wait=false
    sleep 10

    # 6. Restaurer avec mesure du temps
    echo "♻️  Restauration sous charge..."
    local restore_start=$(date +%s)

    if command -v velero &> /dev/null; then
        velero restore create load-restore-$(date +%s) --from-backup load-backup-* --wait
    else
        microk8s kubectl apply -f /tmp/load-backup.yaml
    fi

    local restore_end=$(date +%s)
    local restore_duration=$((restore_end - restore_start))
    echo "⏱️  Durée de restauration: ${restore_duration}s"

    # 7. Vérifier la restauration
    echo "🔍 Vérification de la restauration..."
    local restored_pods=$(microk8s kubectl get deployments -n $test_namespace --no-headers | wc -l)

    # Nettoyer
    microk8s kubectl delete namespace $test_namespace --wait=false

    # Rapport
    echo ""
    echo "📊 RAPPORT DE TEST SOUS CHARGE"
    echo "=============================="
    echo "Pods testés: $pod_count"
    echo "Pods restaurés: $restored_pods"
    echo "Temps de backup: ${backup_duration}s"
    echo "Temps de restauration: ${restore_duration}s"
    echo "Ratio backup: $(echo "scale=2; $backup_duration / $pod_count" | bc)s par pod"
    echo "Ratio restauration: $(echo "scale=2; $restore_duration / $pod_count" | bc)s par pod"

    if [ $restored_pods -eq $pod_count ]; then
        echo "✅ Test réussi!"
        return 0
    else
        echo "❌ Test échoué - tous les pods n'ont pas été restaurés"
        return 1
    fi
}
```

### Test de cohérence multi-composants

```bash
#!/bin/bash
# consistency-test.sh

test_multi_component_consistency() {
    echo "🔬 Test de cohérence multi-composants"

    local test_namespace="consistency-test"

    # 1. Déployer une stack complète
    echo "📦 Déploiement d'une application multi-composants..."
    microk8s kubectl create namespace $test_namespace

    # Base de données
    cat <<EOF | microk8s kubectl apply -n $test_namespace -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: $(echo -n "dbpass123" | base64)
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13-alpine
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: POSTGRES_DB
          value: testdb
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: database
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api
        image: nginx:alpine
        env:
        - name: DB_HOST
          value: database
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: web
        image: nginx:alpine
        env:
        - name: API_URL
          value: http://backend
        ports:
        - containerPort: 80
EOF

    # 2. Attendre que tout soit prêt
    echo "⏳ Attente du démarrage complet..."
    microk8s kubectl wait --for=condition=ready pod -l app=database -n $test_namespace --timeout=60s
    microk8s kubectl wait --for=condition=ready pod -l app=backend -n $test_namespace --timeout=60s
    microk8s kubectl wait --for=condition=ready pod -l app=frontend -n $test_namespace --timeout=60s

    # 3. Capturer l'état de référence
    echo "📸 Capture de l'état de référence..."
    local ref_state="/tmp/reference-state.json"

    cat > "$ref_state" <<EOF
{
    "timestamp": "$(date -Iseconds)",
    "components": {
        "database": {
            "type": "StatefulSet",
            "replicas": $(microk8s kubectl get statefulset database -n $test_namespace -o jsonpath='{.status.readyReplicas}'),
            "version": "$(microk8s kubectl get statefulset database -n $test_namespace -o jsonpath='{.spec.template.spec.containers[0].image}')"
        },
        "backend": {
            "type": "Deployment",
            "replicas": $(microk8s kubectl get deployment backend -n $test_namespace -o jsonpath='{.status.readyReplicas}'),
            "env_db_host": "$(microk8s kubectl get deployment backend -n $test_namespace -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DB_HOST")].value}')"
        },
        "frontend": {
            "type": "Deployment",
            "replicas": $(microk8s kubectl get deployment frontend -n $test_namespace -o jsonpath='{.status.readyReplicas}'),
            "env_api_url": "$(microk8s kubectl get deployment frontend -n $test_namespace -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="API_URL")].value}')"
        }
    },
    "connections": {
        "frontend_to_backend": "http://backend",
        "backend_to_database": "database:5432"
    }
}
EOF

    # 4. Créer un backup
    echo "💾 Création du backup..."
    microk8s kubectl get all,secret,cm -n $test_namespace -o yaml > /tmp/consistency-backup.yaml

    # 5. Simuler une défaillance partielle
    echo "💥 Simulation de défaillance partielle..."
    microk8s kubectl delete deployment backend -n $test_namespace
    microk8s kubectl delete secret db-secret -n $test_namespace

    # 6. Restaurer
    echo "♻️  Restauration..."
    microk8s kubectl apply -f /tmp/consistency-backup.yaml

    # 7. Vérifier la cohérence
    echo "🔍 Vérification de la cohérence..."
    sleep 10  # Attendre la stabilisation

    local errors=0

    # Vérifier que tous les composants sont présents
    for component in database backend frontend; do
        if ! microk8s kubectl get deployment,statefulset -n $test_namespace | grep -q $component; then
            echo "❌ Composant manquant: $component"
            ((errors++))
        else
            echo "✅ Composant restauré: $component"
        fi
    done

    # Vérifier les connexions
    local backend_db_host=$(microk8s kubectl get deployment backend -n $test_namespace -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DB_HOST")].value}')
    if [ "$backend_db_host" != "database" ]; then
        echo "❌ Configuration backend incorrecte"
        ((errors++))
    else
        echo "✅ Configuration backend cohérente"
    fi

    # Vérifier les secrets
    if ! microk8s kubectl get secret db-secret -n $test_namespace &>/dev/null; then
        echo "❌ Secret de base de données manquant"
        ((errors++))
    else
        echo "✅ Secret restauré"
    fi

    # Nettoyer
    microk8s kubectl delete namespace $test_namespace --wait=false

    # Résultat
    if [ $errors -eq 0 ]; then
        echo "✅ Test de cohérence réussi!"
        return 0
    else
        echo "❌ Test de cohérence échoué avec $errors erreurs"
        return 1
    fi
}
```

## Rapports et documentation

### Génération de rapports automatiques

```bash
#!/bin/bash
# generate-test-report.sh

generate_test_report() {
    echo "📄 Génération du rapport de test"

    local report_date=$(date +%Y%m%d-%H%M%S)
    local report_dir="/var/log/restore-tests/reports"
    local report_file="$report_dir/test-report-$report_date.html"

    mkdir -p "$report_dir"

    # Collecter les données
    local total_tests=$(find /var/log/restore-tests -name "test-*.log" | wc -l)
    local successful_tests=$(grep -l "SUCCESS" /var/log/restore-tests/test-*.log 2>/dev/null | wc -l)
    local failed_tests=$(grep -l "FAILURE" /var/log/restore-tests/test-*.log 2>/dev/null | wc -l)
    local success_rate=$(echo "scale=2; $successful_tests * 100 / $total_tests" | bc)

    # Générer le rapport HTML
    cat > "$report_file" <<'HTML'
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Rapport de Tests de Restauration</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            margin: 0;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 12px;
            box-shadow: 0 20px 40px rgba(0,0,0,0.1);
            padding: 30px;
        }
        h1 {
            color: #333;
            border-bottom: 3px solid #667eea;
            padding-bottom: 10px;
        }
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 20px;
            margin: 30px 0;
        }
        .stat-card {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            border-left: 4px solid #667eea;
        }
        .stat-value {
            font-size: 2em;
            font-weight: bold;
            color: #333;
        }
        .stat-label {
            color: #666;
            margin-top: 5px;
        }
        .success { color: #28a745; }
        .failure { color: #dc3545; }
        .warning { color: #ffc107; }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        th, td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #dee2e6;
        }
        th {
            background: #f8f9fa;
            font-weight: 600;
        }
        .badge {
            padding: 4px 8px;
            border-radius: 4px;
            font-size: 0.85em;
            font-weight: 600;
        }
        .badge-success { background: #d4edda; color: #155724; }
        .badge-danger { background: #f8d7da; color: #721c24; }
        .timeline {
            margin-top: 30px;
        }
        .timeline-item {
            padding: 15px;
            border-left: 3px solid #667eea;
            margin-left: 10px;
            margin-bottom: 20px;
            position: relative;
        }
        .timeline-item::before {
            content: "";
            position: absolute;
            left: -7px;
            top: 20px;
            width: 11px;
            height: 11px;
            border-radius: 50%;
            background: #667eea;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📊 Rapport de Tests de Restauration MicroK8s</h1>
        <p>Généré le: <strong>$(date '+%Y-%m-%d %H:%M:%S')</strong></p>

        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-value">$total_tests</div>
                <div class="stat-label">Tests Totaux</div>
            </div>
            <div class="stat-card">
                <div class="stat-value success">$successful_tests</div>
                <div class="stat-label">Tests Réussis</div>
            </div>
            <div class="stat-card">
                <div class="stat-value failure">$failed_tests</div>
                <div class="stat-label">Tests Échoués</div>
            </div>
            <div class="stat-card">
                <div class="stat-value">${success_rate}%</div>
                <div class="stat-label">Taux de Réussite</div>
            </div>
        </div>

        <h2>📈 Résultats Détaillés</h2>
        <table>
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Type de Test</th>
                    <th>Durée</th>
                    <th>Statut</th>
                    <th>Détails</th>
                </tr>
            </thead>
            <tbody>
HTML

    # Ajouter les lignes du tableau
    for log_file in $(ls -t /var/log/restore-tests/test-*.log | head -20); do
        local test_date=$(stat -c %y "$log_file" | cut -d' ' -f1,2 | cut -d'.' -f1)
        local test_type=$(grep "Test:" "$log_file" | head -1 | cut -d':' -f2 | xargs)
        local test_duration=$(grep "Durée:" "$log_file" | head -1 | cut -d':' -f2 | xargs)
        local test_status=$(grep -q "SUCCESS" "$log_file" && echo "SUCCESS" || echo "FAILURE")
        local badge_class=$([ "$test_status" = "SUCCESS" ] && echo "badge-success" || echo "badge-danger")
        local status_icon=$([ "$test_status" = "SUCCESS" ] && echo "✅" || echo "❌")

        cat >> "$report_file" <<HTML
                <tr>
                    <td>$test_date</td>
                    <td>$test_type</td>
                    <td>$test_duration</td>
                    <td><span class="badge $badge_class">$status_icon $test_status</span></td>
                    <td><a href="file://$log_file">Voir les logs</a></td>
                </tr>
HTML
    done

    # Continuer le rapport HTML
    cat >> "$report_file" <<'HTML'
            </tbody>
        </table>

        <h2>📅 Historique Récent</h2>
        <div class="timeline">
HTML

    # Ajouter les événements récents
    for i in {0..6}; do
        local date=$(date -d "$i days ago" +%Y-%m-%d)
        local day_tests=$(find /var/log/restore-tests -name "test-*.log" -newermt "$date 00:00" ! -newermt "$date 23:59" | wc -l)
        local day_success=$(find /var/log/restore-tests -name "test-*.log" -newermt "$date 00:00" ! -newermt "$date 23:59" -exec grep -l "SUCCESS" {} \; | wc -l)

        if [ $day_tests -gt 0 ]; then
            cat >> "$report_file" <<HTML
            <div class="timeline-item">
                <strong>$date</strong><br>
                Tests exécutés: $day_tests<br>
                Réussis: $day_success<br>
                Taux: $(echo "scale=1; $day_success * 100 / $day_tests" | bc)%
            </div>
HTML
        fi
    done

    # Finaliser le rapport HTML
    cat >> "$report_file" <<'HTML'
        </div>

        <h2>🔍 Analyses et Recommandations</h2>
        <div style="background: #f8f9fa; padding: 20px; border-radius: 8px; margin-top: 20px;">
HTML

    # Ajouter les recommandations basées sur les résultats
    if (( $(echo "$success_rate < 95" | bc -l) )); then
        cat >> "$report_file" <<'HTML'
            <p>⚠️ <strong>Attention:</strong> Le taux de réussite est inférieur à 95%. Actions recommandées:</p>
            <ul>
                <li>Vérifier la validité des backups</li>
                <li>Examiner les logs des tests échoués</li>
                <li>Valider la configuration de l'outil de backup</li>
                <li>Augmenter la fréquence des tests</li>
            </ul>
HTML
    else
        cat >> "$report_file" <<'HTML'
            <p>✅ <strong>Excellent:</strong> Le taux de réussite est supérieur à 95%.</p>
            <ul>
                <li>Continuer les tests réguliers</li>
                <li>Envisager des scénarios de test plus complexes</li>
                <li>Documenter les procédures qui fonctionnent bien</li>
            </ul>
HTML
    fi

    cat >> "$report_file" <<'HTML'
        </div>

        <h2>📊 Métriques de Performance</h2>
        <canvas id="performanceChart" width="400" height="200"></canvas>

        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
        <script>
            // Graphique de performance (exemple)
            const ctx = document.getElementById('performanceChart').getContext('2d');
            const chart = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: ['Lun', 'Mar', 'Mer', 'Jeu', 'Ven', 'Sam', 'Dim'],
                    datasets: [{
                        label: 'Durée des tests (secondes)',
                        data: [45, 52, 48, 55, 47, 50, 49],
                        borderColor: '#667eea',
                        tension: 0.1
                    }]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false
                }
            });
        </script>

        <footer style="margin-top: 50px; padding-top: 20px; border-top: 1px solid #dee2e6; text-align: center; color: #666;">
            <p>Rapport généré automatiquement par le système de tests de restauration MicroK8s</p>
            <p>Pour plus d'informations, consultez les logs dans: <code>/var/log/restore-tests/</code></p>
        </footer>
    </div>
</body>
</html>
HTML

    echo "✅ Rapport généré: $report_file"

    # Optionnel: ouvrir le rapport dans le navigateur
    if command -v xdg-open &> /dev/null; then
        xdg-open "$report_file" 2>/dev/null &
    fi
}

# Générer un rapport PDF
generate_pdf_report() {
    local html_report=$1
    local pdf_report="${html_report%.html}.pdf"

    if command -v wkhtmltopdf &> /dev/null; then
        wkhtmltopdf "$html_report" "$pdf_report"
        echo "✅ Rapport PDF généré: $pdf_report"
    else
        echo "⚠️  wkhtmltopdf non installé. Installation: sudo apt-get install wkhtmltopdf"
    fi
}

# Envoyer le rapport par email
send_report_email() {
    local report_file=$1
    local recipient=${2:-$NOTIFICATION_EMAIL}

    if [ -z "$recipient" ]; then
        echo "⚠️  Aucun destinataire email configuré"
        return 1
    fi

    local subject="Rapport de Tests de Restauration - $(date +%Y-%m-%d)"
    local body="Veuillez trouver ci-joint le rapport des tests de restauration.

Résumé:
- Tests totaux: $total_tests
- Tests réussis: $successful_tests
- Taux de réussite: ${success_rate}%

Cordialement,
Système de Tests Automatisés"

    # Envoyer l'email avec pièce jointe
    echo "$body" | mail -s "$subject" -a "$report_file" "$recipient"
    echo "✅ Rapport envoyé à: $recipient"
}
```

### Documentation des procédures

```bash
#!/bin/bash
# generate-documentation.sh

generate_test_documentation() {
    echo "📚 Génération de la documentation des tests"

    local doc_dir="/var/log/restore-tests/documentation"
    local doc_file="$doc_dir/test-procedures-$(date +%Y%m%d).md"

    mkdir -p "$doc_dir"

    cat > "$doc_file" <<'MARKDOWN'
# Documentation des Tests de Restauration MicroK8s

## Vue d'ensemble

Cette documentation décrit les procédures de test de restauration pour l'environnement MicroK8s.

**Dernière mise à jour:** $(date +%Y-%m-%d)

## Table des matières

1. [Objectifs des tests](#objectifs)
2. [Types de tests](#types)
3. [Procédures](#procedures)
4. [Métriques](#metriques)
5. [Troubleshooting](#troubleshooting)
6. [Contacts](#contacts)

## Objectifs des tests {#objectifs}

Les tests de restauration visent à :
- ✅ Valider la capacité de récupération après incident
- ✅ Mesurer les temps de restauration (RTO)
- ✅ Vérifier l'intégrité des données restaurées
- ✅ Former l'équipe aux procédures d'urgence
- ✅ Identifier les points d'amélioration

## Types de tests {#types}

### 1. Test Partiel
- **Fréquence:** Quotidienne
- **Durée:** 5-10 minutes
- **Scope:** Un namespace ou une application
- **Automatisation:** Complète

### 2. Test Complet
- **Fréquence:** Hebdomadaire
- **Durée:** 30-60 minutes
- **Scope:** Cluster entier
- **Automatisation:** Semi-automatique

### 3. Test de Disaster Recovery
- **Fréquence:** Mensuelle
- **Durée:** 2-4 heures
- **Scope:** Simulation de perte totale
- **Automatisation:** Manuelle avec scripts

## Procédures {#procedures}

### Procédure de Test Partiel

```bash
# 1. Sélectionner le namespace à tester
NAMESPACE="test-namespace"

# 2. Créer un backup
velero backup create test-backup --include-namespaces=$NAMESPACE

# 3. Supprimer des ressources
kubectl delete deployment test-app -n $NAMESPACE

# 4. Restaurer
velero restore create --from-backup test-backup

# 5. Valider
kubectl get all -n $NAMESPACE
```

### Procédure de Test Complet

```bash
# 1. Notification de maintenance
echo "Maintenance programmée pour test de restauration"

# 2. Backup complet
velero backup create full-backup --wait

# 3. Simulation de perte
for ns in app-1 app-2 app-3; do
    kubectl delete namespace $ns --wait=false
done

# 4. Restauration
velero restore create --from-backup full-backup --wait

# 5. Validation
./validate-restoration.sh
```

### Procédure de Disaster Recovery

Voir le document séparé: `disaster-recovery-procedure.md`

## Métriques {#metriques}

### Métriques clés à surveiller

| Métrique | Objectif | Seuil d'alerte |
|----------|----------|----------------|
| RTO (Recovery Time Objective) | < 1 heure | > 2 heures |
| RPO (Recovery Point Objective) | < 4 heures | > 8 heures |
| Taux de réussite des tests | > 95% | < 90% |
| Temps de backup | < 30 min | > 1 heure |
| Temps de restauration | < 45 min | > 90 min |

### Dashboard Grafana

Accès: https://monitoring.local/dashboard/restore-tests

## Troubleshooting {#troubleshooting}

### Problème: Test échoué avec timeout

**Symptômes:**
- Le test se termine avec une erreur de timeout
- Les pods ne deviennent pas Ready

**Solutions:**
1. Vérifier les ressources disponibles
2. Augmenter les timeouts dans les scripts
3. Vérifier les logs des pods

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```

### Problème: Données corrompues après restauration

**Symptômes:**
- Les checksums ne correspondent pas
- Applications en erreur après restauration

**Solutions:**
1. Vérifier l'intégrité du backup
2. Tester avec un backup plus ancien
3. Vérifier la configuration de l'outil de backup

### Problème: Restauration incomplète

**Symptômes:**
- Certaines ressources manquent
- Erreurs de dépendances

**Solutions:**
1. Vérifier l'ordre de restauration
2. S'assurer que tous les namespaces sont inclus
3. Vérifier les RBAC et permissions

## Contacts {#contacts}

| Rôle | Nom | Email | Téléphone |
|------|-----|-------|-----------|
| Responsable Infrastructure | Jean Dupont | jean@example.com | +33 1 23 45 67 89 |
| Admin Kubernetes | Marie Martin | marie@example.com | +33 1 23 45 67 90 |
| Support 24/7 | - | support@example.com | +33 1 23 45 67 00 |

## Historique des modifications

| Date | Version | Modifications | Auteur |
|------|---------|--------------|---------|
| $(date +%Y-%m-%d) | 1.0 | Création initiale | System |

---

*Cette documentation est générée automatiquement. Pour toute modification, éditer le script `generate-documentation.sh`*
MARKDOWN

    echo "✅ Documentation générée: $doc_file"
}
```

## Bonnes pratiques et recommandations

### Checklist des bonnes pratiques

```bash
#!/bin/bash
# best-practices-check.sh

check_restore_test_best_practices() {
    echo "🔍 Vérification des bonnes pratiques"
    echo "===================================="

    local score=0
    local max_score=10

    # 1. Tests réguliers
    echo -n "1. Tests réguliers programmés... "
    if [ -f /etc/cron.d/restore-tests ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Configurer les tests automatiques"
    fi

    # 2. Tests variés
    echo -n "2. Différents types de tests... "
    local test_types=$(ls /var/log/restore-tests/test-*.log 2>/dev/null | xargs grep "Test:" | cut -d':' -f3 | sort -u | wc -l)
    if [ $test_types -ge 3 ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Diversifier les scénarios de test"
    fi

    # 3. Documentation à jour
    echo -n "3. Documentation récente... "
    local doc_age=$(find /var/log/restore-tests/documentation -name "*.md" -mtime -30 2>/dev/null | wc -l)
    if [ $doc_age -gt 0 ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Mettre à jour la documentation"
    fi

    # 4. Monitoring actif
    echo -n "4. Métriques collectées... "
    if [ -f /var/lib/prometheus/node-exporter/restore_tests.prom ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Configurer le monitoring"
    fi

    # 5. Notifications configurées
    echo -n "5. Notifications configurées... "
    if [ -n "$NOTIFICATION_EMAIL" ] || [ -n "$SLACK_WEBHOOK" ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Configurer les alertes"
    fi

    # 6. Logs archivés
    echo -n "6. Logs archivés... "
    local old_logs=$(find /var/log/restore-tests -name "*.log" -mtime +30 | wc -l)
    if [ $old_logs -eq 0 ]; then
        echo "✅"
        ((score++))
    else
        echo "⚠️  $old_logs anciens logs à archiver"
    fi

    # 7. Tests réussis récemment
    echo -n "7. Tests récents réussis... "
    local recent_success=$(find /var/log/restore-tests -name "test-*.log" -mtime -7 -exec grep -l "SUCCESS" {} \; | wc -l)
    if [ $recent_success -gt 0 ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Aucun test réussi cette semaine"
    fi

    # 8. Environnement de test isolé
    echo -n "8. Environnement de test dédié... "
    if microk8s kubectl get namespace | grep -q "restore-test"; then
        echo "✅"
        ((score++))
    else
        echo "⚠️  Créer un namespace de test dédié"
    fi

    # 9. Procédures de rollback
    echo -n "9. Procédures de rollback documentées... "
    if [ -f /var/log/restore-tests/documentation/rollback-procedure.md ]; then
        echo "✅"
        ((score++))
    else
        echo "❌ Documenter les procédures de rollback"
    fi

    # 10. Formation de l'équipe
    echo -n "10. Équipe formée... "
    if [ -f /var/log/restore-tests/training-log.txt ]; then
        echo "✅"
        ((score++))
    else
        echo "⚠️  Planifier une formation"
    fi

    # Résultat
    echo ""
    echo "======================================"
    echo "Score: $score/$max_score"

    if [ $score -ge 8 ]; then
        echo "🏆 Excellent! Continuez ainsi."
    elif [ $score -ge 6 ]; then
        echo "👍 Bien, mais des améliorations sont possibles."
    elif [ $score -ge 4 ]; then
        echo "⚠️  Attention, plusieurs points à améliorer."
    else
        echo "❌ Critique! Actions urgentes requises."
    fi

    # Recommandations
    echo ""
    echo "📋 Recommandations prioritaires:"

    if [ $score -lt $max_score ]; then
        echo "1. Automatiser les tests manquants"
        echo "2. Documenter toutes les procédures"
        echo "3. Former l'équipe régulièrement"
        echo "4. Surveiller les métriques de performance"
        echo "5. Tester des scénarios de plus en plus complexes"
    else
        echo "• Maintenir le niveau actuel"
        echo "• Explorer des scénarios avancés"
        echo "• Partager les bonnes pratiques"
    fi
}

# Exécution
check_restore_test_best_practices
```

### Plan de test mensuel

```yaml
# monthly-test-plan.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: monthly-test-plan
  namespace: kube-system
data:
  plan.yaml: |
    monthly_test_schedule:
      week_1:
        monday:
          - test: partial_restore
            namespaces: ["default", "production"]
            time: "03:00"
        wednesday:
          - test: data_integrity
            scope: all_databases
            time: "03:00"
        friday:
          - test: performance_baseline
            load: normal
            time: "03:00"

      week_2:
        monday:
          - test: full_restore
            scope: non_critical_apps
            time: "03:00"
        thursday:
          - test: corruption_recovery
            target: test_database
            time: "03:00"

      week_3:
        tuesday:
          - test: multi_component
            apps: ["frontend", "backend", "database"]
            time: "03:00"
        friday:
          - test: load_test
            pods: 100
            time: "03:00"

      week_4:
        monday:
          - test: disaster_recovery
            scenario: complete_failure
            time: "02:00"
            duration: "4h"
            notification: team
        friday:
          - test: cross_cluster_restore
            source: production
            target: staging
            time: "20:00"

      monthly_review:
        last_friday:
          - generate_reports: true
          - team_meeting: true
          - update_documentation: true
          - plan_next_month: true
```

## Conclusion

### Checklist finale des tests de restauration

- [ ] **Configuration de base**
  - [ ] Environnement de test isolé créé
  - [ ] Scripts de test installés
  - [ ] Permissions configurées

- [ ] **Tests automatisés**
  - [ ] Tests quotidiens programmés
  - [ ] Tests hebdomadaires configurés
  - [ ] Tests mensuels planifiés

- [ ] **Monitoring**
  - [ ] Métriques collectées
  - [ ] Dashboard configuré
  - [ ] Alertes actives

- [ ] **Documentation**
  - [ ] Procédures documentées
  - [ ] Contacts à jour
  - [ ] Runbooks créés

- [ ] **Validation**
  - [ ] Tests réussis > 95%
  - [ ] RTO/RPO respectés
  - [ ] Équipe formée

- [ ] **Amélioration continue**
  - [ ] Revues mensuelles
  - [ ] Scénarios mis à jour
  - [ ] Leçons apprises documentées

Les tests de restauration ne sont pas une option mais une nécessité. Un backup non testé est comme une assurance dont vous n'êtes pas sûr qu'elle fonctionne. Avec les outils et procédures décrits dans cette section, vous pouvez transformer les tests de restauration d'une corvée ponctuelle en un processus automatisé et fiable qui vous donne confiance dans votre capacité à récupérer après un incident.

**Rappel important:** La régularité est la clé. Mieux vaut des tests simples mais fréquents que des tests complexes mais rares. Commencez petit, automatisez progressivement, et améliorez continuellement vos procédures basées sur les résultats et les leçons apprises.

⏭️
