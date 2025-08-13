🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.3 Problèmes réseau courants

## Introduction aux défis réseau dans Kubernetes

Le réseau est l'une des parties les plus complexes de Kubernetes, et par conséquent, une source fréquente de problèmes. Dans MicroK8s, bien que le réseau soit simplifié par rapport à une installation Kubernetes complète, vous rencontrerez tout de même des défis. Cette section vous aidera à comprendre, diagnostiquer et résoudre les problèmes réseau les plus courants, depuis les simples erreurs de connectivité jusqu'aux problèmes plus subtils de routage et de DNS.

## Comprendre l'architecture réseau de MicroK8s

### Vue d'ensemble du modèle réseau

Avant de plonger dans les problèmes, il est essentiel de comprendre comment fonctionne le réseau dans MicroK8s. Le modèle réseau Kubernetes repose sur plusieurs principes fondamentaux :

**Communication pod-to-pod** : Tous les pods peuvent communiquer entre eux sans NAT (Network Address Translation). Chaque pod reçoit sa propre adresse IP dans un réseau virtuel interne. Dans MicroK8s, ce réseau utilise généralement la plage 10.1.0.0/16 par défaut.

**Services et endpoints** : Les Services Kubernetes fournissent une abstraction stable pour accéder aux pods. Ils utilisent un système de proxy (kube-proxy) pour router le trafic vers les pods appropriés. Les Services obtiennent des IPs dans une plage différente, typiquement 10.152.183.0/24.

**DNS interne** : CoreDNS fournit la résolution de noms à l'intérieur du cluster. Chaque service est accessible via un nom DNS de la forme `<service>.<namespace>.svc.cluster.local`.

**Accès externe** : L'accès depuis l'extérieur du cluster se fait via NodePort, LoadBalancer (avec MetalLB dans MicroK8s), ou Ingress Controller.

### Composants réseau critiques

Les composants suivants sont essentiels au bon fonctionnement du réseau :

- **Calico/Flannel** : Le plugin CNI (Container Network Interface) qui gère le réseau des pods
- **CoreDNS** : Le serveur DNS interne du cluster
- **kube-proxy** : Gère les règles iptables pour le routage des services
- **Ingress Controller** : Gère l'accès HTTP/HTTPS externe

## Problème 1 : Les pods ne peuvent pas communiquer entre eux

### Symptômes

- Les pods ne peuvent pas se ping mutuellement
- Les connexions TCP/UDP échouent entre pods
- Messages d'erreur type "connection refused" ou "no route to host"

### Diagnostic

Commencez par vérifier la configuration réseau de base :

```bash
# Vérifier que le plugin CNI est actif
microk8s kubectl get pods -n kube-system | grep calico

# Vérifier les interfaces réseau dans un pod
microk8s kubectl exec -it <pod-name> -- ip addr show

# Tester la connectivité entre deux pods
# Depuis le pod A, obtenir son IP
microk8s kubectl get pod <pod-a> -o wide

# Depuis le pod B, ping le pod A
microk8s kubectl exec -it <pod-b> -- ping <ip-pod-a>
```

Vérifiez les Network Policies qui pourraient bloquer le trafic :

```bash
# Lister toutes les Network Policies
microk8s kubectl get networkpolicies --all-namespaces

# Détailler une policy spécifique
microk8s kubectl describe networkpolicy <policy-name> -n <namespace>
```

### Résolution

Si le plugin CNI n'est pas fonctionnel :

```bash
# Redémarrer Calico
microk8s kubectl delete pods -n kube-system -l k8s-app=calico-node
# Les pods seront recréés automatiquement

# Si le problème persiste, vérifier les logs
microk8s kubectl logs -n kube-system -l k8s-app=calico-node
```

Pour les problèmes de Network Policy :

```bash
# Temporairement supprimer une policy pour tester
microk8s kubectl delete networkpolicy <policy-name> -n <namespace>

# Créer une policy qui permet tout le trafic (pour debug uniquement)
cat <<EOF | microk8s kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - {}
EOF
```

## Problème 2 : Le DNS ne fonctionne pas

### Symptômes

- Impossible de résoudre les noms de services
- Erreurs "nslookup: can't resolve"
- Les pods ne peuvent pas accéder aux services par leur nom

### Diagnostic

Testez d'abord le DNS depuis un pod de test :

