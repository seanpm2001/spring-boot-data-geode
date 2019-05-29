def projectProperties = [
	[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', numToKeepStr: '5']],
	pipelineTriggers([cron('@daily')])
]

properties(projectProperties)

def SUCCESS = hudson.model.Result.SUCCESS.toString()
currentBuild.result = SUCCESS

try {
	parallel check: {
		stage('Check') {
			timeout(time: 10, unit: 'MINUTES') {
				node {
					checkout scm
					try {
						withEnv(["JAVA_HOME=${tool 'jdk8'}"]) {
							sh './gradlew clean check --no-daemon --refresh-dependencies --stacktrace'
						}
					}
					catch (e) {
						currentBuild.result = 'FAILED: check'
						throw e
					}
					finally {
						junit '**/build/test-results/*/*.xml'
					}
				}
			}
		}
	}

	if (currentBuild.result == 'SUCCESS') {
		parallel artifacts: {
			stage('Deploy Artifacts') {
				node('linux') {
					checkout scm
					try {
						withCredentials([file(credentialsId: 'spring-signing-secring.gpg', variable: 'SIGNING_KEYRING_FILE')]) {
							withCredentials([string(credentialsId: 'spring-gpg-passphrase', variable: 'SIGNING_PASSWORD')]) {
								withCredentials([usernamePassword(credentialsId: 'oss-token', passwordVariable: 'OSSRH_PASSWORD', usernameVariable: 'OSSRH_USERNAME')]) {
									withCredentials([usernamePassword(credentialsId: '02bd1690-b54f-4c9f-819d-a77cb7a9822c', usernameVariable: 'ARTIFACTORY_USERNAME', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
										withEnv(["JAVA_HOME=${tool 'jdk8'}"]) {
											sh './gradlew deployArtifacts finalizeDeployArtifacts --no-daemon --refresh-dependencies --stacktrace -Psigning.secretKeyRingFile=$SIGNING_KEYRING_FILE -Psigning.keyId=$SPRING_SIGNING_KEYID -Psigning.password=$SIGNING_PASSWORD -PossrhUsername=$OSSRH_USERNAME -PossrhPassword=$OSSRH_PASSWORD -PartifactoryUsername=$ARTIFACTORY_USERNAME -PartifactoryPassword=$ARTIFACTORY_PASSWORD'
										}
									}
								}
							}
						}
					}
					catch (e) {
						currentBuild.result = 'FAILED: artifacts'
						throw e
					}
				}
			}
		},
		docs: {
			stage('Deploy Docs') {
				node('linux') {
					checkout scm
					try {
						withCredentials([file(credentialsId: 'docs.spring.io-jenkins_private_ssh_key', variable: 'DEPLOY_SSH_KEY')]) {
							withEnv(["JAVA_HOME=${tool 'jdk8'}"]) {
								sh './gradlew deployDocs --no-daemon --refresh-dependencies --stacktrace -PdeployDocsSshKeyPath=$DEPLOY_SSH_KEY -PdeployDocsSshUsername=$SPRING_DOCS_USERNAME'
							}
						}
					}
					catch (e) {
						currentBuild.result = 'FAILED: docs'
						throw e
					}
				}
			}
		}
	}
}
finally {

	def buildStatus = currentBuild.result
	def buildNotSuccess = !SUCCESS.equals(buildStatus)
	def lastBuildNotSuccess = !SUCCESS.equals(currentBuild.previousBuild?.result)

	if (buildNotSuccess || lastBuildNotSuccess) {
		stage('Notify') {
			node {

				final def RECIPIENTS = [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]

				def subject = "${buildStatus}: Build ${env.JOB_NAME} ${env.BUILD_NUMBER} status is now ${buildStatus}"
				def details = "The build status changed to ${buildStatus}. For details see ${env.BUILD_URL}"

				emailext(
						subject: subject,
						body: details,
						recipientProviders: RECIPIENTS,
						to: "$GEODE_TEAM_EMAILS"
				)
			}
		}
	}
}
