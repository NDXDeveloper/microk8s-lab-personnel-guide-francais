🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7 Bonnes pratiques sécurité

## Vue d'ensemble de la sécurité Kubernetes

La sécurité dans Kubernetes n'est pas une fonctionnalité qu'on active, c'est une approche globale qui touche chaque aspect de votre cluster. Pensez à la sécurité comme à la construction d'un château fort : vous avez besoin de murs solides (infrastructure), de gardes vigilants (monitoring), de portes verrouillées (authentification), de passages secrets sécurisés (chiffrement), et d'un plan de défense (incident response).

### Les principes fondamentaux

**Defense in Depth** : Plusieurs couches de sécurité plutôt qu'une seule barrière
**Least Privilege** : Accorder uniquement les permissions strictement nécessaires
**Zero Trust** : Ne jamais faire confiance, toujours vérifier
**Shift Left** : Intégrer la sécurité dès le début du développement
**Continuous Improvement** : La sécurité évolue constamment

## Architecture sécurisée de référence

### Schéma de sécurité multicouche

```yaml
# Architecture de sécurité complète
security_layers:
  1_perimeter:
    - firewall: "Filtrage des IPs sources"
    - loadbalancer: "DDoS protection"
    - waf: "Web Application Firewall"

  2_cluster:
    - api_server: "Authentification et autorisation"
    - admission_controllers: "Validation et mutation"
    - network_policies: "Segmentation réseau"

  3_nodes:
    - os_hardening: "CIS benchmarks"
    - kernel_security: "AppArmor/SELinux"
    - container_runtime: "Rootless containers"

  4_workloads:
    - pod_security: "SecurityContext strict"
    - secrets_management: "Vault/External Secrets"
    - image_scanning: "Vulnerability detection"

  5_data:
    - encryption_at_rest: "etcd encryption"
    - encryption_in_transit: "mTLS everywhere"
    - backup_encryption: "Encrypted backups"
```

## Sécurisation de l'infrastructure

### Hardening du système hôte

```bash
#!/bin/bash
# os-hardening.sh - Script de durcissement pour Ubuntu/Debian

echo "=== OS Hardening Script ==="

# 1. Mise à jour système
echo "1. Updating system packages..."
apt update && apt upgrade -y
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades

# 2. Configuration du firewall
echo "2. Configuring firewall..."
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp  # SSH
ufw allow 16443/tcp  # Kubernetes API
ufw allow 10250:10255/tcp  # Kubelet
ufw --force enable

# 3. Désactiver les services non nécessaires
echo "3. Disabling unnecessary services..."
systemctl disable bluetooth.service
systemctl disable cups.service
systemctl disable avahi-daemon.service

# 4. Configuration sysctl pour la sécurité
echo "4. Applying sysctl security settings..."
cat >> /etc/sysctl.d/99-kubernetes-security.conf <<EOF
# Protection contre les attaques réseau
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Protection contre l'épuisement mémoire
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1

# Augmentation des limites pour Kubernetes
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
EOF
sysctl -p /etc/sysctl.d/99-kubernetes-security.conf

# 5. Configuration des limites
echo "5. Setting resource limits..."
cat >> /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 65536
* soft nproc 32768
* hard nproc 32768
* soft memlock unlimited
* hard memlock unlimited
EOF

# 6. Installation d'outils de sécurité
echo "6. Installing security tools..."
apt install -y \
  aide \
  auditd \
  fail2ban \
  rkhunter \
  clamav \
  apparmor-utils

# 7. Configuration de l'audit
echo "7. Configuring auditd..."
cat >> /etc/audit/rules.d/kubernetes.rules <<EOF
# Surveillance des fichiers Kubernetes
-w /var/snap/microk8s/current/ -p wa -k microk8s_changes
-w /etc/kubernetes/ -p wa -k kubernetes_config
-w /etc/systemd/system/ -p wa -k systemd_changes

# Surveillance des commandes privilégiées
-a always,exit -F arch=b64 -S execve -F uid=0 -k root_commands
EOF
systemctl restart auditd

echo "=== Hardening Complete ==="
```

### Configuration AppArmor pour les conteneurs

```yaml
# apparmor-profile.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apparmor-profiles
  namespace: kube-system
data:
  k8s-default: |
    #include <tunables/global>

    profile k8s-default flags=(attach_disconnected,mediate_deleted) {
      #include <abstractions/base>

      # Réseau
      network inet tcp,
      network inet udp,
      network inet icmp,
      network inet6 tcp,
      network inet6 udp,

      # Système de fichiers - Lecture seule pour la plupart
      / r,
      /** r,

      # Écriture limitée
      /tmp/** rw,
      /var/tmp/** rw,
      /run/** rw,

      # Deny access to sensitive files
      deny /etc/shadow r,
      deny /etc/passwd w,
      deny /etc/sudoers r,
      deny /root/** rwx,

      # Deny kernel module operations
      deny capability sys_module,
      deny capability sys_admin,

      # Allow limited capabilities
      capability net_bind_service,
      capability setuid,
      capability setgid,
    }
```

## Configuration sécurisée de MicroK8s

### Activation des fonctionnalités de sécurité

