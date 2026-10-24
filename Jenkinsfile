pipeline {
    agent { label 'k8s-master' }

    parameters {
        booleanParam(name: 'FORCE_RELEASE', defaultValue: false,
                     description: 'Deploy to production even if today is not the 25th')
    }

    triggers {
        cron('H 6 25 * *')
    }

    environment {
        DOCKERHUB = credentials('dockerhub-creds')
    }

    stages {
        stage('Build image') {
            steps {
                script {
                    env.TAG = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.IMAGE = "${DOCKERHUB_USR}/website"
                }
                sh 'docker build -t $IMAGE:$TAG -t $IMAGE:latest .'
            }
        }

        stage('Push image') {
            when { branch 'master' }
            steps {
                sh 'echo $DOCKERHUB_PSW | docker login -u $DOCKERHUB_USR --password-stdin'
                sh 'docker push $IMAGE:$TAG'
                sh 'docker push $IMAGE:latest'
            }
        }

        stage('Deploy to production') {
            when {
                allOf {
                    branch 'master'
                    expression {
                        def day = sh(returnStdout: true, script: 'date +%d').trim()
                        return day == '25' || params.FORCE_RELEASE
                    }
                }
            }
            steps {
                sh "sed 's|IMAGE_PLACEHOLDER|${IMAGE}:${TAG}|' k8s/deployment.yaml | kubectl apply -f -"
                sh 'kubectl rollout status deployment/website --timeout=120s'
            }
        }
    }

    post {
        always { sh 'docker logout || true' }
    }
}
