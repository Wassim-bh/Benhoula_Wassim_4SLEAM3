pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'      
        maven 'M2_HOME'     
    }

    environment {
        DOCKERHUB_CREDENTIALS = "docker-credentials"
        DOCKER_IMAGE = "wassimbenhoula/4sleam3"
    }

    stages {
        stage('GIT Clone') {
            steps {
                git branch: 'main',   
                    url: 'https://github.com/Wassim-bh/Benhoula_Wassim_4SLEAM3.git'
            }
        }

        stage('Compile Stage') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE:${BUILD_NUMBER} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag $DOCKER_IMAGE:${BUILD_NUMBER} $DOCKER_IMAGE:latest
                        docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                        docker push $DOCKER_IMAGE:latest
                        """
                    }
                }
            }
        }

        // =============== NOUVEAU STAGE KUBERNETES ===============
        stage('Setup Kubernetes Environment') {
            steps {
                script {
                    echo "=== SETUP KUBERNETES ENVIRONMENT ==="
                    
                    sh '''
                    # Vérifier et installer kubectl si nécessaire
                    if ! command -v kubectl &> /dev/null; then
                        echo "Installing kubectl..."
                        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        sudo mv kubectl /usr/local/bin/
                    fi
                    
                    # Vérifier et installer Minikube si nécessaire
                    if ! command -v minikube &> /dev/null; then
                        echo "Installing Minikube..."
                        curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
                        sudo install minikube-linux-amd64 /usr/local/bin/minikube
                    fi
                    
                    # Démarrer Minikube
                    echo "Starting Minikube..."
                    minikube start --memory=4096 --cpus=2 --driver=docker || minikube status
                    
                    # Vérifier
                    echo "Kubernetes cluster status:"
                    kubectl get nodes
                    echo ""
                    '''
                }
            }
        }

        stage('Deploy MySQL to Kubernetes') {
            steps {
                script {
                    echo "=== DEPLOYING MYSQL ==="
                    
                    sh '''
                    # Déployer MySQL
                    echo "1. Creating MySQL secret..."
                    kubectl apply -f k8s/mysql-secret.yaml
                    
                    echo "2. Creating persistent volume..."
                    kubectl apply -f k8s/mysql-pv.yaml
                    
                    echo "3. Deploying MySQL..."
                    kubectl apply -f k8s/mysql-deployment.yaml
                    
                    # Attendre
                    echo "Waiting for MySQL to start (30 seconds)..."
                    sleep 30
                    
                    # Vérifier
                    echo "MySQL status:"
                    kubectl get pods -l app=mysql
                    kubectl get svc mysql-service
                    echo ""
                    '''
                }
            }
        }

        stage('Deploy Spring Boot to Kubernetes') {
            steps {
                script {
                    echo "=== DEPLOYING SPRING BOOT ==="
                    
                    sh '''
                    # Déployer Spring Boot
                    echo "1. Deploying Spring Boot application..."
                    kubectl apply -f k8s/springboot-deployment.yaml
                    
                    # Attendre
                    echo "Waiting for Spring Boot to start (40 seconds)..."
                    sleep 40
                    
                    # Vérifier
                    echo "Spring Boot status:"
                    kubectl get pods -l app=springboot-app
                    kubectl get svc springboot-service
                    echo ""
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "=== VERIFYING DEPLOYMENT ==="
                    
                    sh '''
                    # Afficher tout
                    echo "All Kubernetes resources:"
                    kubectl get all
                    
                    # Obtenir l'URL
                    MINIKUBE_IP=$(minikube ip)
                    NODE_PORT=$(kubectl get svc springboot-service -o jsonpath="{.spec.ports[0].nodePort}")
                    
                    echo ""
                    echo "========================================"
                    echo "🚀 DEPLOYMENT SUCCESSFUL!"
                    echo "========================================"
                    echo "Spring Boot Application URL:"
                    echo "  http://$MINIKUBE_IP:$NODE_PORT"
                    echo ""
                    echo "Health Check:"
                    echo "  http://$MINIKUBE_IP:$NODE_PORT/actuator/health"
                    echo ""
                    echo "MySQL Database:"
                    echo "  Service: mysql-service:3306"
                    echo "========================================"
                    
                    # Tester l'application
                    echo "Testing application connectivity..."
                    sleep 10
                    
                    if curl -f http://$MINIKUBE_IP:$NODE_PORT/actuator/health; then
                        echo "✅ Application is responding!"
                    else
                        echo "⚠️ Application not responding, checking logs..."
                        SPRING_POD=$(kubectl get pods -l app=springboot-app -o jsonpath="{.items[0].metadata.name}")
                        kubectl logs $SPRING_POD --tail=20
                    fi
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ PIPELINE COMPLETED SUCCESSFULLY!'
            sh '''
            echo "=== FINAL STATUS ==="
            kubectl get pods
            kubectl get svc
            '''
        }
        failure {
            echo '❌ PIPELINE FAILED'
            sh '''
            echo "=== TROUBLESHOOTING ==="
            echo "Recent events:"
            kubectl get events --sort-by='.lastTimestamp' | tail -20
            echo ""
            echo "Pod details:"
            kubectl describe pods
            '''
        }
    }
}
