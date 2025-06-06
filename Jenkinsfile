pipeline {
    agent any
    tools {
        maven 'MAVEN3'
    }

    environment {
        // Nexus
        NEXUS_HOSTPORT      = '54.211.27.221:8081'
        NEXUS_DOWNLOAD_CRED = 'nexus-user-pass'
        NEXUS_UPLOAD_CRED   = 'nexus-ci-creds'
        REPO_RELEASE        = 'vprofile-release'
        REPO_SNAPSHOT       = 'vprofile-snapshot'

        // AWS/S3
        BUCKET_NAME         = 'cdp-project-artifacts'
        AWS_CREDS_ID        = 'aws-creds'

        // SonarQube (Jenkins → Configure System → SonarQube servers and SonarScanner)
        SONAR_SERVER_ID     = 'MySonarQube'
        SONAR_SCANNER_TOOL  = 'sonar-scanner'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Extract Maven coordinates') {
            steps {
                dir('complete') {
                    script {
                        env.ARTIFACT_ID = sh(
                            script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout",
                            returnStdout: true
                        ).trim()

                        env.VERSION = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true
                        ).trim()

                        env.GROUP_ID = sh(
                            script: "mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout",
                            returnStdout: true
                        ).trim()

                        echo "Detected artifactId = ${env.ARTIFACT_ID}"
                        echo "Detected version    = ${env.VERSION}"
                        echo "Detected groupId    = ${env.GROUP_ID}"
                    }
                }
            }
        }

        // ---------- SonarQube Analysis ----------
        stage('Code Analysis with SonarQube') {
            when { expression { env.SONAR_SERVER_ID != null && env.SONAR_SCANNER_TOOL != null } }
            environment {
                scannerHome = tool "${SONAR_SCANNER_TOOL}"
            }
            steps {
                dir('complete') {
                    withSonarQubeEnv("${SONAR_SERVER_ID}") {
                        sh """
                          mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=${env.ARTIFACT_ID} \
                            -Dsonar.projectName=${env.ARTIFACT_ID} \
                            -Dsonar.projectVersion=${env.VERSION} \
                            -Dsonar.host.url=\\$SONAR_HOST_URL \
                            -Dsonar.login=\\$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            when { expression { env.SONAR_SERVER_ID != null } }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build JAR') {
            steps {
                dir('complete') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: env.NEXUS_DOWNLOAD_CRED,
                            usernameVariable: 'NEXUS_USR',
                            passwordVariable: 'NEXUS_PSW'
                        )
                    ]) {
                        sh """
                          cat > settings-build.xml <<EOF
<settings>
  <mirrors>
    <mirror>
      <id>nexus-public</id>
      <name>Nexus Public Mirror</name>
      <url>http://${NEXUS_HOSTPORT}/repository/maven-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>nexus-public</id>
      <username>\\\${NEXUS_USR}</username>
      <password>\\\${NEXUS_PSW}</password>
    </server>
  </servers>
</settings>
EOF
                          mvn -s settings-build.xml clean package -DskipTests
                        """
                    }
                }
            }
        }

        stage('Verify JAR in target') {
            steps {
                dir('complete/target') {
                    echo '=== Files in complete/target ==='
                    sh 'ls -lh'
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                dir('complete') {
                    script {
                        def isSnapshot = env.VERSION.endsWith('-SNAPSHOT')
                        def repoId     = isSnapshot ? env.REPO_SNAPSHOT : env.REPO_RELEASE
                        def repoUrl    = "http://${env.NEXUS_HOSTPORT}/repository/${repoId}/"

                        echo "Uploading ${env.ARTIFACT_ID}-${env.VERSION}.jar to Nexus repo '${repoId}' (URL: ${repoUrl})"

                        withCredentials([
                            usernamePassword(
                                credentialsId: env.NEXUS_UPLOAD_CRED,
                                usernameVariable: 'NEXUS_DEPLOY_USR',
                                passwordVariable: 'NEXUS_DEPLOY_PSW'
                            )
                        ]) {
                            sh """
                              cat > settings-deploy.xml <<EOF
<settings>
  <servers>
    <server>
      <id>${repoId}</id>
      <username>\\\${NEXUS_DEPLOY_USR}</username>
      <password>\\\${NEXUS_DEPLOY_PSW}</password>
    </server>
  </servers>
</settings>
EOF
                              mvn -s settings-deploy.xml deploy:deploy-file \\
                                -Durl=${repoUrl} \\
                                -DrepositoryId=${repoId} \\
                                -Dfile=target/${env.ARTIFACT_ID}-${env.VERSION}.jar \\
                                -DgroupId=${env.GROUP_ID} \\
                                -DartifactId=${env.ARTIFACT_ID} \\
                                -Dversion=${env.VERSION} \\
                                -Dpackaging=jar
                            """
                        }
                    }
                }
            }
        }

        stage('Upload JAR to S3') {
            when { expression { currentBuild.currentResult == 'SUCCESS' } }
            steps {
                dir('complete/target') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: env.AWS_CREDS_ID,
                            usernameVariable: 'AWS_ACCESS_KEY_ID',
                            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {
                        sh """
                          aws s3 cp ${env.ARTIFACT_ID}-${env.VERSION}.jar \\
                                    s3://${BUCKET_NAME}/${env.ARTIFACT_ID}-${env.VERSION}.jar \\
                                    --region us-east-1
                        """
                    }
                }
            }
        }

        // ---------- Deploy to Staging ----------
        stage('Deploy to Staging') {
            when { expression { currentBuild.currentResult == 'SUCCESS' } }
            steps {
                script {
                    def beAppName    = 'rest-service-initial'
                    def beEnvName    = 'rest-service-staging'
                    def versionLabel = "${env.ARTIFACT_ID}-${env.VERSION}-${env.BUILD_ID}"
                    def s3Key        = "${env.ARTIFACT_ID}-${env.VERSION}.jar"

                    withCredentials([
                        usernamePassword(
                            credentialsId: env.AWS_CREDS_ID,
                            usernameVariable: 'AWS_ACCESS_KEY_ID',
                            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {
                        sh """
                          aws elasticbeanstalk create-application-version \\
                            --application-name ${beAppName} \\
                            --version-label ${versionLabel} \\
                            --source-bundle S3Bucket=${BUCKET_NAME},S3Key=${s3Key} \\
                            --region us-east-1
                        """
                        sh """
                          aws elasticbeanstalk update-environment \\
                            --application-name ${beAppName} \\
                            --environment-name ${beEnvName} \\
                            --version-label ${versionLabel} \\
                            --region us-east-1
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Artifact ${env.ARTIFACT_ID}:${env.VERSION} successfully published to Nexus, uploaded to S3 и deployed to staging"
        }
        failure {
            echo "❌ Build, deploy, or upload finished with an error"
        }
    }
}


