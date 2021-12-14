#!/usr/bin/env groovy

pipeline {

    agent {
        label 'jenkins-slave-frontend'
    }

    tools {
        nodejs 'NodeJs14'
    }

    environment {
        CI = 'true'
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {

        stage('Build') {
            steps {
                methodOrchestrator('build')
            }
        }
    }

}


//==============================================================================================



void methodOrchestrator(String operation){
    try {
        if(operation.equals('build')){
            build()
        }else if(operation.equals('sonarScanner')){
            sonarScanner()
        }else if(operation.equals('stash')){
            methodStash()
        }else if(operation.equals('unstash')){
            methodUnstash()
        }else if(operation.equals('deploy')){
            deploy()
        }
    } catch (Exception e) {
        println e
        throw e
    }
}



/* Etapa para preparar Compilar la aplicacion
    Dependencia: npm
    En esta etapa se realiza la complilacion y se generan los artefactos para el despliegue.
*/
def build() {
    dir("${env.WORKSPACE}"){
        sh 'npm install'
        sh 'set -x'
        sh 'npm run build'
        sh 'set +x'
    //sh 'zip -r deploy.zip ./* -x config\\*'
    }
}

/* Etapa para realizar la verificacion de codigo estatico
    Dependencia: sonar
    En esta etapa se realiza la ejecucion del sonar scanner y se envia el reporte al servidor de Sonarqube
    Variables: SONAR_SERVER Esta variable tiene el nombre de las credenciales que contienen la configuracion del
    Servidor de Sonar en Jenkins
*/
void sonarScanner() {
    dir("${env.WORKSPACE}") {
            String scannerHome = tool 'sonarqubescanner'
            withSonarQubeEnv("${env.SONAR_SERVER}") {
                /* groovylint-disable-next-line LineLength */
                sh "${scannerHome}/bin/sonar-scanner -Dsonar.exclusions=config/**,node_modules/**,dist/** -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} -Dsonar.sources=."
            }
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
                error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
    }
}


def methodStash(){
    stash includes: 'dist/**/*', useDefaultExcludes: false, name: 'repository'  
}


def methodUnstash(){
    unstash 'repository'
}



def deploy() {
    withCredentials([azureServicePrincipal(
                        credentialsId: CREDENTIAL_SERVICE_PRINCIPAL,
                        subscriptionIdVariable: 'AZURE_SUBSCRIPTION_ID',
                        clientIdVariable: 'AZURE_CLIENT_ID',
                        clientSecretVariable: 'AZURE_CLIENT_SECRET',
                        tenantIdVariable: 'AZURE_TENANT_ID')]) {

        // some block

                          
        sh '''
            az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
            az account set -s $DEPLOY_AZURE_SUBSCRIPTION_ID
         '''
        sh "az storage blob delete-batch -s '${env.CONTAINER_NAME}' --account-name ${env.ACCOUNT_NAME}"
        sh "az storage blob upload-batch -d '${env.CONTAINER_NAME}' --account-name ${env.ACCOUNT_NAME} -s dist/"
        sh 'az logout'
    }
    
}
