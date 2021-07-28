def builderImage
def productionImage
def ACCOUNT_REGISTRY_PREFIX
def GIT_COMMIT_HASH

pipeline {
	agent any
	stages {
		stage('Checkout Source Code and Logging Into Registry') {
			steps {
				echo 'Logging Into the Private ECR Registry'
				script {
					GIT_COMMIT_HASH = sh (script: "git log -n 1 --pretty=format:'%H'", returnStdout: true)
					ACCOUNT_REGISTRY_PREFIX = "431681714777.dkr.ecr.ap-south-1.amazonaws.com"
					sh """
					\$(aws ecr get-login --no-include-email --region ap-south-1)
					"""
				}
			}
		}

		stage('Make A Builder Image') {
			steps {
				echo 'Starting to build the project builder docker image'
				script {
					builderImage = docker.build("${ACCOUNT_REGISTRY_PREFIX}/example_webapp_builder:9b58a263f9c7359b24de168d0660d4382ddebd37", "-f ./Dockerfile.builder .")
					builderImage.push()
					builderImage.push("${env.GIT_BRANCH}")
					builderImage.inside('-v $WORKSPACE:/output -u root') {
						sh """
						cd /output
						lein uberjar
						"""
					}
				}
			}
		}

		stage('Unit Tests') {
			steps {
				echo 'running unit tests in the builder image.'
				script {
					builderImage.inside('-v $WORKSPACE:/output -u root') {
					sh """
					cd /output
					lein test
					"""
					}
				}
			}
		}

		stage('Build Production Image') {
			steps {
				echo 'Starting to build docker image'
				script {
					productionImage = docker.build("${ACCOUNT_REGISTRY_PREFIX}/example_webapp:9b58a263f9c7359b24de168d0660d4382ddebd37")
					productionImage.push()
					productionImage.push("${env.GIT_BRANCH}")
				}
			}
		}

 
		stage('Deploy to Production fixed server') {
			when {
				branch 'release'
			}
			steps {
				echo 'Deploying release to production'
				script {
					productionImage.push("deploy")
					sh """
					aws ec2 reboot-instances --region ap-south-1 --instance-ids i-049e373668a5194ec
					"""
				}
			}
		}


		stage('Integration Tests') {
			when {
				branch 'master'
			}
			steps {
				echo 'Deploy to test environment and run integration tests'
				script {
					TEST_ALB_LISTENER_ARN="arn:aws:elasticloadbalancing:ap-south-1:431681714777:listener/app/testing-website-jenkins/0692fad27c727db2/6651df6316462db8"
					sh """
					./run-stack.sh example-webapp-test ${TEST_ALB_LISTENER_ARN}
					"""
				}
				echo 'Running tests on the integration test environment'
				script {
					sh """
					curl -v testing-website-jenkins-1879284367.ap-south-1.elb.amazonaws.com | grep '<title>Welcome to example-webapp</title>'
					if [ \$? -eq 0 ]
					then
						echo tests pass
					else
						echo tests failed
						exit 1
					fi
					"""
				}
			}
		}
	}
}
