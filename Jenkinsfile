pipeline {
    agent any

    tools {
        maven 'Maven'
        jdk 'JDK25'
    }

    environment {
        MAVEN_OPTS = '-Dmaven.test.failure.ignore=false'
        SONARQUBE_ENV = 'SonarQube' // SonarQube server name configured in Jenkins
        DOCKERHUB_NAMESPACE = 'luongtrz'
    }

    triggers {
        // GitHub webhooks trigger immediately when reachable. Polling is the
        // reliable fallback for this on-premise Jenkins behind WSL/NAT.
        githubPush()
        pollSCM('H/5 * * * *')
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        // ============================================================
        // PHASE 1: DETECT CHANGES (monorepo - only build what changed)
        // ============================================================
        stage('Detect Changes') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()
                    env.SOURCE_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    // Services that have unit tests and can be built
                    def allServices = [
                        'media', 'product', 'cart', 'order', 'customer',
                        'inventory', 'location', 'payment', 'payment-paypal',
                        'promotion', 'rating', 'search', 'tax', 'webhook',
                        'recommendation'
                    ]

                    // Detect which services have changed files
                    changedServices = []
                    commonChanged = false

                    if (env.CHANGE_TARGET) {
                        // PR build: compare against target branch
                        sh "git fetch origin +refs/heads/${env.CHANGE_TARGET}:refs/remotes/origin/${env.CHANGE_TARGET} --no-tags"
                        def changes = sh(
                            script: "git diff --name-only origin/${env.CHANGE_TARGET}...HEAD",
                            returnStdout: true
                        ).trim()

                        if (changes.contains('common-library/') || changes.contains('pom.xml')) {
                            commonChanged = true
                        }

                        for (svc in allServices) {
                            if (changes.contains("${svc}/") || commonChanged) {
                                changedServices.add(svc)
                            }
                        }
                    } else {
                        // Branch build: compare against previous commit
                        def changes = sh(
                            script: "git diff --name-only HEAD~1 HEAD || echo ''",
                            returnStdout: true
                        ).trim()

                        if (changes.contains('common-library/') || changes.contains('pom.xml')) {
                            commonChanged = true
                        }

                        for (svc in allServices) {
                            if (changes.contains("${svc}/") || commonChanged) {
                                changedServices.add(svc)
                            }
                        }
                    }

                    if (changedServices.isEmpty()) {
                        echo "No service changes detected. Skipping build."
                    } else {
                        echo "Changed services: ${changedServices.join(', ')}"
                        echo "Immutable image tag: ${env.IMAGE_TAG}"
                    }
                }
            }
        }

        // ============================================================
        // PHASE 2: BUILD - Compile all changed services
        // ============================================================
        stage('Build') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            steps {
                script {
                    def modules = changedServices.join(',')
                    sh "./mvnw clean compile -pl ${modules} -am -DskipTests"
                }
            }
        }

        // ============================================================
        // PHASE 3: TEST - Run unit tests + generate coverage
        // ============================================================
        stage('Test') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            steps {
                script {
                    def modules = changedServices.join(',')
                    // Run tests + JaCoCo coverage report (verify phase triggers jacoco:report)
                    sh "./mvnw verify -pl ${modules} -am"
                }
            }
            post {
                always {
                    // Upload JUnit test results for each changed service
                    junit(
                        testResults: '**/target/surefire-reports/TEST-*.xml, **/target/failsafe-reports/TEST-*.xml',
                        allowEmptyResults: true
                    )

                    // Publish JaCoCo coverage report
                    jacoco(
                        execPattern: '**/target/jacoco.exec',
                        classPattern: '**/target/classes',
                        sourcePattern: '**/src/main/java',
                        exclusionPattern: '**/config/**,**/exception/**,**/constants/**,**/*Application.*',
                        minimumLineCoverage: '70',
                        minimumBranchCoverage: '50',
                        changeBuildStatus: true
                    )
                }
            }
        }

        // ============================================================
        // PHASE 4: IMAGE - tag every changed service by commit id
        // ============================================================
        stage('Build and Push Commit Images') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            steps {
                script {
                    sh 'mkdir -p work && : > work/ci-image-evidence.txt'
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'

                        for (svc in changedServices) {
                            def image = "${env.DOCKERHUB_NAMESPACE}/yas-${svc}:${env.IMAGE_TAG}"
                            sh """
                                docker build \
                                  --label org.opencontainers.image.revision=${env.SOURCE_COMMIT} \
                                  -t ${image} \
                                  ${svc}
                                docker push ${image}
                                printf '%s commit=%s\n' '${image}' '${env.SOURCE_COMMIT}' \
                                  >> work/ci-image-evidence.txt
                            """
                        }
                    }
                }
            }
            post {
                always {
                    sh 'docker logout || true'
                    archiveArtifacts(
                        artifacts: 'work/ci-image-evidence.txt',
                        allowEmptyArchive: false,
                        fingerprint: true
                    )
                }
            }
        }

        // ============================================================
        // PHASE 5: CODE QUALITY - Checkstyle + SonarQube
        // ============================================================
        stage('Code Quality') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            parallel {
                stage('Checkstyle') {
                    steps {
                        script {
                            def modules = changedServices.join(',')
                            sh "./mvnw checkstyle:checkstyle -pl ${modules} -am"
                        }
                    }
                    post {
                        always {
                            recordIssues(
                                tools: [checkStyle(pattern: '**/checkstyle-result.xml')],
                                qualityGates: [[threshold: 1, type: 'TOTAL_HIGH', unstable: true]]
                            )
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        script {
                            def modules = changedServices.join(',')
                            withSonarQubeEnv("${SONARQUBE_ENV}") {
                                sh "./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -pl ${modules} -am -Dsonar.organization=devop14s -Dsonar.projectKey=devop14s_yas"
                            }
                        }
                    }
                }
            }
        }

        // SonarQube Quality Gate - wait for result
        stage('Quality Gate') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ============================================================
        // PHASE 6: SECURITY SCAN - Gitleaks + Snyk
        // ============================================================
        stage('Security Scan') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            parallel {
                stage('Gitleaks') {
                    steps {
                        sh '''
                            echo "Downloading Gitleaks binary..."
                            curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.4/gitleaks_8.18.4_linux_x64.tar.gz \
                                -o /tmp/gitleaks.tar.gz
                            tar -xzf /tmp/gitleaks.tar.gz -C /tmp gitleaks
                            chmod +x /tmp/gitleaks

                            echo "Running Gitleaks scan..."
                            /tmp/gitleaks detect \
                                --source="." \
                                --config=gitleaks.toml \
                                --report-format=json \
                                --report-path=gitleaks-report.json \
                                --verbose || true

                            echo "Gitleaks scan complete."
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts(
                                artifacts: 'gitleaks-report.json',
                                allowEmptyArchive: true
                            )
                        }
                    }
                }

                stage('Snyk Security Scan') {
                    steps {
                        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                            sh '''
                                echo "Downloading Snyk CLI..."
                                curl -sSfL https://github.com/snyk/cli/releases/latest/download/snyk-linux \
                                    -o /tmp/snyk
                                chmod +x /tmp/snyk

                                echo "Authenticating Snyk..."
                                /tmp/snyk auth ${SNYK_TOKEN}

                                echo "Running Snyk test..."
                                /tmp/snyk test \
                                    --all-projects \
                                    --severity-threshold=high \
                                    --json-file-output=snyk-report.json || true

                                echo "Snyk scan complete."
                            '''
                        }
                    }
                    post {
                        always {
                            archiveArtifacts(
                                artifacts: 'snyk-report.json',
                                allowEmptyArchive: true
                            )
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }
        success {
            echo "Build successful! Changed services: ${changedServices?.join(', ') ?: 'none'}"
        }
        failure {
            echo 'Build failed. Please check the console output for errors.'
        }
        cleanup {
            cleanWs()
        }
    }
}