```bash
#!/bin/bash
# secure-microk8s-setup.sh

echo "=== Securing MicroK8s Installation ==="

# 1. Activer RBAC
echo "1. Enabling RBAC..."
microk8s enable rbac

# 2. Activer Pod Security Policies (ou Pod Security Standards)
echo "2. Configuring Pod Security..."
microk8s kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# 3. Activer l'audit logging
echo "3. Enabling audit logging..."
cat > /var/snap/microk8s/current/args/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    omitStages:
      - RequestReceived
EOF

echo "--audit-policy-file=/var/snap/microk8s/current/args/audit-policy.yaml" >> /var/snap/microk8s/current/args/kube-apiserver
echo "--audit-log-path=/var/log/microk8s/audit.log" >> /var/snap/microk8s/current/args/kube-apiserver
echo "--audit-log-maxage=30" >> /var/snap/microk8s/current/args/kube-apiserver

# 4. Configuration de l'API server
echo "4. Hardening API server..."
cat >> /var/snap/microk8s/current/args/kube-apiserver <<EOF
--anonymous-auth=false
--enable-admission-plugins=NodeRestriction,ResourceQuota,ServiceAccount,AlwaysPullImages,SecurityContextDeny
--tls-min-version=VersionTLS12
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
--request-timeout=60s
--service-account-lookup=true
EOF

# 5. Configuration du kubelet
echo "5. Hardening kubelet..."
cat >> /var/snap/microk8s/current/args/kubelet <<EOF
--anonymous-auth=false
--authorization-mode=Webhook
--read-only-port=0
--streaming-connection-idle-timeout=5m
--protect-kernel-defaults=true
--event-qps=0
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
EOF

# 6. Redémarrer MicroK8s
echo "6. Restarting MicroK8s..."
microk8s stop
microk8s start

echo "=== Security configuration complete ==="
```

### Mise en place du Network Policies par défaut

```yaml
# default-network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
# Autoriser uniquement le trafic interne au namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
---
# Isoler les namespaces système
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-kube-system
  namespace: kube-system
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          component: kube-apiserver
```

## Sécurisation des workloads

### Template de déploiement sécurisé

```yaml
# secure-deployment-template.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
  labels:
    app: secure-app
    security-scan: "passed"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
      annotations:
        # Annotations de sécurité
        container.apparmor.security.beta.kubernetes.io/app: runtime/default
        seccomp.security.alpha.kubernetes.io/pod: runtime/default
    spec:
      # Service Account dédié
      serviceAccountName: secure-app-sa
      automountServiceAccountToken: false

      # Security Context au niveau du Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
        supplementalGroups: [10001]

      # Affinité pour distribution
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - secure-app
              topologyKey: kubernetes.io/hostname

      initContainers:
      # Scan de vulnérabilités au démarrage
      - name: vulnerability-scan
        image: aquasec/trivy:latest
        command:
        - trivy
        - image
        - --exit-code
        - "0"
        - --severity
        - "HIGH,CRITICAL"
        - "myapp:latest"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
            - ALL

      containers:
      - name: app
        image: myapp:latest
        imagePullPolicy: Always

        # Security Context du conteneur
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # Seulement si nécessaire

        # Ressources
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"

        # Health checks
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTPS
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
            scheme: HTTPS
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3

        # Variables d'environnement depuis secrets
        envFrom:
        - secretRef:
            name: app-secrets
        - configMapRef:
            name: app-config

        # Montages
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true

      # Volumes
      volumes:
      - name: tmp
        emptyDir:
          sizeLimit: 1Gi
      - name: cache
        emptyDir:
          sizeLimit: 2Gi
      - name: tls-certs
        secret:
          secretName: app-tls
          defaultMode: 0400
```

### Dockerfile sécurisé

```dockerfile
# Dockerfile sécurisé avec multi-stage build
# Stage 1: Build
FROM golang:1.21-alpine AS builder

# Installation des dépendances de build uniquement
RUN apk add --no-cache git ca-certificates tzdata

# Utilisateur non-root pour le build
RUN adduser -D -g '' appuser

WORKDIR /build

# Copie des dépendances d'abord (cache Docker)
COPY go.mod go.sum ./
RUN go mod download

# Copie du code source
COPY . .

# Build avec flags de sécurité
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static"' \
    -a -installsuffix cgo \
    -o app .

# Stage 2: Runtime minimal
FROM scratch

# Import depuis le builder
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/passwd /etc/passwd

# Copie de l'application
COPY --from=builder /build/app /app

# Utilisateur non-root
USER appuser

# Port non-privilégié
EXPOSE 8080

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/app", "healthcheck"]

# Démarrage
ENTRYPOINT ["/app"]
```

## Gestion sécurisée des secrets

### Architecture de secrets Zero-Trust

```yaml
# secret-management-architecture.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: secret-architecture
  namespace: security
data:
  architecture.yaml: |
    components:
      external_vault:
        type: "HashiCorp Vault"
        features:
          - auto_rotation: true
          - dynamic_secrets: true
          - encryption: "AES-256-GCM"
          - audit: comprehensive

      operators:
        - name: "External Secrets Operator"
          sync_interval: "1m"
          retry_policy: "exponential"

        - name: "Sealed Secrets"
          encryption: "RSA-2048"
          rotation: "30d"

      policies:
        rotation:
          database_passwords: "30d"
          api_keys: "90d"
          certificates: "365d"

        access:
          principle: "least_privilege"
          authentication: "mTLS"
          authorization: "RBAC"
```

