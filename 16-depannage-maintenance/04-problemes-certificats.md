🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.4 Problèmes de certificats

## Introduction aux certificats dans Kubernetes

Les certificats SSL/TLS sont essentiels pour sécuriser les communications dans votre cluster MicroK8s. Ils garantissent le chiffrement des données et l'authentification des services. Cependant, les certificats sont aussi une source fréquente de problèmes : expiration, mauvaise configuration, chaîne de certificats incomplète, ou incompatibilité avec cert-manager. Cette section vous guidera à travers les problèmes de certificats les plus courants, comment les diagnostiquer et les résoudre efficacement.

## Comprendre l'écosystème des certificats dans MicroK8s

### Types de certificats dans Kubernetes

Dans un cluster MicroK8s, vous rencontrerez plusieurs types de certificats, chacun avec un rôle spécifique :

**Certificats internes du cluster** : Kubernetes utilise des certificats pour sécuriser les communications entre ses composants (API server, kubelet, etcd). Ces certificats sont généralement gérés automatiquement par MicroK8s et ont une durée de vie d'un an.

**Certificats pour les applications** : Vos applications web nécessitent des certificats pour HTTPS. Ces certificats peuvent être auto-signés pour le développement, ou signés par une autorité de certification (CA) comme Let's Encrypt pour la production.

**Certificats Ingress** : L'Ingress Controller utilise des certificats pour terminer les connexions SSL/TLS. Un certificat par défaut est utilisé si aucun certificat spécifique n'est configuré pour un hôte.

**Certificats de webhook** : Les webhooks d'admission et de validation nécessitent des certificats pour sécuriser leurs communications avec l'API server.

### Le cycle de vie des certificats

Comprendre le cycle de vie des certificats est crucial pour éviter les interruptions de service :

1. **Création** : Le certificat est généré et signé
2. **Distribution** : Le certificat est stocké dans un Secret Kubernetes
3. **Utilisation** : Les applications référencent le Secret
4. **Surveillance** : La date d'expiration doit être surveillée
5. **Renouvellement** : Le certificat doit être renouvelé avant expiration
6. **Rotation** : Le nouveau certificat remplace l'ancien

### Cert-Manager : l'automatisation des certificats

Cert-Manager est l'addon recommandé pour gérer automatiquement les certificats dans MicroK8s. Il peut :
- Obtenir des certificats de Let's Encrypt automatiquement
- Renouveler les certificats avant expiration
- Créer des certificats auto-signés pour le développement
- Gérer des autorités de certification internes

## Problème 1 : Certificat expiré

### Symptômes

- Erreur "certificate has expired" dans les navigateurs
- Code d'erreur SSL_ERROR_EXPIRED_CERT_ALERT
- Les connexions HTTPS échouent soudainement
- Les logs montrent "x509: certificate has expired"

### Diagnostic

Vérifiez la date d'expiration des certificats :

```bash
# Lister tous les certificats gérés par cert-manager
microk8s kubectl get certificates --all-namespaces

# Voir les détails d'un certificat
microk8s kubectl describe certificate <cert-name> -n <namespace>

# Examiner le Secret contenant le certificat
microk8s kubectl get secret <tls-secret> -n <namespace> -o yaml

# Décoder et examiner le certificat
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Vérifier spécifiquement les dates
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -dates -noout
```

Pour vérifier les certificats depuis l'extérieur :

```bash
# Vérifier un certificat via HTTPS
echo | openssl s_client -connect <domain>:443 2>/dev/null | openssl x509 -dates -noout

# Avec curl
curl -vI https://<domain> 2>&1 | grep -A 5 "SSL certificate"

# Test complet avec openssl
openssl s_client -connect <domain>:443 -servername <domain> </dev/null 2>/dev/null | openssl x509 -text
```

### Résolution

Pour renouveler un certificat géré par cert-manager :

```bash
# Forcer le renouvellement en supprimant le certificat
microk8s kubectl delete certificate <cert-name> -n <namespace>
# Cert-manager le recréera automatiquement

# Vérifier le statut du renouvellement
microk8s kubectl describe certificate <cert-name> -n <namespace>

# Surveiller les logs de cert-manager
microk8s kubectl logs -n cert-manager deployment/cert-manager -f
```

