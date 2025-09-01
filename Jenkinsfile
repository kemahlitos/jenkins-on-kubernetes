pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins

  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command: ["/bin/sh"]
      args: ["-c","sleep infinity"]
      tty: true
      volumeMounts:
        - name: kaniko-docker-config
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: kubectl
      image: bitnami/kubectl:1.30-debian-12
      command: ["/bin/sh"]
      args: ["-c","sleep infinity"]
      tty: true
      securityContext:
        runAsUser: 0
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: jnlp
      image: jenkins/inbound-agent:3327.v868139a_d00e0-6
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

  volumes:
    - name: kaniko-docker-config
      emptyDir: {}
    - name: workspace-volume
      emptyDir: {}
"""
      defaultContainer 'kubectl'
    }
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  parameters {
    string(name: 'TAG',          defaultValue: '',                 description: 'İsteğe bağlı image tag (boşsa otomatik)')
    string(name: 'INGRESS_HOST', defaultValue: 'hello-web.local',  description: 'Ingress FQDN (örn. hello-web.local)')
  }

  environment {
    DOCKERHUB_NAMESPACE = "kemahlitos"
    APP_NAME            = "hello-web"
    REGISTRY            = "docker.io"
    NAMESPACE           = "demo"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push (Kaniko)') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKERHUB_USR',
          passwordVariable: 'DOCKERHUB_PSW'
        )]) {
          container('kaniko') {
            sh '''#!/bin/sh
set -euo pipefail

# 1) Docker Hub auth
mkdir -p /kaniko/.docker
AUTH="$(printf "%s:%s" "$DOCKERHUB_USR" "$DOCKERHUB_PSW" | base64 | tr -d '\\n')"
cat > /kaniko/.docker/config.json <<EOF
{"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF

# 2) TAG: parametre > git SHA > fallback
if [ -z "${TAG:-}" ]; then
  SHORT_SHA="$(git rev-parse --short=7 HEAD 2>/dev/null || true)"
  : "${SHORT_SHA:=local}"
  : "${BUILD_NUMBER:=0}"
  TAG="${BUILD_NUMBER}-${SHORT_SHA}"
fi
echo "Using TAG: $TAG"

# 3) IMAGE
: "${REGISTRY:?REGISTRY boş}"
: "${DOCKERHUB_NAMESPACE:?DOCKERHUB_NAMESPACE boş}"
: "${APP_NAME:?APP_NAME boş}"
IMAGE="${REGISTRY}/${DOCKERHUB_NAMESPACE}/${APP_NAME}:${TAG}"
echo "IMAGE=${IMAGE}" > "$WORKSPACE/image.env"
echo "Pushing: ${IMAGE}"

/kaniko/executor \
  --context "$WORKSPACE" \
  --dockerfile "$WORKSPACE/Dockerfile" \
  --destination "$IMAGE" \
  --cache=true --single-snapshot --use-new-run
'''
          }
        }
      }
    }

    stage('Deploy (Deployment + Service + Ingress)') {
      steps {
        container('kubectl') {
          sh '''#!/bin/sh
set -eux
. "$WORKSPACE/image.env"

# Manifests
mkdir -p "$WORKSPACE/k8s"

# --- Deployment ---
cat > "$WORKSPACE/k8s/deployment.yaml" <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${IMAGE}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
EOF

# --- Service (ClusterIP) ---
cat > "$WORKSPACE/k8s/service.yaml" <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  type: ClusterIP
  selector:
    app: ${APP_NAME}
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF

# --- Ingress (nginx) ---
cat > "$WORKSPACE/k8s/ingress.yaml" <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  ingressClassName: nginx
  rules:
  - host: ${INGRESS_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}
            port:
              number: 80
EOF

# Uygula
kubectl apply -f "$WORKSPACE/k8s/deployment.yaml"
kubectl apply -f "$WORKSPACE/k8s/service.yaml"
kubectl apply -f "$WORKSPACE/k8s/ingress.yaml"

# Rollout bekle
kubectl -n "${NAMESPACE}" rollout status "deploy/${APP_NAME}"

# Ingress erişim bilgisi (NodePort vs LoadBalancer)
CTRL_TYPE="$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.spec.type}' 2>/dev/null || true)"
LB_IP="$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)"
LB_HOST="$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true)"

if [ "$CTRL_TYPE" = "LoadBalancer" ] && [ -n "${LB_IP}${LB_HOST}" ]; then
  INGRESS_URL="http://${INGRESS_HOST}"
  INGRESS_HINT="DNS'te ${INGRESS_HOST} -> ${LB_IP:-$LB_HOST} eşle"
else
  HTTP_NODEPORT="$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}')"
  NODE_IP="$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')"
  INGRESS_URL="http://${INGRESS_HOST}:${HTTP_NODEPORT}"
  INGRESS_HINT="/etc/hosts satırı: ${NODE_IP} ${INGRESS_HOST}  → sonra tarayıcıdan ${INGRESS_URL}"
fi

cat > "$WORKSPACE/ingress.env" <<EOF
INGRESS_URL="${INGRESS_URL}"
INGRESS_HINT="${INGRESS_HINT}"
EOF

'''
        }
      }
    }

    stage('Info') {
      steps {
        sh '''#!/bin/sh
set -eux
. "$WORKSPACE/image.env"
[ -f "$WORKSPACE/ingress.env" ] && . "$WORKSPACE/ingress.env" || true

echo "Deployed image: ${IMAGE}"
echo "Namespace:      ${NAMESPACE}"
echo "Service:        ${APP_NAME} (ClusterIP)"
echo "Ingress host:   ${INGRESS_HOST}"
echo "Open URL:       ${INGRESS_URL:-http://${INGRESS_HOST}}"
echo "Hint:           ${INGRESS_HINT:-'Gerekirse /etc/hosts ile host->IP eşlemesi yapın.'}"
'''
      }
    }
  }

  post {
    success { echo "SUCCESS → ${env.BUILD_URL}" }
    failure { echo "FAILED → logları kontrol et" }
  }
}