### Rotation automatique des credentials

```bash
#!/bin/bash
# rotate-credentials.sh

# Fonction pour générer un mot de passe sécurisé
generate_password() {
    openssl rand -base64 32 | tr -d "=+/" | cut -c1-25
}

# Rotation des secrets de base de données
rotate_database_credentials() {
    local namespace=$1
    local secret_name=$2

    echo "Rotating database credentials for $namespace/$secret_name"

    # Générer nouveau mot de passe
    NEW_PASSWORD=$(generate_password)

    # Mettre à jour dans Kubernetes
    kubectl create secret generic ${secret_name}-new \
        --from-literal=password=$NEW_PASSWORD \
        --namespace=$namespace \
        --dry-run=client -o yaml | kubectl apply -f -

    # Mettre à jour la base de données
    kubectl exec -n $namespace deployment/database -- \
        mysql -u root -p$OLD_PASSWORD \
        -e "ALTER USER 'app'@'%' IDENTIFIED BY '$NEW_PASSWORD';"

    # Redéployer l'application
    kubectl rollout restart deployment/app -n $namespace

    # Supprimer l'ancien secret après vérification
    kubectl delete secret $secret_name -n $namespace
    kubectl patch secret ${secret_name}-new -n $namespace \
        -p '{"metadata":{"name":"'$secret_name'"}}'
}

# Rotation des certificats TLS
rotate_tls_certificates() {
    local namespace=$1
    local domain=$2

    echo "Rotating TLS certificates for $domain"

    # Générer nouveau certificat avec cert-manager
    cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ${domain}-tls-new
  namespace: $namespace
spec:
  secretName: ${domain}-tls-new
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - $domain
  - www.$domain
EOF

    # Attendre que le certificat soit prêt
    kubectl wait --for=condition=ready \
        certificate/${domain}-tls-new \
        -n $namespace --timeout=300s

    # Mettre à jour l'Ingress
    kubectl patch ingress ${domain}-ingress -n $namespace \
        --type='json' \
        -p='[{"op": "replace", "path": "/spec/tls/0/secretName", "value":"'${domain}-tls-new'"}]'
}

# Exécution principale
NAMESPACES=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}')

for ns in $NAMESPACES; do
    # Ignorer les namespaces système
    if [[ "$ns" == kube-* ]]; then
        continue
    fi

    # Rotation des secrets de l'application
    if kubectl get secret app-db-credentials -n $ns &>/dev/null; then
        rotate_database_credentials $ns app-db-credentials
    fi

    # Rotation des certificats
    if kubectl get ingress -n $ns &>/dev/null; then
        DOMAIN=$(kubectl get ingress -n $ns -o jsonpath='{.items[0].spec.rules[0].host}')
        if [ -n "$DOMAIN" ]; then
            rotate_tls_certificates $ns $DOMAIN
        fi
    fi
done

echo "Credential rotation complete"
```

## Monitoring et détection

### Stack de sécurité complète

```yaml
# security-monitoring-stack.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: security-monitoring
---
# Falco pour la détection runtime
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security-monitoring
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-socket
          mountPath: /host/var/run/docker.sock
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: boot
          mountPath: /host/boot
          readOnly: true
        - name: lib-modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: usr
          mountPath: /host/usr
          readOnly: true
        - name: etc
          mountPath: /host/etc
          readOnly: true
        - name: config
          mountPath: /etc/falco
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: dev
        hostPath:
          path: /dev
      - name: proc
        hostPath:
          path: /proc
      - name: boot
        hostPath:
          path: /boot
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: usr
        hostPath:
          path: /usr
      - name: etc
        hostPath:
          path: /etc
      - name: config
        configMap:
          name: falco-config
```

### Règles de détection personnalisées

```yaml
# falco-custom-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: security-monitoring
data:
  custom-rules.yaml: |
    - rule: Unauthorized Process in Container
      desc: Detect unauthorized process execution
      condition: >
        spawned_process and
        container and
        not proc.name in (allowed_processes)
      output: >
        Unauthorized process started in container
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: WARNING
      tags: [process, container]

    - rule: Sensitive File Access
      desc: Detect access to sensitive files
      condition: >
        open_read and
        (fd.name startswith /etc/shadow or
         fd.name startswith /etc/passwd or
         fd.name startswith /root/)
      output: >
        Sensitive file accessed
        (user=%user.name file=%fd.name container=%container.name)
      priority: CRITICAL
      tags: [filesystem, secrets]

    - rule: Container Escape Attempt
      desc: Detect potential container escape
      condition: >
        syscall.type in (mount, umount2) and
        container
      output: >
        Potential container escape attempt
        (user=%user.name syscall=%syscall.type container=%container.name)
      priority: CRITICAL
      tags: [escape, container]

    - rule: Crypto Mining Detection
      desc: Detect crypto mining activity
      condition: >
        spawned_process and
        (proc.name in (cryptomining_binaries) or
         proc.cmdline contains "stratum+tcp")
      output: >
        Crypto mining detected
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: CRITICAL
      tags: [cryptomining, malware]
```

