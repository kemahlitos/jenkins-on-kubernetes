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
    string(name: 'TAG', defaultValue: '', description: 'İsteğe bağlı image tag (boşsa otomatik üretilecek)')
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
set -eu

# 1) Docker Hub auth
mkdir -p /kaniko/.docker
AUTH="$(printf "%s:%s" "$DOCKERHUB_USR" "$DOCKERHUB_PSW" | base64 | tr -d '\\n')"
cat > /kaniko/.docker/config.json <<EOF
{"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF

# 2) TAG üretimi
if [ -z "${TAG:-}" ]; then
  SHORT_SHA="$(git rev-parse --short=7 HEAD 2>/dev/null || true)"
  : "${SHORT_SHA:=local}"
  : "${BUILD_NUMBER:=0}"
  TAG="${BUILD_NUMBER}-${SHORT_SHA}"
fi
echo "Using TAG: $TAG"

# 3) IMAGE ismi
IMAGE="${REGISTRY}/${DOCKERHUB_NAMESPACE}/${APP_NAME}:${TAG}"
echo "IMAGE=${IMAGE}" > "$WORKSPACE/image.env"
echo "Pushing: ${IMAGE}"

# 4) Kaniko build & push
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

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''#!/bin/sh
set -eu
. "$WORKSPACE/image.env"

# Namespace varsa geç, yoksa oluştur
kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"

# Deployment idempotent
kubectl -n "${NAMESPACE}" create deploy "${APP_NAME}" --image="${IMAGE}" \
  --dry-run=client -o yaml | kubectl apply -f -

# Image update ve rollout bekle
kubectl -n "${NAMESPACE}" set image "deploy/${APP_NAME}" "${APP_NAME}=${IMAGE}" --record
kubectl -n "${NAMESPACE}" rollout status "deploy/${APP_NAME}"

# Service: yoksa oluştur, varsa ClusterIP olarak güncelle
if ! kubectl -n "${NAMESPACE}" get svc "${APP_NAME}" >/dev/null 2>&1; then
  kubectl -n "${NAMESPACE}" expose deploy "${APP_NAME}" --port=80 --type=ClusterIP
else
  kubectl -n "${NAMESPACE}" patch svc "${APP_NAME}" -p '{"spec":{"type":"ClusterIP"}}' >/dev/null
fi
'''
        }
      }
    }

    stage('Expose via Ingress') {
      steps {
        container('kubectl') {
          sh '''#!/bin/sh
set -eu
. "$WORKSPACE/image.env"

cat <<YAML | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: ${APP_NAME}.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}
            port:
              number: 80
YAML

# Ingress controller’ın HTTP nodePort’unu bul
INGRESS_HTTP_NODEPORT="$(kubectl -n ingress-nginx get svc ingress-nginx-controller \
  -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}')"

echo "INGRESS_HTTP_NODEPORT=${INGRESS_HTTP_NODEPORT}" > "$WORKSPACE/ingress.env"
'''
        }
      }
    }

    stage('Info') {
      steps {
        sh '''#!/bin/sh
set -eu
. "$WORKSPACE/image.env"
. "$WORKSPACE/ingress.env"

echo "Deployed image: ${IMAGE}"
echo "Namespace: ${NAMESPACE}"
echo "Service: ${APP_NAME} (ClusterIP)"
echo "Ingress Host: ${APP_NAME}.local"
echo
echo "Browse URLs (hosts kaydın varsa):"
echo "  http://192.168.64.31:${INGRESS_HTTP_NODEPORT}"
echo "  http://192.168.64.30:${INGRESS_HTTP_NODEPORT} (eğer master'da açık ise)"
echo "  http://${APP_NAME}.local:${INGRESS_HTTP_NODEPORT}"
'''
      }
    }
  }

  post {
    success { echo "SUCCESS → ${env.BUILD_URL}" }
    failure { echo "FAILED → logları kontrol et" }
  }
}