Pour un certificat manuel :

```bash
# Générer un nouveau certificat auto-signé (développement uniquement)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=<domain>/O=<organization>"

# Créer ou mettre à jour le Secret
microk8s kubectl create secret tls <secret-name> \
  --cert=tls.crt --key=tls.key -n <namespace> \
  --dry-run=client -o yaml | microk8s kubectl apply -f -

# Redémarrer les pods qui utilisent le certificat
microk8s kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

## Problème 2 : Cert-Manager ne peut pas obtenir de certificat Let's Encrypt

### Symptômes

- Le certificat reste en état "Pending" ou "False"
- Erreur "acme: challenge failed"
- Message "Waiting for HTTP-01 challenge propagation"
- Rate limit exceeded de Let's Encrypt

### Diagnostic

Examinez l'état de cert-manager :

```bash
# Vérifier que cert-manager est installé et actif
microk8s kubectl get pods -n cert-manager

# Examiner les logs de cert-manager
microk8s kubectl logs -n cert-manager deployment/cert-manager

# Vérifier l'Issuer ou ClusterIssuer
microk8s kubectl describe clusterissuer letsencrypt-prod
microk8s kubectl describe issuer letsencrypt-staging -n <namespace>

# Examiner le Certificate
microk8s kubectl describe certificate <cert-name> -n <namespace>

# Vérifier les CertificateRequests
microk8s kubectl get certificaterequests -n <namespace>
microk8s kubectl describe certificaterequest <cr-name> -n <namespace>

# Vérifier les Orders (commandes ACME)
microk8s kubectl get orders -n <namespace>
microk8s kubectl describe order <order-name> -n <namespace>

# Vérifier les Challenges
microk8s kubectl get challenges -n <namespace>
microk8s kubectl describe challenge <challenge-name> -n <namespace>
```

### Résolution

Pour les problèmes de validation HTTP-01 :

```bash
# Vérifier que l'Ingress est correctement configuré
microk8s kubectl get ingress -n <namespace>

# S'assurer que l'annotation est présente
microk8s kubectl annotate ingress <ingress-name> \
  cert-manager.io/cluster-issuer="letsencrypt-prod" -n <namespace>

# Vérifier que le domaine est accessible depuis Internet
curl -H "Host: <domain>" http://<public-ip>/.well-known/acme-challenge/test

# Créer un Ingress temporaire pour le challenge
cat <<EOF | microk8s kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: acme-challenge
  namespace: <namespace>
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  rules:
  - host: <domain>
    http:
      paths:
      - path: /.well-known/acme-challenge
        pathType: Prefix
        backend:
          service:
            name: cm-acme-http-solver
            port:
              number: 8089
EOF
```

Pour utiliser le staging de Let's Encrypt (recommandé pour les tests) :

```bash
# Créer un Issuer staging
cat <<EOF | microk8s kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: <your-email>
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Utiliser cet Issuer pour tester
microk8s kubectl annotate ingress <ingress-name> \
  cert-manager.io/cluster-issuer="letsencrypt-staging" --overwrite
```

Pour les problèmes de rate limit :

```bash
# Vérifier les limites actuelles
curl -s https://crt.sh/?q=<domain> | grep -c "Let's Encrypt"

# Solutions:
# 1. Utiliser staging pour les tests
# 2. Attendre (les limites se réinitialisent après une semaine)
# 3. Utiliser un wildcard certificate pour couvrir plusieurs sous-domaines
```

## Problème 3 : Erreur de chaîne de certificats

### Symptômes

- "Unable to verify the first certificate"
- "Certificate chain incomplete"
- Certains clients acceptent le certificat, d'autres non
- Erreur SSL_ERROR_BAD_CERT_DOMAIN

### Diagnostic

Vérifiez la chaîne complète :

```bash
# Examiner la chaîne de certificats
echo | openssl s_client -connect <domain>:443 -servername <domain> -showcerts

# Vérifier la chaîne dans le Secret
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d

