@Library('jenkins-pipeline-shared-libraries')_

pipeline {
    agent {
        label 'kie-rhel7 && kie-mem24g && !master'
    }
    tools {
        maven 'kie-maven-3.6.0'
        jdk 'kie-jdk1.8'
    }
    options {
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')
        timeout(time: 181, unit: 'MINUTES')
    }
    stages {
        stage('Initialize') {
            steps {
                sh 'printenv'

            }
        }
        stage('Build kie-wb-distribution prod profile') {
            steps {
                script {
                    def SETTINGS_XML_ID = "prod-maven-settings"
                    println "Building kie-wb-distributions project"
                    treebuild.downstreamBuild(['kiegroup/kie-wb-distributions'], "${SETTINGS_XML_ID}", "clean install -Dproductized=true ", true)
                }
            }
        }
        stage('Build Production projects') {
            steps {
                script {
                    def SETTINGS_XML_ID = "prod-maven-settings"

                    // This is the map project, variable to store the version from this project
                    configFileProvider([configFile(fileId: "49737697-ebd6-4396-9c22-11f7714808eb", variable: 'PRODUCTION_PROJECT_LIST')]) {
                        println "Reading file ${PRODUCTION_PROJECT_LIST} jenkins file"
                        def projectCollection = readFile "${env.PRODUCTION_PROJECT_LIST}"
                        println "Project collection ${projectCollection}"
                        treebuild.downstreamBuild(projectCollection.readLines(), "${SETTINGS_XML_ID}", "clean deploy -Dproductized=true -DaltDeploymentRepository=local::default::file://${env.WORKSPACE}/deployDirectory")
                    }
                }
            }
        }
        stage('Upload Files to repository') {
            steps {
                script {
                    echo "[INFO] Start uploading ${env.WORKSPACE}/deployDirectory"
                    dir("${env.WORKSPACE}/deployDirectory") {
                        withCredentials([usernameColonPassword(credentialsId: "${env.NIGHTLY_DEPLOYMENT_CREDENTIAL}", variable: 'deploymentCredentials')]) {
                            sh "zip -r deploy ."
                            sh "curl --upload-file deploy.zip -u $deploymentCredentials -v ${RHBA_PROD_DEPLOYMENT_REPO_URL}"
                        }
                    }
                }
            }
        }
    }
    post {
        unstable {
            script {
                mailer.sendEmailFailure()
            }
        }
        failure {
            script {
                mailer.sendEmailFailure()
            }
        }
        always {
            echo 'Archiving logs...'
            archiveArtifacts excludes: '**/target/checkstyle.log', artifacts: '**/*.maven.log,**/target/*.log', fingerprint: false, defaultExcludes: true, caseSensitive: true, allowEmptyArchive: true
        }
        cleanup {
            cleanWs()
        }
    }
}
