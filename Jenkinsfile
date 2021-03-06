properties(
    [
        buildDiscarder(
            logRotator(
                daysToKeepStr: '5',
                numToKeepStr: '5'
            )
        )
    ]
)

node('mainnode') {
    def mvnHome = tool 'Apache Maven 3.5.4'
    def scanerHome = tool 'Sonar Scanner 4.2.0.1873 LTS'
    def server = Artifactory.server 'main_artifactory'
    def projectName = 'war_petclinic'
    def rtMaven = Artifactory.newMavenBuild()
    def buildInfo
    def AppVersion = sh script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout', returnStdout: true
    def AppName = sh script: 'mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout', returnStdout: true
    currentBuild.displayName = "${AppName}_${AppVersion}_${BUILD_TIMESTAMP}_build#${BUILD_NUMBER}" 
    
    withEnv(["JAVA_HOME=${ tool 'Java 8' }", "PATH+MAVEN=${tool 'Apache Maven 3.5.4'}/bin:${env.JAVA_HOME}/bin"]) {

    stage('GIT Clone') {
        git branch: 'master', url: 'https://github.com/devilinpijama/spring_boot_war.git'
    }

    stage ('Artifactory configuration') {
        rtMaven.tool = 'Apache Maven 3.5.4'
        rtMaven.deployer releaseRepo: 'spring-boot', snapshotRepo: 'spring-boot', server: server
        buildInfo = Artifactory.newBuildInfo()
    }

    stage('SonarQube') { 
        withSonarQubeEnv('Sonar Main Server') { 
            sh """
            mvn compile && \
            $scanerHome/bin/sonar-scanner \
            -Dsonar.projectKey=$projectName \
            -Dsonar.java.binaries=target/classes \
            -Dsonar.sources=.
            """
        }
    }
    stage('SonarQube quality gate') {   // Catches webhook from SonarQube
        timeout(time:1, unit: "MINUTES") {
            def qg = waitForQualityGate ()
            if (qg.status != 'OK') {
                error "Pipeline aborted due to the QualityGate failure: ${qg.status}"
            }
        }
        
    }
    stage('Maven build') {
        rtMaven.run pom: '${WORKSPACE}/pom.xml', goals: 'clean install', buildInfo: buildInfo
    }

    stage ('Publish build info') {
        server.publishBuildInfo buildInfo
    }

    stage ('Deploying to Tomcat') {
        deploy adapters: [tomcat9(credentialsId: '439ff615-e2f6-40f1-8988-8d03628abde9', path: '', url: 'http://10.128.0.100:8080')], contextPath: 'petclinic', war: 'target/*.war'
    }    
    stage ('Sending emails to recepients') {
        emailext attachLog: true, body: '${JELLY_SCRIPT,template="html-with-health-and-console2.jelly"}', recipientProviders: [brokenBuildSuspects(), brokenTestsSuspects(), upstreamDevelopers()], subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', to: '$DEFAULT_RECIPIENTS'
    }
  }       
}