## Incident Response

### Plan de réponse aux incidents

```yaml
# incident-response-plan.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: incident-response-plan
  namespace: security
data:
  plan.yaml: |
    incident_levels:
      P1_Critical:
        description: "Compromission active, fuite de données"
        response_time: "15 minutes"
        team: ["security-lead", "platform-lead", "cto"]

      P2_High:
        description: "Tentative d'intrusion, vulnérabilité critique"
        response_time: "1 heure"
        team: ["security-team", "platform-team"]

      P3_Medium:
        description: "Comportement suspect, vulnérabilité moyenne"
        response_time: "4 heures"
        team: ["security-team"]

      P4_Low:
        description: "Anomalie mineure, faux positif potentiel"
        response_time: "24 heures"
        team: ["security-analyst"]

    response_steps:
      1_Detection:
        - alert_triggered
        - initial_assessment
        - severity_classification

      2_Containment:
        - isolate_affected_resources
        - preserve_evidence
        - prevent_spread

      3_Eradication:
        - identify_root_cause
        - remove_threat
        - patch_vulnerabilities

      4_Recovery:
        - restore_services
        - verify_integrity
        - monitor_closely

      5_Lessons_Learned:
        - post_mortem_analysis
        - update_procedures
        - improve_detection
```

### Script de réponse automatique

```bash
#!/bin/bash
# incident-response.sh

INCIDENT_TYPE=$1
NAMESPACE=$2
POD=$3

case $INCIDENT_TYPE in
  "suspicious-exec")
    echo "Responding to suspicious exec in $NAMESPACE/$POD"

    # 1. Isoler le pod
    kubectl label pod $POD -n $NAMESPACE quarantine=true
    kubectl patch pod $POD -n $NAMESPACE \
      -p '{"spec":{"nodeSelector":{"quarantine":"true"}}}'

    # 2. Capturer les logs
    kubectl logs $POD -n $NAMESPACE --all-containers > /tmp/incident-$POD-logs.txt
    kubectl describe pod $POD -n $NAMESPACE > /tmp/incident-$POD-describe.txt

    # 3. Créer une copie forensique
    kubectl debug $POD -n $NAMESPACE -it --image=busybox --share-processes -- \
      tar czf /tmp/forensic-backup.tar.gz /proc /etc

    # 4. Notifier l'équipe
    curl -X POST https://hooks.slack.com/services/XXX \
      -H 'Content-Type: application/json' \
      -d "{\"text\":\"🚨 Incident: Suspicious exec detected in $NAMESPACE/$POD\"}"
    ;;

  "privilege-escalation")
    echo "Responding to privilege escalation attempt"

    # 1. Bloquer immédiatement
    kubectl delete pod $POD -n $NAMESPACE --grace-period=0 --force

    # 2. Vérifier les RBAC
    kubectl get rolebindings,clusterrolebindings -A | grep $NAMESPACE

    # 3. Audit trail
    grep $POD /var/log/microk8s/audit/audit.log > /tmp/incident-audit.log
    ;;
esac

# Créer un rapport d'incident
cat > /tmp/incident-report-$(date +%Y%m%d-%H%M%S).md <<EOF
# Incident Report

**Date**: $(date)
**Type**: $INCIDENT_TYPE
**Location**: $NAMESPACE/$POD

## Actions Taken
1. Pod isolated/terminated
2. Logs collected
3. Team notified

## Evidence
- Logs: /tmp/incident-$POD-logs.txt
- Description: /tmp/incident-$POD-describe.txt
- Audit: /tmp/incident-audit.log

## Next Steps
- Review collected evidence
- Identify root cause
- Update security policies
EOF

echo "Incident response completed. Report saved to /tmp/"
```

## Conformité et audits

### Checklist CIS Benchmark