```bash
# Créer un pod de debug avec des outils réseau
microk8s kubectl run dnstest --image=busybox:latest --rm -it -- /bin/sh

# Dans le pod, tester les résolutions DNS
nslookup kubernetes.default
nslookup google.com
cat /etc/resolv.conf
```

Vérifiez l'état de CoreDNS :

```bash
# Vérifier que CoreDNS est en cours d'exécution
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Examiner les logs de CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Vérifier la configuration de CoreDNS
microk8s kubectl get configmap coredns -n kube-system -o yaml
```

Testez la résolution DNS directement :

```bash
# Obtenir l'IP du service DNS
microk8s kubectl get service -n kube-system kube-dns

# Tester avec dig ou nslookup directement vers CoreDNS
microk8s kubectl run test-dns --image=tutum/dnsutils --rm -it -- /bin/sh
# Dans le pod:
nslookup kubernetes.default 10.152.183.10  # Remplacer par l'IP de kube-dns
dig @10.152.183.10 kubernetes.default.svc.cluster.local
```

### Résolution

Pour réparer CoreDNS :

```bash
# Redémarrer CoreDNS
microk8s kubectl rollout restart deployment/coredns -n kube-system

# Si CoreDNS crash en boucle, vérifier les ressources
microk8s kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Augmenter les ressources si nécessaire
microk8s kubectl edit deployment coredns -n kube-system
# Modifier les sections resources.requests et resources.limits
```

Pour les problèmes de configuration DNS dans les pods :

```bash
# Vérifier la configuration DNS du pod
microk8s kubectl get pod <pod-name> -o yaml | grep -A 10 dnsPolicy

# Forcer une politique DNS spécifique
# Dans le manifest du pod, ajouter:
spec:
  dnsPolicy: ClusterFirst  # ou Default, None, ClusterFirstWithHostNet
  dnsConfig:
    nameservers:
      - 10.152.183.10  # IP de CoreDNS
    searches:
      - default.svc.cluster.local
      - svc.cluster.local
      - cluster.local
```

## Problème 3 : Les Services ne sont pas accessibles

### Symptômes

- Impossible d'accéder à un service via son ClusterIP
- Le service ne route pas vers les pods
- Erreur "connection refused" lors de l'accès au service

### Diagnostic

Vérifiez d'abord la configuration du service :

```bash
# Lister les services et leurs endpoints
microk8s kubectl get services -o wide
microk8s kubectl get endpoints

# Vérifier qu'un service a des endpoints
microk8s kubectl get endpoints <service-name>

# Détailler le service
microk8s kubectl describe service <service-name>
```

Testez l'accès au service :

```bash
# Créer un pod de test
microk8s kubectl run test-svc --image=nicolaka/netshoot --rm -it -- /bin/bash

# Dans le pod, tester l'accès au service
curl http://<service-name>.<namespace>.svc.cluster.local:<port>
nc -zv <service-name>.<namespace>.svc.cluster.local <port>
```

Vérifiez que les sélecteurs correspondent :

```bash
# Comparer les labels du service avec ceux des pods
echo "Service selector:"
microk8s kubectl get service <service-name> -o jsonpath='{.spec.selector}'

echo -e "\nPod labels:"
microk8s kubectl get pods --show-labels
```

### Résolution

Si les endpoints sont vides :

```bash
# Vérifier que les labels des pods correspondent au sélecteur du service
microk8s kubectl label pod <pod-name> <label-key>=<label-value>

# Recréer le service avec les bons sélecteurs
microk8s kubectl delete service <service-name>
microk8s kubectl expose deployment <deployment-name> --port=<port> --target-port=<target-port>
```

Pour les problèmes de kube-proxy :

```bash
# Vérifier les règles iptables (sur le nœud hôte)
sudo iptables -t nat -L -n | grep <service-cluster-ip>

# Redémarrer kube-proxy (intégré dans kubelite)
sudo systemctl restart snap.microk8s.daemon-kubelite

# Vérifier les logs de kube-proxy
journalctl -u snap.microk8s.daemon-kubelite | grep proxy
```

## Problème 4 : L'Ingress ne fonctionne pas

### Symptômes

- Impossible d'accéder aux applications depuis l'extérieur
- Erreur 404 ou 502 Bad Gateway
- Le certificat SSL ne fonctionne pas

### Diagnostic

Vérifiez l'état de l'Ingress Controller :

