🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.7 Test de connectivité externe

## Introduction

Après avoir configuré DNS, redirection de ports et firewall, il est temps de valider que votre lab MicroK8s est réellement accessible depuis Internet. Cette section vous guidera à travers une série de tests méthodiques pour vérifier chaque composant de votre configuration réseau et diagnostiquer les problèmes éventuels.

## Méthodologie de test

### Approche progressive

```
Stratégie de test en cascade :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Local (localhost)
   ↓ Si OK
2. Réseau local (LAN)
   ↓ Si OK
3. DNS public
   ↓ Si OK
4. Connectivité externe
   ↓ Si OK
5. Services applicatifs
   ↓ Si OK
6. Certificats SSL
   ↓ Si OK
✅ Lab pleinement opérationnel !

À chaque étape : Diagnostic si échec
```

### Checklist pré-tests

```
Avant de commencer les tests :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration :
☐ MicroK8s démarré (microk8s status)
☐ Application déployée
☐ Ingress configuré
☐ DNS configuré (A records)
☐ Ports redirigés (80, 443)
☐ Firewall configuré

Outils nécessaires :
☐ Terminal/Console
☐ Navigateur web
☐ Smartphone (pour test 4G)
☐ Accès interface routeur
```

## Étape 1 : Tests locaux

### Vérifier les services MicroK8s

```bash
# État du cluster
━━━━━━━━━━━━━━━━

microk8s status --wait-ready

# Services essentiels doivent être "enabled" :
# dns, ingress, storage

# Vérifier les pods système
kubectl get pods -A

# Tous les pods doivent être "Running"
# Aucun pod en "CrashLoopBackOff" ou "Error"
```

### Test localhost

```bash
# Test direct du service
━━━━━━━━━━━━━━━━━━━━━━━

# Si NodePort configuré
curl http://localhost:30080

# Via Ingress Controller
curl -H "Host: app.monlab.com" http://localhost

# Test HTTPS (accepter certificat auto-signé)
curl -k https://localhost

# Résultat attendu :
# HTML de votre application
# Pas de "Connection refused"
```

### Vérifier l'Ingress

```bash
# Configuration Ingress
━━━━━━━━━━━━━━━━━━━━━

# Lister les Ingress
kubectl get ingress -A

# Détails d'un Ingress
kubectl describe ingress mon-ingress

# Vérifier les backends
kubectl get endpoints

# Logs Ingress Controller
kubectl logs -n ingress deployment/nginx-ingress-microk8s-controller -f

# Points à vérifier :
# - ADDRESS doit avoir une IP
# - HOSTS correspond à vos domaines
# - Pas d'erreurs dans les logs
```

## Étape 2 : Tests réseau local

### Test depuis une autre machine

```bash
# Depuis un autre PC/laptop du réseau
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Par IP directe
curl http://192.168.1.100
wget http://192.168.1.100

# Avec nom d'hôte (si DNS local configuré)
curl http://app.lab.local

# Test navigateur
# Ouvrir : http://192.168.1.100

# Si échec, vérifier :
# - Firewall autorise LAN
# - IP correcte
# - Service écoute sur toutes interfaces
```

### Diagnostic connectivité LAN

```bash
# Tests de diagnostic réseau
━━━━━━━━━━━━━━━━━━━━━━━━━━━

# 1. Ping la machine
ping 192.168.1.100

# 2. Scan des ports ouverts
nmap 192.168.1.100
# ou
nc -zv 192.168.1.100 80
nc -zv 192.168.1.100 443

# 3. Traceroute
traceroute 192.168.1.100

# 4. Test avec telnet
telnet 192.168.1.100 80
# Taper : GET / HTTP/1.0 [Enter][Enter]

# Problèmes courants :
# "Destination unreachable" → Problème réseau/routage
# "Connection refused" → Port fermé ou service down
# "Timeout" → Firewall bloque
```

## Étape 3 : Tests DNS

### Vérification propagation DNS

```bash
# Test résolution DNS locale
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Résolution simple
nslookup monlab.com
dig monlab.com
host monlab.com

# Résolution avec serveur DNS spécifique
dig @8.8.8.8 monlab.com     # Google DNS
dig @1.1.1.1 monlab.com     # Cloudflare
dig @208.67.222.222 monlab.com  # OpenDNS

# Vérifier tous les enregistrements
dig monlab.com ANY

# Trace complète de résolution
dig +trace monlab.com
```

### Test propagation mondiale

```
Sites web de vérification :
━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. whatsmydns.net
   - Carte mondiale des serveurs DNS
   - Voir propagation en temps réel
   - ✅ Vert = Propagé
   - ❌ Rouge = Pas encore

2. dnschecker.org
   - Plus de serveurs testés
   - Historique de propagation

3. dnsmap.io
   - Visualisation graphique
   - Timeline de propagation

Objectif : >80% de propagation
Patience : Jusqu'à 48h pour 100%
```