```bash
#!/bin/bash
# cis-benchmark-check.sh

echo "=== CIS Kubernetes Benchmark Check ==="
echo

# Score global
PASSED=0
FAILED=0

# Fonction de vérification
check() {
    local description=$1
    local command=$2
    local expected=$3

    result=$(eval $command 2>/dev/null)
    if [[ "$result" == *"$expected"* ]]; then
        echo "✓ $description"
        ((PASSED++))
    else
        echo "✗ $description"
        echo "  Expected: $expected"
        echo "  Got: $result"
        ((FAILED++))
    fi
}

# 1. Control Plane Security Configuration
echo "1. Control Plane Security"
check "1.1 API Server: Anonymous auth disabled" \
    "grep 'anonymous-auth' /var/snap/microk8s/current/args/kube-apiserver" \
    "false"

check "1.2 API Server: RBAC enabled" \
    "microk8s status | grep rbac" \
    "enabled"

check "1.3 API Server: Audit logging configured" \
    "grep 'audit-log-path' /var/snap/microk8s/current/args/kube-apiserver" \
    "audit.log"

check "1.4 API Server: TLS version >= 1.2" \
    "grep 'tls-min-version' /var/snap/microk8s/current/args/kube-apiserver" \
    "VersionTLS12"

check "1.5 API Server: Strong cipher suites" \
    "grep 'tls-cipher-suites' /var/snap/microk8s/current/args/kube-apiserver" \
    "TLS_ECDHE"

check "1.6 API Server: AlwaysPullImages admission" \
    "grep 'enable-admission-plugins' /var/snap/microk8s/current/args/kube-apiserver | grep -q AlwaysPullImages && echo 'true'" \
    "true"

# 2. etcd Security
echo ""
echo "2. etcd Security"
check "2.1 etcd: Client cert auth enabled" \
    "grep 'client-cert-auth' /var/snap/microk8s/current/args/etcd" \
    "true"

check "2.2 etcd: Auto TLS disabled" \
    "grep 'auto-tls' /var/snap/microk8s/current/args/etcd || echo 'not-set'" \
    "not-set"

check "2.3 etcd: Peer cert auth enabled" \
    "grep 'peer-client-cert-auth' /var/snap/microk8s/current/args/etcd" \
    "true"

# 3. Kubelet Security
echo ""
echo "3. Kubelet Security"
check "3.1 Kubelet: Anonymous auth disabled" \
    "grep 'anonymous-auth' /var/snap/microk8s/current/args/kubelet" \
    "false"

check "3.2 Kubelet: Authorization mode Webhook" \
    "grep 'authorization-mode' /var/snap/microk8s/current/args/kubelet" \
    "Webhook"

check "3.3 Kubelet: Read only port disabled" \
    "grep 'read-only-port' /var/snap/microk8s/current/args/kubelet" \
    "0"

check "3.4 Kubelet: Protect kernel defaults" \
    "grep 'protect-kernel-defaults' /var/snap/microk8s/current/args/kubelet" \
    "true"

# 4. Network Policies
echo ""
echo "4. Network Policies"
NETPOL_COUNT=$(microk8s kubectl get networkpolicies --all-namespaces --no-headers 2>/dev/null | wc -l)
if [ "$NETPOL_COUNT" -gt 0 ]; then
    echo "✓ 4.1 Network policies exist ($NETPOL_COUNT found)"
    ((PASSED++))
else
    echo "✗ 4.1 No network policies found"
    ((FAILED++))
fi

# 5. Pod Security
echo ""
echo "5. Pod Security Standards"
PSS_COUNT=$(microk8s kubectl get namespaces -o json | jq '.items[].metadata.labels' | grep -c pod-security 2>/dev/null || echo 0)
if [ "$PSS_COUNT" -gt 0 ]; then
    echo "✓ 5.1 Pod Security Standards enabled ($PSS_COUNT namespaces)"
    ((PASSED++))
else
    echo "✗ 5.1 Pod Security Standards not configured"
    ((FAILED++))
fi

# 6. Secrets Management
echo ""
echo "6. Secrets Management"
check "6.1 Encryption at rest configured" \
    "grep 'encryption-provider-config' /var/snap/microk8s/current/args/kube-apiserver" \
    "encryption"

# 7. RBAC Configuration
echo ""
echo "7. RBAC Configuration"
DEFAULT_SA=$(microk8s kubectl get serviceaccount default -n default -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)
if [ "$DEFAULT_SA" == "false" ]; then
    echo "✓ 7.1 Default ServiceAccount token mounting disabled"
    ((PASSED++))
else
    echo "✗ 7.1 Default ServiceAccount token mounting not disabled"
    ((FAILED++))
fi

# 8. Resource Quotas
echo ""
echo "8. Resource Management"
QUOTA_COUNT=$(microk8s kubectl get resourcequotas --all-namespaces --no-headers 2>/dev/null | wc -l)
if [ "$QUOTA_COUNT" -gt 0 ]; then
    echo "✓ 8.1 Resource quotas configured ($QUOTA_COUNT found)"
    ((PASSED++))
else
    echo "⚠ 8.1 No resource quotas found (recommended)"
fi

# Résultat final
echo ""
echo "=== Results ==="
echo "Passed: $PASSED"
echo "Failed: $FAILED"
SCORE=$((PASSED * 100 / (PASSED + FAILED)))
echo "Compliance Score: ${SCORE}%"

# Génération du rapport
REPORT_FILE="cis-compliance-$(date +%Y%m%d).json"
cat > $REPORT_FILE <<EOF
{
  "date": "$(date --iso-8601=seconds)",
  "score": $SCORE,
  "passed": $PASSED,
  "failed": $FAILED,
  "details": {
    "control_plane": $([ $PASSED -gt 0 ] && echo "true" || echo "false"),
    "etcd": $(grep -q 'client-cert-auth=true' /var/snap/microk8s/current/args/etcd 2>/dev/null && echo "true" || echo "false"),
    "kubelet": $(grep -q 'anonymous-auth=false' /var/snap/microk8s/current/args/kubelet 2>/dev/null && echo "true" || echo "false"),
    "network_policies": $([ $NETPOL_COUNT -gt 0 ] && echo "true" || echo "false"),
    "pod_security": $([ $PSS_COUNT -gt 0 ] && echo "true" || echo "false")
  }
}
EOF

echo ""
echo "Report saved to: $REPORT_FILE"

if [ $SCORE -lt 80 ]; then
    echo ""
    echo "⚠️  Warning: Compliance score below 80%. Review failed checks."
    echo ""
    echo "Recommended actions:"
    echo "1. Enable missing security features"
    echo "2. Configure network policies"
    echo "3. Implement Pod Security Standards"
    echo "4. Review and harden API server configuration"
fi
```

