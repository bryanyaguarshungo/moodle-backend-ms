pipeline {
    agent any
    environment {
        IMAGE = "bryanyaguarshungo/moodle-backend:latest"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/bryanyaguarshungo/moodle-backend-ms.git'
            }
        }

        stage('Build Docker image') {
            steps {
                script {
                    docker.build("${IMAGE}")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('', 'docker-hub') {
                        docker.image("${IMAGE}").push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    kubectl delete deployment moodle-ci || true

                    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: moodle-ci
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moodle-ci
  template:
    metadata:
      labels:
        app: moodle-ci
    spec:
      containers:
      - name: moodle
        image: ${IMAGE}
        ports:
        - containerPort: 8080
EOF

                    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: moodle-ci
spec:
  type: NodePort
  selector:
    app: moodle-ci
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 31080
EOF

                    kubectl rollout status deployment/moodle-ci --timeout=120s
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl get svc moodle-ci'
                }
            }
        }
    }
}
