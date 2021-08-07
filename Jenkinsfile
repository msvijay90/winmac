pipeline {
   agent none
    stages {
		stage("Build and deploy on Windows and MacOS") {
            parallel {
				stage("MacOS") {
					agent {
						label "macos-agent"
					}
					stage('Clone Sources') {
						steps {
							echo "Code Clone & Checkout"
							checkout([
								 $class: 'GitSCM',
								 branches: [[name: 'master']],
								 userRemoteConfigs: [[
									url: 'git@github.com:msvijay90/winmac.git',
									credentialsId: '',
								 ]]
							]}
					}

					stage("Code analysis (SonarQube)") {
           				steps {
             		 		withSonarQubeEnv('SonarQube Server') {
                				sh 'mvn clean package sonar:sonar'
              				}
           				}
					}

          			stage("Quality Gate") {
            			steps {
              				timeout(time: 1, unit: 'HOURS') {
               				waitForQualityGate abortPipeline: true
              				}
            			}
          			}

					stage('Build') {
						steps {
							rtServer (
								id: "artifactory-server-id", 
								url: "https://test.jfrog.io/artifactory",
								credentialsId: "testadmin.jfrog"
							)
							rtMavenResolver (
								id: 'maven-resolver',
								serverId: 'artifactory-server-id',
								releaseRepo: libs-releases,
								snapshotRepo: libs-snapshot
							)  
							 
							rtMavenDeployer (
								id: 'maven-deployer',
								serverId: 'artifactory-server-id',
								releaseRepo: libs-releases-local,
								snapshotRepo: libs-snapshot-local,
							)
							
							echo "Build Running ${env.BUILD_ID} ${env.BUILD_DISPLAY_NAME} on ${env.NODE_NAME} and JOB ${env.JOB_NAME}"
							
							rtMavenRun (
								tool: 'Maven 3.3.9',
								pom: 'demoApp/macos/pom.xml',
								goals: '-U clean install',
								deployerId: "maven-deployer",
								resolverId: "maven-resolver"
							)
							
							rtUpload (
								serverId: 'artifactory-server-id',
								spec: '''{
									  "files": [
										{
										  "pattern": "libs-releases-local/demoapp/macos/demoapp.pkg",
										  "target": "libs-releases-local"
										}
									  ]
								}'''
							)
							
							sh(label: 'Push docker image to ECR Repo', script:
							 '''
							 #!/bin/bash
							 
								eval $(aws ecr get-login --region "$AWS_REGION")
								
								cd ${WORKSPACE}/${PROJ_NAME}/backend
								
								docker build  -t 123456789101.dkr.ecr.us-east-1.amazonaws.com/backend-node-app:master .
								
								docker push 123456789101.dkr.ecr.eu-east-1.amazonaws.com/backend-node-app:master
								
								docker tag 123456789101.dkr.ecr.eu-east-1.amazonaws.com/backend-node-app:master 123456789101.dkr.ecr.eu-east-1.amazonaws.com/backend-node-app:latest
								
								docker push 123456789101.dkr.ecr.eu-east-1.amazonaws.com/backend-node-app:latest
								'''
							)
						}
					}

					stage('Test') {
						when {
							branch 'master'
						}
						steps {
							echo 'Testing...'
							steps {
								sh 'mvn test'
							}
							post {
								always {
									junit '**/mac/target/surefire-reports/*.xml'
								}
							}
						}
					}
					
					stage('Code Signing & Notarization') {
						when {
							branch 'master'
						}
						steps {
							//Reference https://github.com/drud/signing_tools
							echo 'Code Siging'
							sh './macos_sign.sh --signing-password="${SIGNING_TOOLS_SIGNING_PASSWORD}" --cert-file=${CERTFILE} --cert-name="${CERTNAME}" --target-binary="${TARGET_BINARY}"'
							
							echo 'Notarization'
							./macos_notarize.sh --app-specific-password=${APP_SPECIFIC_PASSWORD} --apple-id=${APPLE_ID} --primary-bundle-id=com.ddev.test-signing-tools --target-binary=${TARGET_BINARY}
						}
					}
					
					stage('Deploy') {
						when {
							branch 'master'
						}
						steps {
						   echo 'Deploying...'
						   
						   sh(label: 'Deploying backend app on Cloud', script:
							'''
								#!/bin/bash
								aws cloudformation create-stack \
								--stack-name demo-backend-app \
								--template-body /jenkins/demoapp/backend/demo-backend-app.json
							)

						   sh(label: 'Deploying application on customer system', script:
							'''
								#!/bin/bash
								ansible-playbook -i inventory ansible-playbook.yml
							)
							
						}
					}
					
				}
				
				stage("Windows") {
					agent {
						label "windows-agent"
					}

					stage('Clone Sources') {
						steps {
							echo "Code Checkout"
						}
					}

					stage("Security Code analysis (SonarQube)") {
           				steps {
             		 		echo "Static code analysis"
           				}
					}

          			stage("Quality Gate") {
            			steps {
              				timeout(time: 1, unit: 'HOURS') {
               				waitForQualityGate abortPipeline: true
              				}
            			}
          			}

					stage('Build') {
						steps {
							echo "Connect jfrog Artifactory"
							echo "Maven Build"
							echo "Upload artifacts to jfrog Artifactory repo"
							echo "Push docker image to ECR repo"
						}
					}
					
					stage('Test') {
						when {
							branch 'master'
						}
						steps {
							echo 'Testing...'
							steps {
								bat 'mvn test'
							}
							post {
								always {
									junit '**/windows/target/surefire-reports/*.xml'
								}
							}
						}
					}
					
					stage('CodeSigning') {
						when {
							branch 'master'
						}
						steps {
							echo 'CodeSigning'
							steps {
								echo 'Signing exe file'
								
								bat 'signtool sign /tr http://timestamp.digicert.com /td sha256 /fd sha256 /f "/path/to/signingCert.pfx" /p signingCertPassword "/path/to/demoapp.exe'
							}
							
						}
					}
					
					stage('Deploy') {
						when {
							branch 'master'
						}
						steps {
						   echo 'Deploy application on windows'
						   echo 'Deploy backend application on cloud (Linux)'
						}
					}
					
				}
				
			}
		}
	}
}