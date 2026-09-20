pipeline {
  agent any

  parameters {
    string(name: 'REPO_URL',    defaultValue: 'https://github.com/wangyong-fy/learn-k8s-frontend.git', description: '前端仓库地址')
    string(name: 'REPO_BRANCH', defaultValue: 'main',                                                  description: '分支')
    string(name: 'IMAGE_TAG',   defaultValue: '',                                                      description: '留空则用构建号')
  }

  environment {
    REGISTRY = 'wanyongdoker'
    APP_NS   = 'app'
    KANIKO   = 'gcr.io/kaniko-project/executor:v1.23.2'
    CACHE    = 'wanyongdoker/kaniko-cache'
  }

  options {
    disableConcurrentBuilds()
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: "${params.REPO_BRANCH}", url: "${params.REPO_URL}"
        script {
          env.TAG = params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : env.BUILD_NUMBER
          echo "前端本次构建 TAG = ${env.TAG}"
        }
      }
    }

    stage('Build & Push Frontend') {
      steps {
        sh '''
          set -e
          HOSTPATH="${REPO_URL#*://}"
          CTX="git://${HOSTPATH}#refs/heads/${REPO_BRANCH}"
          POD=kaniko-frontend-${TAG}
          echo "frontend context = ${CTX}"

          cat > /tmp/kaniko-fe.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ${POD}
  namespace: jenkins
spec:
  restartPolicy: Never
  containers:
    - name: kaniko
      image: ${KANIKO}
      args:
        - "--context=${CTX}"
        - "--dockerfile=Dockerfile"
        - "--destination=${REGISTRY}/learn-k8s-frontend:${TAG}"
        - "--snapshot-mode=time"
        - "--compressed-caching=false"
        - "--cache=true"
        - "--cache-repo=${CACHE}"
        - "--verbosity=info"
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "2"
          memory: "2Gi"
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      secret:
        secretName: dockerhub
        items:
          - key: .dockerconfigjson
            path: config.json
EOF

          kubectl -n jenkins delete pod ${POD} --ignore-not-found
          kubectl -n jenkins apply -f /tmp/kaniko-fe.yaml

          i=0
          while [ $i -lt 180 ]; do
            PH=$(kubectl -n jenkins get pod ${POD} -o jsonpath='{.status.phase}' 2>/dev/null || echo "")
            if [ "$PH" = "Succeeded" ]; then echo "frontend 镜像推送成功"; break; fi
            if [ "$PH" = "Failed" ]; then
              echo "frontend 构建失败，日志："
              kubectl -n jenkins logs ${POD} --tail=200
              exit 1
            fi
            if [ $((i % 6)) -eq 0 ]; then
              echo "[$((i*10))s] 状态=${PH:-Pending} 最近日志:"
              kubectl -n jenkins logs ${POD} --tail=3 2>/dev/null | sed 's/^/    /' || true
            fi
            sleep 10
            i=$((i+1))
          done
          [ "$PH" = "Succeeded" ] || { echo "frontend 构建超时"; kubectl -n jenkins logs ${POD} --tail=200; exit 1; }
          kubectl -n jenkins delete pod ${POD} --ignore-not-found
        '''
      }
    }

    stage('Deploy Frontend & Ingress') {
      steps {
        sh '''
          set -e
          kubectl -n "$APP_NS" apply -f k8s/frontend.yaml
          kubectl -n "$APP_NS" apply -f k8s/ingress.yaml
          kubectl -n "$APP_NS" set image deployment/frontend frontend=${REGISTRY}/learn-k8s-frontend:${TAG}
          kubectl -n "$APP_NS" rollout status deployment/frontend --timeout=300s
        '''
      }
    }
  }

  post {
    success {
      echo "前端 CI/CD 完成：${REGISTRY}/learn-k8s-frontend:${TAG}"
    }
    failure {
      echo "前端 CI/CD 失败，请查看上方日志"
    }
  }
}