### Audit de conformité GDPR/SOC2

```bash
#!/bin/bash
# compliance-audit.sh

echo "=== Compliance Audit Report ==="
echo "Date: $(date)"
echo ""

# Fonction pour vérifier la conformité
audit_check() {
    local category=$1
    local check=$2
    local command=$3

    echo -n "[$category] $check: "
    if eval $command > /dev/null 2>&1; then
        echo "✓ COMPLIANT"
        return 0
    else
        echo "✗ NON-COMPLIANT"
        return 1
    fi
}

# GDPR Compliance
echo "== GDPR Compliance =="
audit_check "GDPR" "Data encryption at rest" \
    "grep -q encryption-provider-config /var/snap/microk8s/current/args/kube-apiserver"

audit_check "GDPR" "Audit logging enabled" \
    "test -f /var/log/microk8s/audit/audit.log"

audit_check "GDPR" "Access controls (RBAC)" \
    "microk8s kubectl get clusterrolebindings | grep -q admin"

audit_check "GDPR" "Data retention policy" \
    "grep -q audit-log-maxage /var/snap/microk8s/current/args/kube-apiserver"

# SOC2 Compliance
echo ""
echo "== SOC2 Type II Compliance =="
audit_check "SOC2" "Change management process" \
    "test -f /etc/kubernetes/admission-control/change-policy.yaml"

audit_check "SOC2" "Vulnerability scanning" \
    "which trivy"

audit_check "SOC2" "Incident response plan" \
    "test -f /etc/kubernetes/incident-response/plan.yaml"

audit_check "SOC2" "Backup and recovery" \
    "test -d /backup/kubernetes"

audit_check "SOC2" "Monitoring and alerting" \
    "microk8s kubectl get pods -n monitoring | grep -q prometheus"

# PCI-DSS Compliance (si applicable)
echo ""
echo "== PCI-DSS Compliance =="
audit_check "PCI" "Network segmentation" \
    "microk8s kubectl get networkpolicies --all-namespaces | grep -q deny"

audit_check "PCI" "Strong cryptography" \
    "grep -q 'tls-min-version=VersionTLS12' /var/snap/microk8s/current/args/kube-apiserver"

audit_check "PCI" "Regular security testing" \
    "test -f /var/log/security-scans/latest.log"

audit_check "PCI" "Access logging" \
    "grep -q audit-log-path /var/snap/microk8s/current/args/kube-apiserver"

# Générer le certificat de conformité
cat > compliance-certificate-$(date +%Y%m%d).md <<EOF
# Certificate of Compliance

**Organization**: $(hostname)
**Date**: $(date)
**Auditor**: Automated Compliance Scanner v1.0

## Compliance Status

- GDPR: $(audit_check "GDPR" "Overall" "test -f /var/log/microk8s/audit/audit.log" && echo "COMPLIANT" || echo "PARTIAL")
- SOC2 Type II: $(which prometheus > /dev/null && echo "COMPLIANT" || echo "PARTIAL")
- PCI-DSS: $(microk8s kubectl get networkpolicies --all-namespaces | grep -q . && echo "COMPLIANT" || echo "NON-COMPLIANT")
- CIS Benchmark: See separate report

## Recommendations

1. Enable all required security features
2. Implement missing network policies
3. Configure comprehensive monitoring
4. Document all security procedures
5. Schedule regular compliance reviews

## Next Audit Date

$(date -d "+90 days" +%Y-%m-%d)

---
*This certificate is valid for 90 days from the date of issue.*
EOF

echo ""
echo "Compliance certificate generated: compliance-certificate-$(date +%Y%m%d).md"
```

## Formation et sensibilisation

### Programme de formation sécurité

```yaml
# security-training-program.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-training
  namespace: education
data:
  program.yaml: |
    training_curriculum:
      level_1_basics:
        duration: "2 days"
        topics:
          - kubernetes_security_fundamentals
          - rbac_basics
          - network_policies_introduction
          - secrets_management_101
        hands_on:
          - create_service_account
          - apply_network_policy
          - use_secrets_safely

      level_2_intermediate:
        duration: "3 days"
        topics:
          - pod_security_standards
          - admission_controllers
          - vulnerability_scanning
          - audit_logging
        hands_on:
          - configure_pss
          - implement_opa_policy
          - scan_images_with_trivy
          - analyze_audit_logs

      level_3_advanced:
        duration: "5 days"
        topics:
          - incident_response
          - forensics_in_kubernetes
          - supply_chain_security
          - zero_trust_architecture
        hands_on:
          - incident_simulation
          - forensic_analysis
          - implement_signing
          - design_zero_trust

      certification_path:
        - cks: "Certified Kubernetes Security Specialist"
        - cka: "Certified Kubernetes Administrator"
        - ckad: "Certified Kubernetes Application Developer"
```

### Exercices de simulation

