🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6 Plan de reprise d'activité

## Introduction au plan de reprise d'activité (PRA)

Un Plan de Reprise d'Activité (PRA), aussi appelé Disaster Recovery Plan (DRP) en anglais, est comme un plan d'évacuation pour votre infrastructure informatique. De la même façon qu'un bâtiment a des procédures en cas d'incendie, votre cluster MicroK8s a besoin de procédures claires pour faire face aux catastrophes : panne matérielle, corruption de données, erreur humaine, ou même sinistre physique.

### Pourquoi un PRA pour un lab personnel ?

Même pour un environnement de lab, un PRA est essentiel :

**Valeur de votre travail :**
- Des heures de configuration et d'apprentissage
- Des projets personnels importants
- Des données de test précieuses
- Une infrastructure de développement fonctionnelle

**Apprentissage professionnel :**
- Comprendre les concepts de disaster recovery
- Pratiquer des procédures d'entreprise
- Développer des réflexes professionnels
- Documenter des processus critiques

**Tranquillité d'esprit :**
- Savoir exactement quoi faire en cas de problème
- Réduire le stress lors d'incidents
- Minimiser les temps d'arrêt
- Protéger vos investissements en temps

### Composants d'un PRA

```
┌─────────────────────────────────────────────────────────┐
│              Plan de Reprise d'Activité                 │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ╔═══════════════════╗    ╔═══════════════════╗       │
│  ║   1. Analyse      ║    ║   2. Stratégie    ║       │
│  ╠═══════════════════╣    ╠═══════════════════╣       │
│  ║ • Risques         ║    ║ • RTO/RPO         ║       │
│  ║ • Impact          ║    ║ • Solutions       ║       │
│  ║ • Criticité       ║    ║ • Budget          ║       │
│  ╚═══════════════════╝    ╚═══════════════════╝       │
│                                                          │
│  ╔═══════════════════╗    ╔═══════════════════╗       │
│  ║   3. Procédures   ║    ║   4. Tests        ║       │
│  ╠═══════════════════╣    ╠═══════════════════╣       │
│  ║ • Détection       ║    ║ • Simulations     ║       │
│  ║ • Escalade        ║    ║ • Validation      ║       │
│  ║ • Restauration    ║    ║ • Amélioration    ║       │
│  ╚═══════════════════╝    ╚═══════════════════╝       │
│                                                          │
│  ╔═══════════════════╗    ╔═══════════════════╗       │
│  ║   5. Maintenance  ║    ║   6. Formation    ║       │
│  ╠═══════════════════╣    ╠═══════════════════╣       │
│  ║ • Mise à jour     ║    ║ • Documentation   ║       │
│  ║ • Révision        ║    ║ • Exercices       ║       │
│  ║ • Audit           ║    ║ • Retours         ║       │
│  ╚═══════════════════╝    ╚═══════════════════╝       │
└─────────────────────────────────────────────────────────┘
```

## Analyse des risques et impacts

### Identification des risques

```bash
#!/bin/bash
# risk-assessment.sh

# Fonction pour évaluer les risques
assess_risks() {
    echo "🔍 Analyse des risques pour MicroK8s"
    echo "====================================="

    local risk_report="/var/log/disaster-recovery/risk-assessment-$(date +%Y%m%d).txt"
    mkdir -p "$(dirname "$risk_report")"

    cat > "$risk_report" <<'EOF'
=== ANALYSE DES RISQUES MICROK8S ===
Date: $(date)

1. RISQUES MATÉRIELS
├─ Probabilité: Moyenne
├─ Impact: Élevé
├─ Exemples:
│  • Panne disque dur (30% par an pour SSD grand public)
│  • Panne alimentation (5-10% par an)
│  • Défaillance RAM (2-5% par an)
│  • Surchauffe (variable selon environnement)
└─ Mitigation:
   • Monitoring température et SMART
   • Backups réguliers
   • Matériel de rechange

2. RISQUES LOGICIELS
├─ Probabilité: Élevée
├─ Impact: Moyen
├─ Exemples:
│  • Bug dans mise à jour Kubernetes
│  • Corruption de etcd
│  • Incompatibilité de version
│  • Fuite mémoire applications
└─ Mitigation:
   • Tests avant mise à jour
   • Snapshots avant changements
   • Monitoring ressources

3. ERREURS HUMAINES
├─ Probabilité: Élevée
├─ Impact: Variable
├─ Exemples:
│  • kubectl delete sur mauvais namespace
│  • Mauvaise configuration réseau
│  • Suppression accidentelle de données
│  • Oubli de renouvellement certificats
└─ Mitigation:
   • Aliases sécurisés
   • Validation avant application
   • RBAC restrictif
   • Automatisation

4. RISQUES RÉSEAU
├─ Probabilité: Moyenne
├─ Impact: Moyen
├─ Exemples:
│  • Panne Internet
│  • Changement IP
│  • Attaque DDoS
│  • Problème DNS
└─ Mitigation:
   • Configuration réseau redondante
   • Cache DNS local
   • Firewall configuré
   • Monitoring connectivité

5. RISQUES SÉCURITÉ
├─ Probabilité: Faible (lab interne)
├─ Impact: Très élevé
├─ Exemples:
│  • Compromission de secrets
│  • Exploitation de vulnérabilité
│  • Ransomware
│  • Accès non autorisé
└─ Mitigation:
   • Mises à jour sécurité
   • Secrets chiffrés
   • Network policies
   • Audit logs

6. RISQUES ENVIRONNEMENTAUX
├─ Probabilité: Très faible
├─ Impact: Total
├─ Exemples:
│  • Coupure électrique prolongée
│  • Inondation/Incendie
│  • Vol de matériel
│  • Déménagement
└─ Mitigation:
   • UPS/Onduleur
   • Backups hors site
   • Chiffrement disques
   • Documentation complète
EOF

    echo "✅ Analyse des risques générée: $risk_report"
}

# Calculer l'impact business (même pour un lab)
calculate_impact() {
    echo "💼 Calcul de l'impact"

    local impact_matrix="/var/log/disaster-recovery/impact-matrix.json"

    cat > "$impact_matrix" <<'JSON'
{
    "services": [
        {
            "name": "Blog Personnel",
            "criticality": "HIGH",
            "max_downtime": "4 heures",
            "data_loss_tolerance": "1 heure",
            "dependencies": ["database", "storage", "ingress"],
            "impact": {
                "1h": "Faible - visiteurs reviendront",
                "4h": "Moyen - perte de trafic",
                "24h": "Élevé - impact SEO",
                "7j": "Critique - abandon des visiteurs"
            }
        },
        {
            "name": "Environnement Dev",
            "criticality": "MEDIUM",
            "max_downtime": "24 heures",
            "data_loss_tolerance": "4 heures",
            "dependencies": ["git", "ci-cd", "registry"],
            "impact": {
                "1h": "Nul - travail local possible",
                "4h": "Faible - retard développement",
                "24h": "Moyen - blocage projets",
                "7j": "Élevé - perte de momentum"
            }
        },
        {
            "name": "Monitoring",
            "criticality": "LOW",
            "max_downtime": "48 heures",
            "data_loss_tolerance": "24 heures",
            "dependencies": ["prometheus", "grafana"],
            "impact": {
                "1h": "Nul",
                "24h": "Faible - pas de métriques",
                "7j": "Moyen - diagnostic difficile"
            }
        }
    ],
    "cost_per_hour_downtime": {
        "reputation": "Non quantifiable",
        "learning_time": "1-2 heures de re-configuration",
        "data_recreation": "Variable selon le service"
    }
}
JSON

    echo "✅ Matrice d'impact créée: $impact_matrix"
}
```

### Définition des objectifs RTO/RPO

