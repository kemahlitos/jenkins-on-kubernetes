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
    - name: git
      image: alpine/git:2.45.2
      command: ["/bin/sh"]
      args: ["-c","sleep infinity"]
      tty: true
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
      defaultContainer 'git'
    }
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  parameters {
    string(name: 'TAG', defaultValue: '', description: 'Opsiyonel: özel image tag. Boşsa build-<BUILD>-<gitSHA> üretilir')
  }

  environment {
    REGISTRY            = 'docker.io'
    DOCKERHUB_NAMESPACE = 'kemahlitos'
    APP_NAME            = 'hello-web'
    IMAGE_NAME          = 'docker.io/kemahlitos/hello-web' // kustomization.yaml ile birebir
    MANIFEST_REPO_URL   = 'https://github.com/kemahlitos/hello-web-manifests'
    MANIFEST_PATH       = 'base'
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

mkdir -p /kaniko/.docker
AUTH="$(printf "%s:%s" "$DOCKERHUB_USR" "$DOCKERHUB_PSW" | base64 | tr -d '\\n')"
cat > /kaniko/.docker/config.json <<EOF
{"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF

if [ -z "${TAG:-}" ]; then
  : "${BUILD_NUMBER:=0}"
  SHORT_SHA="$(git rev-parse --short=7 HEAD 2>/dev/null || true)"; : "${SHORT_SHA:=local}"
  TAG="build-${BUILD_NUMBER}-${SHORT_SHA}"
fi
echo "Using TAG: $TAG"

IMAGE="${REGISTRY}/${DOCKERHUB_NAMESPACE}/${APP_NAME}:${TAG}"
echo "IMAGE=${IMAGE}" > "$WORKSPACE/image.env"
echo "TAG=${TAG}"     >> "$WORKSPACE/image.env"
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

    stage('Update manifest repo (bump image tag)') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'git-creds',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          container('git') {
            sh '''#!/bin/sh
set -euo pipefail
. "$WORKSPACE/image.env"

: "${MANIFEST_REPO_URL:=https://github.com/kemahlitos/hello-web-manifests}"
: "${MANIFEST_PATH:=base}"

rm -rf manifests
AUTH_URL="$(printf "%s" "${MANIFEST_REPO_URL}" | sed -E "s#https://#https://${GIT_USER}:${GIT_TOKEN}#")"
git clone "${AUTH_URL}" manifests
cd "manifests/${MANIFEST_PATH}"

if [ ! -f kustomization.yaml ]; then
  echo "kustomization.yaml bulunamadı! (path: $(pwd))" >&2
  exit 1
fi

awk -v img="${IMAGE_NAME}" -v tag="${TAG}" '
  BEGIN{inimages=0; target=0}
  /^images:/ {inimages=1; print; next}
  {
    line=$0
    if (inimages==1) {
      if (match(line, /^[[:space:]]*- name:[[:space:]]*/)) {
        n=line
        gsub(/"/,"", n)
        sub(/^[[:space:]]*- name:[[:space:]]*/, "", n)
        target=(n==img)?1:0
        print $0
        next
      }
      if (target==1 && match(line, /^[[:space:]]*newTag:[[:space:]]*/)) {
        sub(/newTag:[[:space:]]*.*/, "newTag: " tag)
        target=0
        print $0
        next
      }
    }
    print $0
  }
' kustomization.yaml > kustomization.yaml.tmp && mv kustomization.yaml.tmp kustomization.yaml

echo "----- kustomization.yaml (after) -----"
sed -n '1,120p' kustomization.yaml

git config user.name  "jenkins-bot"
git config user.email "jenkins-bot@local"
git add -A
git commit -m "chore(cd): hello-web image -> ${IMAGE}" || echo "No changes to commit"
git push
'''
          }
        }
      }
    }
  }

  post {
    success { echo "SUCCESS → ${env.BUILD_URL}" }
    failure { echo "FAILED → logları kontrol et" }
  }
}
