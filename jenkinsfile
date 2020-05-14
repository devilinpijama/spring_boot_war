properties(
    [
        buildDiscarder(
            logRotator(
                daysToKeepStr: '10',
                numToKeepStr: '10'
            )
        )
    ]
)

node('mainnode') {
    def mvnHome = tool 'Apache Maven 3.5.4'
    def server = Artifactory.server 'main_artifactory'
    def rtMaven = Artifactory.newMavenBuild()
    def buildInfo

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
            sh("cd ${WORKSPACE} ; $mvnHome/bin/mvn sonar:sonar")
        }
    }

    stage('Maven build') {
        sh("ls -la ${WORKSPACE}")
        rtMaven.run pom: '${WORKSPACE}/pom.xml', goals: 'clean install', buildInfo: buildInfo
    }

    stage ('Publish build info') {
        server.publishBuildInfo buildInfo
    }

    stage ('Deploying to Tomcat'){
        deploy adapters: [tomcat9(credentialsId: '439ff615-e2f6-40f1-8988-8d03628abde9', path: '', url: 'http://10.128.0.100:8080')], contextPath: 'petclinic', war: 'target/*.war'
    }    
}