```yaml
# rto-rpo-objectives.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: disaster-recovery-objectives
  namespace: kube-system
data:
  objectives.yaml: |
    # RTO: Recovery Time Objective - Temps maximum acceptable d'interruption
    # RPO: Recovery Point Objective - Perte de données maximum acceptable

    global:
      rto_target: 4h        # Objectif global de restauration
      rpo_target: 1h        # Perte de données max 1 heure

    tiers:
      tier_1_critical:
        description: "Services critiques - Production"
        rto: 1h
        rpo: 15m
        services:
          - database-production
          - api-gateway
          - authentication-service
        backup_frequency: "*/15 * * * *"  # Toutes les 15 minutes

      tier_2_important:
        description: "Services importants - Applications"
        rto: 4h
        rpo: 1h
        services:
          - web-frontend
          - blog-wordpress
          - nextcloud
        backup_frequency: "0 * * * *"     # Toutes les heures

      tier_3_standard:
        description: "Services standards - Dev/Test"
        rto: 24h
        rpo: 4h
        services:
          - dev-environment
          - staging-apps
          - jenkins
        backup_frequency: "0 */4 * * *"   # Toutes les 4 heures

      tier_4_low:
        description: "Services non critiques"
        rto: 72h
        rpo: 24h
        services:
          - monitoring
          - logging
          - documentation
        backup_frequency: "0 2 * * *"     # Une fois par jour
```

## Stratégies de reprise

### Architecture de haute disponibilité

```bash
#!/bin/bash
# ha-architecture-setup.sh

setup_ha_architecture() {
    echo "🏗️ Configuration de l'architecture HA"

    # Configuration pour différents niveaux de HA
    cat > /etc/microk8s/ha-config.yaml <<'YAML'
# Configuration Haute Disponibilité MicroK8s

# Option 1: HA Locale (Budget minimal)
local_ha:
  description: "Redondance sur machine unique"
  components:
    storage:
      - type: "RAID 1"
        disks: 2
        purpose: "Miroir des données critiques"
    backup:
      - location: "Disque externe USB"
        frequency: "Quotidien"
      - location: "NAS local"
        frequency: "Horaire"
    monitoring:
      - service: "Prometheus local"
      - alerts: "Email/SMS"
  rto: "1-4 heures"
  rpo: "1 heure"
  cost: "€€"

# Option 2: HA Distribuée (Budget moyen)
distributed_ha:
  description: "Cluster multi-nœuds"
  components:
    nodes:
      - master: 3  # Quorum etcd
      - worker: 2  # Redondance applications
    storage:
      - type: "Ceph/GlusterFS"
        replicas: 2
    backup:
      - location: "S3 compatible (MinIO)"
        frequency: "Continu"
    loadbalancer:
      - type: "MetalLB"
      - vip: "192.168.1.100"
  rto: "5-30 minutes"
  rpo: "5 minutes"
  cost: "€€€"

# Option 3: HA Hybride (Budget flexible)
hybrid_ha:
  description: "Local + Cloud"
  components:
    primary:
      - location: "Local MicroK8s"
      - role: "Production"
    secondary:
      - location: "Cloud (AWS/GCP/Azure)"
      - role: "Standby"
    replication:
      - method: "Velero + Restic"
      - frequency: "Toutes les 15 minutes"
    failover:
      - type: "Manuel ou automatique"
      - dns_update: "CloudFlare API"
  rto: "15-60 minutes"
  rpo: "15 minutes"
  cost: "€€€€"

# Option 4: HA Cloud-Native (Budget élevé)
cloud_native_ha:
  description: "Multi-région cloud"
  components:
    regions:
      - primary: "eu-west-1"
      - secondary: "eu-central-1"
    kubernetes:
      - service: "EKS/GKE/AKS"
      - multi_az: true
    data:
      - database: "RDS Multi-AZ"
      - storage: "S3 cross-region"
    cdn:
      - service: "CloudFlare"
  rto: "0-5 minutes"
  rpo: "0-1 minute"
  cost: "€€€€€"
YAML

    echo "✅ Configuration HA documentée"
}

# Script de basculement (failover)
implement_failover() {
    echo "🔄 Implémentation du failover"

    cat > /usr/local/bin/failover.sh <<'BASH'
#!/bin/bash
# Failover automatique ou manuel

FAILOVER_MODE=${1:-manual}
PRIMARY_CLUSTER="microk8s-primary"
SECONDARY_CLUSTER="microk8s-secondary"
DNS_PROVIDER="cloudflare"

# Fonction de vérification de santé
check_primary_health() {
    echo "🔍 Vérification de la santé du cluster primaire..."

    # Test de connectivité API
    if ! timeout 5 kubectl --context=$PRIMARY_CLUSTER get nodes &>/dev/null; then
        return 1
    fi

    # Test des services critiques
    local critical_services=("database" "api-gateway" "ingress-nginx")
    for service in "${critical_services[@]}"; do
        if ! kubectl --context=$PRIMARY_CLUSTER get deployment $service &>/dev/null; then
            echo "❌ Service critique manquant: $service"
            return 1
        fi
    done

    return 0
}

# Procédure de basculement
perform_failover() {
    echo "🚨 DÉMARRAGE DU FAILOVER"
    echo "======================="

    # 1. Confirmer la panne
    echo "1️⃣ Confirmation de la panne du primaire..."
    local retry=3
    while [ $retry -gt 0 ]; do
        if check_primary_health; then
            echo "✅ Primaire répond - annulation du failover"
            return 0
        fi
        ((retry--))
        sleep 10
    done

    # 2. Activer le secondaire
    echo "2️⃣ Activation du cluster secondaire..."
    kubectl --context=$SECONDARY_CLUSTER scale deployment --all --replicas=1 -n production

    # 3. Vérifier le secondaire
    echo "3️⃣ Vérification du secondaire..."
    kubectl --context=$SECONDARY_CLUSTER wait --for=condition=ready pod --all -n production --timeout=300s

    # 4. Basculer le DNS
    echo "4️⃣ Mise à jour DNS..."
    case $DNS_PROVIDER in
        cloudflare)
            # Utiliser l'API CloudFlare
            curl -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
                 -H "Authorization: Bearer $CF_API_TOKEN" \
                 -H "Content-Type: application/json" \
                 --data '{"content":"'$SECONDARY_IP'"}'
            ;;
        route53)
            # AWS Route53
            aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
                --change-batch file://dns-failover.json
            ;;
    esac

    # 5. Notification
    echo "5️⃣ Envoi des notifications..."
    send_notification "FAILOVER COMPLETE" "Basculement vers cluster secondaire effectué"

    echo "✅ FAILOVER TERMINÉ"
}

# Mode automatique
if [ "$FAILOVER_MODE" = "auto" ]; then
    while true; do
        if ! check_primary_health; then
            perform_failover
            break
        fi
        sleep 60
    done
else
    # Mode manuel
    echo "Mode manuel - Confirmer le failover? (yes/no)"
    read -r response
    if [ "$response" = "yes" ]; then
        perform_failover
    fi
fi
BASH

    chmod +x /usr/local/bin/failover.sh
    echo "✅ Script de failover installé"
}
```

### Stratégies de sauvegarde multi-niveaux

