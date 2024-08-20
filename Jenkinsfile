def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-" + ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
def EMAIL_RECIPIENTS = "your_email@gmail.com"


node {
    try {
        stage('Initialize') {
            def dockerHome = tool 'dockerlatest'
            def mavenHome = tool 'mavenlatest'
            env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"
        }

        stage('Checkout') {
            checkout scm
        }

        stage('Build with test') {
            sh "mvn clean install"
        }
        // après les tests on va envoyer les metriques chez sonar pour être analysé
        stage('Sonarqube Analysis') {
            withSonarQubeEnv('localhost_sonarqube') { // localhost_sonarqube le nom de plugin de sonar dans jenkins
                sh " mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"  // cette commande permet d'envoyer pour analyse du code à sonar
            }
            timeout(time: 1, unit: 'MINUTES') {
                def qg = waitForQualityGate() // cette fonction permet d'attendre le retour du sonar 1mn maximun
                if (qg.status != 'OK') {  // dans le cas ou c'est pas ok on arrete le pipline, cela veut dire la qualité de code est dégradée
                                          // on ne recupère pas le code du developpeur
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }
        // dans le cas ou c'est ok, ici on va s'assurer que toutes les images precedentes sont supprimées avants de regener d'autres images et containers
        stage("Image Prune") {
            imagePrune(CONTAINER_NAME)
        }
        // on va construire l'image de l'application
        stage('Image Build') {
            imageBuild(CONTAINER_NAME, CONTAINER_TAG)
        }

        stage('Push to Docker Registry') {
            withCredentials([usernamePassword(credentialsId: 'dockerhubcredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)
            }
        }

        stage('Run App') {
            withCredentials([usernamePassword(credentialsId: 'DockerhubCredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                runApp(CONTAINER_NAME, CONTAINER_TAG, USERNAME, HTTP_PORT, ENV_NAME)

            }
        }

    } finally {
        deleteDir()
        sendEmail(EMAIL_RECIPIENTS);
    }

}

def imagePrune(containerName) {
    try {
        sh "docker image prune -f"   // suppressions de tous les images pour économiser de l'espace
        sh "docker stop $containerName" // on va arreter le container
    } catch (ignored) {
    }
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag  -t $containerName --pull --no-cache ."
    echo "Image build complete"
}

def pushToImage(containerName, tag, dockerUser, dockerPassword) {
    sh "docker login -u $dockerUser -p $dockerPassword"
    sh "docker tag $containerName:$tag $dockerUser/$containerName:$tag"
    sh "docker push $dockerUser/$containerName:$tag"
    echo "Image push complete"
}

def runApp(containerName, tag, dockerHubUser, httpPort, envName) {
    sh "docker pull $dockerHubUser/$containerName"
    sh "docker run --rm --env SPRING_ACTIVE_PROFILES=$envName -d -p $httpPort:$httpPort --name $containerName $dockerHubUser/$containerName:$tag"
    echo "Application started on port: ${httpPort} (http)"
}

def sendEmail(recipients) {
    mail(
            to: recipients,
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
            body: "Check console output at: ${env.BUILD_URL}/console" + "\n")
}

String getEnvName(String branchName) {
    if (branchName == 'master') {
        return 'prod'
    }
    return (branchName == 'develop') ? 'uat' : 'dev'
}

String getHTTPPort(String branchName) {
    if (branchName == 'master') {
        return '9003'
    }
    return (branchName == 'develop') ? '9002' : '9001'
}

String getTag(String buildNumber, String branchName) {
    if (branchName == 'master') {
        return buildNumber + '-stable'
    }
    return buildNumber + '-unstable'
}
