🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6 Automated Testing

## Introduction aux tests automatisés dans Kubernetes

Les tests automatisés sont essentiels pour garantir que vos applications fonctionnent correctement avant, pendant et après leur déploiement sur Kubernetes. Dans un environnement MicroK8s, vous pouvez mettre en place plusieurs niveaux de tests qui s'exécutent automatiquement à chaque modification de votre code ou configuration.

### Pourquoi automatiser les tests ?

Imaginez que vous déployez une nouvelle version de votre application. Sans tests automatisés, vous devriez :
- Vérifier manuellement que l'application démarre correctement
- Tester toutes les fonctionnalités une par une
- Vérifier que les connexions aux bases de données fonctionnent
- S'assurer que les API répondent correctement
- Contrôler les performances et la charge

Avec des tests automatisés, tout cela se fait automatiquement en quelques minutes, vous alertant immédiatement si quelque chose ne va pas.

## Les différents types de tests

### Tests unitaires
Tests du code source de votre application, exécutés avant la création de l'image Docker. Ils vérifient que chaque fonction ou composant fonctionne individuellement.

### Tests d'intégration
Vérifient que les différents composants de votre application fonctionnent ensemble correctement (API + Base de données, microservices entre eux, etc.).

### Tests de conteneur
S'assurent que votre image Docker se construit correctement et que le conteneur démarre sans erreur.

### Tests de déploiement
Valident que vos manifestes Kubernetes sont corrects et que l'application se déploie correctement dans le cluster.

### Tests fonctionnels (End-to-End)
Simulent le comportement d'un utilisateur réel pour vérifier que l'application complète fonctionne comme attendu.

### Tests de performance
Mesurent les temps de réponse, la consommation de ressources et la capacité de charge.

### Tests de sécurité
Recherchent les vulnérabilités dans vos images et configurations.

## Configuration de l'environnement de test

### Namespace dédié aux tests

Créez un namespace isolé pour vos tests :

```bash
# Créer un namespace pour les tests
kubectl create namespace test-automation

# Définir des quotas de ressources pour éviter la surconsommation
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: test-automation
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
EOF
```

### Configuration des accès pour les tests

```yaml
# test-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-runner
  namespace: test-automation
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: test-runner-role
  namespace: test-automation
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-runner-binding
  namespace: test-automation
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: test-runner-role
subjects:
- kind: ServiceAccount
  name: test-runner
  namespace: test-automation
```

## Tests de conteneurs avec structure-test

### Installation de container-structure-test

Container-structure-test est un outil de Google pour tester la structure des images Docker :

```bash
# Télécharger l'outil
wget https://storage.googleapis.com/container-structure-test/latest/container-structure-test-linux-amd64
chmod +x container-structure-test-linux-amd64
sudo mv container-structure-test-linux-amd64 /usr/local/bin/container-structure-test
```

### Configuration des tests

Créez un fichier `structure-test.yaml` :

```yaml
# structure-test.yaml
schemaVersion: "2.0.0"

# Tests de métadonnées
metadataTest:
  env:
    - key: "NODE_ENV"
      value: "production"
  labels:
    - key: "version"
      value: "1.0.0"
    - key: "maintainer"
      isPresent: true

# Tests de fichiers
fileExistenceTests:
  - name: "Application principale présente"
    path: "/app/server.js"
    shouldExist: true
  - name: "Fichier de configuration présent"
    path: "/app/config/settings.json"
    shouldExist: true
  - name: "Pas de fichiers temporaires"
    path: "/tmp/build"
    shouldExist: false

# Tests de contenu de fichiers
fileContentTests:
  - name: "Version correcte dans package.json"
    path: "/app/package.json"
    expectedContents: ["\"version\": \"1.0.0\""]

# Tests de commandes
commandTests:
  - name: "Node.js installé"
    command: "node"
    args: ["--version"]
    expectedOutput: ["v18.*"]
  - name: "Application démarre sans erreur"
    command: "node"
    args: ["/app/server.js", "--dry-run"]
    exitCode: 0
  - name: "Healthcheck endpoint fonctionne"
    command: "curl"
    args: ["-f", "http://localhost:3000/health"]
    exitCode: 0
```

### Exécution des tests

