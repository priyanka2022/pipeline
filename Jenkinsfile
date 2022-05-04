pipeline {
  agent any

  //Configure the following environment variables before executing the Jenkins Job
  environment {
    IntegrationFlowID = "Kafka_Producer_and_Consumer"
    FailJobOnFailedMPL = true //if you are expecting your message to fail, set this to false, so that your job won't fail
    DeploymentCheckRetryCounter = 50 //multiply by 3 to get the maximum deployment time
    MPLCheckRetryCounter = 20 //multiply by 3 to get the maximum processing time. Example: 10 would be sufficient for message processings <30s
    CPIHost = "${env.CPI_HOST}"
    CPIOAuthHost = "${env.CPI_OAUTH_HOST}"
    CPIOAuthCredentials = "${env.CPI_OAUTH_CRED}"
    GITCredentials = "${env.GIT_CRED}"
    GITRepositoryURL = "${env.GIT_REPOSITORY_URL}"
    GITBranch = "${env.GIT_BRANCH_NAME}"
    GITFolder = "IntegrationArtefacts"
    GITComment = "Integration Artefacts update from CICD pipeline"
  }

  stages {
    stage('deploy Iflow and check MPL Status and Submit to GIT if MPL Status Completed') {
      steps {
        deleteDir()
        script {
          //clone repo 
          checkout([
            $class: 'GitSCM',
            branches: [[name: env.GITBranch]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
              [$class: 'RelativeTargetDirectory', relativeTargetDir: "."],
              [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [
                [$class: 'SparseCheckoutPath', path: env.GITFolder]
              ]]
            ],
            submoduleCfg: [],
            userRemoteConfigs: [
              [
                credentialsId: env.GITCredentials,
                url: 'https://' + env.GITRepositoryURL
              ]
            ]
          ])

          //get token
          def getTokenResp = httpRequest acceptType: 'APPLICATION_JSON',
            authentication: env.CPIOAuthCredentials,
            contentType: 'APPLICATION_JSON',
            httpMode: 'POST',
            responseHandle: 'LEAVE_OPEN',
            timeout: 30,
            url: 'https://' + env.CPIOAuthHost + '/oauth/token?grant_type=client_credentials';
          def jsonObjToken = readJSON text: getTokenResp.content
          def token = "Bearer " + jsonObjToken.access_token

          //deploy integration flow
          println("Deploy integration flow");

          def deployResp = httpRequest httpMode: 'POST',
            customHeaders: [
              [maskValue: false, name: 'Authorization', value: token]
            ],
            ignoreSslErrors: false,
            timeout: 30,
            url: 'https://' + env.CPIHost + '/api/v1/DeployIntegrationDesigntimeArtifact?Id=\'' + env.IntegrationFlowID + '\'&Version=\'active\'';

          //check if the flow was deployed 
          Integer counter = 0;
          def deploymentStatus;
          def continueLoop = true;
          println("Start checking integration artefact status.");
          while (counter < env.DeploymentCheckRetryCounter.toInteger() & continueLoop == true) {
            Thread.sleep(3000);
            counter = counter + 1;
            def statusResp = httpRequest acceptType: 'APPLICATION_JSON',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: token]
              ],
              httpMode: 'GET',
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + env.CPIHost + '/api/v1/IntegrationRuntimeArtifacts(\'' + env.IntegrationFlowID + '\')';

            def jsonStatusObj = readJSON text: statusResp.content;
            deploymentStatus = jsonStatusObj.d.Status;

            println("Deployment status: " + deploymentStatus);
            if (deploymentStatus.equalsIgnoreCase("Error")) {
              //get error details
              def deploymentErrorResp = httpRequest acceptType: 'APPLICATION_JSON',
                customHeaders: [
                  [maskValue: false, name: 'Authorization', value: token]
                ],
                httpMode: 'GET',
                responseHandle: 'LEAVE_OPEN',
                timeout: 30,
                url: 'https://' + env.CPIHost + '/api/v1/IntegrationRuntimeArtifacts(\'' + env.IntegrationFlowID + '\')' + '/ErrorInformation/$value';
              def jsonErrObj = readJSON text: deploymentErrorResp.content
              def deployErrorInfo = jsonErrObj.parameter;
              println("Error Details: " + deployErrorInfo);
              statusResp.close();
              deploymentErrorResp.close();
              //end job
              currentBuild.result = 'FAILURE'
              return
            } else if (deploymentStatus.equalsIgnoreCase("Started")) {
              println("Integration flow deployment successful")
              statusResp.close();
              continueLoop = false
            } else {
              println("The integration flow is not yet started. Will wait 3s and then check again.")
            }
          }

          if (!deploymentStatus.equalsIgnoreCase("Started")) {
            println("No final deployment status reached. Current status: \'" + deploymentStatus);
            currentBuild.result = 'FAILURE'
            return
          }

          //Get latest MPL status of the deployed integration flow
          println("Checking message processing log status");
          counter = 0;
          def mplStatus = '';
          continueLoop = true;
          def mplId = '';

          while (counter < env.MPLCheckRetryCounter.toInteger() & continueLoop == true) {
            //get the latest MPL
            def checkMPLResp = httpRequest acceptType: 'APPLICATION_JSON',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: token]
              ],
              contentType: 'APPLICATION_JSON',
              ignoreSslErrors: false,
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + env.CPIHost + '/api/v1/MessageProcessingLogs?$filter=IntegrationArtifact/Id%20eq%20\'' + env.IntegrationFlowID + '\'and%20Status%20ne%20\'DISCARDED\'&$orderby=LogEnd+desc&$top=1';

            def jsonMPLStatus = readJSON text: checkMPLResp.content
            jsonMPLStatus.d.results.each {
              value ->
                mplStatus = value.Status;
              mplId = value.MessageGuid;
              //if status processing, keep going
              if (mplStatus.equalsIgnoreCase("Processing")) {
                println("message processing not over yet, trying again in a short moment");
                Thread.sleep(3000);
                counter = counter + 1;
              }
            else {
                //we got a final state, ending the loop
                continueLoop = false;
                checkMPLResp.close();
              }
            }
          }
          println("message status of MPL ID \'" + mplId + "\' : \'" + mplStatus + "\'");
          if (mplStatus.equalsIgnoreCase("Processing")) {
            error("The message processing did not finish within the check frame. If it is a long running flow, increase the retry counter in the job configuration.");
          } else if (mplStatus.equalsIgnoreCase("Failed") || mplStatus.equalsIgnoreCase("Retry")) {
            //get error information
            def cpiMplError = httpRequest acceptType: 'APPLICATION_ZIP',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: token]
              ],
              ignoreSslErrors: false,
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + env.CPIHost + '/api/v1/MessageProcessingLogs(\'' + mplId + '\')/ErrorInformation/$value';
            println("Message processing failed! Error information: " + cpiMplError.content);
            cpiMplError.close();
			if (env.FailJobOnFailedMPL.equalsIgnoreCase("true")) {
              error("The job is configured to fail on a failed MPL processing.");
            }
          } else if (mplStatus.equalsIgnoreCase("Discarded") || mplStatus.equalsIgnoreCase("Abandoned")) {
            error("Message processing not successful. Either a different message is blocking the processing or the processing was interrupted");
          } else if (mplStatus.equalsIgnoreCase("Completed")) {
            println("Message processing successful.");
		  } else {
              error("Unexpected error. Kindly check the tenant.");
            }
		  
            //Proceed with the download. Delete the old flow content first so that only the latest content gets stored
            dir(env.GITFolder + '/' + env.FlowId) {
              deleteDir();
            }
            //download and extract artefact from tenant
            println("Downloading artefact");
            def tempfile = UUID.randomUUID().toString() + ".zip";
            def cpiDownloadResponse = httpRequest acceptType: 'APPLICATION_ZIP',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: token]
              ],
              ignoreSslErrors: false,
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              outputFile: tempfile,
              url: 'https://' + env.CPIHost + '/api/v1/IntegrationDesigntimeArtifacts(Id=\'' + env.IntegrationFlowID + '\',Version=\'active\')/$value';
            def disposition = cpiDownloadResponse.headers.toString();
            def index = disposition.indexOf('filename') + 9;
            def lastindex = disposition.indexOf('.zip', index);
            def filename = disposition.substring(index + 1, lastindex + 4);
            def folder = env.GITFolder + '/' + filename.substring(0, filename.indexOf('.zip'));
            fileOperations([fileUnZipOperation(filePath: tempfile, targetLocation: folder)])
            cpiDownloadResponse.close();

            //remove the zip
            fileOperations([fileDeleteOperation(excludes: '', includes: tempfile)])

            dir(folder) {
              sh 'git add .'
            }
            println("Store integration artefact in Git")
            withCredentials([
              [$class: 'UsernamePasswordMultiBinding', credentialsId: env.GITCredentials, usernameVariable: 'GitUserName', passwordVariable: 'GitPassword']
            ]) {
              sh 'git diff-index --quiet HEAD || git commit -am ' + '\'' + env.GITComment + '\''
                 sh('git push https://${GitUserName}:${GitPassword}@' + env.GITRepositoryURL  + ' HEAD:' + env.GITBranch)
            }
        }
      }
    }
  }
}
