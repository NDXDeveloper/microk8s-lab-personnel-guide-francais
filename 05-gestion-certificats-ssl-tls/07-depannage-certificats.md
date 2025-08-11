🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.7 Dépannage des certificats

## Introduction au dépannage des certificats

Le dépannage des certificats SSL/TLS peut sembler intimidant au début, mais avec une approche méthodique et les bons outils, la plupart des problèmes peuvent être résolus rapidement. Cette section vous fournira une boîte à outils complète pour diagnostiquer et résoudre les problèmes les plus courants.

### Analogie du diagnostic médical

Dépanner un problème de certificat, c'est comme diagnostiquer un problème de santé. Un médecin ne prescrit pas un traitement au hasard : il commence par écouter les symptômes, examine le patient, fait des tests spécifiques, puis pose un diagnostic avant de proposer un traitement. De la même manière, nous allons apprendre à "examiner" nos certificats et "diagnostiquer" leurs problèmes.

### Philosophie du dépannage

**Approche systématique**
- Observer les symptômes sans supposer la cause
- Collecter des informations objectives
- Tester les hypothèses une par une
- Documenter les solutions pour l'avenir

**Patience et méthode**
- Ne pas modifier plusieurs choses à la fois
- Attendre les délais de propagation (DNS, cache)
- Vérifier après chaque modification
- Garder une trace des actions effectuées

## Méthodologie de diagnostic

### Étapes de diagnostic recommandées

```
1. Identification des symptômes
   ↓
2. Collecte d'informations
   ↓
3. Vérification de l'état Kubernetes
   ↓
4. Test de connectivité
   ↓
5. Analyse des logs
   ↓
6. Application des corrections
   ↓
7. Validation du fonctionnement
```

### Outils essentiels

**Outils Kubernetes natifs**
- `kubectl` : commande principale pour interagir avec le cluster
- `describe` : informations détaillées sur les ressources
- `logs` : consultation des journaux des pods
- `get events` : événements du cluster

**Outils SSL/TLS**
- `openssl` : vérification et analyse des certificats
- `curl` : test de connectivité HTTPS
- `dig` : vérification DNS pour validation ACME
- `nslookup` : résolution de noms de domaine

**Outils de monitoring**
- `ss` ou `netstat` : vérification des ports ouverts
- `tcpdump` : capture de trafic réseau si nécessaire
- Navigateur web : test utilisateur final

## Problèmes courants et solutions

### 1. Certificat non créé

**Symptômes**
- Erreur SSL/TLS dans le navigateur
- Service inaccessible en HTTPS
- Secret TLS inexistant

**Diagnostic**
```bash
# Vérifier l'existence du certificat
kubectl get certificate mon-certificat

# Si absent, vérifier l'Ingress ou la ressource Certificate
kubectl get ingress mon-ingress -o yaml

# Vérifier les événements récents
kubectl get events --sort-by='.lastTimestamp' | tail -20
```

**Causes possibles et solutions**

**Annotation manquante ou incorrecte**
```yaml
# Vérification de l'annotation dans l'Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Vérifier le nom
    # OU
    cert-manager.io/issuer: "mon-issuer"  # Pour un Issuer spécifique au namespace
```

**Issuer inexistant ou non fonctionnel**
```bash
# Vérifier l'existence des issuers
kubectl get clusterissuers
kubectl get issuers --all-namespaces

# Vérifier l'état d'un issuer
kubectl describe clusterissuer letsencrypt-prod
```

**Section TLS manquante dans l'Ingress**
```yaml
spec:
  tls:
  - hosts:
    - mon-domaine.com
    secretName: mon-certificat-tls  # Secret qui sera créé
```

### 2. Certificat en état "False" ou "Unknown"

**Symptômes**
- `kubectl get certificates` montre `READY: False`
- Erreurs SSL persistantes
- Secret créé mais certificat invalide

**Diagnostic détaillé**
```bash
# État détaillé du certificat
kubectl describe certificate mon-certificat

# Regarder la section "Status" et "Events"
# Vérifier les CertificateRequest associées
kubectl get certificaterequests -l cert-manager.io/certificate-name=mon-certificat

# Examiner les détails de la demande
kubectl describe certificaterequest nom-de-la-request
```

**Causes et solutions courantes**