```bash
# Tester une image locale
container-structure-test test --image mon-app:latest --config structure-test.yaml

# Tester une image du registry MicroK8s
container-structure-test test --image localhost:32000/mon-app:v1.0.0 --config structure-test.yaml
```

## Tests de déploiement Kubernetes

### Utilisation de kubectl dry-run

Validez vos manifestes avant le déploiement :

```bash
# Valider la syntaxe YAML
kubectl apply --dry-run=client -f deployment.yaml

# Valider côté serveur (vérifie les permissions, quotas, etc.)
kubectl apply --dry-run=server -f deployment.yaml

# Générer un manifest depuis une commande
kubectl create deployment test-app --image=nginx --dry-run=client -o yaml > test-deployment.yaml
```

### Tests avec Kustomize

Si vous utilisez Kustomize pour gérer vos configurations :

```bash
# Structure de test avec Kustomize
mkdir -p tests/kustomize/{base,overlays/test}

# base/kustomization.yaml
cat > tests/kustomize/base/kustomization.yaml <<EOF
resources:
  - deployment.yaml
  - service.yaml
EOF

# overlays/test/kustomization.yaml
cat > tests/kustomize/overlays/test/kustomization.yaml <<EOF
bases:
  - ../../base
replicas:
  - name: mon-app
    count: 1
patches:
  - patch-test-env.yaml
EOF

# Valider la configuration générée
kubectl kustomize tests/kustomize/overlays/test --enable-helm | kubectl apply --dry-run=server -f -
```

## Tests Helm avec helm test

### Création de tests Helm

Dans votre Chart Helm, créez des tests dans `templates/tests/` :

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mon-app.fullname" . }}-test-connection"
  labels:
    {{- include "mon-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:
        - '--timeout=5'
        - '-O-'
        - 'http://{{ include "mon-app.fullname" . }}:{{ .Values.service.port }}/health'
  restartPolicy: Never
```

```yaml
# templates/tests/test-database.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mon-app.fullname" . }}-test-db"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test-db-connection
      image: postgres:13-alpine
      env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .Values.database.secretName }}
              key: password
      command: ["psql"]
      args:
        - "-h"
        - "{{ .Values.database.host }}"
        - "-U"
        - "{{ .Values.database.user }}"
        - "-d"
        - "{{ .Values.database.name }}"
        - "-c"
        - "SELECT 1"
  restartPolicy: Never
```

### Exécution des tests Helm

```bash
# Installer l'application
helm install mon-app ./mon-chart

# Exécuter les tests
helm test mon-app

# Voir les logs des tests
kubectl logs mon-app-test-connection -n default
kubectl logs mon-app-test-db -n default

# Nettoyer les pods de test
helm test mon-app --cleanup
```

## Tests fonctionnels avec Jobs Kubernetes

### Création d'un Job de test

```yaml
# test-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-fonctionnel
  namespace: test-automation