```bash
#!/bin/bash
# security-drill.sh

echo "=== Kubernetes Security Drill ==="
echo "Scenario: Suspected container compromise"
echo ""

# Sélectionner un pod aléatoire pour la simulation
NAMESPACE="production"
POD=$(kubectl get pods -n $NAMESPACE -o name | shuf -n 1 | cut -d/ -f2)

echo "Alert: Suspicious activity detected in pod $POD"
echo "Starting incident response drill..."
echo ""

# Chronomètre
START_TIME=$(date +%s)

# Étapes de réponse
echo "Step 1: Isolate the pod"
echo "  kubectl label pod $POD -n $NAMESPACE quarantine=true"
sleep 2

echo "Step 2: Collect evidence"
echo "  kubectl logs $POD -n $NAMESPACE > evidence.log"
echo "  kubectl describe pod $POD -n $NAMESPACE > pod-state.txt"
sleep 2

echo "Step 3: Check for lateral movement"
echo "  kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD"
sleep 2

echo "Step 4: Review audit logs"
echo "  grep $POD /var/log/microk8s/audit/audit.log | tail -20"
sleep 2

echo "Step 5: Terminate if confirmed"
echo "  kubectl delete pod $POD -n $NAMESPACE --grace-period=0"
sleep 2

# Calculer le temps de réponse
END_TIME=$(date +%s)
RESPONSE_TIME=$((END_TIME - START_TIME))

echo ""
echo "=== Drill Complete ==="
echo "Response time: ${RESPONSE_TIME} seconds"
echo ""

# Évaluation
if [ $RESPONSE_TIME -lt 300 ]; then
    echo "✓ EXCELLENT: Response within 5 minutes"
elif [ $RESPONSE_TIME -lt 900 ]; then
    echo "✓ GOOD: Response within 15 minutes"
else
    echo "⚠ NEEDS IMPROVEMENT: Response took over 15 minutes"
fi

# Générer un rapport
cat > drill-report-$(date +%Y%m%d-%H%M%S).md <<EOF
# Security Drill Report

**Date**: $(date)
**Scenario**: Container compromise
**Target**: $POD in $NAMESPACE
**Response Time**: ${RESPONSE_TIME} seconds

## Actions Taken
1. Pod isolated
2. Evidence collected
3. Lateral movement checked
4. Audit logs reviewed
5. Pod terminated

## Lessons Learned
- Team response time: $([ $RESPONSE_TIME -lt 300 ] && echo "Excellent" || echo "Needs improvement")
- Communication: To be reviewed
- Documentation: Completed

## Recommendations
1. Practice regular drills
2. Automate response steps
3. Improve communication channels
4. Update runbooks

EOF

echo "Report saved: drill-report-$(date +%Y%m%d-%H%M%S).md"
```

## Outils et ressources

### Scripts d'automatisation

```bash
#!/bin/bash
# security-automation-toolkit.sh

# Menu principal
show_menu() {
    echo "=== Kubernetes Security Toolkit ==="
    echo "1. Run security scan"
    echo "2. Check compliance"
    echo "3. Rotate secrets"
    echo "4. Backup cluster"
    echo "5. Incident response"
    echo "6. Generate reports"
    echo "7. Update security policies"
    echo "8. Exit"
    echo ""
    read -p "Select option: " choice
}

# Fonction pour chaque option
run_security_scan() {
    echo "Running comprehensive security scan..."

    # Scan des images
    echo "- Scanning container images..."
    for image in $(kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u); do
        trivy image --severity HIGH,CRITICAL $image 2>/dev/null || echo "  ⚠ Failed to scan $image"
    done

    # Scan des configurations
    echo "- Scanning Kubernetes configurations..."
    kubectl get all --all-namespaces -o yaml | kubesec scan -

    # Scan réseau
    echo "- Checking network exposure..."
    kubectl get services --all-namespaces -o wide | grep -E "LoadBalancer|NodePort"

    echo "Scan complete!"
}

check_compliance() {
    echo "Checking compliance standards..."
    ./cis-benchmark-check.sh
    ./compliance-audit.sh
}

rotate_secrets() {
    echo "Starting secret rotation..."
    ./rotate-credentials.sh
}

backup_cluster() {
    echo "Starting cluster backup..."
    ./disaster-recovery-backup.sh
}

incident_response() {
    echo "Starting incident response..."
    ./incident-response.sh suspicious-activity default nginx-pod
}

generate_reports() {
    echo "Generating security reports..."

    # Rapport de vulnérabilités
    echo "- Vulnerability report..."
    trivy image --format template --template "@contrib/html.tpl" -o vulnerability-report.html \
        $(kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u | head -1)

    # Rapport de conformité
    echo "- Compliance report..."
    ./cis-benchmark-check.sh > compliance-report.txt

    # Rapport d'audit
    echo "- Audit report..."
    cat /var/log/microk8s/audit/audit.log | \
        jq -r '[.requestReceivedTimestamp, .user.username, .verb, .objectRef.resource] | @csv' > audit-report.csv

    echo "Reports generated!"
}

update_policies() {
    echo "Updating security policies..."

    # Appliquer les dernières policies
    kubectl apply -f /etc/kubernetes/policies/

    # Redémarrer les admission controllers
    kubectl rollout restart deployment/opa -n opa-system
    kubectl rollout restart deployment/kyverno -n kyverno

    echo "Policies updated!"
}

# Boucle principale
while true; do
    show_menu
    case $choice in
        1) run_security_scan ;;
        2) check_compliance ;;
        3) rotate_secrets ;;
        4) backup_cluster ;;
        5) incident_response ;;
        6) generate_reports ;;
        7) update_policies ;;
        8) echo "Goodbye!"; exit 0 ;;
        *) echo "Invalid option" ;;
    esac
    echo ""
    read -p "Press Enter to continue..."
    clear
done
```