**Challenge ACME échoué**
```bash
# Vérifier les challenges en cours
kubectl get challenges

# Détails d'un challenge spécifique
kubectl describe challenge nom-du-challenge

# Pour HTTP-01 : vérifier l'accessibilité
curl -I http://mon-domaine.com/.well-known/acme-challenge/test

# Pour DNS-01 : vérifier la propagation DNS
dig TXT _acme-challenge.mon-domaine.com
```

**Problème de validation DNS (DNS-01)**
```bash
# Vérifier les permissions API DNS
kubectl logs -n cert-manager deployment/cert-manager | grep -i dns

# Vérifier le secret contenant les clés API
kubectl get secret cloudflare-api-token-secret -n cert-manager -o yaml
```

**Problème de validation HTTP (HTTP-01)**
```bash
# Vérifier que l'Ingress Controller fonctionne
kubectl get pods -n ingress-nginx

# Tester l'accessibilité du domaine
curl -v http://mon-domaine.com

# Vérifier les règles de firewall/NAT
```

### 3. Limite de débit atteinte (Rate Limit)

**Symptômes**
- Erreur explicite "rate limit exceeded" dans les logs
- Certificats qui ne se renouvellent plus
- Nouveaux certificats refusés

**Diagnostic**
```bash
# Rechercher les erreurs de rate limit dans les logs
kubectl logs -n cert-manager deployment/cert-manager | grep -i "rate limit"

# Vérifier l'historique des certificats émis
curl -s "https://crt.sh/?q=%.mon-domaine.com&output=json" | jq length
```

**Solutions**

**Passer à l'environnement staging temporairement**
```yaml
# Modifier l'issuer pour utiliser staging
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging-temp
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # ... reste de la configuration
```

**Optimiser l'utilisation des certificats**
- Utiliser des certificats wildcard pour réduire le nombre de demandes
- Éviter de supprimer/recréer les secrets inutilement
- Grouper plusieurs domaines dans un seul certificat

**Attendre la remise à zéro des quotas**
- Quotas Let's Encrypt remis à zéro chaque semaine
- Documenter les limites atteintes pour éviter la répétition

### 4. Problèmes de propagation DNS

**Symptômes**
- Validation DNS-01 qui échoue de manière intermittente
- Délais d'obtention de certificats très longs
- Timeouts pendant la validation

**Diagnostic**
```bash
# Vérifier la propagation DNS depuis différents serveurs
dig @8.8.8.8 TXT _acme-challenge.mon-domaine.com
dig @1.1.1.1 TXT _acme-challenge.mon-domaine.com
dig @208.67.222.222 TXT _acme-challenge.mon-domaine.com

# Vérifier le TTL des enregistrements
dig TXT _acme-challenge.mon-domaine.com | grep -i ttl
```

**Solutions**

**Réduire le TTL des enregistrements DNS**
- Configurer un TTL bas (300 secondes) pour les zones utilisées par ACME
- Faire le changement avant d'avoir besoin des certificats

**Ajuster les timeouts cert-manager**
```yaml
# Configuration avec timeouts étendus
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-extended
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: email@example.com
    privateKeySecretRef:
      name: letsencrypt-dns01-key
    solvers:
    - dns01:
        cloudflare:
          email: email@cloudflare.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
        # Timeouts étendus pour DNS lent
        cnameStrategy: Follow
        timeout: 300s  # 5 minutes au lieu de 60s par défaut
```

**Choisir un provider DNS plus rapide**
- Cloudflare : propagation généralement sous 1 minute
- Route53 : propagation rapide dans la région AWS
- Éviter les providers avec propagation lente (>5 minutes)

### 5. Problèmes de certificats auto-signés

**Symptômes**
- Avertissements de sécurité dans le navigateur
- Applications qui refusent les connexions
- Erreurs "certificate signed by unknown authority"

**Diagnostic**
```bash
# Vérifier la chaîne de certificats
openssl s_client -connect mon-service.local:443 -servername mon-service.local

# Examiner le certificat
kubectl get secret mon-certificat-tls -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout
```

**Solutions**

**Installer la CA racine dans le trust store**
```bash
# Extraire le certificat CA
kubectl get secret my-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > my-ca.crt

# Ubuntu/Debian
sudo cp my-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# macOS
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain my-ca.crt
```

**Configurer les applications pour accepter la CA**
```yaml
# ConfigMap avec CA pour les applications
apiVersion: v1
kind: ConfigMap
metadata:
  name: ca-certificates
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAK...
    -----END CERTIFICATE-----
```

