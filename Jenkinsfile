pipeline{
    agent any

    options {
        buildDiscarder(logRotator(artifactDaysToKeepStr: '1', artifactNumToKeepStr: '1', daysToKeepStr: '5', numToKeepStr: '50'))
        // Disable concurrent builds. It will wait until the pipeline finish before start a new one
        disableConcurrentBuilds()
    }

    
    tools {
        oc "OpenShiftv3.11.0"
    }
    

    environment {
        // sonarQube
        // Name of the sonarQube environment
        sonarEnv = "SharesSonar"

        // Nexus 3
        // Maven global settings configuration ID
        globalSettingsId = 'MavenSettings'
        // Maven tool id
        mavenInstallation = 'Maven3'

        
        // Docker
        dockerFileName = 'Dockerfile.ci'
        dockerRegistry = 'docker-registry-shared-services.pl.s2-eu.capgemini.com'
        dockerRegistryCredentials = 'nexus-nexusDeployer'
        
        

        
        // Openshift
        openshiftUrl = 'https://ocp.itaas.s2-eu.capgemini.com'
        openShiftCredentials = 'ocadmin'
        openShiftNamespace = 's2portaldev'
        
    }

    stages {
        stage ('Setup pipeline') {
            when {
               anyOf {
                   branch 'master'
                   branch 'develop'
                   branch 'release/*'
                   branch 'feature/*'
                   branch 'hotfix/*'
               }
            }
            steps {
                
                script {
                    def pom = readMavenPom file: './pom.xml';
                    version = pom.version

                    if (env.BRANCH_NAME.startsWith('release')) {
                        dockerTag = "release"
                        sonarProjectKey = "-release"
                        dockerEnvironment = "-uat"
                        
                        if (!version.endsWith("-RC")) {
                            version = "${version}-RC"
                        }
                    }

                    if (env.BRANCH_NAME == 'develop') {
                        dockerTag = "latest"
                        repositoryName = 'maven-snapshots'
                        dockerEnvironment = "-dev"

                        if (!version.endsWith("-SNAPSHOT")) {
                            version = "${version}-SNAPSHOT"
                        }
                    }

                    if (env.BRANCH_NAME == 'master') {
                        sonarProjectKey = ""
                        dockerTag = "production"
                        dockerEnvironment = "-prod"

                        if (env.BRANCH_NAME == 'master' && (version.endsWith("-RC") || version.endsWith("-SNAPSHOT"))){
                            version = version.replace("-RC", "")
                            version = version.replace("-SNAPSHOT", "")
                        }
                    }

                    // Put the correct version at pom.
                    pom.version = version
                    writeMavenPom model: pom, file: 'pom.xml'

                    def apiPom = readMavenPom file: 'api/pom.xml'
                    apiPom.parent.version = version
                    writeMavenPom model: apiPom, file: 'api/pom.xml'

                    def corePom = readMavenPom file: 'core/pom.xml'
                    corePom.parent.version = version
                    writeMavenPom model: corePom, file: 'core/pom.xml'

                    def serverPom = readMavenPom file: 'server/pom.xml'
                    serverPom.parent.version = version
                    writeMavenPom model: serverPom, file: 'server/pom.xml'
                }
            }
        }

        stage ('Fresh Dependency Installation') {
            when {
               anyOf {
                   branch 'master'
                   branch 'develop'
                   branch 'release/*'
                   branch 'feature/*'
                   branch 'hotfix/*'
                   changeRequest()
               }
            }
            steps {
                withMaven(globalMavenSettingsConfig: globalSettingsId, maven: mavenInstallation) {
                    sh "mvn clean install -Dmaven.test.skip=true"
                }
            }
        }

        stage ('Unit Tests') {
            when {
               anyOf {
                   branch 'master'
                   branch 'develop'
                   branch 'release/*'
                   branch 'feature/*'
                   branch 'hotfix/*'
                   changeRequest()
               }
            }
            steps {
                withMaven(globalMavenSettingsConfig: globalSettingsId, maven: mavenInstallation) {
                    sh "mvn clean test"
                }
            }
        }

        stage ('SonarQube code analysis') {
            when {
               anyOf {
                   branch 'master'
                   branch 'develop'
                   branch 'release/*'
               }
            }
            steps {
                script {
                    withMaven(globalMavenSettingsConfig: globalSettingsId, maven: mavenInstallation) {
                        withSonarQubeEnv(sonarEnv) {
                            // Change the project name (in order to simulate branches with the free version)
                            sh "cp pom.xml pom.xml.bak"
                            sh "cp api/pom.xml api/pom.xml.bak"
                            sh "cp core/pom.xml core/pom.xml.bak"
                            sh "cp server/pom.xml server/pom.xml.bak"

                            def pom = readMavenPom file: './pom.xml';
                            pom.artifactId = "${pom.artifactId}${sonarProjectKey}"
                            writeMavenPom model: pom, file: 'pom.xml'

                            def apiPom = readMavenPom file: 'api/pom.xml'
                            apiPom.parent.artifactId = pom.artifactId
                            apiPom.artifactId = "${pom.artifactId}-api"
                            writeMavenPom model: apiPom, file: 'api/pom.xml'

                            def corePom = readMavenPom file: 'core/pom.xml'
                            corePom.parent.artifactId = pom.artifactId
                            corePom.artifactId = "${pom.artifactId}-core"
                            writeMavenPom model: corePom, file: 'core/pom.xml'

                            def serverPom = readMavenPom file: 'server/pom.xml'
                            serverPom.parent.artifactId = pom.artifactId
                            serverPom.artifactId = "${pom.artifactId}-server"
                            writeMavenPom model: serverPom, file: 'server/pom.xml'

                            sh "mvn sonar:sonar"

                            sh "mv pom.xml.bak pom.xml"
                            sh "mv api/pom.xml.bak api/pom.xml"
                            sh "mv core/pom.xml.bak core/pom.xml"
                            sh "mv server/pom.xml.bak server/pom.xml"
                        }
                    }
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate() 
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        
        stage ('Deliver application into Nexus') {
            when {
               anyOf {
                   branch 'master'
                   branch 'develop'
                   branch 'release/*'
               }
            }
            steps {
                withMaven(globalMavenSettingsConfig: globalSettingsId, maven: mavenInstallation) {
                    sh "mvn deploy -Dmaven.test.skip=true"
                    sh """
                        cp ${dockerFileName} server/target/Dockerfile
                    """
                }
            }
        }

        

        
        stage ('Create the Docker image') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'

                }
            }
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    dir('server/target') {
                        withCredentials([usernamePassword(credentialsId: "${openShiftCredentials}", passwordVariable: 'pass', usernameVariable: 'user')]) {
                            sh "oc login -u ${user} -p ${pass} ${openshiftUrl} --insecure-skip-tls-verify"
                            try {
                                sh "oc start-build ${pom.artifactId}${dockerEnvironment} --namespace=${openShiftNamespace} --from-dir=. --wait"
                            } catch (e) {
                                sh """
                                    oc logs \$(oc get builds -l build=${pom.artifactId}${dockerEnvironment} --namespace=${openShiftNamespace} --sort-by=.metadata.creationTimestamp -o name | tail -n 1) --namespace=${namespace}
                                    throw e
                                """
                            }
                        }
                    }
                }
            }
        }

        stage ('Deploy the new image') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps{
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    withCredentials([usernamePassword(credentialsId: "${openShiftCredentials}", passwordVariable: 'pass', usernameVariable: 'user')]) {
                        sh "oc login -u ${user} -p ${pass} ${openshiftUrl} --insecure-skip-tls-verify"
                        try {
                            sh "oc import-image ${pom.artifactId}${dockerEnvironment}:${dockerTag} --namespace=${openShiftNamespace} --from=${dockerRegistry}/${pom.artifactId}:${dockerTag} --confirm"
                        } catch (e) {
                            sh """
                                oc logs \$(oc get builds -l build=${pom.artifactId}${dockerEnvironment} --namespace=${openShiftNamespace} --sort-by=.metadata.creationTimestamp -o name | tail -n 1) --namespace=${openShiftNamespace}
                                throw e
                            """
                        }
                    }
                }
            }
        }
        
        stage ('Check pod status') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps{
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    sleep 60
                    withCredentials([usernamePassword(credentialsId: "${openShiftCredentials}", passwordVariable: 'pass', usernameVariable: 'user')]) {
                        sh "oc login -u ${user} -p ${pass} ${openshiftUrl} --insecure-skip-tls-verify"
                        sh "oc project ${openShiftNamespace}"
                        
                        def oldRetry = -1;
                        def oldState = "";
                        
                        sh "oc get pods -l app=${pom.artifactId}${dockerEnvironment} > out"
                        def status = sh (
                            script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f3",
                            returnStdout: true
                        ).trim()
                        
                        def retry = sh (
                            script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f4",
                            returnStdout: true
                        ).trim().toInteger();
                        
                        while (retry < 5 && (oldRetry != retry || oldState != status)) {
                            sleep 30
                            oldRetry = retry
                            oldState = status
                            
                            sh """oc get pods -l app=${pom.artifactId}${dockerEnvironment} > out"""
                            status = sh (
                                script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f3",
                                returnStdout: true
                            ).trim()
                            
                            retry = sh (
                                script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f4",
                                returnStdout: true
                            ).trim().toInteger();
                        }
                        
                        if(status != "Running"){
                            try {
                                sh """oc logs \$(oc get pods -l app=${pom.artifactId}${dockerEnvironment} --sort-by=.metadata.creationTimestamp -o name | tail -n 1)"""
                            } catch (e) {
                                sh "echo error reading logs"
                            }
                            error("The pod is not running, cause: " + status)
                        }
                    }
                }
            }
        }
        
    }

    post {
        cleanup {
            cleanWs()
        }
    }
}