### Dashboard de sécurité centralisé

```yaml
# security-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-security-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Kubernetes Security Overview",
        "panels": [
          {
            "title": "Security Score",
            "type": "stat",
            "targets": [{
              "expr": "(sum(security_checks_passed) / sum(security_checks_total)) * 100"
            }]
          },
          {
            "title": "Critical Vulnerabilities",
            "type": "graph",
            "targets": [{
              "expr": "sum(vulnerability_count{severity='CRITICAL'})"
            }]
          },
          {
            "title": "Failed Auth Attempts",
            "type": "graph",
            "targets": [{
              "expr": "sum(rate(apiserver_audit_event_total{response_code='403'}[5m]))"
            }]
          },
          {
            "title": "Network Policy Coverage",
            "type": "piechart",
            "targets": [{
              "expr": "sum(pods_with_network_policy) by (namespace)"
            }]
          },
          {
            "title": "Secret Rotation Status",
            "type": "table",
            "targets": [{
              "expr": "secret_age_days"
            }]
          },
          {
            "title": "Compliance Status",
            "type": "heatmap",
            "targets": [{
              "expr": "compliance_score by (standard)"
            }]
          }
        ]
      }
    }
```

## Conclusion et recommandations finales

### Matrice de maturité sécurité

```yaml
# security-maturity-model.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-maturity
  namespace: security
data:
  maturity-levels.yaml: |
    level_1_initial:
      description: "Ad-hoc, réactif"
      characteristics:
        - no_formal_security_process
        - reactive_incident_response
        - basic_authentication_only
        - no_vulnerability_scanning

    level_2_developing:
      description: "Processus en développement"
      characteristics:
        - rbac_implemented
        - basic_network_policies
        - periodic_vulnerability_scans
        - documented_procedures

    level_3_defined:
      description: "Processus définis et documentés"
      characteristics:
        - comprehensive_rbac
        - network_segmentation
        - automated_scanning
        - incident_response_plan
        - regular_audits

    level_4_managed:
      description: "Métriques et amélioration"
      characteristics:
        - continuous_monitoring
        - automated_remediation
        - supply_chain_security
        - regular_drills
        - compliance_automation

    level_5_optimized:
      description: "Amélioration continue"
      characteristics:
        - predictive_security
        - zero_trust_architecture
        - chaos_engineering
        - ml_based_detection
        - security_as_code
```

### Plan d'action sur 12 mois

```markdown
# Plan de Sécurisation Kubernetes - 12 Mois

## Trimestre 1 : Fondations (Mois 1-3)
- [ ] Mois 1: RBAC et Network Policies de base
- [ ] Mois 2: Pod Security Standards et audit logging
- [ ] Mois 3: Scan de vulnérabilités et secrets management

## Trimestre 2 : Renforcement (Mois 4-6)
- [ ] Mois 4: Admission controllers (OPA/Kyverno)
- [ ] Mois 5: Supply chain security et image signing
- [ ] Mois 6: Monitoring avancé avec Falco

## Trimestre 3 : Automatisation (Mois 7-9)
- [ ] Mois 7: CI/CD security integration
- [ ] Mois 8: Automated compliance checking
- [ ] Mois 9: Incident response automation

## Trimestre 4 : Optimisation (Mois 10-12)
- [ ] Mois 10: Service mesh et mTLS
- [ ] Mois 11: Zero trust implementation
- [ ] Mois 12: Chaos engineering et tests
```

### Ressources essentielles

```yaml
# resources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-resources
data:
  tools: |
    scanning:
      - trivy: "Vulnerability scanner"
      - kubesec: "Security risk analysis"
      - kube-bench: "CIS benchmark"
      - kube-hunter: "Penetration testing"

    runtime:
      - falco: "Runtime security"
      - sysdig: "Container intelligence"
      - tracee: "Runtime security and forensics"

    policies:
      - opa: "Policy engine"
      - kyverno: "Policy management"
      - polaris: "Best practices validation"

    secrets:
      - vault: "Secret management"
      - sealed-secrets: "GitOps secrets"
      - external-secrets: "External secret integration"

  documentation: |
    - CIS Kubernetes Benchmark
    - NIST Cybersecurity Framework
    - OWASP Kubernetes Security Top 10
    - Kubernetes Security Best Practices

  training: |
    - Linux Foundation CKS
    - SANS SEC555
    - Cloud Native Security Course
```

La sécurité dans Kubernetes est un voyage continu qui demande vigilance, apprentissage constant et amélioration progressive. Commencez par les fondamentaux, automatisez ce qui peut l'être, et construisez une culture de sécurité dans votre équipe. La sécurité n'est pas une destination mais un processus d'amélioration continue.

---

*Fin du chapitre 11 : Sécurité dans MicroK8s*

⏭️