### 6. Problèmes de renouvellement

**Symptômes**
- Certificats qui expirent sans être renouvelés
- État "Ready: True" mais certificat expiré
- Services qui deviennent inaccessibles subitement

**Diagnostic**
```bash
# Vérifier les dates d'expiration
kubectl get certificates -o custom-columns=\
"NAME:.metadata.name,READY:.status.conditions[0].status,EXPIRES:.status.notAfter"

# Vérifier la configuration de renouvellement
kubectl get certificate mon-certificat -o yaml | grep -A 5 renewBefore

# Vérifier les événements de renouvellement
kubectl get events --field-selector involvedObject.name=mon-certificat
```

**Solutions**

**Ajuster les paramètres de renouvellement**
```yaml
# Configuration corrigée
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-certificat
spec:
  duration: 2160h     # 90 jours
  renewBefore: 720h   # 30 jours (au lieu de trop tard)

  # Forcer le renouvellement si nécessaire
  secretName: mon-certificat-tls
```

**Forcer un renouvellement immédiat**
```bash
# Méthode 1 : annotation de renouvellement forcé
kubectl annotate certificate mon-certificat cert-manager.io/force-renewal=$(date +%s)

# Méthode 2 : supprimer le secret pour recréation
kubectl delete secret mon-certificat-tls
```

### 7. Problèmes de performance et latence

**Symptômes**
- Établissement de connexion SSL lent
- Timeouts lors des handshakes TLS
- Performance dégradée des applications

**Diagnostic**
```bash
# Tester la performance de la négociation TLS
time openssl s_client -connect mon-service.com:443 -servername mon-service.com < /dev/null

# Vérifier la taille et le type de certificat
kubectl get secret mon-certificat-tls -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout | grep -E "(Public Key|Signature Algorithm)"
```

**Solutions**

**Optimiser l'algorithme de clé**
```yaml
# Utiliser ECDSA pour de meilleures performances
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: performance-cert
spec:
  privateKey:
    algorithm: ECDSA
    size: 256  # Plus rapide que RSA 2048
```

**Configurer la mise en cache des certificats**
```yaml
# Configuration NGINX avec cache SSL
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ssl-config
data:
  ssl_session_cache: "shared:SSL:10m"
  ssl_session_timeout: "10m"
  ssl_session_tickets: "off"
```

## Outils de diagnostic avancés

### Scripts de vérification automatisée

```bash
#!/bin/bash
# Script de diagnostic complet des certificats

NAMESPACE=${1:-default}
CERTIFICATE=${2}

echo "=== Diagnostic des certificats ==="
echo "Namespace: $NAMESPACE"

if [ -n "$CERTIFICATE" ]; then
  echo "Certificat: $CERTIFICATE"

  # Vérification spécifique d'un certificat
  echo -e "\n--- État du certificat ---"
  kubectl get certificate $CERTIFICATE -n $NAMESPACE

  echo -e "\n--- Détails du certificat ---"
  kubectl describe certificate $CERTIFICATE -n $NAMESPACE

  echo -e "\n--- CertificateRequest associées ---"
  kubectl get certificaterequests -n $NAMESPACE \
    -l cert-manager.io/certificate-name=$CERTIFICATE

  echo -e "\n--- Challenges en cours ---"
  kubectl get challenges -n $NAMESPACE

  echo -e "\n--- Vérification du secret ---"
  SECRET_NAME=$(kubectl get certificate $CERTIFICATE -n $NAMESPACE \
    -o jsonpath='{.spec.secretName}')
  if kubectl get secret $SECRET_NAME -n $NAMESPACE &>/dev/null; then
    echo "Secret $SECRET_NAME existe"

    # Vérifier la validité du certificat
    CERT_DATA=$(kubectl get secret $SECRET_NAME -n $NAMESPACE \
      -o jsonpath='{.data.tls\.crt}' | base64 -d)

    echo "Émetteur: $(echo "$CERT_DATA" | openssl x509 -noout -issuer | cut -d= -f2-)"
    echo "Expire: $(echo "$CERT_DATA" | openssl x509 -noout -enddate | cut -d= -f2)"
    echo "Domaines: $(echo "$CERT_DATA" | openssl x509 -noout -text | \
      grep -A1 "Subject Alternative Name" | tail -1 | sed 's/DNS://g')"
  else
    echo "❌ Secret $SECRET_NAME n'existe pas"
  fi

else
  # Vérification globale
  echo -e "\n--- Tous les certificats du namespace ---"
  kubectl get certificates -n $NAMESPACE -o wide

  echo -e "\n--- Certificats en échec ---"
  kubectl get certificates -n $NAMESPACE -o json | \
    jq -r '.items[] | select(.status.conditions[]?.status == "False") | .metadata.name'

  echo -e "\n--- Challenges en cours ---"
  kubectl get challenges -n $NAMESPACE
fi

echo -e "\n--- État de cert-manager ---"
kubectl get pods -n cert-manager

echo -e "\n--- Événements récents ---"
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp' | tail -10
```

