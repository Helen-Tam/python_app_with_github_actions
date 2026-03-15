pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins-agent-pod.yaml'
            label 'jenkins-agent'
        }
    }

    environment {
        GITOPS_REPO = "https://github.com/Helen-Tam/gitops-weather-app.git"
        GITOPS_DIR  = "gitops-weather-app"
        GIT_CREDENTIALS = 'git-creds'

        DOCKER_REPO = "helentam93/weather"
        DOCKER_TEST_REPO = "helentam93/weather-test"
        IMAGE_TAG = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"  
        NAMESPACE = "jenkins"
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
                    when { anyOf { branch 'feature/*'; branch 'develop'; branch 'hotfix/*' } }
                    steps {
                        container('python-tools') {
                            sh '''
                                echo "Installing pylint..."
                                pip install --upgrade pip
                                pip install pylint

                                echo "Running pylint on app.py..."
                                SCORE=$(pylint app.py | awk '/rated at/ {print $7}' | cut -d'/' -f1)

                                echo "Pylint score: $SCORE"
                                python3 -c "import sys; sys.exit(1 if float('$SCORE') < 7.0 else 0)" 
                            '''
                        }
                    }
                }

                stage('Secret Scan (TruffleHog)') {
                    when { not { branch 'main' } }
                    steps {
                        container('python-tools') {
                            sh '''
                                python3 -m venv venv
                                . venv/bin/activate
                                pip install --upgrade pip
                                pip install truffleHog

                                echo "Running secret scan..."
                                trufflehog discover --repo_path . --json --max_depth 10 --fail
                            '''
                        }
                    }
                }
            }
        }

        stage('Pre-Build Dependency Scan (Trivy)') {
            steps {
                container('kaniko') {
                    sh '''
                        echo "Running the dependency file-system scan..."
                        trivy fs --exit-code 1 --severity CRITICAL .

                        echo "Scanning Dockerfile ..."
                        trivy config --exit-code 1 --severity CRITICAL Dockerfile
                    '''
                }
            }
        }

        stage('Build, Scan & Push Docker Image') {
            steps {
                container('kaniko') {
                    sh '''
                        # Determine repo based on branch
                        if [[ "$BRANCH_NAME" == "develop" || "$BRANCH_NAME" == feature/* ]]; then
                            IMAGE_REPO="${DOCKER_TEST_REPO}"
                        else
                            IMAGE_REPO="${DOCKER_REPO}"
                        fi

                        FULL_IMAGE="${IMAGE_REPO}:${IMAGE_TAG}"

                        echo "Building Docker image: $FULL_IMAGE"
                        /kaniko/executor \
                          --dockerfile=Dockerfile \
                          --context=$WORKSPACE \
                          --destination=$FULL_IMAGE \
                          --docker-config=/kaniko/.docker \
                          --digest-file=/home/jenkins/agent/image-digest.txt

                        echo "Running Trivy scan on pushed image..."
                        trivy image $FULL_IMAGE --exit-code 1 --severity CRITICAL
                    '''
                }
            }
        }


        stage('Smoke Test') {
            steps {
                container('git-ops') {
                    sh '''
                        if [[ "$BRANCH_NAME" == "develop" || "$BRANCH_NAME" == feature/* ]]; then
                            TEST_IMAGE="${DOCKER_TEST_REPO}:${IMAGE_TAG}"
                        else
                            TEST_IMAGE="${DOCKER_REPO}:${IMAGE_TAG}"
                        fi
                        # in case there are still some Pods running from previous pipeline
                        kubectl delete pod test-pod --namespace=$NAMESPACE --ignore-not-found=true

                        echo "Running smoke test on $TEST_IMAGE..."
                        kubectl run test-pod --image=$TEST_IMAGE --restart=Never --namespace=$NAMESPACE
                        kubectl wait --for=condition=Ready pod/test-pod --timeout=60s --namespace=$NAMESPACE

                        if kubectl exec -n $NAMESPACE test-pod -- curl -f http://localhost:8000; then
                            echo "Smoke test PASSED"
                        else
                            echo "Smoke test FAILED"
                            kubectl logs test-pod --namespace=$NAMESPACE || true
                            exit 1
                        fi

                        echo "Removing the test Pod..."
                        kubectl delete pod test-pod --namespace=$NAMESPACE
                    '''
                }
            }
        }

        stage('Sign the Image') {
            when { anyOf { branch 'main'; branch pattern: 'release/.*'; branch pattern: 'hotfix/.*' } }
            steps {
                container('kaniko') {
                    sh '''
                        # Sign only for production/release branches
                        IMAGE_DIGEST=$(cat /home/jenkins/agent/image-digest.txt | cut -d@ -f2)
                        FULL_IMAGE="${DOCKER_REPO}:${IMAGE_TAG}"
                        echo "Image digest: $IMAGE_DIGEST"
                        echo "Signing image ${FULL_IMAGE} with Cosign..."
                        cosign sign --key /kaniko/.cosign/cosign.key ${FULL_IMAGE}@${IMAGE_DIGEST}

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
                        def gitBranch = ""
                        if (env.BRANCH_NAME == "develop") {
                            envName = "dev"
                            valuesFile = "values-dev.yaml"
                            gitBranch = "develop"
                        } else if (env.BRANCH_NAME.startsWith("release/")) {
                            envName = "staging"
                            valuesFile = "values-staging.yaml"
                            gitBranch = "main"    // release changes go to main branch in GitOps
                        } else if (env.BRANCH_NAME.startsWith("hotfix/")) {
                            envName = "hotfix-staging"
                            valuesFile = "values-hotfix.yaml"
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
                                git clone --depth 1 -b ${gitBranch} https://${GIT_USER}:${GIT_TOKEN}@github.com/Helen-Tam/gitops-weather-app.git ${GITOPS_DIR}
                                cd ${GITOPS_DIR}/weather-app

                                git config user.email "jenkins@ci.local"
                                git config user.name "Jenkins CI"

                                if [ "$envName" = "dev" ]; then
                                    yq e ".image.tag = \"${IMAGE_TAG}\" | .image.digest = \"\"" -i ${valuesFile}
                                else
                                    DIGEST=$(cat /home/jenkins/agent/image-digest.txt | cut -d@ -f2)
                                    yq e ".image.digest = \\"\$DIGEST\\" | .image.tag = \\"\\"" -i ${valuesFile}
                                fi

                                git add ${valuesFile}

                                if git diff --cached --quiet; then
                                    echo "No GitOps changes detected; skipping commit."
                                else
                                    git commit -m "ci: update ${envName} image to ${IMAGE_TAG} (build ${BUILD_NUMBER})"
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
                kubectl delete pod test-pod --namespace=$NAMESPACE || true
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


