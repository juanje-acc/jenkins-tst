
import groovy.json.JsonSlurper
import hudson.AbortException
import org.jenkinsci.plugins.workflow.steps.FlowInterruptedException


node("!MASTER") {
    // Definitions
    GITLAB_HOST_HTTPS = env.GITLAB_HOST_HTTPS
    GITLAB_CREDENTIALS = "GitLabJenkinsUser"
    GROUP = "${group}"
    REPO_NAME = "${repo_name}.git"

    try {

         showParametersInfo()

            stage "Clone repository"
                println '''
                #######################################################
                [INFO] Start download repository
                #######################################################
                '''
                sh 'git config --global http.sslVerify false'
                checkoutRepo()
                processSkips()
                println '''
                #######################################################
                [INFO] End download repository
                #######################################################
                '''
            stage "Security & Compliance Checks" 
                println '''
                #######################################################
                [INFO] Start check Rules
                #######################################################
                '''
                if (!skipSecurityCheck) {
                    println '''
                    #######################################################
                    [INFO] End check Rules
                    #######################################################
                    '''
                } else {
                    println '''
                    #######################################################
                    [INFO] Skipped
                    #######################################################
                    '''
                }
            stage "Build Image" 
                println '''
                #######################################################
                [INFO] Start Image Build
                #######################################################
                '''
                    buildImage()
                    println '''
                    #######################################################
                    [INFO] End Image Build
                    #######################################################
                    '''
                    //Stop pipeline until proceed
                    input message: "The Docker image is builded, Do you wanna publish it in Sandbox?"
            stage "Test reports & coverage"
                println '''
                #######################################################
                [INFO] Start Coverage validation
                #######################################################
                '''
                if (!skipSecurityCheck) {
                    testsReport()
                    println '''
                    #######################################################
                    [INFO] End Coverage validation
                    #######################################################
                    '''
                } else {
                    println '''
                    #######################################################
                    [INFO] Skipped
                    #######################################################
                    '''
                }
             
              stage "TAG / Signing & Publish"
                println '''
                #######################################################
                [INFO] Start TAG & Signing Image

                #######################################################
                '''
                if (!skipTag) {
                    tag_image()
                    println '''
                    #######################################################
                    [INFO] Fin TAG & Signing Image
                    #######################################################
                    '''
                } else {
                    println '''
                    #######################################################
                    [INFO] Skipped
                    #######################################################
                    '''
                }
               println '''
                #######################################################
                [INFO] Start publishing on Sandbox Registry
                #######################################################'''
                if (!skipPublish) {
                    publish()
                    println '''
                    #######################################################
                    [INFO] End publishing Sandbox Registry
                    #######################################################
                    '''
                } else {
                    println '''
                    #######################################################
                    [INFO] Skipped
                    #######################################################
                    '''
                }
            
            } catch(FlowInterruptedException fe){
                println '''
                +++++++++++++++++++++++++++++++++++++++++++++++++++++++
                [JOB CANCELADO] El job fue cancelado
                +++++++++++++++++++++++++++++++++++++++++++++++++++++++'''
                println fe
                jobStatus = "CANCEL"
        
        } catch (Exception e) {
            println '''
            +++++++++++++++++++++++++++++++++++++++++++++++++++++++
            [ERROR] El job produjo el siguiente error 
            +++++++++++++++++++++++++++++++++++++++++++++++++++++++'''
            println e
            currentBuild.result = "FAILURE"

        } finally {
            stage "Clean workspace"
                println '''
                #######################################################
                [INFO] Start clean workspace
                #######################################################'''
                
               // cleanAll()
               cleanWs()
                
                println '''
                #######################################################
                [INFO] End clean workspace
                #######################################################'''
      }
     /*   withCredentials([[$class: 'StringBinding', credentialsId: API_TOKEN, variable: 'API_TOKEN']]) {
            if(jobStatus == "CANCEL"){
                ci_apiGitlab.setCommitStatusCanceled(env.API_TOKEN, GITLAB_HOST_HTTPS, repoId, repoLastCommit, branch)
            }else if(currentBuild.result == "FAILURE"){
                ci_apiGitlab.setCommitStatusFailed(env.API_TOKEN, GITLAB_HOST_HTTPS, repoId, repoLastCommit, branch)
            }else {
                ci_apiGitlab.setCommitStatusSuccess(env.API_TOKEN, GITLAB_HOST_HTTPS, repoId, repoLastCommit, branch)
          }
    */     
    //}//Try 
} //node

    /**
     * Download code from GitLab
     */
    def checkoutRepo() {
          checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git_user', url: "${GITLAB_HOST_HTTPS}/${GROUP}/${repo_name}.git"]]])
    }

    def processSkips() {
            skipRules = true
            skipSecurityCheck = true
            skipTag = false
            skipPublish = false
            skipLint = false
    }

    /*
    def checkARQRules(projectType, project){

    }
    */

    def buildImage() {
          /* This builds the actual image; synonymous to
             * docker build on the command line */
              docker.withRegistry('https://sandbox.docker.caixabank.com', 'sandbox-registry') {
                app = docker.build("containers/apps/${repo_name}")
                }

        
    }
    def testsReport () {
        //TODO Report security Checks
    }

    def showParametersInfo() {
        println """
        #######################################################
        [INFO] ENTRY PARAMETERS LIST
        #######################################################

        GROUP=$group
        REPO_NAME=$repo_name
        WORKSPACE_PATH=pwd()

        Parametros especificos del job:
        JENKINS_HOME=${env.JENKINS_HOME}
        BUILD_URL=${env.BUILD_URL}
        JENKINS_URL=${env.JENKINS_URL}
        JOB_NAME=${env.JOB_NAME}
        JOB_URL=${env.JOB_URL}

        #######################################################
        #######################################################

        """
    }

    def tag_image() {
         docker.withRegistry('https://sandbox.docker.caixabank.com:8443', 'sandbox-registry') {
                app.push("snapshot-${env.BUILD_NUMBER}")
        }
    }

    /**
     * Publish on Sandbox registry 
     */
    def publish() {
        sh 'export DOCKER_CONTENT_TRUST=1' //needed for signing image
        docker.withRegistry('https://sandbox.docker.caixabank.com:8443', 'sandbox-registry') {
           app.push("last-signed")
        }

    }