```bash
#!/bin/bash
# multi-tier-backup-strategy.sh

implement_backup_strategy() {
    echo "💾 Implémentation de la stratégie de sauvegarde"

    cat > /etc/disaster-recovery/backup-strategy.yaml <<'YAML'
# Stratégie de sauvegarde 3-2-1 améliorée

levels:
  # Niveau 1: Snapshots locaux (Rapide)
  level_1_local:
    type: "Snapshots système"
    frequency: "Toutes les heures"
    retention: "24 derniers"
    location: "/backup/snapshots"
    tools:
      - LVM snapshots
      - ZFS snapshots
      - Velero local
    restore_time: "< 5 minutes"
    use_cases:
      - Erreur de configuration
      - Suppression accidentelle
      - Test de changements

  # Niveau 2: Backup serveur secondaire
  level_2_secondary:
    type: "Réplication complète"
    frequency: "Toutes les 4 heures"
    retention: "7 jours"
    location: "NAS ou serveur backup"
    tools:
      - Rsync
      - Velero + MinIO
      - Borg Backup
    restore_time: "< 30 minutes"
    use_cases:
      - Panne disque principal
      - Corruption de données
      - Restauration sélective

  # Niveau 3: Backup cloud
  level_3_cloud:
    type: "Archive cloud"
    frequency: "Quotidien"
    retention: "30 jours + archives mensuelles"
    location: "S3/GCS/Azure Blob"
    tools:
      - Velero + S3
      - Restic
      - Duplicati
    restore_time: "< 4 heures"
    use_cases:
      - Disaster recovery
      - Archive long terme
      - Conformité

  # Niveau 4: Backup offline
  level_4_offline:
    type: "Archive déconnectée"
    frequency: "Hebdomadaire"
    retention: "4 dernières + mensuelles"
    location: "Disque externe déconnecté"
    tools:
      - Script manuel
      - Tar + GPG
    restore_time: "< 8 heures"
    use_cases:
      - Protection ransomware
      - Archive légale
      - Dernier recours

automation:
  scheduler: "Systemd timers / Cron"
  orchestrator: "Velero schedules"
  monitoring: "Prometheus + AlertManager"
  validation: "Tests automatiques hebdomadaires"
YAML

    echo "✅ Stratégie de sauvegarde configurée"
}
```

## Procédures de reprise

### Procédure de reprise standard

```bash
#!/bin/bash
# standard-recovery-procedure.sh

# Procédure pas à pas pour la reprise
execute_standard_recovery() {
    echo "📋 PROCÉDURE DE REPRISE STANDARD"
    echo "================================="

    local incident_id="INC-$(date +%Y%m%d-%H%M%S)"
    local log_file="/var/log/disaster-recovery/incident-$incident_id.log"

    # Créer le répertoire de logs
    mkdir -p "$(dirname "$log_file")"

    # Démarrer l'enregistrement
    exec > >(tee -a "$log_file")
    exec 2>&1

    cat <<'PROCEDURE'
╔════════════════════════════════════════════════════════════╗
║           PROCÉDURE DE REPRISE D'ACTIVITÉ                 ║
╚════════════════════════════════════════════════════════════╝

PHASE 1: ÉVALUATION (0-15 minutes)
───────────────────────────────────
□ 1.1 Identifier le problème
  └─ Symptômes observés: _______________________
  └─ Heure de détection: _______________________
  └─ Services impactés: ________________________

□ 1.2 Évaluer l'impact
  └─ Criticité: [Faible|Moyen|Élevé|Critique]
  └─ Utilisateurs affectés: ____________________
  └─ Données à risque: _________________________

□ 1.3 Décision de reprise
  └─ Type de reprise: [Partielle|Complète|Failover]
  └─ RTO cible: ________________________________
  └─ RPO acceptable: ___________________________

PHASE 2: COMMUNICATION (15-30 minutes)
───────────────────────────────────────
□ 2.1 Notifier les parties prenantes
  └─ Email envoyé: □
  └─ Slack/Teams notifié: □
  └─ Status page mise à jour: □

□ 2.2 Créer le war room
  └─ Canal de communication: ___________________
  └─ Responsable incident: _____________________
  └─ Équipe mobilisée: _________________________

PHASE 3: ISOLATION (30-45 minutes)
───────────────────────────────────
□ 3.1 Isoler le problème
  └─ Services arrêtés: _________________________
  └─ Trafic rerouté: □
  └─ Backups sécurisés: □

□ 3.2 Préserver les preuves
  └─ Logs collectés: □
  └─ Snapshots créés: □
  └─ Métriques sauvegardées: □

PHASE 4: RESTAURATION (45 minutes - 4 heures)
──────────────────────────────────────────────
□ 4.1 Préparer l'environnement
  └─ Espace disque vérifié: □
  └─ Ressources disponibles: □
  └─ Outils prêts: □

□ 4.2 Restaurer les données
  └─ Source de backup: _________________________
  └─ Point de restauration: ____________________
  └─ Commande: velero restore create --from-backup

□ 4.3 Restaurer les services
  └─ Ordre de restauration:
    1. Base de données: □
    2. Services backend: □
    3. Services frontend: □
    4. Ingress/LoadBalancer: □

PHASE 5: VALIDATION (4-5 heures)
─────────────────────────────────
□ 5.1 Tests de santé
  └─ Tous les pods Running: □
  └─ Services accessibles: □
  └─ Données intègres: □

□ 5.2 Tests fonctionnels
  └─ Connexion utilisateur: □
  └─ Transactions test: □
  └─ Performance acceptable: □

PHASE 6: RETOUR À LA NORMALE (5-6 heures)
──────────────────────────────────────────
□ 6.1 Réactiver le trafic
  └─ DNS mis à jour: □
  └─ Load balancer reconfiguré: □
  └─ CDN purgé: □

□ 6.2 Monitoring renforcé
  └─ Alertes actives: □
  └─ Dashboards surveillés: □
  └─ Logs analysés: □

PHASE 7: POST-MORTEM (J+1 à J+7)
─────────────────────────────────
□ 7.1 Analyse de l'incident
  └─ Cause racine identifiée: __________________
  └─ Timeline reconstituée: □
  └─ Impact total calculé: _____________________

□ 7.2 Actions correctives
  └─ Court terme: ______________________________
  └─ Moyen terme: ______________________________
  └─ Long terme: _______________________________

□ 7.3 Documentation
  └─ Rapport d'incident: □
  └─ Procédures mises à jour: □
  └─ Lessons learned partagées: □
PROCEDURE
}

# Automatisation partielle de la récupération
automated_recovery() {
    local recovery_type=$1

    echo "🤖 Démarrage de la récupération automatisée: $recovery_type"

    case $recovery_type in
        "partial")
            # Récupération partielle
            echo "📦 Récupération partielle..."
            velero restore create partial-restore-$(date +%s) \
                --from-backup $(velero backup get -o json | jq -r '.items[0].metadata.name') \
                --include-namespaces production
            ;;

        "complete")
            # Récupération complète
            echo "📦 Récupération complète..."
            velero restore create complete-restore-$(date +%s) \
                --from-backup $(velero backup get -o json | jq -r '.items[0].metadata.name')
            ;;

        "failover")
            # Basculement vers secondaire
            echo "🔄 Basculement vers site secondaire..."
            /usr/local/bin/failover.sh auto
            ;;
    esac

    # Attendre la fin de la restauration
    echo "⏳ Restauration en cours..."
    sleep 30

    # Validation automatique
    echo "✅ Validation de la restauration..."
    kubectl get pods --all-namespaces
    kubectl get pvc --all-namespaces
    kubectl get ingress --all-namespaces
}
```

### Runbook d'urgence

```markdown
# RUNBOOK D'URGENCE - DISASTER RECOVERY MICROK8S

## 🚨 NUMÉROS D'URGENCE
- Responsable Infrastructure: +33 6 XX XX XX XX
- Support Cloud Provider: +1 800 XXX XXXX
- Hébergeur: support@provider.com

## 🔥 INCIDENT CRITIQUE - ACTIONS IMMÉDIATES

### 1. CLUSTER COMPLÈTEMENT DOWN
```bash
# Vérifier l'état
microk8s status
systemctl status snap.microk8s.daemon-kubelite

# Tentative de redémarrage
microk8s stop
microk8s start

