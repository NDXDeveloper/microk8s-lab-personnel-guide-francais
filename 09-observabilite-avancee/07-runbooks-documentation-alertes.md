🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.7 Runbooks et documentation des alertes

## Introduction : Qu'est-ce qu'un runbook ?

Un **runbook** est un document de référence qui contient des procédures standardisées pour résoudre des problèmes ou répondre à des alertes spécifiques. Imaginez-le comme un "manuel d'urgence" que n'importe qui dans votre équipe (ou vous-même dans 6 mois) peut suivre pour diagnostiquer et résoudre un problème, même sans connaître tous les détails du système.

Dans le contexte de votre lab MicroK8s avec Prometheus et Grafana, les runbooks deviennent essentiels pour :
- Documenter la signification de chaque alerte
- Fournir des étapes de résolution claires
- Réduire le temps de résolution des incidents
- Capitaliser sur les expériences passées

## Pourquoi les runbooks sont-ils importants ?

### Pour un lab personnel
Même dans un environnement de lab, vous allez oublier pourquoi vous avez configuré certaines alertes ou comment résoudre certains problèmes. Les runbooks vous permettent de :
- Ne pas perdre de temps à redécouvrir des solutions
- Apprendre de manière structurée
- Préparer votre lab pour une utilisation professionnelle

### Avantages clés
- **Réduction du stress** : Face à une alerte, vous avez une procédure claire à suivre
- **Cohérence** : Les problèmes sont résolus de la même manière à chaque fois
- **Apprentissage** : Les nouveaux utilisateurs peuvent comprendre rapidement le système
- **Amélioration continue** : Les runbooks évoluent avec votre expérience

## Structure d'un runbook efficace

Un bon runbook suit toujours la même structure pour faciliter sa lecture en situation de stress :

### 1. Métadonnées de l'alerte
```yaml
# Exemple de structure en en-tête du runbook
Nom de l'alerte: HighMemoryUsage
Sévérité: Warning
Service impacté: Application Web
Dernière mise à jour: 2025-01-15
Auteur: VotreNom
```

### 2. Description du problème
Une explication claire de ce que signifie l'alerte, accessible aux débutants :
- Qu'est-ce qui est surveillé ?
- Pourquoi est-ce important ?
- Quel est l'impact potentiel ?

### 3. Seuils et conditions de déclenchement
Les valeurs exactes qui déclenchent l'alerte :
- Seuil critique : > 90% d'utilisation mémoire
- Durée : pendant plus de 5 minutes
- Fréquence de vérification : toutes les 30 secondes

### 4. Impact sur le système
Description concrète de ce qui peut arriver si le problème n'est pas résolu :
- Performance dégradée
- Risque de crash de l'application
- Perte de données potentielle

### 5. Étapes de diagnostic
Liste ordonnée des vérifications à effectuer :
1. Vérifier les métriques actuelles
2. Identifier les pods consommateurs
3. Analyser les logs récents
4. Vérifier l'historique des déploiements

### 6. Actions de résolution
Procédures détaillées pour résoudre le problème, avec les commandes exactes :
```bash
# Exemple de commandes de résolution
kubectl top pods -n production
kubectl describe pod <nom-du-pod>
kubectl logs <nom-du-pod> --tail=100
```

### 7. Escalade
Quand et comment escalader si le problème persiste :
- Après combien de temps d'investigation
- Qui contacter
- Quelles informations fournir

### 8. Prévention
Comment éviter que le problème se reproduise :
- Ajustement des limites de ressources
- Optimisation du code
- Mise à jour de la configuration

## Intégration avec Prometheus Alertmanager

### Lier les runbooks aux alertes

Dans vos règles d'alerte Prometheus, vous pouvez directement référencer les runbooks :

```yaml
groups:
- name: example
  rules:
  - alert: HighMemoryUsage
    expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Utilisation mémoire élevée (instance {{ $labels.instance }})"
      description: "La mémoire est utilisée à {{ $value | humanizePercentage }}"
      runbook_url: "https://votre-wiki/runbooks/high-memory-usage"
```