### Diagnostic problèmes DNS

```bash
# Script de diagnostic DNS
#!/bin/bash

DOMAIN="monlab.com"
EXPECTED_IP="82.123.45.67"

echo "=== Test DNS pour $DOMAIN ==="

# Test différents serveurs DNS
for dns in 8.8.8.8 1.1.1.1 208.67.222.222 9.9.9.9; do
    echo -n "DNS $dns : "
    RESULT=$(dig @$dns +short $DOMAIN)
    if [ "$RESULT" = "$EXPECTED_IP" ]; then
        echo "✅ OK ($RESULT)"
    else
        echo "❌ Erreur ($RESULT)"
    fi
done

# Vérifier TTL
echo -e "\n=== TTL actuel ==="
dig $DOMAIN | grep "^$DOMAIN" | awk '{print "TTL: "$2" secondes"}'

# Vérifier nameservers
echo -e "\n=== Nameservers ==="
dig NS $DOMAIN +short
```

## Étape 4 : Test connectivité externe

### Test depuis mobile 4G

```
Test smartphone (méthode simple) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Désactiver WiFi (utiliser 4G/5G)
2. Ouvrir navigateur
3. Taper : http://monlab.com
4. Résultats possibles :

✅ Site s'affiche → Succès total !
⏱️ Timeout → Ports bloqués ou firewall
🔒 Erreur certificat → Normal, accepter
📄 Page routeur → Mauvaise redirection
❌ Site introuvable → DNS pas propagé
```

### Test avec services en ligne

```bash
# Services de test HTTP
━━━━━━━━━━━━━━━━━━━━━━━

# 1. Down for everyone or just me
https://downforeveryoneorjustme.com/monlab.com

# 2. Is it up?
https://isitup.org/monlab.com

# 3. Uptime Robot (monitoring gratuit)
# Créer compte et moniteur

# 4. GTmetrix (test performance)
https://gtmetrix.com
# Entrer URL, analyser
```

### Test avec curl externe

```bash
# Via service web de test
━━━━━━━━━━━━━━━━━━━━━━━━

# 1. reqbin.com
# Interface web pour tester requêtes HTTP

# 2. webhook.site
# Créer URL unique
# Configurer webhook dans votre app
# Voir requêtes arriver

# 3. Via VPS ou ami
ssh user@vps-externe
curl -v http://monlab.com
```

### Test des ports

```
Vérification ports ouverts :
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Sites web :
1. canyouseeme.org
   - Port: 80 → Check
   - Port: 443 → Check

2. portchecker.co
   - Entrer IP et port
   - Résultat immédiat

3. yougetsignal.com/tools/open-ports
   - Test multiple ports

Commande (depuis externe) :
nmap -Pn -p 80,443 monlab.com

Résultats :
• "Open" = ✅ Accessible
• "Filtered" = 🔥 Firewall bloque
• "Closed" = ❌ Rien n'écoute
```

## Étape 5 : Tests applicatifs

### Test HTTP complet

```bash
# Test avec headers détaillés
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

curl -v http://monlab.com

# Analyser la réponse :
> GET / HTTP/1.1
> Host: monlab.com
> User-Agent: curl/7.68.0
>
< HTTP/1.1 200 OK
< Server: nginx
< Date: Mon, 11 Aug 2025 10:00:00 GMT
< Content-Type: text/html

# Points importants :
# - Code 200 = OK
# - Server nginx = Ingress fonctionne
# - Content = Votre application
```

### Test HTTPS et certificats

```bash
# Test certificat SSL
━━━━━━━━━━━━━━━━━━━━

# Infos certificat
openssl s_client -connect monlab.com:443 -servername monlab.com

# Vérifier certificat
curl -vI https://monlab.com

# Test avec navigateur
# Cadenas 🔒 doit apparaître
# Cliquer pour voir détails certificat

# Si auto-signé :
# - Warning normal
# - Accepter exception
# - Procéder au site
```

### Test de charge basique

```bash
# Test performance simple
━━━━━━━━━━━━━━━━━━━━━━━━

# Apache Bench (ab)
sudo apt install apache2-utils

# 100 requêtes, 10 simultanées
ab -n 100 -c 10 http://monlab.com/

# Analyser :
# - Requests per second
# - Time per request
# - Failed requests (doit être 0)

# Avec curl en boucle
for i in {1..10}; do
    time curl -s http://monlab.com > /dev/null
done
```

## Étape 6 : Tests avancés