# Si échec -> Restauration depuis backup
velero restore create emergency-$(date +%s) --from-backup latest
```

### 2. CORRUPTION ETCD
```bash
# Sauvegarder l'état actuel (si possible)
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-corrupted.db

# Restaurer depuis snapshot
microk8s stop
sudo rm -rf /var/snap/microk8s/current/var/kubernetes/backend/*
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
    --data-dir=/var/snap/microk8s/current/var/kubernetes/backend
microk8s start
```

### 3. PERTE DE DONNÉES CRITIQUES
```bash
# Identifier le dernier bon backup
velero backup get
velero backup describe <backup-name>

# Restaurer spécifiquement les données
velero restore create data-restore-$(date +%s) \
    --from-backup <backup-name> \
    --include-resources persistentvolumeclaims,persistentvolumes
```

### 4. RANSOMWARE DÉTECTÉ
```bash
# ISOLER IMMÉDIATEMENT
# 1. Déconnecter du réseau
sudo iptables -I INPUT -j DROP
sudo iptables -I OUTPUT -j DROP

# 2. Arrêter tous les pods
kubectl delete pods --all --all-namespaces --force --grace-period=0

# 3. Identifier l'étendue
find / -name "*.encrypted" -o -name "*ransom*" 2>/dev/null

# 4. Restaurer depuis backup offline
# Utiliser UNIQUEMENT les backups déconnectés!
```

## 📊 COMMANDES DE DIAGNOSTIC

### État du cluster
```bash
# Vue d'ensemble
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running
kubectl top nodes
kubectl top pods --all-namespaces

# Événements récents
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Logs des composants critiques
kubectl logs -n kube-system deployment/coredns
kubectl logs -n kube-system deployment/metrics-server
journalctl -u snap.microk8s.daemon-kubelite -n 100

# Vérification du stockage
df -h
kubectl get pv,pvc --all-namespaces
```

### Réseau et connectivité
```bash
# Test de connectivité
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -O- http://kubernetes.default

# Vérifier les services
kubectl get svc --all-namespaces
kubectl get endpoints --all-namespaces

# DNS
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

## 🔄 PROCÉDURES DE RESTAURATION PAR SCÉNARIO

### Scénario 1: Panne matérielle
```bash
# 1. Basculer sur matériel de secours
# 2. Restaurer MicroK8s
sudo snap install microk8s --classic --channel=1.28/stable
sudo usermod -a -G microk8s $USER

# 3. Restaurer la configuration
kubectl apply -f /backup/configs/

# 4. Restaurer les données
velero restore create hardware-recovery-$(date +%s) --from-backup latest
```

### Scénario 2: Mise à jour échouée
```bash
# Rollback de MicroK8s
sudo snap revert microk8s

# Si échec, réinstaller version précédente
sudo snap remove microk8s
sudo snap install microk8s --classic --channel=1.27/stable

# Restaurer l'état
velero restore create rollback-$(date +%s) --from-backup pre-upgrade-backup
```

### Scénario 3: Suppression accidentelle
```bash
# Récupération rapide si moins de 30 min
velero restore create quick-restore-$(date +%s) \
    --from-backup $(velero backup get -o json | jq -r '.items[0].metadata.name') \
    --include-namespaces affected-namespace

# Restauration sélective
velero restore create selective-$(date +%s) \
    --from-backup backup-name \
    --include-resources deployment,service,configmap \
    --selector app=deleted-app
```

## Tests et validation du PRA

### Programme de tests réguliers

```bash
#!/bin/bash
# dpr-test-schedule.sh

schedule_drp_tests() {
    echo "📅 Planification des tests PRA"

    cat > /etc/disaster-recovery/test-schedule.yaml <<'YAML'
# Programme de tests du Plan de Reprise d'Activité

test_schedule:
  monthly:
    - name: "Test de restauration partielle"
      day: 1
      scope: "Un namespace non-critique"
      duration: "30 minutes"
      validation:
        - Pods running
        - Services accessibles
        - Données présentes

    - name: "Test de failover"
      day: 15
      scope: "Basculement DNS"
      duration: "1 heure"
      validation:
        - DNS mis à jour
        - Services accessibles depuis nouvelle IP
        - Monitoring actif

  quarterly:
    - name: "Test de restauration complète"
      month: [3, 6, 9, 12]
      scope: "Environnement de test complet"
      duration: "4 heures"
      validation:
        - Tous les namespaces restaurés
        - Intégrité des données
        - Performance acceptable

  annually:
    - name: "Simulation disaster complet"
      month: 11
      scope: "Simulation black-out"
      duration: "8 heures"
      participants:
        - Équipe infrastructure
        - Équipe développement
        - Management
      validation:
        - Procédures suivies
        - Communications efficaces
        - RTO/RPO respectés
        - Lessons learned documentées

alerts:
  pre_test:
    - email: team@example.com
    - slack: "#dr-tests"
    - advance_notice: "7 jours"

  post_test:
    - report_to: management@example.com
    - metrics_dashboard: grafana.local/dr-tests
    - documentation: confluence/DR-Reports
YAML
}

# Script de test automatisé
run_drp_test() {
    local test_type=$1
    local test_id="DRP-TEST-$(date +%Y%m%d-%H%M%S)"

    echo "🧪 Exécution du test PRA: $test_type"
    echo "ID du test: $test_id"

    # Créer un rapport
    local report="/var/log/disaster-recovery/test-$test_id.md"

    cat > "$report" <<'REPORT'
# Rapport de Test PRA

## Informations du test
- **ID:** $test_id
- **Type:** $test_type
- **Date:** $(date)
- **Exécutant:** $(whoami)

## Checklist pré-test
- [ ] Backup vérifié
- [ ] Équipe notifiée
- [ ] Environnement de test prêt
- [ ] Monitoring actif

## Étapes du test

### 1. Préparation (T-10 minutes)
```
Actions:
- Vérification des backups disponibles
- Création snapshot de l'état actuel
- Notification début du test
```

### 2. Simulation incident (T0)
```
Actions:
- Suppression/corruption simulée
- Documentation de l'heure exacte
- Capture des métriques initiales
```

### 3. Détection (T+X minutes)
```
Temps de détection: ______
Méthode de détection:
- [ ] Alerte automatique
- [ ] Détection manuelle
- [ ] Rapport utilisateur
```

### 4. Restauration (T+Y minutes)
```
Début restauration: ______
Fin restauration: ______
Durée totale: ______

Commandes exécutées:
[Documenter les commandes]
```

### 5. Validation (T+Z minutes)
```
Tests effectués:
- [ ] Connectivité réseau
- [ ] Services disponibles
- [ ] Données intègres
- [ ] Performance acceptable
```

## Résultats

### Métriques
- **RTO réel:** ______
- **RTO cible:** ______
- **RPO réel:** ______
- **RPO cible:** ______

### Problèmes rencontrés
[Liste des problèmes]

### Actions correctives
[Actions à implémenter]

## Conclusion
- [ ] Test réussi
- [ ] Test partiellement réussi
- [ ] Test échoué

**Signature:** _________________
**Date:** _________________
REPORT

    echo "✅ Rapport créé: $report"
}
```

### Métriques et KPIs du PRA

```yaml
# drp-metrics.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drp-metrics
  namespace: monitoring
data:
  metrics.yaml: |
    # Métriques clés du Plan de Reprise d'Activité

    availability_metrics:
      uptime_target: 99.9%  # 8.76 heures de downtime par an
      current_uptime: calculated_from_prometheus

    recovery_metrics:
      last_successful_backup:
        timestamp: prometheus_query
        size: prometheus_query

      last_successful_restore:
        timestamp: prometheus_query
        duration: prometheus_query

      average_rto:
        30_days: prometheus_query
        90_days: prometheus_query

      average_rpo:
        30_days: prometheus_query
        90_days: prometheus_query

    test_metrics:
      tests_planned_monthly: 2
      tests_executed_monthly: prometheus_query
      test_success_rate: prometheus_query

    risk_metrics:
      unpatched_vulnerabilities: prometheus_query
      backup_age_hours: prometheus_query
      last_dr_test_days: prometheus_query

    # Seuils d'alerte
    thresholds:
      backup_age_warning: 25h  # Alerte si backup > 25h
      backup_age_critical: 48h # Critique si backup > 48h

      test_frequency_warning: 45d  # Alerte si pas de test depuis 45j
      test_frequency_critical: 60d # Critique si pas de test depuis 60j

      recovery_time_warning: 2h  # Alerte si RTO > 2h
      recovery_time_critical: 4h # Critique si RTO > 4h
```

## Documentation et formation

### Documentation du PRA

```bash
#!/bin/bash
# generate-drp-documentation.sh

generate_drp_docs() {
    echo "📚 Génération de la documentation PRA"

    local doc_dir="/var/www/html/drp-docs"
    mkdir -p "$doc_dir"

    # Page d'accueil HTML
    cat > "$doc_dir/index.html" <<'HTML'
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Plan de Reprise d'Activité - MicroK8s</title>
    <style>
        body {
            font-family: -apple-system, system-ui, sans-serif;
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
            background: #f5f5f5;
        }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            border-radius: 10px;
            margin-bottom: 30px;
        }
        .card {
            background: white;
            padding: 20px;
            margin: 20px 0;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .emergency {
            background: #fee;
            border-left: 4px solid #f44336;
        }
        .info {
            background: #e3f2fd;
            border-left: 4px solid #2196f3;
        }
        .success {
            background: #e8f5e9;
            border-left: 4px solid #4caf50;
        }
        .grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 20px;
        }
        code {
            background: #f4f4f4;
            padding: 2px 6px;
            border-radius: 3px;
            font-family: 'Courier New', monospace;
        }
        .contact-list {
            list-style: none;
            padding: 0;
        }
        .contact-list li {
            padding: 10px;
            margin: 5px 0;
            background: #f9f9f9;
            border-radius: 5px;
        }
        .status-indicator {
            display: inline-block;
            width: 10px;
            height: 10px;
            border-radius: 50%;
            margin-right: 10px;
        }
        .status-ok { background: #4caf50; }
        .status-warning { background: #ff9800; }
        .status-critical { background: #f44336; }
    </style>
</head>
<body>
    <div class="header">
        <h1>🚨 Plan de Reprise d'Activité</h1>
        <p>Documentation complète pour la gestion des incidents et la reprise d'activité</p>
        <p>Dernière mise à jour: <strong id="last-update"></strong></p>
    </div>

    <div class="card emergency">
        <h2>☎️ Contacts d'Urgence</h2>
        <ul class="contact-list">
            <li>
                <strong>Responsable Infrastructure:</strong>
                Jean Dupont - +33 6 XX XX XX XX - jean@example.com
            </li>
            <li>
                <strong>Support Cloud (24/7):</strong>
                +1 800 XXX XXXX - support@cloudprovider.com
            </li>
            <li>
                <strong>Escalade Management:</strong>
                Marie Martin - +33 6 YY YY YY YY - marie@example.com
            </li>
        </ul>
    </div>

    <div class="grid">
        <div class="card">
            <h3>📊 Statut Actuel</h3>
            <p><span class="status-indicator status-ok"></span>Cluster: Opérationnel</p>
            <p><span class="status-indicator status-ok"></span>Dernier backup: <span id="last-backup">Il y a 2h</span></p>
            <p><span class="status-indicator status-warning"></span>Dernier test DR: <span id="last-test">Il y a 15j</span></p>
            <p><span class="status-indicator status-ok"></span>RTO moyen: 45 min</p>
            <p><span class="status-indicator status-ok"></span>RPO moyen: 1h</p>
        </div>

        <div class="card">
            <h3>🔧 Actions Rapides</h3>
            <ul>
                <li><a href="#backup-now">Lancer un backup manuel</a></li>
                <li><a href="#check-health">Vérifier la santé du cluster</a></li>
                <li><a href="#view-logs">Consulter les logs récents</a></li>
                <li><a href="#test-restore">Tester une restauration</a></li>
                <li><a href="#failover">Initier un failover</a></li>
            </ul>
        </div>

        <div class="card">
            <h3>📚 Documentation</h3>
            <ul>
                <li><a href="procedures/standard-recovery.html">Procédure de récupération standard</a></li>
                <li><a href="procedures/emergency-failover.html">Failover d'urgence</a></li>
                <li><a href="procedures/data-corruption.html">Corruption de données</a></li>
                <li><a href="procedures/ransomware-response.html">Réponse ransomware</a></li>
                <li><a href="runbooks/">Runbooks complets</a></li>
            </ul>
        </div>

        <div class="card">
            <h3>📈 Métriques</h3>
            <canvas id="rtoChart" width="300" height="200"></canvas>
            <p>RTO sur les 30 derniers jours</p>
        </div>
    </div>

    <div class="card info">
        <h2>🎯 Objectifs de Récupération</h2>
        <table style="width: 100%; border-collapse: collapse;">
            <thead>
                <tr style="background: #e3f2fd;">
                    <th style="padding: 10px; text-align: left;">Service</th>
                    <th style="padding: 10px;">RTO</th>
                    <th style="padding: 10px;">RPO</th>
                    <th style="padding: 10px;">Criticité</th>
                </tr>
            </thead>
            <tbody>
                <tr>
                    <td style="padding: 10px;">Base de données production</td>
                    <td style="padding: 10px; text-align: center;">1h</td>
                    <td style="padding: 10px; text-align: center;">15min</td>
                    <td style="padding: 10px; text-align: center;"><span style="color: red;">Critique</span></td>
                </tr>
                <tr style="background: #f5f5f5;">
                    <td style="padding: 10px;">Applications web</td>
                    <td style="padding: 10px; text-align: center;">2h</td>
                    <td style="padding: 10px; text-align: center;">1h</td>
                    <td style="padding: 10px; text-align: center;"><span style="color: orange;">Élevé</span></td>
                </tr>
                <tr>
                    <td style="padding: 10px;">Services de développement</td>
                    <td style="padding: 10px; text-align: center;">4h</td>
                    <td style="padding: 10px; text-align: center;">4h</td>
                    <td style="padding: 10px; text-align: center;"><span style="color: #ffc107;">Moyen</span></td>
                </tr>
                <tr style="background: #f5f5f5;">
                    <td style="padding: 10px;">Monitoring</td>
                    <td style="padding: 10px; text-align: center;">24h</td>
                    <td style="padding: 10px; text-align: center;">24h</td>
                    <td style="padding: 10px; text-align: center;"><span style="color: green;">Faible</span></td>
                </tr>
            </tbody>
        </table>
    </div>

    <div class="card">
        <h2>⚡ Commandes d'Urgence</h2>
        <pre style="background: #2d2d2d; color: #f8f8f2; padding: 15px; border-radius: 5px; overflow-x: auto;">
# Vérifier l'état du cluster
microk8s status
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running

# Backup d'urgence
velero backup create emergency-$(date +%s) --wait

# Restauration rapide
velero restore create --from-backup latest

# Logs des dernières erreurs
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | head -20

# Redémarrage d'urgence
microk8s stop && microk8s start
        </pre>
    </div>

    <div class="card success">
        <h2>✅ Checklist Post-Incident</h2>
        <form id="post-incident-checklist">
            <label><input type="checkbox"> Services restaurés et opérationnels</label><br>
            <label><input type="checkbox"> Données vérifiées et intègres</label><br>
            <label><input type="checkbox"> Monitoring réactivé</label><br>
            <label><input type="checkbox"> Utilisateurs notifiés du retour à la normale</label><br>
            <label><input type="checkbox"> Logs de l'incident sauvegardés</label><br>
            <label><input type="checkbox"> Rapport d'incident créé</label><br>
            <label><input type="checkbox"> Post-mortem planifié</label><br>
            <label><input type="checkbox"> Actions correctives identifiées</label><br>
        </form>
    </div>

    <script>
        // Mise à jour de la date
        document.getElementById('last-update').textContent = new Date().toLocaleString('fr-FR');

        // Graphique RTO simple
        const canvas = document.getElementById('rtoChart');
        const ctx = canvas.getContext('2d');

        // Données simulées
        const data = [45, 52, 38, 65, 42, 55, 48, 51, 43, 47, 40, 46];
        const max = Math.max(...data);

        // Dessiner le graphique
        ctx.strokeStyle = '#667eea';
        ctx.lineWidth = 2;
        ctx.beginPath();

        data.forEach((value, index) => {
            const x = (index / (data.length - 1)) * canvas.width;
            const y = canvas.height - (value / max) * canvas.height * 0.8;

            if (index === 0) {
                ctx.moveTo(x, y);
            } else {
                ctx.lineTo(x, y);
            }
        });

        ctx.stroke();

        // Ligne RTO cible
        ctx.strokeStyle = '#f44336';
        ctx.setLineDash([5, 5]);
        ctx.beginPath();
        ctx.moveTo(0, canvas.height * 0.4);
        ctx.lineTo(canvas.width, canvas.height * 0.4);
        ctx.stroke();
    </script>
</body>
</html>
HTML

    echo "✅ Documentation HTML générée: $doc_dir/index.html"
}
```

### Programme de formation

```yaml
# drp-training-program.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drp-training
  namespace: default
data:
  program.yaml: |
    # Programme de Formation PRA

    modules:
      module_1:
        title: "Introduction au Disaster Recovery"
        duration: "2 heures"
        audience: "Tous"
        content:
          - Concepts RTO/RPO
          - Types de disasters
          - Stratégies de récupération
          - Rôles et responsabilités
        exercises:
          - Quiz sur les concepts
          - Calcul RTO/RPO pour cas d'usage

      module_2:
        title: "Outils et Technologies"
        duration: "4 heures"
        audience: "Équipe technique"
        content:
          - Velero et backups
          - Monitoring avec Prometheus
          - Scripts d'automatisation
          - Outils de communication
        exercises:
          - Création d'un backup
          - Restauration test
          - Lecture de métriques

      module_3:
        title: "Procédures d'Urgence"
        duration: "3 heures"
        audience: "Équipe ops"
        content:
          - Runbook d'urgence
          - Escalade
          - Communication de crise
          - Décision de failover
        exercises:
          - Simulation d'incident
          - Utilisation du runbook
          - Communication test

      module_4:
        title: "Exercice Pratique"
        duration: "8 heures"
        audience: "Toute l'équipe"
        content:
          - Simulation complète
          - Panne surprise
          - Coordination équipe
          - Débriefing
        validation:
          - Respect des procédures
          - Temps de récupération
          - Communication efficace
          - Lessons learned

    planning:
      onboarding:
        - Module 1: "Première semaine"
        - Module 2: "Première mois"

      récurrent:
        - Module 3: "Trimestriel"
        - Module 4: "Semestriel"

      certification:
        - Test théorique: "80% minimum"
        - Test pratique: "Participation obligatoire"
        - Renouvellement: "Annuel"
```

## Amélioration continue

### Processus d'amélioration

```bash
#!/bin/bash
# continuous-improvement.sh

implement_improvement_process() {
    echo "📈 Mise en place du processus d'amélioration continue"

    cat > /etc/disaster-recovery/improvement-process.md <<'MARKDOWN'
# Processus d'Amélioration Continue du PRA

## 1. COLLECTE DES DONNÉES

### Sources de données
- Logs d'incidents
- Résultats de tests
- Métriques de performance
- Feedback des équipes
- Benchmarks industrie

### Métriques suivies
- MTTR (Mean Time To Repair)
- MTBF (Mean Time Between Failures)
- Taux de réussite des restaurations
- Temps de détection des incidents
- Satisfaction des utilisateurs

## 2. ANALYSE

### Revue mensuelle
- Analyse des incidents du mois
- Performance des backups
- Résultats des tests
- Évolution des métriques

### Identification des améliorations
- Points de défaillance récurrents
- Goulots d'étranglement
- Processus manuels à automatiser
- Documentation manquante

## 3. PLANIFICATION

### Priorisation (Matrice Impact/Effort)
```
         Effort Faible    Effort Élevé
Impact   ┌─────────────┬─────────────┐
Élevé    │ Quick Wins  │ Projets     │
         │ (Faire      │ Stratégiques│
         │ maintenant) │ (Planifier) │
         ├─────────────┼─────────────┤
Faible   │ Fill-ins    │ Éviter      │
         │ (Si temps)  │ (Ou         │
         │             │ simplifier) │
         └─────────────┴─────────────┘
```

### Plan d'action
- Court terme (< 1 mois)
- Moyen terme (1-3 mois)
- Long terme (> 3 mois)

## 4. IMPLÉMENTATION

### Cycle PDCA
- **Plan**: Définir l'amélioration
- **Do**: Implémenter en test
- **Check**: Mesurer les résultats
- **Act**: Déployer ou ajuster

### Gestion du changement
- Communication des changements
- Formation si nécessaire
- Documentation mise à jour
- Validation par tests

## 5. MESURE ET SUIVI

### KPIs d'amélioration
- Réduction du RTO: Objectif -10% par trimestre
- Réduction du RPO: Objectif -15% par an
- Augmentation du taux de réussite: Objectif > 99%
- Réduction des incidents: Objectif -20% par an

### Reporting
- Dashboard temps réel
- Rapport mensuel
- Revue trimestrielle
- Bilan annuel

## 6. STANDARDISATION

### Best practices
- Documentation des succès
- Création de templates
- Automatisation des processus
- Partage de connaissances

### Certification
- ISO 22301 (Business Continuity)
- ISO 27031 (IT Disaster Recovery)
- Audit annuel interne
- Certification externe si pertinent
MARKDOWN
}

# Tableau de bord d'amélioration
create_improvement_dashboard() {
    echo "📊 Création du dashboard d'amélioration"

    cat > /var/www/html/improvement-dashboard.html <<'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>Dashboard Amélioration Continue PRA</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { font-family: Arial; padding: 20px; background: #f0f0f0; }
        .container { max-width: 1400px; margin: 0 auto; }
        .metric-card {
            background: white;
            padding: 20px;
            margin: 10px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            display: inline-block;
            width: calc(25% - 20px);
        }
        .metric-value { font-size: 2em; font-weight: bold; color: #333; }
        .metric-label { color: #666; margin-top: 5px; }
        .trend-up { color: #4caf50; }
        .trend-down { color: #f44336; }
        .chart-container { background: white; padding: 20px; margin: 20px 10px; border-radius: 8px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>📈 Dashboard Amélioration Continue - PRA MicroK8s</h1>

        <div>
            <div class="metric-card">
                <div class="metric-value">42 min <span class="trend-up">↓15%</span></div>
                <div class="metric-label">RTO Moyen</div>
            </div>
            <div class="metric-card">
                <div class="metric-value">98.5% <span class="trend-up">↑2%</span></div>
                <div class="metric-label">Taux de Succès</div>
            </div>
            <div class="metric-card">
                <div class="metric-value">3.2h <span class="trend-down">↓8%</span></div>
                <div class="metric-label">MTTR</div>
            </div>
            <div class="metric-card">
                <div class="metric-value">156 <span class="trend-up">↑</span></div>
                <div class="metric-label">Jours sans incident majeur</div>
            </div>
        </div>

        <div class="chart-container">
            <h2>Évolution du RTO (12 derniers mois)</h2>
            <canvas id="rtoChart"></canvas>
        </div>

        <div class="chart-container">
            <h2>Actions d'Amélioration</h2>
            <table style="width: 100%;">
                <tr>
                    <th>Action</th>
                    <th>Statut</th>
                    <th>Impact</th>
                    <th>Date</th>
                </tr>
                <tr>
                    <td>Automatisation des backups</td>
                    <td>✅ Complété</td>
                    <td>RTO -20%</td>
                    <td>2024-01-15</td>
                </tr>
                <tr>
                    <td>Mise en place monitoring avancé</td>
                    <td>🔄 En cours</td>
                    <td>Détection +40%</td>
                    <td>2024-02-01</td>
                </tr>
                <tr>
                    <td>Formation équipe</td>
                    <td>📅 Planifié</td>
                    <td>MTTR -30%</td>
                    <td>2024-03-01</td>
                </tr>
            </table>
        </div>
    </div>

    <script>
        // Graphique RTO
        const ctx = document.getElementById('rtoChart').getContext('2d');
        new Chart(ctx, {
            type: 'line',
            data: {
                labels: ['Jan', 'Fév', 'Mar', 'Avr', 'Mai', 'Jun', 'Jul', 'Aoû', 'Sep', 'Oct', 'Nov', 'Déc'],
                datasets: [{
                    label: 'RTO Réel (minutes)',
                    data: [65, 62, 58, 55, 52, 48, 47, 45, 44, 43, 42, 42],
                    borderColor: '#4caf50',
                    tension: 0.1
                }, {
                    label: 'RTO Cible',
                    data: [60, 60, 60, 60, 60, 45, 45, 45, 45, 45, 30, 30],
                    borderColor: '#f44336',
                    borderDash: [5, 5]
                }]
            }
        });
    </script>
</body>
</html>
HTML
}
```

## Checklist finale et validation

### Checklist de validation du PRA

```bash
#!/bin/bash
# validate-drp.sh

validate_disaster_recovery_plan() {
    echo "✅ Validation du Plan de Reprise d'Activité"
    echo "==========================================="

    local score=0
    local max_score=20

    # Fonction de vérification
    check_item() {
        local item=$1
        local check_command=$2
        local description=$3

        echo -n "Vérification: $description... "

        if eval "$check_command" &>/dev/null; then
            echo "✅"
            ((score++))
            return 0
        else
            echo "❌"
            return 1
        fi
    }

    # 1. Documentation
    check_item "doc" "[ -f /etc/disaster-recovery/drp-plan.md ]" "Documentation PRA existe"
    check_item "runbook" "[ -f /etc/disaster-recovery/runbook.md ]" "Runbook d'urgence disponible"
    check_item "contacts" "[ -f /etc/disaster-recovery/contacts.txt ]" "Liste contacts à jour"

    # 2. Backups
    check_item "backup-config" "velero backup-location get" "Configuration backup validée"
    check_item "recent-backup" "[ $(velero backup get -o json | jq '.items | length') -gt 0 ]" "Backups récents présents"
    check_item "backup-schedule" "velero schedule get | grep -q Enabled" "Backups programmés actifs"

    # 3. Monitoring
    check_item "prometheus" "kubectl get deployment prometheus -n monitoring" "Prometheus déployé"
    check_item "alertmanager" "kubectl get deployment alertmanager -n monitoring" "AlertManager configuré"
    check_item "alerts" "[ -f /etc/prometheus/alerts/drp-alerts.yml ]" "Alertes PRA définies"

    # 4. Tests
    check_item "test-env" "kubectl get namespace restore-test" "Environnement de test existe"
    check_item "test-logs" "find /var/log/disaster-recovery -name 'test-*.log' -mtime -30 | grep -q ." "Tests récents effectués"
    check_item "test-schedule" "[ -f /etc/cron.d/drp-tests ]" "Tests programmés configurés"

    # 5. Procédures
    check_item "failover-script" "[ -x /usr/local/bin/failover.sh ]" "Script failover disponible"
    check_item "restore-script" "[ -x /usr/local/bin/restore.sh ]" "Script restoration disponible"
    check_item "notification" "[ -n \"$NOTIFICATION_EMAIL\" ] || [ -n \"$SLACK_WEBHOOK\" ]" "Notifications configurées"

    # 6. Objectifs
    check_item "rto-defined" "[ -f /etc/disaster-recovery/rto-rpo.yaml ]" "RTO/RPO définis"
    check_item "tier-classification" "[ -f /etc/disaster-recovery/service-tiers.yaml ]" "Services classifiés par criticité"

    # 7. Formation
    check_item "training-log" "[ -f /var/log/disaster-recovery/training.log ]" "Formation documentée"
    check_item "team-ready" "[ -f /etc/disaster-recovery/team-roles.yaml ]" "Rôles équipe définis"

    # Résultat
    echo ""
    echo "========================================="
    echo "Score: $score/$max_score"

    local percentage=$((score * 100 / max_score))

    if [ $percentage -ge 90 ]; then
        echo "🏆 EXCELLENT - PRA complet et opérationnel"
    elif [ $percentage -ge 75 ]; then
        echo "✅ BON - PRA fonctionnel, quelques améliorations possibles"
    elif [ $percentage -ge 60 ]; then
        echo "⚠️  MOYEN - PRA basique, améliorations nécessaires"
    else
        echo "❌ INSUFFISANT - PRA incomplet, actions urgentes requises"
    fi

    # Recommandations
    echo ""
    echo "📋 Recommandations:"

    if [ $score -lt $max_score ]; then
        [ ! -f /etc/disaster-recovery/drp-plan.md ] && echo "  • Créer la documentation PRA complète"
        [ $(velero backup get -o json | jq '.items | length') -eq 0 ] && echo "  • Configurer les backups automatiques"
        [ ! -f /etc/cron.d/drp-tests ] && echo "  • Planifier des tests réguliers"
        [ ! -f /etc/disaster-recovery/team-roles.yaml ] && echo "  • Former l'équipe et définir les rôles"
    else
        echo "  • Maintenir les tests réguliers"
        echo "  • Réviser le plan trimestriellement"
        echo "  • Continuer la formation de l'équipe"
    fi
}

# Générer un certificat de conformité
generate_compliance_certificate() {
    local validation_date=$(date +%Y-%m-%d)
    local next_review=$(date -d "+90 days" +%Y-%m-%d)

    cat > /etc/disaster-recovery/compliance-certificate.txt <<EOF
╔══════════════════════════════════════════════════════════════╗
║            CERTIFICAT DE CONFORMITÉ PRA                      ║
╚══════════════════════════════════════════════════════════════╝

Organisation: MicroK8s Lab Personnel
Date de validation: $validation_date
Prochaine révision: $next_review

ÉLÉMENTS VALIDÉS:
✓ Plan de Reprise d'Activité documenté
✓ Objectifs RTO/RPO définis et mesurables
✓ Stratégie de sauvegarde multi-niveaux
✓ Procédures de restauration testées
✓ Équipe formée et rôles définis
✓ Tests réguliers programmés
✓ Monitoring et alerting opérationnels

CONFORMITÉ:
• ISO 22301 (Business Continuity): Partielle
• Best Practices Kubernetes: Conforme
• Standards internes: Conforme

Validé par: $(whoami)
Signature: _____________________

Ce certificat atteste que le Plan de Reprise d'Activité
répond aux exigences minimales pour assurer la continuité
des services en cas d'incident majeur.

════════════════════════════════════════════════════════════════
EOF

    echo "📜 Certificat généré: /etc/disaster-recovery/compliance-certificate.txt"
}

# Exécution
validate_disaster_recovery_plan
generate_compliance_certificate
```

### Métriques de succès du PRA

```yaml
# drp-success-metrics.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drp-success-metrics
  namespace: monitoring
data:
  metrics.yaml: |
    # Métriques de Succès du PRA

    operational_metrics:
      # Temps de récupération
      rto_achieved:
        target: "< 1 heure"
        current: "42 minutes"
        trend: "improving"

      # Point de récupération
      rpo_achieved:
        target: "< 1 heure"
        current: "15 minutes"
        trend: "stable"

      # Disponibilité
      availability:
        target: "99.9%"
        current: "99.95%"
        trend: "stable"

    process_metrics:
      # Tests
      test_frequency:
        target: "Mensuel"
        current: "Bi-mensuel"
        compliance: "exceeds"

      test_success_rate:
        target: "> 95%"
        current: "98.5%"
        compliance: "meets"

      # Documentation
      documentation_currency:
        target: "< 30 jours"
        last_update: "15 jours"
        compliance: "meets"

    team_metrics:
      # Formation
      team_trained:
        target: "100%"
        current: "100%"
        last_training: "2 mois"

      # Temps de réponse
      incident_response_time:
        target: "< 15 minutes"
        average: "8 minutes"
        compliance: "exceeds"

    improvement_metrics:
      # Réduction des incidents
      incident_reduction:
        baseline: "10/mois"
        current: "3/mois"
        improvement: "70%"

      # Automatisation
      automation_level:
        baseline: "30%"
        current: "75%"
        target: "85%"

      # Coût de récupération
      recovery_cost:
        baseline: "4 heures-homme"
        current: "1.5 heures-homme"
        saving: "62.5%"
```

## Conclusion et recommandations

### Résumé du PRA

```markdown
# Résumé Exécutif - Plan de Reprise d'Activité MicroK8s

## Vue d'ensemble
Le Plan de Reprise d'Activité (PRA) pour l'environnement MicroK8s fournit un cadre complet pour assurer la continuité des services en cas d'incident majeur. Ce plan couvre tous les aspects critiques de la récupération, de la détection initiale à la restauration complète.

## Points clés

### ✅ Forces du PRA
- **Objectifs clairs**: RTO < 1h, RPO < 1h pour services critiques
- **Automatisation**: 75% des procédures automatisées
- **Tests réguliers**: Programme de tests mensuel établi
- **Documentation complète**: Runbooks, procédures, contacts à jour
- **Formation continue**: Équipe formée et exercices réguliers

### 🎯 Objectifs atteints
- Réduction du temps de récupération de 65% en 12 mois
- Augmentation du taux de succès des restaurations à 98.5%
- Zéro perte de données critique sur les 6 derniers mois
- 100% de l'équipe formée aux procédures d'urgence

### 📈 Améliorations continues
- Migration vers une architecture HA en cours
- Automatisation avancée avec IA/ML planifiée
- Intégration cloud hybride en développement
- Certification ISO 22301 visée pour 2025

## Recommandations prioritaires

### Court terme (0-3 mois)
1. **Augmenter la fréquence des tests**: Passer à des tests hebdomadaires pour les systèmes critiques
2. **Améliorer le monitoring**: Déployer des sondes synthétiques pour détection proactive
3. **Optimiser les backups**: Implémenter la déduplication pour réduire l'espace de stockage

### Moyen terme (3-6 mois)
1. **Architecture multi-site**: Établir un site de reprise secondaire
2. **Orchestration avancée**: Implémenter Ansible/Terraform pour l'IaC complet
3. **Formation avancée**: Certifications Kubernetes pour l'équipe

### Long terme (6-12 mois)
1. **Cloud hybride**: Extension vers AWS/GCP pour resilience maximale
2. **AI/ML pour prédiction**: Détection d'anomalies et prévention des pannes
3. **Conformité complète**: Obtenir les certifications ISO pertinentes

## Budget et ROI

### Investissement actuel
- Infrastructure: ~500€/an (stockage, backup)
- Outils: ~200€/an (licences, cloud)
- Formation: ~1000€/an (cours, certifications)
- **Total**: ~1700€/an

### Retour sur investissement
- Temps économisé: 200 heures/an (~10,000€ valeur)
- Incidents évités: 5-10 majeurs/an
- Tranquillité d'esprit: Inestimable
- **ROI**: >500%

## Prochaines étapes

1. **Semaine 1**: Valider tous les contacts d'urgence
2. **Semaine 2**: Exécuter un test de failover complet
3. **Mois 1**: Réviser et mettre à jour toute la documentation
4. **Trimestre 1**: Implémenter les recommandations court terme
5. **Année 1**: Atteindre tous les objectifs de maturité

## Conclusion

Le Plan de Reprise d'Activité MicroK8s est opérationnel et efficace. Avec une amélioration continue et des investissements ciblés, nous pouvons atteindre un niveau de résilience enterprise-grade tout en maintenant la simplicité et l'efficacité d'un environnement de lab.

**Message clé**: Un PRA n'est pas une assurance qu'on espère ne jamais utiliser, c'est un muscle qu'on entraîne régulièrement pour qu'il soit fort quand on en a besoin.

---

*"La meilleure façon de prédire l'avenir est de le créer" - Peter Drucker*

*Dans le contexte du disaster recovery: La meilleure façon de survivre à un désastre est de s'y préparer.*
```

### Checklist finale complète

```bash
#!/bin/bash
# final-drp-checklist.sh

echo "📋 CHECKLIST FINALE - PLAN DE REPRISE D'ACTIVITÉ"
echo "================================================="
echo ""
echo "□ ANALYSE ET PLANIFICATION"
echo "  □ Analyse des risques complétée"
echo "  □ Impact business évalué"
echo "  □ RTO/RPO définis pour chaque service"
echo "  □ Budget alloué"
echo ""
echo "□ STRATÉGIE ET ARCHITECTURE"
echo "  □ Stratégie de sauvegarde multi-niveaux"
echo "  □ Architecture HA documentée"
echo "  □ Sites de reprise identifiés"
echo "  □ Redondance implémentée"
echo ""
echo "□ PROCÉDURES ET DOCUMENTATION"
echo "  □ Runbook d'urgence créé"
echo "  □ Procédures de restauration documentées"
echo "  □ Contacts d'urgence à jour"
echo "  □ Documentation accessible hors ligne"
echo ""
echo "□ OUTILS ET AUTOMATISATION"
echo "  □ Velero configuré et opérationnel"
echo "  □ Scripts de failover testés"
echo "  □ Monitoring actif 24/7"
echo "  □ Alerting configuré"
echo ""
echo "□ TESTS ET VALIDATION"
echo "  □ Tests mensuels programmés"
echo "  □ Simulations trimestrielles"
echo "  □ Exercice annuel complet"
echo "  □ Métriques de succès suivies"
echo ""
echo "□ ÉQUIPE ET FORMATION"
echo "  □ Rôles et responsabilités définis"
echo "  □ Formation initiale complétée"
echo "  □ Exercices réguliers"
echo "  □ Certification/validation des compétences"
echo ""
echo "□ AMÉLIORATION CONTINUE"
echo "  □ Process d'amélioration en place"
echo "  □ KPIs définis et suivis"
echo "  □ Revues régulières"
echo "  □ Benchmark avec les best practices"
echo ""
echo "================================================="
echo "✅ Votre PRA est prêt à protéger votre infrastructure!"
```

Le Plan de Reprise d'Activité est maintenant complet. Il couvre tous les aspects essentiels pour assurer la continuité de votre environnement MicroK8s, depuis l'analyse des risques jusqu'à l'amélioration continue. Rappelez-vous : un PRA n'est jamais "terminé" - c'est un document vivant qui doit évoluer avec votre infrastructure et être testé régulièrement pour garantir son efficacité.

⏭️