### Annotations importantes
Les annotations dans Prometheus permettent d'enrichir vos alertes :
- **summary** : Description courte du problème
- **description** : Détails avec les valeurs actuelles
- **runbook_url** : Lien direct vers le runbook
- **dashboard_url** : Lien vers le dashboard Grafana pertinent

## Formats et outils de documentation

### Format Markdown
Le format le plus simple et portable pour vos runbooks :

```markdown
# Runbook: Pod CrashLoopBackOff

## Symptômes
- Le pod redémarre constamment
- Status: CrashLoopBackOff dans kubectl get pods

## Causes possibles
1. Erreur dans l'application
2. Mauvaise configuration
3. Ressources insuffisantes

## Résolution
### Étape 1: Vérifier les logs
`kubectl logs <pod-name> --previous`
...
```

### Wiki interne
Pour un lab plus structuré, utilisez un wiki :
- **DokuWiki** : Simple, sans base de données
- **Wiki.js** : Moderne avec interface agréable
- **BookStack** : Organisé comme des livres
- **MkDocs** : Génère un site statique depuis Markdown

### Intégration dans Git
Stockez vos runbooks dans un repository Git pour :
- Versionner les modifications
- Réviser les changements
- Automatiser le déploiement
- Collaborer facilement

Structure recommandée :
```
runbooks/
├── alerts/
│   ├── high-memory-usage.md
│   ├── pod-crashloop.md
│   └── certificate-expiry.md
├── procedures/
│   ├── backup-restore.md
│   └── cluster-upgrade.md
└── README.md
```

## Templates de runbooks pour alertes courantes

### Template 1: Problème de ressources
```markdown
# Alerte: [NOM_ALERTE]

## Signal
- **Métrique**: memory/cpu/disk usage > X%
- **Durée**: Y minutes
- **Namespace**: production/staging

## Impact
- Applications lentes
- Risque d'OOMKill
- Indisponibilité potentielle

## Investigation
1. Identifier le pod/node concerné
   ```bash
   kubectl top nodes
   kubectl top pods --all-namespaces
   ```
2. Vérifier les limites configurées
3. Analyser l'historique Grafana

## Résolution immédiate
1. Si pod unique: restart
2. Si multiple pods: scale horizontal
3. Si node: cordon et drain

## Prévention
- Ajuster les resource limits
- Implémenter HPA
- Revoir l'architecture
```

### Template 2: Problème de disponibilité
```markdown
# Alerte: Service Unavailable

## Signal
- Probe failure
- 5xx errors > threshold
- Response time > Xs

## Vérifications immédiates
1. État des pods
2. Ingress configuration
3. Certificats SSL
4. DNS resolution

## Actions
1. Rollback si déploiement récent
2. Scale up les replicas
3. Vérifier les dépendances externes
```

## Bonnes pratiques pour la rédaction

### Langage et style
- **Soyez concis** : En situation de crise, personne ne lit les longs paragraphes
- **Utilisez des listes** : Plus faciles à suivre sous stress
- **Commandes copiables** : Toujours fournir les commandes exactes
- **Évitez le jargon** : Expliquez les termes techniques

### Maintenance des runbooks
- **Revue régulière** : Tous les 3 mois minimum
- **Test des procédures** : Vérifiez que les commandes fonctionnent
- **Feedback loop** : Mettez à jour après chaque incident
- **Versioning** : Gardez l'historique des modifications

### Organisation et accessibilité
- **Naming convention** : alert-[service]-[problème].md
- **Index central** : Liste de tous les runbooks disponibles
- **Recherche facile** : Tags et catégories
- **Accès rapide** : Liens directs depuis les alertes

## Automatisation et runbooks dynamiques

### Runbooks as Code
Générez automatiquement certaines parties de vos runbooks :

