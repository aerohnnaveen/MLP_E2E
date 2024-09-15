pipeline {
    agent {
        // Use Docker to ensure a consistent build environment
        docker {
            image 'python:3.8-slim-buster' // Or any other relevant base image
            args '-v $HOME/.cache/pip:/root/.cache/pip' // Cache pip dependencies between builds
        }
    }

    environment {
        VENV_DIR = '.venv'
    }

    options {
        // Time out after 1 hour to avoid hanging jobs
        timeout(time: 1, unit: 'HOURS')
        // Keep only the last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/aerohnnaveen/MLP_E2E.git'
            }
        }

        stage('Setup Python Environment') {
            steps {
                sh '''
                python -m venv ${VENV_DIR}
                . ${VENV_DIR}/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }

        stage('Lint') {
            steps {
                sh '''
                . ${VENV_DIR}/bin/activate
                pip install flake8
                flake8 src/ tests/
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                . ${VENV_DIR}/bin/activate
                pytest --junitxml=results.xml
                '''
            }
        }

        stage('Data Preprocessing') {
            steps {
                sh '''
                . ${VENV_DIR}/bin/activate
                python src/preprocess.py
                '''
            }
        }

        stage('Train Model') {
            steps {
                sh '''
                . ${VENV_DIR}/bin/activate
                python src/train.py --config config/train_config.yaml
                '''
            }
        }

        stage('Evaluate Model') {
            steps {
                sh '''
                . ${VENV_DIR}/bin/activate
                python src/evaluate.py --model models/trained_model.pkl --test-data data/test/
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build('mlp_e2e:latest', '.')
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        docker.image('mlp_e2e:latest').push("${env.BUILD_NUMBER}")
                    }
                }
            }
        }
    }

    post {
        always {
            // Archive test results and other artifacts
            archiveArtifacts artifacts: 'results.xml', allowEmptyArchive: true
            cleanWs()
        }

        success {
            echo 'Build succeeded!'
        }

        failure {
            echo 'Build failed!'
        }

        unstable {
            echo 'Build is unstable!'
        }
    }
}
