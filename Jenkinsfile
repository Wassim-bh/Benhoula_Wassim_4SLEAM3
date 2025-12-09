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
        }

        stage('Setup Kubernetes Environment') {
            steps {
                sh '''
                echo "=== SETUP KUBERNETES ==="
                
                # Vérifier si kubectl est installé
                if ! command -v kubectl &> /dev/null; then
                    echo "❌ kubectl n'est pas installé"
                    echo "Installation en cours..."
                    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    sudo mv kubectl /usr/local/bin/
                else
                    echo "✅ kubectl est installé"
                fi
                
                # Vérifier si Minikube est installé
                if ! command -v minikube &> /dev/null; then
                    echo "❌ Minikube n'est pas installé"
                    echo "Installation en cours..."
                    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
                    sudo install minikube-linux-amd64 /usr/local/bin/minikube
                else
                    echo "✅ Minikube est installé"
                fi
                
                # Démarrer Minikube
                echo "Démarrage de Minikube..."
                minikube start --memory=4096 --cpus=2 --driver=docker || echo "Minikube déjà démarré ou erreur"
                
                # Vérifier le cluster
                echo "Vérification du cluster Kubernetes..."
                kubectl cluster-info || { echo "❌ Impossible de se connecter au cluster"; exit 1; }
                kubectl get nodes || { echo "❌ Aucun nœud trouvé"; exit 1; }
                
                echo "✅ Environnement Kubernetes prêt"
                '''
            }
        }

        stage('Deploy MySQL to Kubernetes') {
            steps {
                sh '''
                echo "=== DÉPLOIEMENT MYSQL ==="
                
                # Vérifier que les fichiers YAML existent
                echo "Vérification des fichiers YAML..."
                ls -la k8s/ || { echo "❌ Dossier k8s non trouvé"; exit 1; }
                
                # Déployer MySQL avec vérification d'erreur
                echo "1. Création du secret MySQL..."
                kubectl apply -f k8s/mysql-secret.yaml || { echo "❌ Échec de création du secret"; exit 1; }
                
                echo "2. Création du volume persistant..."
                kubectl apply -f k8s/mysql-pv.yaml || { echo "❌ Échec de création du volume"; exit 1; }
                
                echo "3. Déploiement de MySQL..."
                kubectl apply -f k8s/mysql-deployment.yaml || { echo "❌ Échec du déploiement MySQL"; exit 1; }
                
                echo "⏳ Attente du démarrage de MySQL (40 secondes)..."
                sleep 40
                
                # Vérifier que MySQL tourne
                echo "Vérification de l'état MySQL:"
                kubectl get pods -l app=mysql || { echo "❌ Impossible de récupérer les pods MySQL"; exit 1; }
                kubectl get svc mysql-service || { echo "❌ Service MySQL non trouvé"; exit 1; }
                
                # Afficher les logs MySQL pour vérification
                echo "Logs MySQL (démarrage):"
                kubectl logs -l app=mysql --tail=20 || echo "⚠️  Impossible de récupérer les logs MySQL"
                
                echo "✅ MySQL déployé avec succès"
                '''
            }
        }

        stage('Deploy Spring Boot to Kubernetes') {
            steps {
                sh '''
                echo "=== DÉPLOIEMENT SPRING BOOT ==="
                
                # Vérifier que le fichier YAML existe
                if [ ! -f "k8s/springboot-deployment.yaml" ]; then
                    echo "❌ Fichier springboot-deployment.yaml non trouvé"
                    exit 1
                fi
                
                # Déployer Spring Boot
                echo "Déploiement de l'application Spring Boot..."
                kubectl apply -f k8s/springboot-deployment.yaml || { echo "❌ Échec du déploiement Spring Boot"; exit 1; }
                
                echo "⏳ Attente du démarrage de Spring Boot (50 secondes)..."
                sleep 50
                
                # Vérifier que Spring Boot tourne
                echo "Vérification de l'état Spring Boot:"
                kubectl get pods -l app=springboot-app || { echo "❌ Impossible de récupérer les pods Spring Boot"; exit 1; }
                kubectl get svc springboot-service || { echo "❌ Service Spring Boot non trouvé"; exit 1; }
                
                # Afficher les logs Spring Boot pour vérification
                echo "Logs Spring Boot (démarrage):"
                SPRING_POD=$(kubectl get pods -l app=springboot-app -o jsonpath="{.items[0].metadata.name}" 2>/dev/null || echo "")
                if [ -n "$SPRING_POD" ]; then
                    kubectl logs $SPRING_POD --tail=30 || echo "⚠️  Impossible de récupérer les logs Spring Boot"
                else
                    echo "⚠️  Pod Spring Boot non trouvé"
                fi
                
                echo "✅ Spring Boot déployé avec succès"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "=== VÉRIFICATION DU DÉPLOIEMENT ==="
                
                # Afficher toutes les ressources
                echo "Toutes les ressources Kubernetes:"
                kubectl get all || { echo "❌ Impossible de récupérer les ressources"; exit 1; }
                
                # Obtenir l'URL d'accès
                MINIKUBE_IP=$(minikube ip 2>/dev/null || echo "127.0.0.1")
                NODE_PORT=$(kubectl get svc springboot-service -o jsonpath="{.spec.ports[0].nodePort}" 2>/dev/null || echo "30080")
                
                echo ""
                echo "========================================"
                echo "🌐 INFORMATIONS D'ACCÈS"
                echo "========================================"
                echo "Adresse IP Minikube: $MINIKUBE_IP"
                echo "Port NodePort: $NODE_PORT"
                echo "URL Spring Boot: http://$MINIKUBE_IP:$NODE_PORT"
                echo "Health check: http://$MINIKUBE_IP:$NODE_PORT/actuator/health"
                echo "========================================"
                
                # Tester l'application
                echo "Test de connexion à l'application..."
                for i in {1..10}; do
                    echo "Tentative $i/10..."
                    if curl -s -f http://$MINIKUBE_IP:$NODE_PORT/actuator/health > /dev/null 2>&1; then
                        echo "✅ Application accessible!"
                        curl -s http://$MINIKUBE_IP:$NODE_PORT/actuator/health | head -5
                        break
                    else
                        echo "⏳ Application non encore prête..."
                        sleep 10
                    fi
                done
                
                # Afficher les logs finaux
                echo ""
                echo "=== LOGS FINAUX ==="
                echo "Pods:"
                kubectl get pods -o wide
                echo ""
                echo "Services:"
                kubectl get svc
                echo ""
                echo "Secrets:"
                kubectl get secrets
                echo ""
                echo "Volumes persistants:"
                kubectl get pv,pvc
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== RAPPORT FINAL ==="
            sh '''
            echo "Date: $(date)"
            echo ""
            echo "État final des pods:"
            kubectl get pods 2>/dev/null || echo "kubectl non disponible"
            echo ""
            echo "Services exposés:"
            kubectl get svc 2>/dev/null || echo "kubectl non disponible"
            echo ""
            MINIKUBE_IP=$(minikube ip 2>/dev/null || echo "Non disponible")
            echo "IP Minikube: $MINIKUBE_IP"
            '''
        }
        
        success {
            echo '✅ PIPELINE RÉUSSI !'
            // J'ai retiré l'email qui causait l'erreur de syntaxe
            // Vous pouvez l'ajouter plus tard si nécessaire
        }
        
        failure {
            echo '❌ PIPELINE ÉCHOUÉ'
            sh '''
            echo "=== DÉPANNAGE ==="
            echo "1. Vérifiez Minikube:"
            minikube status 2>/dev/null || echo "Minikube non disponible"
            echo ""
            echo "2. Vérifiez les pods en erreur:"
            kubectl get pods 2>/dev/null | grep -v Running | grep -v Completed || echo "kubectl non disponible"
            echo ""
            echo "3. Événements récents:"
            kubectl get events --sort-by='.lastTimestamp' 2>/dev/null | tail -10 || echo "kubectl non disponible"
            echo ""
            echo "4. Logs MySQL (si disponible):"
            kubectl logs -l app=mysql --tail=20 2>/dev/null || echo "Pas de logs MySQL"
            echo ""
            echo "5. Logs Spring Boot (si disponible):"
            kubectl logs -l app=springboot-app --tail=20 2>/dev/null || echo "Pas de logs Spring Boot"
            '''
        }
    }
}