### Tests de connectivité automatisés

```bash
#!/bin/bash
# Script de test de connectivité SSL

DOMAINS_FILE="domains.txt"  # Liste des domaines à tester

echo "=== Test de connectivité SSL ==="

while IFS= read -r domain; do
  echo -n "Testing $domain... "

  # Test de connectivité basique
  if curl -s --max-time 10 "https://$domain" > /dev/null 2>&1; then
    echo -n "✅ HTTPS OK "

    # Vérifier le certificat
    EXPIRY=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null | \
      openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)

    if [ -n "$EXPIRY" ]; then
      EXPIRY_TIMESTAMP=$(date -d "$EXPIRY" +%s 2>/dev/null)
      CURRENT_TIMESTAMP=$(date +%s)
      DAYS_LEFT=$(( ($EXPIRY_TIMESTAMP - $CURRENT_TIMESTAMP) / 86400 ))

      if [ $DAYS_LEFT -gt 30 ]; then
        echo "✅ Expire dans $DAYS_LEFT jours"
      elif [ $DAYS_LEFT -gt 7 ]; then
        echo "⚠️  Expire dans $DAYS_LEFT jours"
      else
        echo "❌ Expire dans $DAYS_LEFT jours - URGENT"
      fi
    else
      echo "❓ Impossible de vérifier l'expiration"
    fi
  else
    echo "❌ HTTPS KO"
  fi
done < "$DOMAINS_FILE"
```

### Monitoring avec métriques personnalisées

```bash
#!/bin/bash
# Script pour exporter des métriques personnalisées

# Fichier de métriques pour Prometheus
METRICS_FILE="/tmp/cert_metrics.prom"

echo "# HELP certificate_days_until_expiry Days until certificate expiry" > $METRICS_FILE
echo "# TYPE certificate_days_until_expiry gauge" >> $METRICS_FILE

kubectl get certificates --all-namespaces -o json | \
  jq -r '.items[] | select(.status.notAfter != null) |
    [
      .metadata.namespace,
      .metadata.name,
      .status.notAfter
    ] | @tsv' | \
while IFS=$'\t' read namespace name expiry; do

  EXPIRY_TIMESTAMP=$(date -d "$expiry" +%s 2>/dev/null)
  CURRENT_TIMESTAMP=$(date +%s)
  DAYS_LEFT=$(( ($EXPIRY_TIMESTAMP - $CURRENT_TIMESTAMP) / 86400 ))

  echo "certificate_days_until_expiry{namespace=\"$namespace\",certificate=\"$name\"} $DAYS_LEFT" >> $METRICS_FILE
done

echo "Métriques exportées vers $METRICS_FILE"
```

## Prévention des problèmes

### Checklist de déploiement

```markdown
## Checklist pré-déploiement certificat

### Configuration de base
- [ ] Issuer/ClusterIssuer configuré et fonctionnel
- [ ] Annotations correctes dans l'Ingress
- [ ] Section TLS présente avec hosts et secretName
- [ ] DNS pointant vers le cluster

### Validation ACME
- [ ] Pour HTTP-01 : port 80 accessible depuis Internet
- [ ] Pour DNS-01 : API DNS configurée avec bonnes permissions
- [ ] Test de propagation DNS effectué
- [ ] Limites de débit Let's Encrypt vérifiées

### Monitoring
- [ ] Alertes configurées pour expiration
- [ ] Dashboard de monitoring en place
- [ ] Scripts de vérification automatisée déployés
- [ ] Runbook de dépannage documenté

### Tests
- [ ] Test de connectivité HTTPS
- [ ] Validation du certificat avec navigateur
- [ ] Test de renouvellement en staging
- [ ] Vérification des logs cert-manager
```