# Compter le nombre de certificats dans la chaîne
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | grep -c "BEGIN CERTIFICATE"

# Vérifier avec un outil en ligne
# curl https://www.ssllabs.com/ssltest/analyze.html?d=<domain>
```

### Résolution

Reconstruire la chaîne de certificats :

```bash
# Si vous avez les certificats séparés
cat server.crt intermediate.crt root.crt > fullchain.crt

# Mettre à jour le Secret avec la chaîne complète
microk8s kubectl create secret tls <secret-name> \
  --cert=fullchain.crt --key=server.key \
  -n <namespace> --dry-run=client -o yaml | \
  microk8s kubectl apply -f -

# Pour Let's Encrypt avec cert-manager, vérifier l'Issuer
microk8s kubectl get clusterissuer letsencrypt-prod -o yaml
# S'assurer que preferredChain n'est pas mal configuré
```

## Problème 4 : Incompatibilité de domaine (SAN)

### Symptômes

- ERR_CERT_COMMON_NAME_INVALID
- "Certificate is not valid for this domain"
- Le certificat fonctionne pour www.domain.com mais pas domain.com

### Diagnostic

Examinez les domaines couverts par le certificat :

```bash
# Voir le CN et les SAN du certificat
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout | grep -A 2 "Subject:"

# Vérifier spécifiquement les Subject Alternative Names
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout | grep -A 1 "Subject Alternative Name"

# Depuis l'extérieur
echo | openssl s_client -connect <domain>:443 -servername <domain> 2>/dev/null | \
  openssl x509 -text -noout | grep -A 1 "Subject Alternative Name"
```

### Résolution

Créer un certificat avec les bons domaines :

```bash
# Pour cert-manager, mettre à jour le Certificate
cat <<EOF | microk8s kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <cert-name>
  namespace: <namespace>
spec:
  secretName: <tls-secret>
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - domain.com
  - www.domain.com
  - api.domain.com
  - "*.domain.com"  # Wildcard si supporté
EOF

# Pour un certificat auto-signé avec plusieurs domaines
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=domain.com" \
  -addext "subjectAltName=DNS:domain.com,DNS:www.domain.com,DNS:api.domain.com"
```

## Problème 5 : Certificat auto-signé non accepté

### Symptômes

- NET::ERR_CERT_AUTHORITY_INVALID
- "Security certificate is not trusted"
- curl: (60) SSL certificate problem: self signed certificate

### Diagnostic

Vérifiez le type de certificat :

```bash
# Vérifier l'émetteur du certificat
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout | grep "Issuer:"

# Vérifier si auto-signé
microk8s kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout | grep -q "Issuer:.*CN = .*Subject:.*CN = " && \
  echo "Auto-signé" || echo "Signé par une CA"
```

### Résolution

Pour le développement, accepter les certificats auto-signés :

```bash
# Avec curl
curl -k https://<domain>  # -k ignore les erreurs de certificat

# Avec wget
wget --no-check-certificate https://<domain>

# Dans un pod
microk8s kubectl exec -it <pod> -- curl -k https://<service>

# Pour une application Node.js, définir la variable d'environnement
env:
- name: NODE_TLS_REJECT_UNAUTHORIZED
  value: "0"  # UNIQUEMENT en développement
```

Pour la production, utiliser Let's Encrypt :

```bash
# Installer cert-manager si pas déjà fait
microk8s enable cert-manager

# Créer un ClusterIssuer Let's Encrypt
cat <<EOF | microk8s kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <your-email>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Problème 6 : Webhook certificates invalides

### Symptômes

- "x509: certificate signed by unknown authority" dans les logs
- Webhook admission refusé
- Impossible de créer ou modifier des ressources

### Diagnostic

Vérifiez les webhooks :

```bash
# Lister les webhooks
microk8s kubectl get validatingwebhookconfigurations
microk8s kubectl get mutatingwebhookconfigurations

# Examiner un webhook
microk8s kubectl describe validatingwebhookconfiguration <webhook-name>

# Vérifier le certificat CA dans le webhook
microk8s kubectl get validatingwebhookconfiguration <webhook-name> -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | base64 -d | openssl x509 -text -noout

# Vérifier le service du webhook
microk8s kubectl get service -n <webhook-namespace> <webhook-service>
```

