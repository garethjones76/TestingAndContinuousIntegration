pipeline {
    agent { label 'master' }
    // pin the pipeline to the master until we either create custom AMI images with Git and Java installed or use Docker
    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
        disableConcurrentBuilds()
    }

    parameters {
        string(name: 'Profile', defaultValue: 'jenkins', description: 'Maven profile as defined in settings.xml')

        booleanParam(name: 'RunAcceptanceTests', defaultValue: true, description: 'Whether to run the Acceptance Tests')

        booleanParam(name: 'RunIntegrationTests', defaultValue: true, description: 'Whether to run the Integration Tests')
    }
    tools {
        maven 'maven_3_5_0'
        jdk 'jdk8'
    }
    environment {
        GIT_COMMIT_SHORT = "${GIT_COMMIT}".substring(0,7)
        RELEASE_VERSION = "0.24.0.${GIT_COMMIT_SHORT}"
    }
    stages {

        stage('Initialize') {
            steps {
                echo "PATH = ${PATH}"
                echo "workspace directory is ${WORKSPACE}"
                echo "Release Version is ${RELEASE_VERSION}"
            }
        }

        stage('Maven Build') {
            steps {
                sh "mvn -P${params.Profile} -DskipTests -s ${WORKSPACE}/build/apache-maven/conf/settings.xml clean install -Drevision=${RELEASE_VERSION}"
            }
            post {
                failure {
                    emailext body: "<b>Foundation Engine Build FAILED</b><br><br>Maven build for project ${env.JOB_NAME} has failed because of you.<br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br>Link to Build: ${env.BUILD_URL}<br>Changesets: ${currentBuild.changeSets}",
                             recipientProviders: [[$class: 'CulpritsRecipientProvider']],
                             replyTo: 'jenkins@smogdev.co.uk',
                             from: 'jenkins@smogdev.co.uk',
                             mimeType: 'text/html',
                             subject: "Blame ${env.JOB_NAME} Acceptance Test FAILED"
                }
            }
        }

        stage('Unit Test') {
            steps {
                echo 'Starting unit tests '
                #sh 'mvn --no-snapshot-updates -pl :foundation-engine-device-data -s ${WORKSPACE}/build/apache-maven/conf/settings.xml org.jacoco:jacoco-maven-plugin:prepare-agent surefire:test -Drevision=${RELEASE_VERSION}'
            }
            post {
                failure {
                    emailext body: "<b>Foundation Engine Test FAILED</b><br><br>Unit tests for project ${env.JOB_NAME} are now failing because of you.<br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br>Link to Build: ${env.BUILD_URL}<br>Changesets: ${currentBuild.changeSets}",
                             recipientProviders: [[$class: 'CulpritsRecipientProvider']],
                             replyTo: 'gareth_jones76@hotmail.com',
                             from: 'gareth_jones76@hotmail.com',
                             mimeType: 'text/html',
                             subject: "Blame ${env.JOB_NAME} Unit Test FAILED"
                }
            }
        }

        stage('Integration Test') {
                    when {
                        expression { params.RunIntegrationTests == true }
                    }
                    steps {
                        echo 'Starting Integration tests '
                        £sh 'mvn --no-snapshot-updates -pl :foundation-engine-mpm-test -s ${WORKSPACE}/build/apache-maven/conf/settings.xml org.jacoco:jacoco-maven-plugin:prepare-agent surefire:test -Drevision=${RELEASE_VERSION}'
                    }
                    post {
                        failure {
                            emailext body: "<b>Foundation Engine Test FAILED</b><br><br>Integration tests for project ${env.JOB_NAME} are now failing because of you.<br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br>Link to Build: ${env.BUILD_URL}<br>Changesets: ${currentBuild.changeSets}",
                                     recipientProviders: [[$class: 'CulpritsRecipientProvider']],
                                     replyTo: 'gareth_jones76@hotmail.com',
                                     from: 'gareth_jones76@hotmail.com',
                                     mimeType: 'text/html',
                                     subject: "Blame ${env.JOB_NAME} Integration Test FAILED"
                        }
                    }
                }

        stage('Acceptance Test') {
            when {
                expression { params.RunAcceptanceTests == true }
            }
            steps {
                echo 'Starting Acceptance tests '
                sh 'mvn --no-snapshot-updates -pl :foundation-engine-acceptance-test -s ${WORKSPACE}/build/apache-maven/conf/settings.xml org.jacoco:jacoco-maven-plugin:prepare-agent surefire:test -Drevision=${RELEASE_VERSION}'
            }
            post {
                failure {
                    emailext body: "<b>Foundation Engine Test FAILED</b><br><br>Acceptance tests for project ${env.JOB_NAME} are now failing because of you.<br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br>Link to Build: ${env.BUILD_URL}<br>Changesets: ${currentBuild.changeSets}",
                             recipientProviders: [[$class: 'CulpritsRecipientProvider']],
                             replyTo: 'gareth_jones76@hotmail.com',
                             from: 'gareth_jones76@hotmail.com',
                             mimeType: 'text/html',
                             subject: "Blame ${env.JOB_NAME} Acceptance Test FAILED"
                }
            }
        }

        stage('Analyse with Sonar') {
            steps {
                echo 'Analysing with SonarQube..'
                withSonarQubeEnv('smogdev-sonar') {
                    #sh 'mvn -s ${WORKSPACE}/build/apache-maven/conf/settings.xml org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -Drevision=${RELEASE_VERSION}'
                }
            }
            post {
                failure {
                    emailext body: "<b>Foundation Engine Sonar Analysis FAILED</b><br><br>Sonar analysis for project ${env.JOB_NAME} is now failing because of you.<br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br>Link to Build: ${env.BUILD_URL}<br>Changesets: ${currentBuild.changeSets}",
                             recipientProviders: [[$class: 'CulpritsRecipientProvider']],
                             replyTo: 'gareth_jones76@hotmail.com',
                             from: 'gareth_jones76@hotmail.com',
                             mimeType: 'text/html',
                             subject: "${env.JOB_NAME} Analysis FAILED"
                }
            }
        }

        stage('Initiate Image Bake') {
            steps {
                echo 'Pushing Application Bundles to the AWS AMI Factory to initiate AMI image bakes of each node type.'
                withAWS(credentials:  'AWS-Jerry-Test', region: 'eu-west-2') {
                    #s3Upload(bucket: 'cgi-london-amibuild', workingDir: 'java/foundation-engine-distribution/target', file: 'foundation-engine', path: 'foundation-engine/')
                }
            }
            post {
                failure {
                    emailext body: "<b>Foundation Engine AWS-Bundle-Push Step FAILED</b><br><br>Fix the build pipeline<br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br>Link to Build: ${env.BUILD_URL}<br>Changesets: ${currentBuild.changeSets}",
                             recipientProviders: [[$class: 'CulpritsRecipientProvider']],
                             replyTo: 'gareth_jones76@hotmail.com',
                             from: 'gareth_jones76@hotmail.com',
                             mimeType: 'text/html',
                             subject: "${env.JOB_NAME} Pre-AWS-Deploy Steps FAILED"
                }
            }
        }
    }
    post {
        always {
            // Need to sort this out as it isn't descending far enough into the module hierarchy
            archive '**/**/**/target/**/*'
            junit '**/**/**/target/surefire-reports/*.xml'
        }
        success {
            echo "Link to Build: ${env.BUILD_URL}"
            echo "Tagging successful build with version: ${env.RELEASE_VERSION}"
            //sh "${env.WORKSPACE}/java/foundation-engine-distribution/tag-release.sh"
            emailext body: "<b>Foundation Engine Build Success</b><br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Link to Build: ${env.BUILD_URL}",
                     mimeType: 'text/html',
                     replyTo: 'gareth_jones76@hotmail.com',
                     from: 'gareth_jones76@hotmail.com',
                     subject: "${env.JOB_NAME} Pipeline Success",
                     to: 'build@smets1gateway.define.cgi.com'
        }
        failure {
            emailext body: "<b>Foundation Engine Pipeline Failed</b><br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Link to Build: ${env.BUILD_URL}",
                     mimeType: 'text/html',
                     replyTo: 'gareth_jones76@hotmail.com',
                     from: 'gareth_jones76@hotmail.com',
                     subject: "${env.JOB_NAME} Pipeline Failed",
                     to: 'build@smets1gateway.define.cgi.com'
        }
        unstable {
            emailext body: "<b>Foundation Engine Pipeline Unstable</b><br><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Link to Build: ${env.BUILD_URL}",
                     mimeType: 'text/html',
                     replyTo: 'gareth_jones76@hotmail.com',
                     from: 'gareth_jones76@hotmail.com',
                     subject: "${env.JOB_NAME} Pipeline Unstable",
                     to: 'build@smets1gateway.define.cgi.com'
        }
    }
}