```bash
# Vérifier que l'addon ingress est activé
microk8s status | grep ingress

# Vérifier les pods de l'ingress controller
microk8s kubectl get pods -n ingress

# Examiner les logs de l'ingress controller
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s
```

Vérifiez la configuration Ingress :

```bash
# Lister tous les Ingress
microk8s kubectl get ingress --all-namespaces

# Détailler un Ingress spécifique
microk8s kubectl describe ingress <ingress-name> -n <namespace>

# Vérifier que l'Ingress a une adresse IP
microk8s kubectl get ingress <ingress-name> -n <namespace>
```

Testez la résolution et l'accès :

```bash
# Vérifier la résolution DNS externe
nslookup <your-domain>

# Tester l'accès direct à l'Ingress Controller
curl -H "Host: <your-domain>" http://<node-ip>

# Avec SSL/TLS
curl -k -H "Host: <your-domain>" https://<node-ip>

# Voir les backends configurés
microk8s kubectl exec -n ingress <nginx-pod> -- cat /etc/nginx/nginx.conf | grep -A 5 "upstream"
```

### Résolution

Pour les problèmes de routage Ingress :

```bash
# Vérifier que le service backend existe et a des endpoints
microk8s kubectl get endpoints <backend-service> -n <namespace>

# Recréer l'Ingress avec la bonne configuration
cat <<EOF | microk8s kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: <namespace>
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: <your-domain>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <service-name>
            port:
              number: <port>
EOF
```

Pour les problèmes SSL/TLS :

```bash
# Vérifier le certificat
microk8s kubectl get certificate --all-namespaces
microk8s kubectl describe certificate <cert-name> -n <namespace>

# Vérifier le secret TLS
microk8s kubectl get secret <tls-secret> -n <namespace>
microk8s kubectl get secret <tls-secret> -n <namespace> -o yaml

# Forcer le renouvellement avec cert-manager
microk8s kubectl delete certificate <cert-name> -n <namespace>
# Le certificat sera recréé automatiquement
```

## Problème 5 : Impossible d'accéder à Internet depuis les pods

### Symptômes

- Les pods ne peuvent pas télécharger des images
- Impossible de faire des requêtes HTTP vers l'extérieur
- Les mises à jour de packages échouent

### Diagnostic

Testez la connectivité externe :

```bash
# Test depuis un pod
microk8s kubectl run test-internet --image=busybox --rm -it -- /bin/sh

# Dans le pod:
ping 8.8.8.8  # Test connectivité IP
nslookup google.com  # Test DNS externe
wget -O- http://www.google.com  # Test HTTP
```

Vérifiez la configuration NAT et le forwarding :

```bash
# Sur le nœud hôte
# Vérifier que le forwarding IP est activé
cat /proc/sys/net/ipv4/ip_forward  # Doit retourner 1

# Vérifier les règles de NAT
sudo iptables -t nat -L POSTROUTING -n -v

# Vérifier la route par défaut dans un pod
microk8s kubectl exec -it <pod-name> -- ip route
```

### Résolution

Activer le forwarding IP si nécessaire :

```bash
# Activation temporaire
sudo sysctl -w net.ipv4.ip_forward=1

# Activation permanente
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Corriger les règles de NAT :

```bash
# Ajouter une règle de masquerade si manquante
sudo iptables -t nat -A POSTROUTING -s 10.1.0.0/16 ! -d 10.1.0.0/16 -j MASQUERADE

# Sauvegarder les règles
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

Pour les problèmes de proxy d'entreprise :

```bash
# Configurer le proxy pour MicroK8s
sudo nano /var/snap/microk8s/current/args/containerd-env

# Ajouter:
HTTP_PROXY=http://proxy.company.com:8080
HTTPS_PROXY=http://proxy.company.com:8080
NO_PROXY=10.0.0.0/8,192.168.0.0/16,127.0.0.1,localhost,.cluster.local

# Redémarrer MicroK8s
microk8s stop
microk8s start
```

## Problème 6 : Performances réseau dégradées

### Symptômes

- Latence élevée entre les pods
- Débit réseau faible
- Timeouts fréquents

### Diagnostic

Testez les performances réseau :

```bash
# Installer iperf3 pour les tests de bande passante
# Pod serveur
microk8s kubectl run iperf-server --image=networkstatic/iperf3 -- -s

# Pod client
microk8s kubectl run iperf-client --image=networkstatic/iperf3 --rm -it -- -c <server-pod-ip>

# Test de latence
microk8s kubectl exec -it <pod-name> -- ping -c 100 <target-pod-ip>
```