### Résolution

Régénérer les certificats du webhook :

```bash
# Script pour générer des certificats webhook
cat <<'EOF' > generate-webhook-certs.sh
#!/bin/bash
SERVICE_NAME=$1
NAMESPACE=$2
CERT_DIR=${3:-/tmp/certs}

mkdir -p $CERT_DIR

# Générer CA
openssl req -nodes -new -x509 -keyout $CERT_DIR/ca.key -out $CERT_DIR/ca.crt \
  -subj "/CN=Webhook CA"

# Générer clé et CSR
openssl req -nodes -new -keyout $CERT_DIR/tls.key \
  -out $CERT_DIR/server.csr \
  -subj "/CN=$SERVICE_NAME.$NAMESPACE.svc"

# Signer le certificat
openssl x509 -req -in $CERT_DIR/server.csr \
  -CA $CERT_DIR/ca.crt -CAkey $CERT_DIR/ca.key \
  -CAcreateserial -out $CERT_DIR/tls.crt \
  -days 365 \
  -extfile <(printf "subjectAltName=DNS:$SERVICE_NAME,DNS:$SERVICE_NAME.$NAMESPACE,DNS:$SERVICE_NAME.$NAMESPACE.svc")

# Créer le Secret
microk8s kubectl create secret tls $SERVICE_NAME-tls \
  --cert=$CERT_DIR/tls.crt \
  --key=$CERT_DIR/tls.key \
  -n $NAMESPACE --dry-run=client -o yaml | \
  microk8s kubectl apply -f -

# Afficher le CA pour le webhook config
echo "CA Bundle pour le webhook:"
cat $CERT_DIR/ca.crt | base64 -w0
EOF

chmod +x generate-webhook-certs.sh
./generate-webhook-certs.sh <service-name> <namespace>
```

## Problème 7 : Rotation des certificats internes

### Symptômes

- "x509: certificate has expired" dans les logs système
- Composants Kubernetes ne peuvent pas communiquer
- API server inaccessible

### Diagnostic

Vérifiez les certificats internes :

```bash
# Vérifier les certificats MicroK8s
sudo microk8s.refresh-certs

# Examiner les certificats dans le dossier MicroK8s
sudo ls -la /var/snap/microk8s/current/certs/

# Vérifier les dates d'expiration
for cert in /var/snap/microk8s/current/certs/*.crt; do
  echo "=== $(basename $cert) ==="
  sudo openssl x509 -in $cert -dates -noout
done

# Vérifier spécifiquement le certificat de l'API server
sudo openssl x509 -in /var/snap/microk8s/current/certs/server.crt -text -noout
```

### Résolution

Renouveler les certificats internes :

```bash
# Arrêter MicroK8s
microk8s stop

# Renouveler les certificats
sudo microk8s.refresh-certs --cert ca.crt
sudo microk8s.refresh-certs --cert server.crt

# Redémarrer MicroK8s
microk8s start

# Vérifier le statut
microk8s status --wait-ready

# Si nécessaire, régénérer tous les certificats
sudo snap set microk8s certs.regenerate=true
microk8s stop
microk8s start
```

## Outils de diagnostic des certificats

### OpenSSL pour l'analyse

Commandes OpenSSL essentielles :

```bash
# Décoder un certificat base64
echo "<base64-cert>" | base64 -d | openssl x509 -text -noout

# Vérifier qu'une clé correspond à un certificat
openssl x509 -noout -modulus -in cert.crt | openssl md5
openssl rsa -noout -modulus -in private.key | openssl md5
# Les MD5 doivent être identiques

# Vérifier la chaîne de certificats
openssl verify -CAfile ca.crt -untrusted intermediate.crt server.crt

# Simuler une connexion TLS
openssl s_client -connect <domain>:443 -servername <domain> \
  -CAfile /etc/ssl/certs/ca-certificates.crt
```

### Script de vérification complète

