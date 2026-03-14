pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins
  labels:
    app: jenkins-agent
spec:
  serviceAccountName: jenkins-agent
  containers:
  - name: jnlp
    image: jenkins/inbound-agent
    args: ['$(JENKINS_SECRET)', '$(JENKINS_NAME)']
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent

  - name: python-tools
    image: python:3.11-slim
    command: ["cat"]
    tty: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent

  - name: kaniko
    image: helentam93/jenkins-agents:kaniko-trivy-cosign    
    command: ["cat"]
    tty: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
      - name: kaniko-secret
        mountPath: /kaniko/.docker
      - name: cosign-key
        mountPath: /kaniko/.cosign

  - name: git-ops
    image: alpine/k8s:1.32.12 # Includes kubectl, yq, and git
    command: ["cat"]
    tty: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent

  volumes:
  - name: workspace-volume
    emptyDir: {}
  - name: kaniko-secret
    secret:
      secretName: dockerhub-creds
  - name: cosign-key
    secret:
      secretName: cosign-key
'''
        }
    }

    environment {
        GITOPS_REPO = "https://github.com/Helen-Tam/gitops-weather-app.git"
        GITOPS_DIR  = "gitops-weather-app"
        GIT_CREDENTIALS = 'git-creds'

        DOCKER_REPO = "helentam93/weather"
        IMAGE_TAG   = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        DOCKER_IMAGE = "${DOCKER_REPO}:${IMAGE_TAG}"
    }

    stages {

        stage('Clone App Repo') {
            steps {
                container('jnlp') {
                    echo "Cloning application repository..."
                    checkout scm
                }
            }
        }


        stage('Security & Linting') {
            parallel {

                stage('Static Code Analysis') {
                    when { 
                        anyOf { branch 'feature/*'; branch 'develop'; branch 'hotfix/*' } 
                    }
                    steps {
                        container('python-tools') {
                            sh '''
                                echo "Installing pylint..."
                                pip install --upgrade pip
                                pip install pylint

                                echo "Running pylint on app.py..."
                                SCORE=$(pylint app.py | awk '/rated at/ {print $7}' | cut -d'/' -f1)

                                echo "Pylint score: $SCORE"
                                (( $(echo "$SCORE < 7.0" | bc -l) )) && exit 1 || echo "Score OK"
                            '''
                        kubectl delete pod test-pod --namespace=jenkins-agent
                        }
                    }
                }

                stage('Secret Scan (TruffleHog)') {
                    when { 
                        not { branch 'main' } 
                    }
                    steps {
                        container('python-tools') {
                            sh '''
                                python3 -m venv venv
                                . venv/bin/activate
                                pip install --upgrade pip
                                pip install truffleHog

                                echo "Running secret scan..."
                                trufflehog discover --repo_path . --json --max_depth 10
                            '''
                        }
                    }
                }
            }
        }

        stage('Pre-Build Dependency Scan (Trivy)') {
            when {
                not { branch 'main' }
            }
            steps {
                container('kaniko') {
                    sh '''
                        echo "Running the dependency file-system scan..."
                        trivy fs --exit-code 1 --severity CRITICAL requirements.txt

                        echo "Scanning Dockerfile ..."
                        trivy config --exit-code 1 --severity CRITICAL Dockerfile
                    '''
                }
            }
        }

        stage('Build and Test Docker Image (Export the image as archive, do not push it yet)') {
            steps {
                container('kaniko') {
                    sh '''
                        echo "Building Docker image with Kaniko..."
                        /kaniko/executor \
                        kubectl delete pod test-pod --namespace=jenkins-agent
                          --dockerfile=Dockerfile \
                          --context=$WORKSPACE \
                          --tarPath=/home/jenkins/agent/app-image.tar

                        echo "Scanning Built Image for OS Vulnerabilities..."
                        trivy image --input /home/jenkins/agent/app-image.tar --exit-code 1 --severity CRITICAL $DOCKER_IMAGE
                    '''🔍
Thursday	7 am–7 pm
Friday	7 am–3 pm
Saturday	Close
                }
            }
        }

        stage('Smoke test') {
            steps {
                container('git-ops') {
                    sh '''
                        echo "Creating test Pod..."
                        kubectl run test-pod --image=$DOCKER_IMAGE --restart=Never --namespace=jenkins-agents
                        kubectl wait --for=condition=Ready pod/test-pod --timeout=30s --namespace=jenkins-agents

                        echo "Testing application reachability..."
                        if kubectl exec -n jenkins-agents test-pod -- curl -f http://localhost:8000; then
                            echo "Reachability test PASSED"
                        else🔍
                             echo "Reachability test FAILED"
                             kubectl logs test-pod --namespace=jenkins-agents || true
                             exit 1
                        fi

                        echo "Removing the test Pod..."
                        kubectl delete pod test-pod --namespace=jenkins-agents
                    '''
                }
            }
        }

        stage('Push and Sign the Docker Image') {
            steps {
                container('kaniko') {
                    sh '''
                        echo "Building Docker image with Kaniko..."
                        /kaniko/executor \
                          --dockerfile=Dockerfile \
                          --context=$WORKSPACE \
                          --destination=$DOCKER_IMAGE \
                          --docker-config=/kaniko/.docker \
                          --digest-file=/home/jenkins/agent/image-digest.txt

                        # Sign image only for main and release/*
                        if [[ "$BRANCH_NAME" == "main" || "$BRANCH_NAME" =~ ^release/ ]]; then
                            echo "Identifying Image Digest..."
                            IMAGE_DIGEST=$(cat /home/jenkins/agent/image-digest.txt)
                            echo "Image digest: $IMAGE_DIGEST"
                            echo "Signing image with Cosign..."

                            cosign sign --key /kaniko/.cosign/cosign.key $IMAGE_DIGEST
                        else
                            echo "Skipping image signing for branch $BRANCH_NAME"
                        fi                        
                    '''
                }
            }
        }

        stage('Hotfix Verification Gate') {
            when { 
                branch pattern: "hotfix/.*", comparator: "REGEXP" 
            }
            steps {
                // Slack Notify the team that an emergency fix is waiting
                slackSend(
                    channel: 'devops-alerts',
                    color: 'warning',
                    message: "🚨 *HOTFIX PENDING:* Branch `${env.BRANCH_NAME}` is ready for verification.\nReview the build here: <${env.BUILD_URL}|View Jenkins Job> and click 'Proceed' to deploy to Staging."
                )
                
                input message: "Deploy this hotfix to STAGING for verification?", ok: "Deploy to Staging"
            }
        }

        stage('Update Helm Chart in GitOps Repo') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                    branch pattern: 'hotfix/.*', comparator: 'REGEXP'
                }
            }
            steps {
                container('git-ops') {
                    script {
                        def envName = ""
                        def valuesFile = ""
                        if (env.BRANCH_NAME == "develop") {
                            envName = "dev"
                            valuesFile = "values-dev.yaml"
                            gitBranch = "develop"
                        } else if (env.BRANCH_NAME.startsWith("release/")) {
                            envName = "staging"
                            valuesFile = "values-staging.yaml"
                            gitBranch = "main"    // release changes go to main branch in GitOps
                        } else if (env.BRANCH_NAME.startsWith("hotfix")) {
                            envName = "hotfix-staging"
                            valuesFile = "values-staging.yaml"
                            gitBranch = "main"    // hotfix changes go to main branch in GitOps
                        } else if (env.BRANCH_NAME == "main") {
                            envName = "prod"
                            valuesFile = "values-prod.yaml"
                            gitBranch = "main"
                        } else {
                            echo "Branch ${env.BRANCH_NAME} does not deploy to GitOps"
                            return 
                        }
                        
                        echo "Targeting Environment: ${envName} (using ${valuesFile})"

                        withCredentials([usernamePassword(
                            credentialsId: "${GIT_CREDENTIALS}",
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )]) {
                            sh """
                                rm -rf ${GITOPS_DIR}
                                git clone -b ${gitBranch} https://${GIT_USER}:${GIT_TOKEN}@github.com/Helen-Tam/gitops-weather-app.git ${GITOPS_DIR}
                                cd ${GITOPS_DIR}/weather-app

                                git config user.email "jenkins@ci.local"
                                git config user.name "Jenkins CI"

                                if [ "$envName" = "dev" ]; then
                                    yq e '.image.tag = "${IMAGE_TAG}" | .image.digest = ""' -i ${valuesFile}
                                else
                                    DIGEST=\$(cat \$WORKSPACE/image-digest.txt | cut -d@ -f2)
                                    yq e '.image.digest = "'$DIGEST'" | .image.tag = ""' -i ${valuesFile}
                                fi

                                git add ${valuesFile}

                                if git diff --cached --quiet; then
                                    echo "No GitOps changes detected; skipping commit."
                                else
                                    git commit -m "Update ${envName} image to ${IMAGE_TAG}"
                                    git push origin ${gitBranch}
                                fi
                            """
                        }
                    }
                }
            }
        }
    }


    post {
        always {
           container('git-ops') {
              sh """
                echo "Cleaning up ephemeral test pod if it exists..."
                kubectl delete pod test-pod --namespace=jenkins-agents || true
              """
           }
        }
        success {
            echo "Pipeline completed successfully!"
            slackSend(
                channel: 'succeeded-build',
                color: 'good',
                message: "Pipeline *${env.JOB_NAME}* #${env.BUILD_NUMBER} succeeded!\n<${env.BUILD_URL}|View Build>"
            )
        }
        failure {
            echo "Pipeline failed. Check the logs."
            slackSend(
                channel: 'devops-alerts',
                color: 'danger',
                message: "Pipeline *${env.JOB_NAME}* #${env.BUILD_NUMBER} failed!\n<${env.BUILD_URL}|View Build>"
            )
        }
    }
}