spec:
  template:
    spec:
      serviceAccountName: test-runner
      containers:
      - name: test-runner
        image: localhost:32000/test-runner:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            echo "Démarrage des tests fonctionnels..."

            # Test 1: Vérifier que l'API répond
            response=$(curl -s -o /dev/null -w "%{http_code}" http://mon-app-service:8080/api/status)
            if [ $response -eq 200 ]; then
              echo "✓ API status: OK"
            else
              echo "✗ API status: FAILED (HTTP $response)"
              exit 1
            fi

            # Test 2: Créer une ressource
            curl -X POST http://mon-app-service:8080/api/items \
              -H "Content-Type: application/json" \
              -d '{"name":"test-item","value":123}' \
              -f || exit 1
            echo "✓ Création de ressource: OK"

            # Test 3: Lire la ressource
            curl -f http://mon-app-service:8080/api/items/test-item || exit 1
            echo "✓ Lecture de ressource: OK"

            # Test 4: Supprimer la ressource
            curl -X DELETE -f http://mon-app-service:8080/api/items/test-item || exit 1
            echo "✓ Suppression de ressource: OK"

            echo "Tous les tests sont passés avec succès!"
      restartPolicy: Never
  backoffLimit: 1
```

### Exécution et monitoring du Job

```bash
# Lancer le job de test
kubectl apply -f test-job.yaml

# Suivre l'exécution
kubectl get job test-fonctionnel -n test-automation -w

# Voir les logs
kubectl logs -f job/test-fonctionnel -n test-automation

# Vérifier le statut
kubectl describe job test-fonctionnel -n test-automation
```

## Tests de charge avec K6

### Préparation de l'image K6

```dockerfile
# Dockerfile.k6
FROM grafana/k6:latest
COPY test-scripts/ /scripts/
```

### Script de test de charge

```javascript
// test-scripts/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Métriques personnalisées
const errorRate = new Rate('errors');

// Options de test
export const options = {
  stages: [
    { duration: '30s', target: 10 },  // Montée progressive à 10 utilisateurs
    { duration: '1m', target: 10 },   // Maintien à 10 utilisateurs
    { duration: '30s', target: 50 },  // Montée à 50 utilisateurs
    { duration: '2m', target: 50 },   // Maintien à 50 utilisateurs
    { duration: '30s', target: 0 },   // Descente à 0
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'], // 95% des requêtes < 500ms
    'errors': ['rate<0.1'],              // Taux d'erreur < 10%
  },
};

export default function () {
  // Test de la page d'accueil
  const homeRes = http.get('http://mon-app-service.default.svc.cluster.local:8080/');
  check(homeRes, {
    'status is 200': (r) => r.status === 200,
    'page loaded': (r) => r.body.includes('Welcome'),
  });
  errorRate.add(homeRes.status !== 200);

  sleep(1);

  // Test de l'API
  const apiRes = http.get('http://mon-app-service.default.svc.cluster.local:8080/api/products');
  check(apiRes, {
    'API responded': (r) => r.status === 200,
    'has products': (r) => JSON.parse(r.body).length > 0,
  });
  errorRate.add(apiRes.status !== 200);

  sleep(2);
}

// Rapport de fin
export function handleSummary(data) {
  return {
    'stdout': textSummary(data, { indent: ' ', enableColors: true }),
    '/scripts/summary.json': JSON.stringify(data),
  };
}
```

### Déploiement du test K6 comme Job

```yaml
# k6-load-test-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: k6-load-test
  namespace: test-automation
spec:
  template:
    spec:
      containers:
      - name: k6
        image: localhost:32000/k6-tests:latest
        command: ["k6", "run", "/scripts/load-test.js"]
        env:
        - name: K6_OUT
          value: "json=/scripts/results.json"
        volumeMounts:
        - name: test-results
          mountPath: /scripts/results
      volumes:
      - name: test-results
        persistentVolumeClaim:
          claimName: test-results-pvc
      restartPolicy: Never
```

## Tests de sécurité

### Scan de vulnérabilités avec Trivy

```bash
# Installer Trivy
wget https://github.com/aquasecurity/trivy/releases/download/v0.45.0/trivy_0.45.0_Linux-64bit.tar.gz
tar zxvf trivy_0.45.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# Scanner une image
trivy image localhost:32000/mon-app:latest

# Scanner avec un seuil de sévérité
trivy image --severity HIGH,CRITICAL localhost:32000/mon-app:latest

# Générer un rapport JSON
trivy image -f json -o scan-results.json localhost:32000/mon-app:latest
```

### Test de configuration avec Polaris

```bash
# Installer Polaris
kubectl apply -f https://github.com/FairwindsOps/polaris/releases/latest/download/dashboard.yaml

# Exposer le dashboard
kubectl port-forward --namespace polaris svc/polaris-dashboard 8080:80

# Scanner un namespace
curl -X POST http://localhost:8080/api/v1/scan \
  -H "Content-Type: application/json" \
  -d '{"namespace": "default"}'
```

### Validation des politiques avec OPA (Open Policy Agent)

```yaml
# opa-test-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policies
  namespace: test-automation
data:
  deployment-policy.rego: |
    package kubernetes.admission

    deny[msg] {
      input.kind == "Deployment"
      not input.spec.template.spec.containers[_].resources.limits.memory
      msg := "Les conteneurs doivent avoir des limites de mémoire"
    }

    deny[msg] {
      input.kind == "Deployment"
      input.spec.replicas < 2
      msg := "Les déploiements de production doivent avoir au moins 2 replicas"
    }

    deny[msg] {
      input.kind == "Deployment"
      not input.spec.template.spec.containers[_].livenessProbe
      msg := "Les conteneurs doivent avoir un livenessProbe"
    }
```

## Pipeline de tests automatisés

### Structure d'un pipeline complet

```yaml
# .gitlab-ci.yml ou Jenkinsfile converti en YAML pour la compréhension
stages:
  - build
  - test-unit
  - test-container
  - deploy-test
  - test-integration
  - test-performance
  - test-security
  - cleanup

variables:
  REGISTRY: "localhost:32000"
  APP_NAME: "mon-app"
  TEST_NAMESPACE: "test-automation"

# 1. Construction de l'image
build:
  stage: build
  script:
    - docker build -t $REGISTRY/$APP_NAME:$CI_COMMIT_SHA .
    - docker push $REGISTRY/$APP_NAME:$CI_COMMIT_SHA

# 2. Tests unitaires
test-unit:
  stage: test-unit
  script:
    - npm install
    - npm test
  artifacts:
    reports:
      junit: test-results.xml

# 3. Tests de structure du conteneur
test-container:
  stage: test-container
  script:
    - container-structure-test test --image $REGISTRY/$APP_NAME:$CI_COMMIT_SHA --config structure-test.yaml

# 4. Déploiement en environnement de test
deploy-test:
  stage: deploy-test
  script:
    - kubectl create namespace $TEST_NAMESPACE || true
    - helm upgrade --install $APP_NAME-test ./chart \
        --namespace $TEST_NAMESPACE \
        --set image.tag=$CI_COMMIT_SHA \
        --wait --timeout 5m

# 5. Tests d'intégration
test-integration:
  stage: test-integration
  script:
    - helm test $APP_NAME-test -n $TEST_NAMESPACE
    - kubectl apply -f test-job.yaml -n $TEST_NAMESPACE
    - kubectl wait --for=condition=complete job/test-fonctionnel -n $TEST_NAMESPACE --timeout=5m

# 6. Tests de performance
test-performance:
  stage: test-performance
  script:
    - kubectl apply -f k6-load-test-job.yaml -n $TEST_NAMESPACE
    - kubectl wait --for=condition=complete job/k6-load-test -n $TEST_NAMESPACE --timeout=10m
    - kubectl logs job/k6-load-test -n $TEST_NAMESPACE

# 7. Tests de sécurité
test-security:
  stage: test-security
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 $REGISTRY/$APP_NAME:$CI_COMMIT_SHA
    - kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/main/deploy/static/trivy-operator.yaml
  allow_failure: true

# 8. Nettoyage
cleanup:
  stage: cleanup
  when: always
  script:
    - helm uninstall $APP_NAME-test -n $TEST_NAMESPACE || true
    - kubectl delete namespace $TEST_NAMESPACE || true
```

## Monitoring des tests

### Dashboard de tests avec Grafana

```yaml
# configmap-grafana-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-dashboard
  namespace: monitoring
data:
  test-dashboard.json: |
    {
      "dashboard": {
        "title": "Tests Automatisés",
        "panels": [
          {
            "title": "Taux de succès des tests",
            "targets": [
              {
                "expr": "sum(rate(test_success_total[5m])) / sum(rate(test_total[5m]))"
              }
            ]
          },
          {
            "title": "Durée moyenne des tests",
            "targets": [
              {
                "expr": "avg(test_duration_seconds)"
              }
            ]
          },
          {
            "title": "Tests par type",
            "targets": [
              {
                "expr": "sum by (test_type) (test_total)"
              }
            ]
          }
        ]
      }
    }
```

### Alertes sur les échecs de tests

```yaml
# prometheus-alert-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-alert-rules
  namespace: monitoring
data:
  test-alerts.yaml: |
    groups:
    - name: test-alerts
      rules:
      - alert: HighTestFailureRate
        expr: |
          (sum(rate(test_failure_total[15m])) / sum(rate(test_total[15m]))) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Taux d'échec des tests élevé"
          description: "Plus de 10% des tests échouent depuis 5 minutes"

      - alert: TestJobFailed
        expr: |
          kube_job_status_failed{job_name=~"test-.*"} > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Job de test échoué"
          description: "Le job {{ $labels.job_name }} a échoué"
```

## Intégration avec les outils CI/CD

### GitLab CI

```yaml
# .gitlab-ci.yml
include:
  - template: Kubernetes.gitlab-ci.yml

test:kubernetes:
  stage: test
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context $KUBE_CONTEXT
    - kubectl apply -f tests/ -n test-automation
    - kubectl wait --for=condition=complete job/all-tests -n test-automation --timeout=15m
  environment:
    name: test
    kubernetes:
      namespace: test-automation
```

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests Automatisés
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Configuration kubectl
      uses: azure/k8s-set-context@v3
      with:
        kubeconfig: ${{ secrets.KUBE_CONFIG }}

    - name: Tests de déploiement
      run: |
        kubectl apply -f manifests/ --dry-run=server

    - name: Tests Helm
      run: |
        helm lint ./chart
        helm template ./chart | kubectl apply --dry-run=server -f -

    - name: Scan de sécurité
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.IMAGE }}
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
```

## Bonnes pratiques

### 1. Organisation des tests

Structurez vos tests de manière logique :
```
tests/
├── unit/               # Tests unitaires
├── integration/        # Tests d'intégration
├── e2e/               # Tests end-to-end
├── performance/       # Tests de charge
├── security/          # Tests de sécurité
├── fixtures/          # Données de test
└── scripts/           # Scripts utilitaires
```

### 2. Isolation des tests

- Utilisez toujours des namespaces dédiés pour les tests
- Nettoyez les ressources après chaque test
- Évitez les dépendances entre tests
- Utilisez des données de test uniques (UUID, timestamps)

### 3. Tests idempotents

Les tests doivent pouvoir être exécutés plusieurs fois avec le même résultat :
```bash
# Mauvais : modifie l'état global
kubectl scale deployment prod-app --replicas=5

# Bon : crée ses propres ressources
kubectl apply -f test-deployment.yaml -n test-$RANDOM
```

### 4. Timeouts appropriés

Définissez toujours des timeouts pour éviter les tests qui restent bloqués :
```yaml
spec:
  activeDeadlineSeconds: 300  # 5 minutes maximum
  backoffLimit: 3             # 3 tentatives maximum
```

### 5. Logging et debugging

```bash
# Activer le mode verbose pour le debugging
kubectl apply -f test.yaml -v=8

# Capturer tous les logs
kubectl logs -n test-automation -l test=true --all-containers=true --tail=-1
```

### 6. Tests en parallèle

Pour accélérer l'exécution, lancez les tests indépendants en parallèle :
```bash
# Lancer plusieurs jobs en parallèle
for test in test-*.yaml; do
  kubectl apply -f $test &
done
wait

# Vérifier tous les résultats
kubectl get jobs -n test-automation
```

## Dépannage courant

### Les tests échouent de manière aléatoire

```bash
# Augmenter les timeouts
kubectl patch job test-job -p '{"spec":{"activeDeadlineSeconds":600}}'

# Vérifier les ressources disponibles
kubectl top nodes
kubectl describe resourcequota -n test-automation

# Analyser les événements
kubectl get events -n test-automation --sort-by='.lastTimestamp'
```

### Les tests ne se lancent pas

```bash
# Vérifier les permissions
kubectl auth can-i create pods -n test-automation --as=system:serviceaccount:test-automation:test-runner

# Vérifier les images
kubectl describe pod test-pod -n test-automation | grep -A5 "Events:"

# Forcer le re-pull des images
kubectl set image job/test-job test-container=localhost:32000/test:latest --record
```

### Nettoyage des ressources de test

```bash
# Script de nettoyage complet
#!/bin/bash
NAMESPACE="test-automation"

# Supprimer tous les jobs terminés
kubectl delete jobs -n $NAMESPACE --field-selector status.successful=1

# Supprimer les pods en erreur
kubectl delete pods -n $NAMESPACE --field-selector status.phase=Failed

# Nettoyer les ressources de test
kubectl delete all -l environment=test -n $NAMESPACE
```

## Conclusion

Les tests automatisés dans MicroK8s vous permettent de :
- Détecter les problèmes avant qu'ils n'atteignent la production
- Gagner du temps en évitant les tests manuels répétitifs
- Améliorer la qualité et la fiabilité de vos applications
- Documenter le comportement attendu de votre système
- Faciliter les mises à jour et les refactoring

Commencez par des tests simples (validation de manifestes, tests de connexion) puis progressez vers des tests plus complexes (charge, sécurité, chaos engineering) au fur et à mesure que votre infrastructure grandit. L'important est d'avoir au minimum des tests de base qui s'exécutent automatiquement à chaque changement.

⏭️