```bash
#!/bin/bash
# check-certificates.sh

NAMESPACE=${1:-default}

echo "=== Vérification des certificats dans namespace $NAMESPACE ==="

# Fonction pour vérifier un secret TLS
check_tls_secret() {
  local secret=$1
  local namespace=$2

  echo -e "\nVérification du secret: $secret"

  # Extraire le certificat
  cert=$(microk8s kubectl get secret $secret -n $namespace -o jsonpath='{.data.tls\.crt}' 2>/dev/null)

  if [ -z "$cert" ]; then
    echo "  ❌ Secret non trouvé ou pas de type TLS"
    return
  fi

  # Décoder et analyser
  cert_text=$(echo $cert | base64 -d | openssl x509 -text -noout 2>/dev/null)

  if [ $? -ne 0 ]; then
    echo "  ❌ Certificat invalide"
    return
  fi

  # Extraire les informations
  subject=$(echo "$cert_text" | grep "Subject:" | sed 's/.*Subject: //')
  issuer=$(echo "$cert_text" | grep "Issuer:" | sed 's/.*Issuer: //')
  not_before=$(echo "$cert_text" | grep "Not Before:" | sed 's/.*Not Before: //')
  not_after=$(echo "$cert_text" | grep "Not After:" | sed 's/.*Not After: //')

  # Vérifier l'expiration
  expiry_date=$(date -d "$not_after" +%s 2>/dev/null)
  current_date=$(date +%s)
  days_left=$(( ($expiry_date - $current_date) / 86400 ))

  echo "  Subject: $subject"
  echo "  Issuer: $issuer"
  echo "  Valid from: $not_before"
  echo "  Valid until: $not_after"

  if [ $days_left -lt 0 ]; then
    echo "  ❌ EXPIRÉ depuis $((-$days_left)) jours"
  elif [ $days_left -lt 30 ]; then
    echo "  ⚠️  Expire dans $days_left jours"
  else
    echo "  ✅ Valide encore $days_left jours"
  fi
}

# Lister tous les secrets TLS
echo "Recherche des secrets TLS..."
secrets=$(microk8s kubectl get secrets -n $NAMESPACE -o json | \
  jq -r '.items[] | select(.type=="kubernetes.io/tls") | .metadata.name')

if [ -z "$secrets" ]; then
  echo "Aucun secret TLS trouvé dans le namespace $NAMESPACE"
else
  for secret in $secrets; do
    check_tls_secret $secret $NAMESPACE
  done
fi

# Vérifier les certificats cert-manager
echo -e "\n=== Certificats gérés par cert-manager ==="
microk8s kubectl get certificates -n $NAMESPACE 2>/dev/null

# Vérifier les Ingress avec TLS
echo -e "\n=== Ingress avec TLS ==="
microk8s kubectl get ingress -n $NAMESPACE -o json | \
  jq -r '.items[] | select(.spec.tls) | .metadata.name + ": " + (.spec.tls[].secretName // "default")'
```

## Monitoring et alerting des certificats

### Alertes Prometheus

Configuration pour surveiller l'expiration des certificats :

```yaml
# prometheus-cert-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cert-expiry-rules
  namespace: monitoring
data:
  cert-rules.yaml: |
    groups:
    - name: certificates
      rules:
      - alert: CertificateExpiringSoon
        expr: certmanager_certificate_expiration_timestamp_seconds - time() < 7 * 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Certificate expiring soon"
          description: "Certificate {{ $labels.name }} in namespace {{ $labels.namespace }} expires in less than 7 days"

      - alert: CertificateExpired
        expr: certmanager_certificate_expiration_timestamp_seconds - time() < 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Certificate has expired"
          description: "Certificate {{ $labels.name }} in namespace {{ $labels.namespace }} has expired"
```

### Script de monitoring automatisé