Vérifiez l'utilisation des ressources :

```bash
# CPU et mémoire du système
top
htop

# Utilisation réseau
iftop
nethogs

# Statistiques des interfaces
ip -s link show
```

Analysez la configuration MTU :

```bash
# Vérifier le MTU des interfaces
ip link show

# Dans un pod
microk8s kubectl exec -it <pod-name> -- ip link show

# Test de MTU
microk8s kubectl exec -it <pod-name> -- ping -M do -s 1472 <target-ip>
```

### Résolution

Optimiser les paramètres réseau :

```bash
# Ajuster le MTU si nécessaire
sudo ip link set dev cni0 mtu 1450

# Configuration permanente dans Calico
microk8s kubectl edit configmap calico-config -n kube-system
# Modifier: "mtu": "1450"

# Redémarrer Calico
microk8s kubectl rollout restart daemonset/calico-node -n kube-system
```

Optimisation des paramètres kernel :

```bash
# Augmenter les buffers réseau
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# Optimiser les connexions
sudo sysctl -w net.core.somaxconn=1024
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=2048
```

## Problème 7 : Port forwarding ne fonctionne pas

### Symptômes

- `kubectl port-forward` échoue ou se déconnecte
- Impossible d'accéder aux services via port-forward
- Connection reset lors de l'utilisation

### Diagnostic

Testez le port-forward :

```bash
# Tentative de port-forward
microk8s kubectl port-forward pod/<pod-name> 8080:80

# Dans un autre terminal
curl http://localhost:8080

# Avec plus de verbosité pour debug
microk8s kubectl port-forward pod/<pod-name> 8080:80 -v=9
```

Vérifiez les ports locaux :

```bash
# Vérifier si le port est déjà utilisé
sudo netstat -tlnp | grep 8080
sudo lsof -i :8080
```

### Résolution

Pour les problèmes de port-forward :

```bash
# Utiliser une adresse IP spécifique
microk8s kubectl port-forward --address 0.0.0.0 pod/<pod-name> 8080:80

# Pour un service au lieu d'un pod
microk8s kubectl port-forward service/<service-name> 8080:80

# Augmenter le timeout
microk8s kubectl port-forward pod/<pod-name> 8080:80 --pod-running-timeout=5m
```

Alternative avec socat :

```bash
# Créer un pod socat pour le forwarding
microk8s kubectl run socat-proxy --image=alpine/socat --port=8080 \
  -- tcp-listen:8080,fork,reuseaddr tcp-connect:<service-name>:80

# Exposer ce pod
microk8s kubectl expose pod socat-proxy --type=NodePort --port=8080
```

## Outils de diagnostic réseau avancés

### Utilisation de tcpdump

Pour une analyse approfondie du trafic :

```bash
# Capturer le trafic sur l'interface CNI
sudo tcpdump -i cni0 -w capture.pcap

# Capturer dans un pod (nécessite des privilèges)
microk8s kubectl run tcpdump --image=corfr/tcpdump --privileged=true \
  --overrides='{"spec":{"hostNetwork":true}}' -- -i any -w /tmp/capture.pcap

# Filtrer par port
sudo tcpdump -i any port 80

# Filtrer par IP
sudo tcpdump -i any host 10.1.1.10
```

### Utilisation de netshoot

Netshoot est une image Docker contenant de nombreux outils réseau :

```bash
# Déployer netshoot
microk8s kubectl run netshoot --image=nicolaka/netshoot --rm -it -- /bin/bash

# Outils disponibles dans netshoot:
# - curl, wget
# - dig, nslookup, host
# - ping, traceroute, mtr
# - iperf3, netcat
# - tcpdump, tshark
# - ss, netstat
```

### Analyse avec Wireshark

Pour une analyse graphique :

```bash
# Capturer et analyser avec Wireshark
sudo tcpdump -i cni0 -w /tmp/k8s-capture.pcap
# Transférer le fichier sur votre machine locale
# Ouvrir avec Wireshark pour analyse

# Filtres Wireshark utiles:
# - tcp.port == 80
# - http
# - dns
# - tcp.flags.reset == 1  # Pour voir les connexions rejetées
```