### Test WebSocket (si applicable)

```javascript
// Test WebSocket depuis console navigateur
const ws = new WebSocket('wss://monlab.com/ws');

ws.onopen = () => {
    console.log('✅ WebSocket connecté');
    ws.send('Test message');
};

ws.onmessage = (event) => {
    console.log('Message reçu:', event.data);
};

ws.onerror = (error) => {
    console.log('❌ Erreur WebSocket:', error);
};
```

### Test API REST

```bash
# Tests API complets
━━━━━━━━━━━━━━━━━━━

# GET simple
curl https://monlab.com/api/health

# POST avec données
curl -X POST https://monlab.com/api/data \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'

# Avec authentification
curl -H "Authorization: Bearer TOKEN" \
  https://monlab.com/api/protected

# Test avec Postman ou Insomnia
# - Créer collection
# - Tester tous endpoints
# - Vérifier temps réponse
```

### Test multi-domaines

```bash
# Si plusieurs sous-domaines
━━━━━━━━━━━━━━━━━━━━━━━━━━━

DOMAINS="monlab.com www.monlab.com api.monlab.com app.monlab.com"

for domain in $DOMAINS; do
    echo "Test $domain :"
    curl -sI $domain | head -1
done

# Tous doivent répondre HTTP/1.1 200 OK
# ou redirection appropriée (301/302)
```

## Diagnostic des problèmes

### Arbre de décision

```
Problème → Solution
━━━━━━━━━━━━━━━━━━

"Connection refused" local
→ Service pas démarré
→ Mauvais port
→ Binding localhost only

"Timeout" depuis LAN
→ Firewall local bloque
→ Mauvaise IP

"Timeout" depuis Internet
→ Ports pas redirigés
→ Firewall routeur
→ FAI bloque ports

"DNS not found"
→ Propagation incomplète
→ Erreur configuration DNS
→ Typo dans l'URL

"Bad Gateway 502"
→ Backend down
→ Mauvaise config Ingress

"Certificate error"
→ Normal si auto-signé
→ Vérifier nom dans cert
```

### Logs à vérifier

```bash
# Ordre de vérification des logs
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Ingress Controller
kubectl logs -n ingress deployment/nginx-ingress-microk8s-controller

2. Application
kubectl logs deployment/mon-app

3. Firewall
sudo tail -f /var/log/ufw.log

4. Système
sudo journalctl -xe

5. DNS (si problème résolution)
kubectl logs -n kube-system deployment/coredns
```

### Script de diagnostic complet

```bash
#!/bin/bash
# diagnostic-complet.sh

DOMAIN="monlab.com"
IP_PUBLIC="82.123.45.67"
IP_LOCAL="192.168.1.100"

echo "=== DIAGNOSTIC CONNECTIVITÉ ==="
echo "Domain: $DOMAIN"
echo "IP Publique: $IP_PUBLIC"
echo "IP Locale: $IP_LOCAL"
echo ""

# 1. Test local
echo "1. TEST LOCAL"
if curl -s http://localhost > /dev/null; then
    echo "✅ Localhost accessible"
else
    echo "❌ Localhost inaccessible"
fi

# 2. Test LAN
echo -e "\n2. TEST LAN"
if ping -c 1 $IP_LOCAL > /dev/null 2>&1; then
    echo "✅ Machine pingable"
else
    echo "❌ Machine non pingable"
fi

# 3. Test ports
echo -e "\n3. TEST PORTS"
for port in 80 443; do
    if nc -zv $IP_LOCAL $port 2>&1 | grep succeeded > /dev/null; then
        echo "✅ Port $port ouvert"
    else
        echo "❌ Port $port fermé"
    fi
done

# 4. Test DNS
echo -e "\n4. TEST DNS"
DNS_RESULT=$(dig +short $DOMAIN)
if [ "$DNS_RESULT" = "$IP_PUBLIC" ]; then
    echo "✅ DNS résout correctement"
else
    echo "❌ DNS incorrect: $DNS_RESULT"
fi

# 5. Test HTTP externe
echo -e "\n5. TEST HTTP EXTERNE"
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$DOMAIN)
if [ "$HTTP_CODE" = "200" ]; then
    echo "✅ HTTP accessible (code $HTTP_CODE)"
else
    echo "❌ HTTP problème (code $HTTP_CODE)"
fi

# 6. Test HTTPS
echo -e "\n6. TEST HTTPS"
if curl -ks https://$DOMAIN > /dev/null; then
    echo "✅ HTTPS accessible"
else
    echo "❌ HTTPS inaccessible"
fi

echo -e "\n=== RÉSUMÉ ==="
echo "Si tous les tests sont ✅ = Configuration réussie !"
echo "Si certains ❌ = Vérifier la section correspondante"
```

