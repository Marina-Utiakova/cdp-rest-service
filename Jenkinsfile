// Jenkinsfile for rest-service-initial

pipeline {
    agent any
    tools { maven 'MAVEN3' }

    environment {
        NEXUS_HOSTPORT      = '34.239.104.64:8081'   // Nexus:port
        NEXUS_DOWNLOAD_CRED = 'nexus-user-pass'
        NEXUS_UPLOAD_CRED   = 'nexus-ci-creds'
        REPO_RELEASE        = 'vprofile-release'
        REPO_SNAPSHOT       = 'vprofile-snapshot'

        GROUP_ID            = 'com.example'
        ARTIFACT_ID         = 'rest-service-initial'
        BUCKET_NAME         = 'cdp-project-artifacts'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                          cat > settings.xml <<EOF
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
                          mvn -s settings.xml clean package -DskipTests
                        """
                    }
                }
            }
        }

        stage('Determine Version') {
            steps {
                dir('complete') {
                    script {
                        env.VERSION = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true
                        ).trim()
                        echo "Project version is ${env.VERSION}"
                    }
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                dir('complete') {
                    script {
                        def repo = env.VERSION.endsWith('-SNAPSHOT') ? env.REPO_SNAPSHOT : env.REPO_RELEASE

                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol:    'http',
                            nexusUrl:    env.NEXUS_HOSTPORT,
                            repository:  repo,
                            credentialsId: env.NEXUS_UPLOAD_CRED,
                            groupId:     env.GROUP_ID,
                            version:     env.VERSION,
                            artifacts: [[
                                artifactId: env.ARTIFACT_ID,
                                file:       "target/${env.ARTIFACT_ID}-${env.VERSION}.jar",
                                type:       'jar'
                            ]]
                        )
                    }
                }
            }
        }

        stage('Upload JAR to S3') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    dir('initial') {
                        sh """
                          aws s3 cp target/${ARTIFACT_ID}-${VERSION}.jar s3://${BUCKET_NAME}/${ARTIFACT_ID}-${VERSION}.jar
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Published ${GROUP_ID}:${ARTIFACT_ID}:${VERSION} to Nexus and uploaded to S3"
        }
        failure {
            echo "❌ Build, deploy or upload failed"
        }
    }
}



