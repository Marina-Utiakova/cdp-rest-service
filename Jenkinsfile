pipeline {
    agent any
    tools { maven 'MAVEN3' }

    environment {
        // Nexus
        NEXUS_HOSTPORT      = '52.90.92.46:8081'
        NEXUS_DOWNLOAD_CRED = 'nexus-user-pass'   // для чтения (build)
        NEXUS_UPLOAD_CRED   = 'nexus-ci-creds'     // для заливки (deploy)
        REPO_RELEASE        = 'vprofile-release'
        REPO_SNAPSHOT       = 'vprofile-snapshot'

        // AWS/S3
        BUCKET_NAME         = 'cdp-project-artifacts'
        AWS_CREDS_ID        = 'aws-creds'
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
                        // Считываем artifactId из pom.xml
                        env.ARTIFACT_ID = sh(
                            script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout",
                            returnStdout: true
                        ).trim()

                        // Считываем version из pom.xml
                        env.VERSION = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true
                        ).trim()

                        // Считываем groupId из pom.xml
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

        stage('Build JAR') {
            steps {
                dir('complete') {
                    withCredentials([usernamePassword(
                        credentialsId: env.NEXUS_DOWNLOAD_CRED,
                        usernameVariable: 'NEXUS_USR',
                        passwordVariable: 'NEXUS_PSW'
                    )]) {
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
      <username>\${NEXUS_USR}</username>
      <password>\${NEXUS_PSW}</password>
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
                        // Выбираем, куда заливать: snapshot или release
                        def isSnapshot = env.VERSION.endsWith('-SNAPSHOT')
                        def repoId     = isSnapshot ? env.REPO_SNAPSHOT : env.REPO_RELEASE
                        def repoUrl    = "http://${env.NEXUS_HOSTPORT}/repository/${repoId}/"

                        echo "Uploading ${env.ARTIFACT_ID}-${env.VERSION}.jar to Nexus repo '${repoId}' (URL: ${repoUrl})"

                        // Берём креденшиалы, чтобы записать их в новый settings-deploy.xml
                        withCredentials([usernamePassword(
                            credentialsId: env.NEXUS_UPLOAD_CRED,
                            usernameVariable: 'NEXUS_DEPLOY_USR',
                            passwordVariable: 'NEXUS_DEPLOY_PSW'
                        )]) {
                            sh """
                              cat > settings-deploy.xml <<EOF
<settings>
  <servers>
    <server>
      <id>${repoId}</id>
      <username>\${NEXUS_DEPLOY_USR}</username>
      <password>\${NEXUS_DEPLOY_PSW}</password>
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
            when {
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                dir('complete/target') {
                    withCredentials([usernamePassword(
                        credentialsId: env.AWS_CREDS_ID,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )]) {
                        sh """
                          aws s3 cp ${env.ARTIFACT_ID}-${env.VERSION}.jar \\
                                    s3://${BUCKET_NAME}/${env.ARTIFACT_ID}-${env.VERSION}.jar \\
                                    --region us-east-1
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Artifact ${env.ARTIFACT_ID}:${env.VERSION} успешно опубликован в Nexus и загружен в S3"
        }
        failure {
            echo "❌ Build, deploy или upload завершился с ошибкой"
        }
    }
}

