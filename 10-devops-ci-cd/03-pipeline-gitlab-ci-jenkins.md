🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.3 Pipeline GitLab CI/Jenkins

## Introduction : L'automatisation complète de votre workflow

Un pipeline CI/CD est comme une chaîne de montage automatisée pour votre code. Depuis le moment où vous effectuez un commit Git jusqu'au déploiement en production sur votre cluster MicroK8s, chaque étape est orchestrée automatiquement. GitLab CI et Jenkins sont deux des solutions les plus populaires pour créer ces pipelines, chacune avec ses forces et sa philosophie.

## Comprendre les pipelines CI/CD

### Qu'est-ce qu'un pipeline ?

Un pipeline est une séquence d'étapes automatisées qui transforment votre code source en application déployée. Imaginez une usine où :

1. **Le code arrive** (git push)
2. **Il est vérifié** (tests unitaires, linting)
3. **Il est assemblé** (compilation, build)
4. **Il est emballé** (création d'image Docker)
5. **Il est stocké** (push vers registry)
6. **Il est livré** (déploiement sur Kubernetes)

Chaque étape dépend du succès de la précédente. Si les tests échouent, le pipeline s'arrête, protégeant votre environnement de production.

### GitLab CI vs Jenkins : Comprendre les différences

**GitLab CI** est intégré nativement à GitLab, offrant une expérience unifiée où code, CI/CD, et gestion de projet coexistent. Il utilise des fichiers YAML pour définir les pipelines, rendant la configuration versionnée avec votre code.

**Jenkins** est le vétéran de l'automatisation, extrêmement flexible et extensible via des milliers de plugins. Il offre une interface graphique pour construire des pipelines et supporte multiple syntaxes (Declarative, Scripted Pipeline).

### Vocabulaire essentiel

**Job/Task** : Une unité de travail atomique (ex: exécuter les tests)
**Stage** : Un groupe de jobs qui s'exécutent en parallèle (ex: tous les tests)
**Pipeline** : L'ensemble complet des stages dans l'ordre
**Runner/Agent** : La machine qui exécute réellement les jobs
**Artifact** : Fichier produit par un job et conservé (ex: binaire compilé)
**Trigger** : Événement qui déclenche le pipeline (push, merge request, schedule)

## GitLab CI : Configuration complète

### Architecture GitLab CI

GitLab CI fonctionne avec trois composants principaux :

1. **GitLab Server** : Orchestrateur central qui gère les pipelines
2. **GitLab Runner** : Exécuteur qui run les jobs sur différentes machines
3. **Registry** : Stockage des images Docker et artifacts

### Installation de GitLab dans votre lab

#### Option 1 : GitLab avec Docker Compose

```yaml
# docker-compose.yml pour GitLab
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab.monlab.local
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.monlab.local'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        # Configuration initiale
        gitlab_rails['initial_root_password'] = 'ChangeMeNow123!'
        # Registry intégré
        registry_external_url 'http://registry.gitlab.monlab.local:5005'
        gitlab_rails['registry_enabled'] = true
        # Prometheus metrics
        prometheus_monitoring['enable'] = true
    ports:
      - '80:80'
      - '443:443'
      - '2222:22'
      - '5005:5005'
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    networks:
      - gitlab-network

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab_runner_config:/etc/gitlab-runner
    networks:
      - gitlab-network

volumes:
  gitlab_config:
  gitlab_logs:
  gitlab_data:
  gitlab_runner_config:

networks:
  gitlab-network:
    driver: bridge
```

#### Option 2 : GitLab sur Kubernetes

```yaml
# gitlab-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab

---
# gitlab-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
  namespace: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
      - name: gitlab
        image: gitlab/gitlab-ce:latest
        ports:
        - containerPort: 80
        - containerPort: 22
        - containerPort: 443
        env:
        - name: GITLAB_OMNIBUS_CONFIG
          value: |
            external_url 'http://gitlab.monlab.local'
            gitlab_rails['initial_root_password'] = 'ChangeMeNow123!'
        volumeMounts:
        - name: gitlab-storage
          mountPath: /var/opt/gitlab
        - name: gitlab-config
          mountPath: /etc/gitlab
        - name: gitlab-logs
          mountPath: /var/log/gitlab
      volumes:
      - name: gitlab-storage
        persistentVolumeClaim:
          claimName: gitlab-data-pvc
      - name: gitlab-config
        persistentVolumeClaim:
          claimName: gitlab-config-pvc
      - name: gitlab-logs
        emptyDir: {}

---
# Services et PVC additionnels...
```

### Configuration du Runner GitLab

Le Runner est l'agent qui exécute vos jobs. Pour votre lab MicroK8s :

```bash
# Enregistrer un runner Docker
docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://gitlab.monlab.local/" \
  --registration-token "VOTRE_TOKEN_REGISTRATION" \
  --executor "docker" \
  --docker-image alpine:latest \
  --description "docker-runner" \
  --tag-list "docker,local" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"

# Configuration avancée du runner
cat > /etc/gitlab-runner/config.toml <<EOF
concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner"
  url = "http://gitlab.monlab.local/"
  token = "TOKEN_DU_RUNNER"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
EOF
```

### Structure d'un pipeline GitLab CI

Le pipeline est défini dans `.gitlab-ci.yml` à la racine de votre projet :

```yaml
# .gitlab-ci.yml - Pipeline complet pour une application
# Variables globales
variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  REGISTRY: registry.gitlab.monlab.local:5005
  IMAGE_NAME: $REGISTRY/$CI_PROJECT_PATH
  KUBECONFIG: /tmp/kubeconfig

# Stages du pipeline
stages:
  - build
  - test
  - package
  - deploy
  - cleanup

# Templates réutilisables
.docker_template:
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# Job: Compilation
build:app:
  stage: build
  image: node:18-alpine
  script:
    - echo "Building application..."
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/

# Job: Tests unitaires
test:unit:
  stage: test
  image: node:18-alpine
  needs: ["build:app"]
  script:
    - echo "Running unit tests..."
    - npm run test:unit
  coverage: '/Coverage: \d+\.\d+%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# Job: Tests d'intégration
test:integration:
  stage: test
  image: node:18-alpine
  needs: ["build:app"]
  services:
    - postgres:14-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testdb
  script:
    - echo "Running integration tests..."
    - npm run test:integration
  allow_failure: false

# Job: Analyse de sécurité
test:security:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy fs --no-progress --security-checks vuln,config .
  allow_failure: true

# Job: Analyse de code
test:sonar:
  stage: test
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_HOST_URL: "http://sonarqube.monlab.local"
  script:
    - sonar-scanner
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.sources=.
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
  only:
    - merge_requests
    - main

# Job: Construction de l'image Docker
package:docker:
  extends: .docker_template
  stage: package
  needs: ["build:app", "test:unit"]
  script:
    - echo "Building Docker image..."
    - docker build
      --cache-from $IMAGE_NAME:latest
      --tag $IMAGE_NAME:$CI_COMMIT_SHA
      --tag $IMAGE_NAME:latest .
    - echo "Pushing image to registry..."
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
    - docker push $IMAGE_NAME:latest
    # Scan de vulnérabilités sur l'image
    - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock
      aquasec/trivy image $IMAGE_NAME:$CI_COMMIT_SHA
  only:
    - main
    - develop
    - tags

# Job: Déploiement en développement
deploy:dev:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: ["package:docker"]
  environment:
    name: development
    url: http://dev.monlab.local
    on_stop: cleanup:dev
  before_script:
    - echo "$KUBE_CONFIG_DEV" | base64 -d > $KUBECONFIG
  script:
    - echo "Deploying to development environment..."
    - kubectl set image deployment/myapp
      myapp=$IMAGE_NAME:$CI_COMMIT_SHA
      -n development
    - kubectl rollout status deployment/myapp -n development
    - kubectl get pods -n development
  only:
    - develop

# Job: Déploiement en staging
deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: ["package:docker"]
  environment:
    name: staging
    url: http://staging.monlab.local
    on_stop: cleanup:staging
  before_script:
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > $KUBECONFIG
  script:
    - echo "Deploying to staging environment..."
    - |
      cat <<EOF | kubectl apply -f -
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
        namespace: staging
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: myapp
        template:
          metadata:
            labels:
              app: myapp
              version: $CI_COMMIT_SHA
          spec:
            containers:
            - name: myapp
              image: $IMAGE_NAME:$CI_COMMIT_SHA
              ports:
              - containerPort: 3000
              env:
              - name: ENVIRONMENT
                value: staging
              resources:
                requests:
                  memory: "128Mi"
                  cpu: "100m"
                limits:
                  memory: "256Mi"
                  cpu: "200m"
      EOF
    - kubectl rollout status deployment/myapp -n staging
  only:
    - main
  when: manual

# Job: Déploiement en production
deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: ["deploy:staging"]
  environment:
    name: production
    url: http://monlab.local
  before_script:
    - echo "$KUBE_CONFIG_PROD" | base64 -d > $KUBECONFIG
  script:
    - echo "🚀 Deploying to production..."
    - kubectl set image deployment/myapp
      myapp=$IMAGE_NAME:$CI_COMMIT_SHA
      -n production --record
    - kubectl rollout status deployment/myapp -n production
    - kubectl get pods -n production
    - echo "✅ Production deployment successful!"
  only:
    - tags
  when: manual
  allow_failure: false

# Job: Nettoyage environnement dev
cleanup:dev:
  stage: cleanup
  image: bitnami/kubectl:latest
  environment:
    name: development
    action: stop
  before_script:
    - echo "$KUBE_CONFIG_DEV" | base64 -d > $KUBECONFIG
  script:
    - kubectl delete deployment myapp -n development --ignore-not-found
  when: manual
  only:
    - develop

# Job: Nettoyage des anciennes images
cleanup:registry:
  stage: cleanup
  image: alpine:latest
  script:
    - apk add --no-cache curl jq
    - |
      # Script de nettoyage des images de plus de 7 jours
      IMAGES=$(curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        "$CI_API_V4_URL/projects/$CI_PROJECT_ID/registry/repositories")
      for repo in $(echo $IMAGES | jq -r '.[].id'); do
        TAGS=$(curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
          "$CI_API_V4_URL/projects/$CI_PROJECT_ID/registry/repositories/$repo/tags")
        for tag in $(echo $TAGS | jq -r '.[] | select(.created_at < (now - 604800)) | .name'); do
          echo "Deleting old tag: $tag"
          curl -s -X DELETE -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
            "$CI_API_V4_URL/projects/$CI_PROJECT_ID/registry/repositories/$repo/tags/$tag"
        done
      done
  only:
    - schedules
```

### Variables et secrets dans GitLab CI

GitLab offre plusieurs niveaux pour gérer les variables :

```yaml
# Variables dans .gitlab-ci.yml (publiques)
variables:
  NODE_ENV: "production"
  API_URL: "https://api.monlab.local"

# Variables spécifiques à un job
test:unit:
  variables:
    TEST_TIMEOUT: "30000"
  script:
    - npm test

# Variables conditionnelles
deploy:prod:
  script:
    - |
      if [ "$CI_COMMIT_TAG" != "" ]; then
        export VERSION=$CI_COMMIT_TAG
      else
        export VERSION=$CI_COMMIT_SHORT_SHA
      fi
    - echo "Deploying version $VERSION"
```

Configuration des variables secrètes dans l'interface GitLab :
1. Projet → Settings → CI/CD → Variables
2. Ajouter les secrets (masqués et protégés)
3. Utiliser dans le pipeline via `$VARIABLE_NAME`

## Jenkins : Configuration complète

### Architecture Jenkins

Jenkins utilise une architecture maître-esclave :

1. **Jenkins Master** : Serveur principal qui orchestre
2. **Jenkins Agents** : Nœuds qui exécutent les jobs
3. **Plugins** : Extensions pour ajouter des fonctionnalités

### Installation de Jenkins sur MicroK8s

```yaml
# jenkins-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        env:
        - name: JAVA_OPTS
          value: "-Djenkins.install.runSetupWizard=false"
        - name: JENKINS_OPTS
          value: "--prefix=/jenkins"
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        - name: docker-sock
          mountPath: /var/run/docker.sock
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
          type: Socket

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
    name: http
  - port: 50000
    targetPort: 50000
    nodePort: 30050
    name: agent
  selector:
    app: jenkins
```

### Configuration initiale avec Configuration as Code

```yaml
# jenkins-casc.yaml - Jenkins Configuration as Code
jenkins:
  systemMessage: "Jenkins CI/CD pour MicroK8s Lab"
  numExecutors: 2
  mode: EXCLUSIVE

  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false

  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "admin123"

  clouds:
    - kubernetes:
        name: "kubernetes"
        serverUrl: "https://kubernetes.default"
        namespace: "jenkins"
        jenkinsUrl: "http://jenkins:8080"
        jenkinsTunnel: "jenkins:50000"
        containerCapStr: "10"
        connectTimeout: 5
        readTimeout: 15
        maxRequestsPerHostStr: "32"
        templates:
          - name: "jenkins-agent"
            label: "jenkins-agent"
            nodeUsageMode: NORMAL
            containers:
              - name: "jnlp"
                image: "jenkins/inbound-agent:latest"
                command: ""
                args: "^${computer.jnlpmac} ^${computer.name}"
                workingDir: "/home/jenkins/agent"
              - name: "docker"
                image: "docker:latest"
                command: "cat"
                ttyEnabled: true
                privileged: true
            volumes:
              - hostPathVolume:
                  hostPath: "/var/run/docker.sock"
                  mountPath: "/var/run/docker.sock"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "github-credentials"
              username: "${GITHUB_USER}"
              password: "${GITHUB_TOKEN}"
              description: "GitHub credentials"
          - string:
              scope: GLOBAL
              id: "kubeconfig"
              secret: "${KUBECONFIG_BASE64}"
              description: "Kubeconfig for MicroK8s"

unclassified:
  location:
    url: "http://jenkins.monlab.local:30080"
  gitHubPluginConfig:
    hookUrl: "http://jenkins.monlab.local:30080/github-webhook/"
```

### Pipeline Jenkins : Syntaxe Declarative

```groovy
// Jenkinsfile - Pipeline Declarative
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  containers:
  - name: docker
    image: docker:latest
    command: ['cat']
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
  - name: node
    image: node:18-alpine
    command: ['cat']
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        REGISTRY = 'registry.monlab.local:5000'
        IMAGE_NAME = "${REGISTRY}/myapp"
        DOCKER_CREDENTIALS = credentials('docker-registry')
        KUBECONFIG = credentials('kubeconfig')
    }

    options {
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()
        cron('H 2 * * 1-5')  // Nightly builds en semaine
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.GIT_BRANCH = sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                container('node') {
                    sh '''
                        echo "Building application..."
                        npm ci
                        npm run build
                    '''
                    stash includes: 'dist/**', name: 'built-app'
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('node') {
                            sh '''
                                echo "Running unit tests..."
                                npm run test:unit
                            '''
                            junit 'test-results/**/*.xml'
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        container('node') {
                            sh '''
                                echo "Running integration tests..."
                                npm run test:integration
                            '''
                        }
                    }
                }

                stage('Code Quality') {
                    steps {
                        container('node') {
                            sh '''
                                echo "Running linting..."
                                npm run lint

                                echo "Running security audit..."
                                npm audit --audit-level=moderate
                            '''
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
                }
            }
            steps {
                container('docker') {
                    unstash 'built-app'
                    sh '''
                        echo "Building Docker image..."
                        docker build -t ${IMAGE_NAME}:${GIT_COMMIT} .
                        docker tag ${IMAGE_NAME}:${GIT_COMMIT} ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Push to Registry') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('docker') {
                    sh '''
                        echo "Logging into Docker registry..."
                        echo ${DOCKER_CREDENTIALS_PSW} | docker login ${REGISTRY} -u ${DOCKER_CREDENTIALS_USR} --password-stdin

                        echo "Pushing images..."
                        docker push ${IMAGE_NAME}:${GIT_COMMIT}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Development') {
            when {
                branch 'develop'
            }
            steps {
                container('kubectl') {
                    sh '''
                        echo "Deploying to development..."
                        echo ${KUBECONFIG} | base64 -d > /tmp/kubeconfig
                        export KUBECONFIG=/tmp/kubeconfig

                        kubectl set image deployment/myapp myapp=${IMAGE_NAME}:${GIT_COMMIT} -n development
                        kubectl rollout status deployment/myapp -n development
                    '''
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to staging?"
                ok "Deploy"
                parameters {
                    choice(name: 'ENVIRONMENT', choices: ['staging', 'skip'], description: 'Target environment')
                }
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'staging') {
                        container('kubectl') {
                            sh '''
                                echo "Deploying to staging..."
                                echo ${KUBECONFIG} | base64 -d > /tmp/kubeconfig
                                export KUBECONFIG=/tmp/kubeconfig

                                kubectl set image deployment/myapp myapp=${IMAGE_NAME}:${GIT_COMMIT} -n staging
                                kubectl rollout status deployment/myapp -n staging
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy to Production') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                submitter "admin,devops"
            }
            steps {
                container('kubectl') {
                    sh '''
                        echo "🚀 Deploying to production..."
                        echo ${KUBECONFIG} | base64 -d > /tmp/kubeconfig
                        export KUBECONFIG=/tmp/kubeconfig

                        # Blue-Green deployment
                        kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp-green
  namespace: production
spec:
  selector:
    app: myapp
    version: green
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: ${IMAGE_NAME}:${GIT_COMMIT}
        ports:
        - containerPort: 3000
EOF

                        kubectl rollout status deployment/myapp-green -n production

                        # Switch traffic
                        kubectl patch service myapp -n production -p '{"spec":{"selector":{"version":"green"}}}'

                        echo "✅ Production deployment complete!"
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
            slackSend(
                color: 'good',
                message: "Build Successful: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        failure {
            echo 'Pipeline failed!'
            slackSend(
                color: 'danger',
                message: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        unstable {
            echo 'Pipeline unstable'
        }
    }
}
```

### Pipeline Jenkins : Syntaxe Scripted

```groovy
// Jenkinsfile - Pipeline Scripted (plus flexible)
node {
    def dockerImage
    def gitCommit

    try {
        stage('Checkout') {
            checkout scm
            gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }

        stage('Build') {
            docker.image('node:18-alpine').inside {
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            parallel(
                'Unit Tests': {
                    docker.image('node:18-alpine').inside {
                        sh 'npm run test:unit'
                    }
                },
                'Integration Tests': {
                    docker.image('node:18-alpine').inside {
                        sh 'npm run test:integration'
                    }
                }
            )
        }

        stage('Build Docker Image') {
            dockerImage = docker.build("${REGISTRY}/myapp:${gitCommit}")
        }

        stage('Push to Registry') {
            docker.withRegistry("http://${REGISTRY}", 'docker-credentials') {
                dockerImage.push()
                dockerImage.push('latest')
            }
        }

        stage('Deploy') {
            def environment = env.BRANCH_NAME

            if (environment == 'develop') {
                deployToKubernetes('development', gitCommit)
            } else if (environment == 'main') {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Deploy to staging?', ok: 'Deploy'
                }
                deployToKubernetes('staging', gitCommit)
            } else if (env.TAG_NAME) {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Deploy to production?',
                          ok: 'Deploy',
                          submitter: 'admin,devops'
                }
                deployToKubernetes('production', gitCommit)
            }
        }

        currentBuild.result = 'SUCCESS'

    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // Notifications
        notifyBuild(currentBuild.result)

        // Cleanup
        cleanWs()
    }
}

def deployToKubernetes(environment, version) {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh """
            echo "Deploying version ${version} to ${environment}..."
            kubectl set image deployment/myapp myapp=${REGISTRY}/myapp:${version} -n ${environment}
            kubectl rollout status deployment/myapp -n ${environment}
        """
    }
}

def notifyBuild(String buildStatus = 'STARTED') {
    buildStatus = buildStatus ?: 'SUCCESS'

    def colorCode = '#FF0000'
    def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
    def summary = "${subject} (${env.BUILD_URL})"

    if (buildStatus == 'STARTED') {
        colorCode = '#FFFF00'
    } else if (buildStatus == 'SUCCESS') {
        colorCode = '#00FF00'
    } else {
        colorCode = '#FF0000'
    }

    // Slack notification
    if (buildStatus != 'STARTED') {
        slackSend(
            color: colorCode,
            message: summary,
            channel: '#ci-cd'
        )
    }

    // Email notification
    emailext(
        subject: subject,
        body: summary,
        to: '${DEFAULT_RECIPIENTS}'
    )
}
```

## Shared Libraries Jenkins

Les Shared Libraries permettent de réutiliser du code entre pipelines :

### Structure d'une Shared Library

```
jenkins-shared-library/
├── vars/                     # Variables globales
│   ├── buildDocker.groovy   # Fonction réutilisable
│   ├── deployK8s.groovy
│   └── runTests.groovy
├── src/                      # Classes Groovy
│   └── com/
│       └── monlab/
│           ├── Docker.groovy
│           └── Kubernetes.groovy
└── resources/               # Fichiers de ressources
    └── templates/
        └── deployment.yaml
```

### Exemple de fonction partagée

```groovy
// vars/buildDocker.groovy
def call(Map config = [:]) {
    def registry = config.registry ?: 'localhost:5000'
    def imageName = config.imageName ?: 'myapp'
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def context = config.context ?: '.'
    def platforms = config.platforms ?: ['linux/amd64']

    // Construction multi-plateforme avec buildx
    sh """
        docker buildx create --use --name multibuilder || true
        docker buildx build \
            --platform ${platforms.join(',')} \
            --tag ${registry}/${imageName}:${env.GIT_COMMIT} \
            --tag ${registry}/${imageName}:latest \
            --file ${dockerfile} \
            --push \
            ${context}
    """

    return "${registry}/${imageName}:${env.GIT_COMMIT}"
}
```

```groovy
// vars/deployK8s.groovy
def call(Map config = [:]) {
    def namespace = config.namespace
    def deployment = config.deployment
    def image = config.image
    def replicas = config.replicas ?: 1
    def strategy = config.strategy ?: 'RollingUpdate'

    // Template Kubernetes
    def manifest = libraryResource('templates/deployment.yaml')
    manifest = manifest.replaceAll('{{NAMESPACE}}', namespace)
                      .replaceAll('{{DEPLOYMENT}}', deployment)
                      .replaceAll('{{IMAGE}}', image)
                      .replaceAll('{{REPLICAS}}', replicas.toString())
                      .replaceAll('{{STRATEGY}}', strategy)

    writeFile file: 'deployment.yaml', text: manifest

    sh """
        kubectl apply -f deployment.yaml
        kubectl rollout status deployment/${deployment} -n ${namespace}
        kubectl get pods -n ${namespace} -l app=${deployment}
    """
}
```

### Utilisation dans un pipeline

```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    def image = buildDocker(
                        registry: 'registry.monlab.local:5000',
                        imageName: 'myapp',
                        platforms: ['linux/amd64', 'linux/arm64']
                    )
                    env.DOCKER_IMAGE = image
                }
            }
        }

        stage('Deploy') {
            steps {
                deployK8s(
                    namespace: 'production',
                    deployment: 'myapp',
                    image: env.DOCKER_IMAGE,
                    replicas: 3,
                    strategy: 'BlueGreen'
                )
            }
        }
    }
}
```

## Plugins essentiels pour Jenkins

### Plugins de base pour Kubernetes

```groovy
// Liste des plugins recommandés
def plugins = [
    'kubernetes',              // Intégration Kubernetes
    'kubernetes-cli',          // kubectl dans Jenkins
    'docker-workflow',         // Support Docker dans pipelines
    'docker-plugin',           // Docker comme cloud provider
    'blueocean',              // Interface moderne
    'pipeline-stage-view',     // Visualisation des stages
    'gitlab-plugin',          // Intégration GitLab
    'github-branch-source',    // Multibranch avec GitHub
    'slack',                  // Notifications Slack
    'email-ext',              // Emails avancés
    'prometheus',             // Métriques Prometheus
    'configuration-as-code',  // JCasC
    'job-dsl',                // Jobs as Code
    'hashicorp-vault-plugin', // Secrets Vault
    'sonarqube',              // Analyse de code
    'dependency-check',       // Scan de dépendances
]
```

### Configuration du plugin Kubernetes

```groovy
// Configuration programmatique via Job DSL
job('kubernetes-pipeline') {
    definition {
        cpsScm {
            scm {
                git {
                    remote {
                        url('https://github.com/monlab/myapp.git')
                    }
                    branch('*/main')
                }
            }
            scriptPath('Jenkinsfile')
        }
    }

    properties {
        pipelineTriggers {
            triggers {
                githubPush()
                cron('H 2 * * 1-5')
            }
        }
    }
}
```

## Optimisation des pipelines

### Parallélisation avancée

```groovy
pipeline {
    agent any

    stages {
        stage('Tests Parallèles') {
            parallel {
                stage('Tests Frontend') {
                    agent {
                        label 'node'
                    }
                    steps {
                        sh 'npm run test:frontend'
                    }
                }

                stage('Tests Backend') {
                    agent {
                        label 'java'
                    }
                    steps {
                        sh './gradlew test'
                    }
                }

                stage('Tests E2E') {
                    agent {
                        label 'selenium'
                    }
                    steps {
                        sh 'npm run test:e2e'
                    }
                }

                stage('Security Scan') {
                    agent {
                        label 'security'
                    }
                    steps {
                        sh '''
                            trivy fs .
                            snyk test
                            safety check
                        '''
                    }
                }
            }
        }
    }
}
```

### Cache et optimisation

```groovy
pipeline {
    agent any

    options {
        // Cache des dépendances
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout avec cache') {
            steps {
                checkout scm

                // Cache npm
                cache(maxCacheSize: 250, caches: [
                    arbitraryFileCache(
                        path: 'node_modules',
                        includes: '**/*',
                        fingerprinting: true
                    )
                ]) {
                    sh 'npm ci'
                }

                // Cache Docker layers
                script {
                    docker.build("myapp:${env.BUILD_ID}",
                        "--cache-from registry.monlab.local:5000/myapp:cache .")
                }
            }
        }
    }
}
```

## Comparaison GitLab CI vs Jenkins

### Tableau comparatif détaillé

| Aspect | GitLab CI | Jenkins |
|--------|-----------|---------|
| **Configuration** | YAML dans le repo | UI ou Jenkinsfile |
| **Intégration Git** | Native et complète | Via plugins |
| **Interface** | Moderne et intuitive | Classique (BlueOcean améliore) |
| **Courbe d'apprentissage** | Douce | Plus raide |
| **Flexibilité** | Bonne | Excellente |
| **Plugins** | Limités mais suffisants | Des milliers disponibles |
| **Runners** | Auto-scaling facile | Configuration manuelle |
| **Maintenance** | Minimale | Régulière |
| **Coût** | Gratuit (CE) | Gratuit (OSS) |
| **Registry intégré** | Oui | Via plugins |
| **Secrets** | Gestion native | Via plugins |
| **Monitoring** | Prometheus intégré | Via plugins |

### Quand choisir GitLab CI

GitLab CI est idéal quand :
- Vous utilisez déjà GitLab pour le code
- Vous voulez une solution tout-en-un
- Vous préférez la configuration en YAML
- Vous avez une équipe peu expérimentée en CI/CD
- Vous voulez démarrer rapidement

### Quand choisir Jenkins

Jenkins est préférable quand :
- Vous avez des besoins très spécifiques
- Vous utilisez déjà Jenkins en entreprise
- Vous avez besoin de plugins particuliers
- Vous voulez un contrôle total
- Vous avez des pipelines complexes

## Intégration avec MicroK8s

### Configuration des credentials Kubernetes

Pour GitLab CI :
```bash
# Récupérer le kubeconfig
microk8s config > kubeconfig

# Encoder en base64
cat kubeconfig | base64 -w0

# Ajouter comme variable CI/CD dans GitLab
# Settings → CI/CD → Variables
# Key: KUBECONFIG_BASE64
# Value: [contenu base64]
# Type: File
# Protected: Yes
# Masked: Yes
```

Pour Jenkins :
```groovy
// Ajouter via Jenkins Credentials
// Kind: Secret file
// ID: kubeconfig
// File: Upload kubeconfig

// Utilisation dans pipeline
withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    sh 'kubectl get pods'
}
```

### Déploiement automatique sur MicroK8s

```yaml
# Template de déploiement réutilisable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  labels:
    app: ${APP_NAME}
    version: ${VERSION}
    managed-by: ${CI_TOOL}
spec:
  replicas: ${REPLICAS}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
        version: ${VERSION}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${DOCKER_IMAGE}
        ports:
        - containerPort: ${APP_PORT}
          name: http
        env:
        - name: ENVIRONMENT
          value: ${ENVIRONMENT}
        - name: VERSION
          value: ${VERSION}
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## Monitoring des pipelines

### Métriques Prometheus pour GitLab CI

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: 'gitlab'
    static_configs:
      - targets: ['gitlab.monlab.local:9090']
    metrics_path: '/-/metrics'
    params:
      token: ['YOUR_GITLAB_TOKEN']
```

### Métriques Jenkins

```groovy
// Installation du plugin Prometheus
// Configuration dans Jenkins
prometheus {
    collectingMetricsPeriodInSeconds = 120
    countAbortedBuilds = true
    countFailedBuilds = true
    countNotBuiltBuilds = true
    countSuccessfulBuilds = true
    countUnstableBuilds = true
    fetchTestResults = true
    processingDisabledBuilds = false
}
```

### Dashboard Grafana pour CI/CD

```json
{
  "dashboard": {
    "title": "CI/CD Pipeline Metrics",
    "panels": [
      {
        "title": "Build Success Rate",
        "targets": [
          {
            "expr": "rate(jenkins_builds_success_total[5m])"
          }
        ]
      },
      {
        "title": "Average Build Duration",
        "targets": [
          {
            "expr": "avg(jenkins_builds_duration_milliseconds_summary)"
          }
        ]
      },
      {
        "title": "Pipeline Queue Length",
        "targets": [
          {
            "expr": "jenkins_queue_size_value"
          }
        ]
      },
      {
        "title": "Deployment Frequency",
        "targets": [
          {
            "expr": "increase(deployments_total[1d])"
          }
        ]
      }
    ]
  }
}
```

## Gestion des secrets dans les pipelines

### Vault Integration

```groovy
// Jenkins avec HashiCorp Vault
pipeline {
    agent any

    stages {
        stage('Get Secrets') {
            steps {
                withVault(
                    configuration: [
                        vaultUrl: 'http://vault.monlab.local:8200',
                        vaultCredentialId: 'vault-token'
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/data/myapp',
                            engineVersion: 2,
                            secretValues: [
                                [envVar: 'DB_PASSWORD', vaultKey: 'db_password'],
                                [envVar: 'API_KEY', vaultKey: 'api_key']
                            ]
                        ]
                    ]
                ) {
                    sh 'echo "Using secrets from Vault"'
                    sh 'kubectl create secret generic myapp-secrets \
                        --from-literal=db-password=$DB_PASSWORD \
                        --from-literal=api-key=$API_KEY'
                }
            }
        }
    }
}
```

### Sealed Secrets

```yaml
# GitLab CI avec Sealed Secrets
deploy:
  script:
    - |
      # Créer le secret
      echo -n "$DB_PASSWORD" | kubectl create secret generic myapp-secret \
        --dry-run=client \
        --from-literal=password=/dev/stdin \
        -o yaml > secret.yaml

      # Sceller le secret
      kubeseal --format=yaml < secret.yaml > sealed-secret.yaml

      # Appliquer le secret scellé
      kubectl apply -f sealed-secret.yaml
```

## Stratégies de déploiement avancées

### Blue-Green Deployment

```groovy
// Pipeline Jenkins pour Blue-Green
def deployBlueGreen(environment, newVersion) {
    // Déterminer la couleur actuelle
    def currentColor = sh(
        script: "kubectl get service myapp -n ${environment} -o jsonpath='{.spec.selector.color}'",
        returnStdout: true
    ).trim()

    def newColor = (currentColor == 'blue') ? 'green' : 'blue'

    // Déployer la nouvelle version
    sh """
        kubectl set image deployment/myapp-${newColor} \
            myapp=registry.monlab.local:5000/myapp:${newVersion} \
            -n ${environment}

        kubectl rollout status deployment/myapp-${newColor} -n ${environment}
    """

    // Tests de smoke
    def healthCheck = sh(
        script: "curl -f http://myapp-${newColor}.${environment}.svc.cluster.local/health",
        returnStatus: true
    )

    if (healthCheck == 0) {
        // Basculer le trafic
        sh """
            kubectl patch service myapp -n ${environment} \
                -p '{"spec":{"selector":{"color":"${newColor}"}}}'
        """

        echo "✅ Déploiement Blue-Green réussi. Nouvelle version: ${newColor}"
    } else {
        error "❌ Health check échoué pour ${newColor}"
    }
}
```

### Canary Deployment

```yaml
# GitLab CI Canary Deployment
deploy:canary:
  stage: deploy
  script:
    - |
      # Déployer la version canary (10% du trafic)
      kubectl apply -f - <<EOF
      apiVersion: v1
      kind: Service
      metadata:
        name: myapp-canary
        namespace: production
      spec:
        selector:
          app: myapp
          version: canary
        ports:
        - port: 80
          targetPort: 8080
      ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp-canary
        namespace: production
      spec:
        replicas: 1  # 10% des replicas totaux
        selector:
          matchLabels:
            app: myapp
            version: canary
        template:
          metadata:
            labels:
              app: myapp
              version: canary
          spec:
            containers:
            - name: myapp
              image: ${IMAGE_NAME}:${CI_COMMIT_SHA}
      EOF

      # Configurer Istio pour le split du trafic
      kubectl apply -f - <<EOF
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: myapp
        namespace: production
      spec:
        http:
        - match:
          - headers:
              canary:
                exact: "true"
          route:
          - destination:
              host: myapp-canary
            weight: 100
        - route:
          - destination:
              host: myapp-stable
            weight: 90
          - destination:
              host: myapp-canary
            weight: 10
      EOF
```

## Troubleshooting des pipelines

### Problèmes courants GitLab CI

**Problème : Runner n'exécute pas les jobs**
```bash
# Vérifier l'état du runner
gitlab-runner verify

# Voir les logs
gitlab-runner --debug run

# Réenregistrer le runner
gitlab-runner register --url http://gitlab.monlab.local \
  --registration-token YOUR_TOKEN
```

**Problème : Échec de connexion au registry**
```yaml
# Solution dans .gitlab-ci.yml
before_script:
  - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
  # Ou pour un registry insecure
  - mkdir -p /etc/docker
  - echo '{"insecure-registries":["'$CI_REGISTRY'"]}' > /etc/docker/daemon.json
  - service docker restart || true
```

### Problèmes courants Jenkins

**Problème : Out of memory**
```groovy
// Augmenter la mémoire Java
JAVA_OPTS="-Xmx2048m -XX:MaxPermSize=512m"

// Dans le pipeline
options {
    timeout(time: 1, unit: 'HOURS')
    disableConcurrentBuilds()
}
```

**Problème : Pipeline bloqué**
```groovy
// Ajouter des timeouts
timeout(time: 10, unit: 'MINUTES') {
    input message: 'Continue?'
}

// Nettoyer les workspaces
post {
    always {
        cleanWs()
    }
}
```

## Bonnes pratiques pour les pipelines

### 1. Fail Fast
```yaml
# GitLab CI
stages:
  - validate  # Vérifications rapides d'abord
  - build
  - test
  - deploy

validate:syntax:
  stage: validate
  script:
    - yamllint .
    - jsonlint *.json
  allow_failure: false
```

### 2. DRY (Don't Repeat Yourself)
```groovy
// Jenkins Shared Library
def buildAndPush(service) {
    docker.build("${REGISTRY}/${service}:${VERSION}")
    docker.withRegistry("https://${REGISTRY}", 'docker-creds') {
        docker.push("${VERSION}")
    }
}

// Utilisation
['frontend', 'backend', 'worker'].each { service ->
    stage("Build ${service}") {
        buildAndPush(service)
    }
}
```

### 3. Idempotence
```yaml
# Toujours pouvoir relancer sans effet de bord
deploy:
  script:
    - kubectl apply -f deployment.yaml  # Idempotent
    # Pas: kubectl create -f deployment.yaml  # Échoue si existe
```

### 4. Versioning sémantique
```groovy
def getVersion() {
    if (env.TAG_NAME) {
        return env.TAG_NAME
    } else if (env.BRANCH_NAME == 'main') {
        return "latest"
    } else {
        return "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    }
}
```

## Conclusion

Les pipelines CI/CD avec GitLab CI ou Jenkins transforment radicalement votre capacité à livrer du logiciel de qualité. Dans votre lab MicroK8s, vous disposez maintenant de tous les éléments pour :

- **Automatiser** complètement le cycle de vie de vos applications
- **Standardiser** les processus de build et déploiement
- **Garantir** la qualité avec des tests automatiques
- **Accélérer** la mise en production des nouvelles fonctionnalités
- **Sécuriser** les déploiements avec des stratégies avancées

Le choix entre GitLab CI et Jenkins dépend de vos besoins spécifiques. GitLab CI offre une expérience intégrée et moderne, idéale pour démarrer rapidement. Jenkins apporte une flexibilité maximale et un écosystème de plugins incomparable pour les besoins complexes.

Les compétences acquises avec ces outils dans votre lab personnel sont directement applicables en entreprise, où l'automatisation CI/CD est devenue indispensable pour rester compétitif.

⏭️
