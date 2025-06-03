// Jenkinsfile для rest-service-initial (с учётом, что pom.xml лежит в подпапке initial)

pipeline {
    agent any
    tools { maven 'MAVEN3' }

    environment {
        NEXUS_HOSTPORT      = '34.239.104.64:8081'   // замените на ваш Nexus:port
        NEXUS_DOWNLOAD_CRED = 'nexus-user-pass'      // ID Jenkins-учётки для mirror
        NEXUS_UPLOAD_CRED   = 'nexus-ci-creds'       // ID Jenkins-учётки для выкладки
        REPO_RELEASE        = 'vprofile-release'     // имя release-репо в Nexus
        REPO_SNAPSHOT       = 'vprofile-snapshot'    // имя snapshot-репо в Nexus

        GROUP_ID            = 'com.example'
        ARTIFACT_ID         = 'rest-service-initial'
    }

    stages {
        stage('Checkout') {
            steps {
                // Скачиваем весь репозиторий
                checkout scm
            }
        }

        stage('Build JAR') {
            steps {
                // Переходим в папку initial, где лежит pom.xml
                dir('initial') {
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
                dir('initial') {
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
                dir('initial') {
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
    }

    post {
        success {
            echo "✅ Published ${GROUP_ID}:${ARTIFACT_ID}:${VERSION} to Nexus"
        }
        failure {
            echo "❌ Build or deploy failed"
        }
    }
}