### Bonnes pratiques préventives

**Tests systématiques en staging**
- Déployer d'abord en environnement de test
- Utiliser l'environnement staging de Let's Encrypt
- Valider le processus complet avant la production

**Monitoring proactif**
- Alertes configurées au minimum 7 jours avant expiration
- Vérification quotidienne automatisée
- Dashboard de vue d'ensemble

**Documentation et formation**
- Runbook de dépannage accessible à l'équipe
- Formation des équipes sur les outils de diagnostic
- Retour d'expérience (postmortem) après chaque incident

**Automatisation maximale**
- Renouvellement automatique configuré
- Tests automatisés de connectivité
- Déploiement automatisé des certificats

## Cas d'étude : Résolution d'un problème complexe

### Scénario : Service inaccessible après renouvellement

**Symptômes observés**
- Service accessible en HTTP mais pas en HTTPS
- Erreur "ERR_SSL_VERSION_OR_CIPHER_MISMATCH" dans le navigateur
- Certificat présent et valide selon kubectl

**Investigation étape par étape**

```bash
# 1. Vérifier l'état du certificat
kubectl get certificate api-cert
# Résultat : READY: True, AGE: 2d

# 2. Vérifier le secret TLS
kubectl get secret api-cert-tls -o yaml
# Résultat : Secret existe avec données tls.crt et tls.key

# 3. Vérifier le contenu du certificat
kubectl get secret api-cert-tls -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout
# Résultat : Certificat valide, domaines corrects, expiration dans 60 jours

# 4. Tester la connectivité directe
curl -v https://api.example.com
# Résultat : Erreur SSL handshake

# 5. Examiner les logs de l'Ingress Controller
kubectl logs -n ingress-nginx deployment/nginx-ingress-controller
# Résultat : Erreurs de chargement de certificat

# 6. Vérifier la configuration de l'Ingress
kubectl get ingress api-ingress -o yaml
# Découverte : secretName pointe vers l'ancien nom de secret

# 7. Corriger la configuration
kubectl patch ingress api-ingress -p '{"spec":{"tls":[{"hosts":["api.example.com"],"secretName":"api-cert-tls"}]}}'

# 8. Vérifier le fonctionnement
curl -I https://api.example.com
# Résultat : HTTP/2 200 - problème résolu
```

**Leçons apprises**
- Toujours vérifier la cohérence entre les noms de secrets
- Les logs de l'Ingress Controller sont cruciaux pour le diagnostic
- Un certificat valide ne garantit pas son utilisation correcte

## Ressources et références

### Commandes de référence rapide

```bash
# État des certificats
kubectl get certificates --all-namespaces
kubectl describe certificate <nom>

# Challenges et demandes
kubectl get challenges --all-namespaces
kubectl get certificaterequests --all-namespaces

# Logs cert-manager
kubectl logs -n cert-manager deployment/cert-manager -f

# Test de certificat
openssl s_client -connect <domain>:443 -servername <domain>

# Forcer un renouvellement
kubectl annotate certificate <nom> cert-manager.io/force-renewal=$(date +%s)
```

### Documentation officielle

- [Cert-Manager Troubleshooting](https://cert-manager.io/docs/troubleshooting/)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [OpenSSL Command Reference](https://www.openssl.org/docs/manmaster/man1/)

### Communauté et support

- [Cert-Manager GitHub Issues](https://github.com/cert-manager/cert-manager/issues)
- [Kubernetes Slack #cert-manager](https://kubernetes.slack.com/channels/cert-manager)
- [Let's Encrypt Community](https://community.letsencrypt.org/)

## Conclusion

Le dépannage des certificats SSL/TLS peut sembler complexe, mais avec une approche méthodique et les bons outils, la plupart des problèmes peuvent être résolus efficacement. La clé du succès réside dans :

- **La prévention** : monitoring proactif et bonnes pratiques de déploiement
- **La méthode** : approche systématique du diagnostic
- **Les outils** : maîtrise des commandes et scripts de diagnostic
- **La documentation** : capitalisation sur les problèmes résolus

Avec ces compétences acquises, vous êtes maintenant capable de gérer une infrastructure SSL/TLS robuste et fiable pour votre lab MicroK8s, capable de gérer automatiquement les certificats tout en étant préparé à résoudre rapidement les problèmes occasionnels.

⏭️