## Scripts de diagnostic réseau

### Script de test complet

Créez un script pour automatiser les tests réseau :

```bash
#!/bin/bash
# network-diagnostic.sh

echo "=== Diagnostic Réseau MicroK8s ==="
echo "Date: $(date)"
echo ""

echo "1. État des composants réseau"
microk8s kubectl get pods -n kube-system | grep -E "calico|dns|proxy"

echo -e "\n2. Services et Endpoints"
microk8s kubectl get services --all-namespaces
echo "---"
microk8s kubectl get endpoints --all-namespaces

echo -e "\n3. Test DNS"
microk8s kubectl run dnstest --image=busybox --rm -it --restart=Never -- \
  nslookup kubernetes.default

echo -e "\n4. Test de connectivité externe"
microk8s kubectl run nettest --image=busybox --rm -it --restart=Never -- \
  ping -c 3 8.8.8.8

echo -e "\n5. Règles Ingress"
microk8s kubectl get ingress --all-namespaces

echo -e "\n6. Network Policies"
microk8s kubectl get networkpolicies --all-namespaces

echo -e "\n7. Configuration réseau du nœud"
ip addr show
echo "---"
ip route show

echo -e "\n8. Règles iptables (NAT)"
sudo iptables -t nat -L POSTROUTING -n | head -20
```

### Script de test de connectivité

```bash
#!/bin/bash
# test-connectivity.sh

NAMESPACE=${1:-default}
POD1=$2
POD2=$3

if [ -z "$POD1" ] || [ -z "$POD2" ]; then
    echo "Usage: $0 [namespace] pod1 pod2"
    exit 1
fi

echo "Test de connectivité entre $POD1 et $POD2 dans namespace $NAMESPACE"

# Obtenir les IPs
IP1=$(microk8s kubectl get pod $POD1 -n $NAMESPACE -o jsonpath='{.status.podIP}')
IP2=$(microk8s kubectl get pod $POD2 -n $NAMESPACE -o jsonpath='{.status.podIP}')

echo "IP de $POD1: $IP1"
echo "IP de $POD2: $IP2"

# Test ping
echo -e "\nTest ping $POD1 -> $POD2"
microk8s kubectl exec -n $NAMESPACE $POD1 -- ping -c 3 $IP2

echo -e "\nTest ping $POD2 -> $POD1"
microk8s kubectl exec -n $NAMESPACE $POD2 -- ping -c 3 $IP1

# Test ports si possible
echo -e "\nTest de port (si nc disponible)"
microk8s kubectl exec -n $NAMESPACE $POD1 -- sh -c "command -v nc && nc -zv $IP2 80"
```

## Prévention des problèmes réseau

### Bonnes pratiques

1. **Monitoring proactif** : Mettez en place des alertes pour détecter les problèmes réseau avant qu'ils n'impactent les utilisateurs

2. **Documentation** : Maintenez une documentation de votre architecture réseau, incluant les plages IP, les services exposés, et les règles de firewall

3. **Tests réguliers** : Exécutez des tests de connectivité automatisés périodiquement

4. **Isolation** : Utilisez les Network Policies pour segmenter le réseau et limiter l'impact des problèmes

5. **Ressources suffisantes** : Assurez-vous que les composants réseau (CoreDNS, Ingress Controller) ont suffisamment de ressources

### Configuration préventive

```yaml
# health-check-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: network-health-check
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: network-health
  template:
    metadata:
      labels:
        app: network-health
    spec:
      containers:
      - name: healthcheck
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true; do nslookup kubernetes.default; sleep 30; done"]
        resources:
          limits:
            memory: "64Mi"
            cpu: "50m"
```

## Conclusion

Les problèmes réseau dans Kubernetes peuvent sembler intimidants au début, mais avec une approche méthodique et les bons outils, ils deviennent gérables. La clé est de comprendre l'architecture réseau, de savoir où chercher les informations de diagnostic, et d'avoir une boîte à outils de commandes et de techniques prêtes à l'emploi. N'oubliez pas que la plupart des problèmes réseau ont des causes communes : mauvaise configuration DNS, sélecteurs de service incorrects, ou règles de firewall trop restrictives. En gardant ces points à l'esprit et en utilisant les techniques présentées dans cette section, vous serez bien équipé pour résoudre la majorité des problèmes réseau que vous rencontrerez dans votre cluster MicroK8s.

⏭️
