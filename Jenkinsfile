@Library('nais')
@Library('deploy')
import deploy
import com.jenkinsci.plugins.badge.action.BadgeAction
deployLib = new deploy()
node {
    def appToken
    def commitHash
    def appConfig = "nais.yaml"
    def dockerRepo = "repo.adeo.no:5443"
    def application = "data-catalog-backend"
    def mvnHome = tool "maven-3.3.9"
    def jdk = tool "11"
    def mvn = "${mvnHome}/bin/mvn"
    def FASIT_ENV = "t5"
    try {
        cleanWs()
        stage("checkout") {
            appToken = github.generateAppToken()
            sh "git init"
            sh "git pull https://x-access-token:$appToken@github.com/navikt/data-catalog-backend.git"
            sh "git fetch --tags https://x-access-token:$appToken@github.com/navikt/data-catalog-backend.git"
            releaseVersion = sh(script: "git describe --always --abbrev=0 --tags", returnStdout:true).trim()
            sh "mvn clean install"
        }
        stage("build and publish docker image") {
            withCredentials([usernamePassword(credentialsId: 'nexusUploader', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                sh "docker build -t ${dockerRepo}/${application}:${releaseVersion} ."
                sh "docker login -u ${env.NEXUS_USERNAME} -p ${env.NEXUS_PASSWORD} ${dockerRepo} && docker push ${dockerRepo}/${application}:${releaseVersion}"
            }
        }
        stage("publish yaml") {
            withCredentials([usernamePassword(credentialsId: 'nexusUploader', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
              sh "curl --user ${env.NEXUS_USERNAME}:${env.NEXUS_PASSWORD} --upload-file ${appConfig} https://repo.adeo.no/repository/raw/nais/${application}/${releaseVersion}/nais.yaml"
            }
        }
       stage('Deploy to nais preprod') {
            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jiraServiceUser', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                sh "curl -k -d \'{\"application\": \"${application}\", \"version\": \"${releaseVersion}\", \"fasitEnvironment\": \"q1\", \"zone\": \"fss\", \"skipFasit\": true, \"namespace\": \"q1\"}\' https://daemon.nais.preprod.local/deploy"
            }
       }
    } catch (err) {
        throw err
    }
}