## Monitoring continu

### Mise en place monitoring

```
Services de monitoring gratuits :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. UptimeRobot
   - 50 moniteurs gratuits
   - Check toutes les 5 min
   - Alertes email
   - Page status publique

2. StatusCake
   - Monitoring HTTP/HTTPS
   - Alertes multiples
   - Historique uptime

3. Pingdom (essai gratuit)
   - Monitoring pro
   - Analyse performance

Configuration type :
- URL: https://monlab.com
- Intervalle: 5 minutes
- Alertes: Email + SMS
- Locations: Multiple
```

### Script monitoring local

```bash
#!/bin/bash
# monitor.sh - À exécuter via cron

URL="https://monlab.com"
LOGFILE="/var/log/monitor.log"
ALERT_EMAIL="admin@example.com"

# Test accessibilité
HTTP_CODE=$(curl -ks -o /dev/null -w "%{http_code}" $URL)
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

if [ "$HTTP_CODE" = "200" ]; then
    echo "$TIMESTAMP - OK - HTTP $HTTP_CODE" >> $LOGFILE
else
    echo "$TIMESTAMP - ERREUR - HTTP $HTTP_CODE" >> $LOGFILE
    echo "Site down! Code: $HTTP_CODE" | mail -s "ALERTE: $URL" $ALERT_EMAIL
fi

# Garder seulement 7 jours de logs
find $LOGFILE -mtime +7 -delete

# Crontab : */5 * * * * /path/to/monitor.sh
```

## Validation finale

### Checklist de succès

```
Configuration validée si :
━━━━━━━━━━━━━━━━━━━━━━━━━━

Accessibilité :
☐ Local : http://localhost ✅
☐ LAN : http://192.168.1.100 ✅
☐ DNS : dig monlab.com → IP correcte ✅
☐ Internet : http://monlab.com ✅
☐ HTTPS : https://monlab.com ✅

Performance :
☐ Temps réponse < 1s
☐ Pas d'erreurs 502/503
☐ Charge supportée (10 req/s minimum)

Sécurité :
☐ Seuls ports 80/443 ouverts
☐ Certificat SSL valide/accepté
☐ Firewall actif

Monitoring :
☐ Uptime monitoring configuré
☐ Alertes testées
☐ Logs accessibles
```

### Test utilisateur final

```
Simulation utilisateur réel :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Ouvrir navigateur privé/incognito
2. Vider cache et cookies
3. Naviguer vers https://monlab.com
4. Vérifier :
   - Page charge complètement
   - Pas d'erreurs console (F12)
   - Images chargent
   - CSS appliqué
   - JavaScript fonctionne
   - Formulaires soumettent
   - Navigation entre pages

5. Test sur différents appareils :
   - PC Windows/Mac/Linux
   - Smartphone iOS/Android
   - Tablette
   - Différents navigateurs
```

## Documentation des tests

### Template de rapport

```markdown
# Rapport de Test Connectivité
Date : 2025-08-11
Domaine : monlab.com
IP Publique : 82.123.45.67

## Tests Effectués

### Local
- [✅] Localhost accessible
- [✅] Services K8s running
- [✅] Ingress configuré

### Réseau
- [✅] LAN accessible
- [✅] Ports 80/443 ouverts
- [✅] Firewall configuré

### DNS
- [✅] Propagation complète
- [✅] Résolution correcte

### Internet
- [✅] HTTP accessible
- [✅] HTTPS fonctionnel
- [✅] Performance acceptable

## Problèmes Rencontrés
1. [Description du problème]
   - Solution : [Comment résolu]

## Métriques
- Temps DNS : 50ms
- Temps réponse : 200ms
- Uptime : 99.9%

## Prochaines Étapes
- [ ] Configurer CDN
- [ ] Optimiser performance
- [ ] Ajouter monitoring avancé
```

## Points clés à retenir

1. **Tester progressivement** : Local → LAN → DNS → Internet
2. **Utiliser multiple sources** : Ne pas se fier qu'à un test
3. **Documenter les problèmes** : Pour référence future
4. **Monitoring continu** : Les problèmes arrivent après
5. **Tester comme un utilisateur** : Navigateur, différents appareils
6. **Patience avec DNS** : Propagation prend du temps

## Résumé

Les tests de connectivité externe valident que votre configuration réseau fonctionne de bout en bout. En suivant une méthodologie progressive et en utilisant les bons outils de diagnostic, vous pouvez rapidement identifier et résoudre les problèmes. Une fois tous les tests passés avec succès, votre lab MicroK8s est officiellement accessible depuis Internet, prêt à héberger vos applications et services pour le monde entier !

⏭️
