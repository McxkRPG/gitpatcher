@Library('forge-shared-library')_

pipeline {
    agent {
        docker {
            image 'gradle:6.9.1-jdk8'
            args '-v gradlecache:/gradlecache'
        }
    }
    environment {
        GRADLE_ARGS = '-Dorg.gradle.daemon.idletimeout=5000'
        DISCORD_WEBHOOK = credentials('forge-discord-jenkins-webhook')
        DISCORD_PREFIX = "Job: GitPatcher Branch: ${BRANCH_NAME} Build: #${BUILD_NUMBER}"
        JENKINS_HEAD = 'https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png'
    }

    stages {
        stage('notify_start') {
            when {
                not {
                    changeRequest()
                }
            }
            steps {
                discordSend(
                    title: "${DISCORD_PREFIX} Started",
                    successful: true,
                    result: 'ABORTED', //White border
                    thumbnail: JENKINS_HEAD,
                    webhookURL: DISCORD_WEBHOOK
                )
            }
        }
        stage('buildandtest') {
            steps {
                withGradle {
                    sh './gradlew ${GRADLE_ARGS} --refresh-dependencies --continue build test'
                    gradleVersion(this)
                }
            }
            post {
                success {
                    writeChangelog(currentBuild, 'build/changelog.txt')
                    archiveArtifacts artifacts: 'build/changelog.txt', fingerprint: false
                }
            }
        }
        stage('publish') {
            when {
                not {
                    changeRequest()
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'maven-forge-user', usernameVariable: 'MAVEN_USER', passwordVariable: 'MAVEN_PASSWORD')]) {
                    withGradle {
                        sh './gradlew ${GRADLE_ARGS} publish'
                    }
                }
            }
            post {
                success {
                    build job: 'filegenerator', parameters: [string(name: 'COMMAND', value: "promote ${env.MYGROUP}:${env.MYARTIFACT} ${env.MYVERSION} latest")], propagate: false, wait: false
                }
            }
        }
    }
    post {
        always {
            script {
                if (env.CHANGE_ID == null) { // This is unset for non-PRs
                    discordSend(
                        title: "${DISCORD_PREFIX} Finished ${currentBuild.currentResult}",
                        description: '```\n' + getChanges(currentBuild) + '\n```',
                        successful: currentBuild.resultIsBetterOrEqualTo("SUCCESS"),
                        result: currentBuild.currentResult,
                        thumbnail: JENKINS_HEAD,
                        webhookURL: DISCORD_WEBHOOK
                    )
                }
            }
        }
    }
}