```bash
#!/bin/bash
# monitor-certificates.sh

# Configuration
ALERT_DAYS=30  # Alerter si expire dans moins de X jours
EMAIL="admin@example.com"
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Fonction d'alerte
send_alert() {
  local message=$1
  local severity=$2

  # Email (nécessite mailutils)
  # echo "$message" | mail -s "Certificate Alert: $severity" $EMAIL

  # Slack
  if [ ! -z "$SLACK_WEBHOOK" ]; then
    curl -X POST $SLACK_WEBHOOK \
      -H 'Content-Type: application/json' \
      -d "{\"text\":\"Certificate Alert ($severity): $message\"}"
  fi

  # Log
  echo "[$(date)] $severity: $message" >> /var/log/cert-monitor.log
}

# Vérifier tous les namespaces
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  # Obtenir tous les certificats cert-manager
  certs=$(microk8s kubectl get certificates -n $ns -o json 2>/dev/null | jq -r '.items[]?.metadata.name')

  for cert in $certs; do
    # Obtenir le statut
    ready=$(microk8s kubectl get certificate $cert -n $ns -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')

    if [ "$ready" != "True" ]; then
      send_alert "Certificate $cert in namespace $ns is not ready" "WARNING"
    fi

    # Vérifier l'expiration
    expiry=$(microk8s kubectl get certificate $cert -n $ns -o jsonpath='{.status.notAfter}')
    if [ ! -z "$expiry" ]; then
      expiry_epoch=$(date -d "$expiry" +%s)
      current_epoch=$(date +%s)
      days_left=$(( ($expiry_epoch - $current_epoch) / 86400 ))

      if [ $days_left -lt 0 ]; then
        send_alert "Certificate $cert in namespace $ns has EXPIRED" "CRITICAL"
      elif [ $days_left -lt $ALERT_DAYS ]; then
        send_alert "Certificate $cert in namespace $ns expires in $days_left days" "WARNING"
      fi
    fi
  done
done
```

## Bonnes pratiques pour éviter les problèmes

### Configuration préventive

1. **Toujours utiliser cert-manager** pour l'automatisation
2. **Configurer des alertes** pour l'expiration à 30, 14, et 7 jours
3. **Utiliser Let's Encrypt staging** pour les tests
4. **Documenter** tous les certificats et leurs utilisations
5. **Tester le renouvellement** régulièrement en environnement de test

### Checklist de déploiement

Avant de déployer une application avec HTTPS :

```bash
# 1. Vérifier que cert-manager est installé
microk8s kubectl get pods -n cert-manager

# 2. Vérifier l'Issuer/ClusterIssuer
microk8s kubectl get clusterissuers

# 3. Vérifier la configuration DNS
nslookup <your-domain>

# 4. Tester l'accessibilité HTTP
curl http://<your-domain>

# 5. Déployer avec staging d'abord
# 6. Vérifier le certificat staging
# 7. Passer en production
```

### Template de Certificate

Template réutilisable pour les certificats :

```yaml
# certificate-template.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ${APP_NAME}-tls
  namespace: ${NAMESPACE}
spec:
  secretName: ${APP_NAME}-tls-secret
  issuerRef:
    name: letsencrypt-${ENVIRONMENT}  # staging ou prod
    kind: ClusterIssuer
  commonName: ${DOMAIN}
  dnsNames:
  - ${DOMAIN}
  - www.${DOMAIN}
  duration: 2160h  # 90 jours
  renewBefore: 720h  # Renouveler 30 jours avant expiration
  privateKey:
    algorithm: RSA
    size: 2048
  usages:
  - digital signature
  - key encipherment
  - server auth
```

## Conclusion

Les problèmes de certificats peuvent sembler complexes au premier abord, mais ils suivent généralement des patterns prévisibles : expiration, mauvaise configuration, ou problèmes de validation. Avec cert-manager correctement configuré, la plupart de ces problèmes peuvent être évités. La clé est de mettre en place une surveillance proactive, de comprendre le cycle de vie des certificats, et d'avoir une procédure claire pour le renouvellement. Les outils et scripts présentés dans cette section vous permettront de diagnostiquer rapidement les problèmes et de maintenir vos certificats en bon état. N'oubliez pas que la sécurité commence par une bonne gestion des certificats, et qu'un certificat expiré peut rendre votre service complètement inaccessible. Investir du temps dans l'automatisation et la surveillance des certificats vous épargnera de nombreux problèmes futurs.

⏭️
