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
      command: ['sleep','infinity']        # pod ayakta kalsın
      tty: true
      volumeMounts:
        - name: kaniko-docker-config
          mountPath: /kaniko/.docker       # auth dosyası için
    - name: kubectl
      image: bitnami/kubectl:1.30-debian-12
      command: ['sleep','infinity']
      tty: true
  volumes:
    - name: kaniko-docker-config
      emptyDir: {}
"""
      defaultContainer 'kubectl'
    }
  }

  options {
    timestamps()
    disableConcurrentBuilds()               // aynı anda 2 build çalışmasın
  }

  environment {
    // ---- KENDİ KULLANICINI DOLDUR ----
    DOCKERHUB_NAMESPACE = "kemahlitos"   // Docker Hub kullanıcı adın
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
        sh '''
          set -euo pipefail

          # 1) Docker Hub auth
          mkdir -p /kaniko/.docker
          AUTH=$(printf "%s:%s" "$DOCKERHUB_USR" "$DOCKERHUB_PSW" | base64 | tr -d "\\n")
          cat > /kaniko/.docker/config.json <<EOF
          {"auths":{"https://index.docker.io/v1/":{"auth":"${AUTH}"}}}
          EOF

          # 2) TAG üret (güvenli)
          SHORT_SHA=$(git rev-parse --short=7 HEAD 2>/dev/null || true)
          [ -n "$SHORT_SHA" ] || SHORT_SHA="local"
          TAG="${BUILD_NUMBER:-0}-${SHORT_SHA}"

          # 3) IMAGE ismi (tüm env'ler dolu mu kontrolü)
          : "${REGISTRY:?REGISTRY boş}"
          : "${DOCKERHUB_NAMESPACE:?DOCKERHUB_NAMESPACE boş}"
          : "${APP_NAME:?APP_NAME boş}"
          IMAGE="${REGISTRY}/${DOCKERHUB_NAMESPACE}/${APP_NAME}:${TAG}"
          echo "IMAGE=${IMAGE}" > image.env
          echo "Pushing: ${IMAGE}"

          # 4) Kaniko build & push
          /kaniko/executor \
            --context "$WORKSPACE" \
            --dockerfile "$WORKSPACE/Dockerfile" \
            --destination "$IMAGE" \
            --single-snapshot \
            --use-new-run \
            --cache=true
        '''
      }
    }
  }
}

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            set -eux
            . image.env

            # Namespace varsa geç, yoksa oluştur
            kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}

            # İlk sefer: deploy + service oluştur (idempotent)
            kubectl -n ${NAMESPACE} create deploy ${APP_NAME} --image="${IMAGE}" \
              --dry-run=client -o yaml | kubectl apply -f -

            # Her seferinde imaj güncelle ve rollout bekle
            kubectl -n ${NAMESPACE} set image deploy/${APP_NAME} ${APP_NAME}="${IMAGE}" --record
            kubectl -n ${NAMESPACE} rollout status deploy/${APP_NAME}

            # Service yoksa oluştur (NodePort)
            kubectl -n ${NAMESPACE} get svc ${APP_NAME} || \
              kubectl -n ${NAMESPACE} expose deploy/${APP_NAME} --port=80 --type=NodePort

            # NodePort'u yaz
            PORT=$(kubectl -n ${NAMESPACE} get svc ${APP_NAME} -o jsonpath='{.spec.ports[0].nodePort}')
            echo "NODEPORT=${PORT}" > service.env
            echo "NodePort: ${PORT}"
          '''
        }
      }
    }

    stage('Info') {
      steps {
        sh '''
          set -eux
          . image.env
          . service.env
          echo "Deployed image: ${IMAGE}"
          echo "Kubernetes namespace: ${NAMESPACE}"
          echo "Service: ${APP_NAME}"
          echo "NodePort: ${NODEPORT}"
          echo "Browser URL (any node IP):"
          echo "  http://192.168.64.30:${NODEPORT}"
          echo "  http://192.168.64.31:${NODEPORT}"
        '''
      }
    }
  }

  post {
    success { echo "SUCCESS → ${env.BUILD_URL}" }
    failure { echo "FAILED → logları kontrol et" }
  }
}