```python
# Exemple de génération automatique
def generate_runbook(alert_name, metrics, thresholds):
    template = f"""
# Alerte: {alert_name}

## Métriques surveillées
{format_metrics(metrics)}

## Seuils configurés
{format_thresholds(thresholds)}

## Commandes de diagnostic
{generate_kubectl_commands(alert_name)}
"""
    return template
```

### Intégration avec ChatOps
Pour les environnements plus avancés, intégrez vos runbooks avec Slack ou Teams :
- Bot qui affiche le runbook lors d'une alerte
- Commandes interactives pour l'investigation
- Historique des actions prises

## Métriques sur l'utilisation des runbooks

### Tracking de l'efficacité
Mesurez l'impact de vos runbooks :
- **MTTR** (Mean Time To Resolution) : Temps moyen de résolution
- **Taux d'utilisation** : Pourcentage d'alertes résolues avec runbook
- **Feedback score** : Utilité perçue par les utilisateurs

### Dashboard Grafana dédié
Créez un dashboard pour suivre :
- Alertes les plus fréquentes
- Runbooks les plus consultés
- Temps de résolution par type d'alerte
- Corrélation entre documentation et MTTR

## Exemples concrets pour votre lab MicroK8s

### Runbook: Certificate Expiry Warning
```markdown
# Certificat SSL proche de l'expiration

## Alerte déclenchée quand
- Certificat expire dans moins de 7 jours
- Vérification toutes les 6 heures

## Impact
- Warning à 7 jours: préventif
- Critical à 2 jours: action urgente requise
- Expiration: service inaccessible en HTTPS

## Résolution
1. Vérifier le certificat actuel:
   ```bash
   kubectl get certificate -A
   kubectl describe certificate <cert-name> -n <namespace>
   ```

2. Forcer le renouvellement (cert-manager):
   ```bash
   kubectl delete secret <cert-secret> -n <namespace>
   # cert-manager va recréer automatiquement
   ```

3. Vérifier le nouveau certificat:
   ```bash
   openssl s_client -connect <domain>:443 -servername <domain>
   ```

## Si le renouvellement échoue
1. Vérifier les logs cert-manager
2. Vérifier la limite de rate Let's Encrypt
3. DNS challenge accessible ?
4. Fallback sur certificat auto-signé temporaire
```

### Runbook: Prometheus Storage Full
```markdown
# Stockage Prometheus saturé

## Symptômes
- Prometheus ne collecte plus de métriques
- Dashboards Grafana vides
- Alerte "Storage Space Low"

## Investigation rapide
```bash
# Vérifier l'espace disque
kubectl exec -n monitoring prometheus-0 -- df -h

# Taille de la base de données
kubectl exec -n monitoring prometheus-0 -- du -sh /prometheus
```

## Résolution immédiate
1. Augmenter la rétention (perte de données anciennes):
   ```bash
   # Éditer la configmap Prometheus
   kubectl edit configmap prometheus-config -n monitoring
   # Modifier: retention.time: 7d (au lieu de 15d)
   ```

2. Nettoyer manuellement:
   ```bash
   kubectl exec -n monitoring prometheus-0 -- \
     curl -X POST http://localhost:9090/-/reload
   ```

## Solution long terme
- Implémenter Thanos pour stockage longue durée
- Augmenter le PVC
- Optimiser les métriques collectées
```

## Conclusion et évolution

Les runbooks sont un investissement qui paie rapidement. Commencez simple avec quelques alertes critiques, puis enrichissez progressivement votre documentation. Dans votre lab MicroK8s, c'est l'occasion parfaite de développer cette compétence essentielle en DevOps.

Rappelez-vous : un runbook n'est jamais "terminé". Il évolue avec votre système, vos connaissances et les incidents rencontrés. L'important est de commencer à documenter, même imparfaitement, puis d'améliorer itérativement.

### Prochaines étapes recommandées
1. Créez votre premier runbook pour l'alerte la plus fréquente
2. Testez-le en conditions réelles
3. Intégrez le lien dans vos alertes Prometheus
4. Mesurez le temps gagné
5. Partagez et améliorez avec les retours d'expérience

⏭️
