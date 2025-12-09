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
        stage('GIT') {
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
	stage('Deploy to Kubernetes') {
    steps {
        script {
            withCredentials([file(credentialsId: 'minikube-kubeconfig', variable: 'KUBECONFIG')]) {
                sh '''
                export KUBECONFIG=$KUBECONFIG
                
                echo "=== Déploiement MySQL ==="
                kubectl apply -f k8s/mysql-secret.yaml
                kubectl apply -f k8s/mysql-pv.yaml
                kubectl apply -f k8s/mysql-deployment.yaml
                
                echo "Attente du démarrage de MySQL (30s)..."
                sleep 30
                
                echo "=== Déploiement Spring Boot ==="
                kubectl apply -f k8s/springboot-deployment.yaml
                
                echo "Attente du démarrage de Spring Boot (40s)..."
                sleep 40
                
                echo "=== Vérification ==="
                kubectl get all
                '''
            }
        }
    }
}
        }

        
    }
}
