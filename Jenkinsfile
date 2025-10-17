pipeline {
   agent { label 'cxo_wbuild010' }
   environment {   
             REPO="sm"
             EXT_PATH=";C:\\Program Files (x86)\\MSBuild\\12.0\\bin\\"
             JAVA_HOME="C:\\Program Files\\Java\\jdk1.8.0_144"
             CERT_ROOT="\\\\cxo-samba-01.eng.nimblestorage.com\\share\\build_env\\CodeSignCert"
             THIRD_PARTY_BUILD_TOOLS="\\\\cxo-samba-01.eng.nimblestorage.com\\share\\build_env\\third_party\\"
             SAMBA_CODE_SIGN_STAGE_DIR="\\\\cxo-samba-01.eng.nimblestorage.com\\artifactswindows-cxo\\code_sign_stage_dir"
             //echo "setting windows version per ticket ESS-2745"
             //"""
   }
   stages {
        stage('Display Environment Variables') {
           steps {
             timestamps { 
                 echo "reposity is ${env.REPO}"
                 bat 'set PATH="${env.PATH}.${env.EXT_PATH}"'
                 echo "PATH is set to ${env.PATH}"
                 echo "changed java env and gradle 4.1 on 9/21/2017 (ESS-18481)"
                 
             }   
           }
        }
        stage('Clone Workspace') {
           steps {
             timestamps {
               bat """
               for /f "tokens=1,2 delims=_ " %%a in ("%SM_JOB_NAME%") do set smbranch=%%b
               echo %smbranch%
               set directory="C:\\var\\workspace\\%JOB_NAME%\\%BRANCH%\\nimos"
               echo Removig the dir %directory% if exists
               IF EXIST %directory% RMDIR /S /Q %directory%
               set branch_dir="C:\\var\\workspace\\%JOB_NAME%\\%BRANCH%"
               IF NOT EXIST %branch_dir% MKDIR %branch_dir%
               cd %branch_dir%
               echo cloning nimos repo to %branch_dir%
               git clone -n -b %smbranch% ssh://phabricator-vcs@phabricator.eng.nimblestorage.com/diffusion/nimos
               cd C:\\var\\workspace\\%JOB_NAME%\\%BRANCH%\\nimos
               echo Current Working dir: %CD%
               git config core.sparseCheckout true
               git config core.protectNTFS false
               git sparse-checkout init --cone
               copy /V C:\\it_scripts\\sparse-checkout C:\\var\\workspace\\%JOB_NAME%\\%BRANCH%\\nimos\\.git\\info\\sparse-checkout
               git checkout -f %REV%
               echo "revision is %REV%"
               echo "branch is %smbranch%"
               echo "Deleting the %smbranch% in the local Workspace"
               git branch -D %smbranch%
               git checkout -b %smbranch% %REV%
               echo "current checkout branch is"
               git branch
               """
             }   
           }
       }
       stage('Build'){
           steps {
               script {
                   timestamps {
                       bat """
                       for /f "tokens=1,2,3,4 delims=. " %%a in ("%VERSION%") do set win_version=%%a.%%b.%%c
                       echo windows version is %win_version%
                       REM echo version=%win_version%>>buildinfo.txt
                       set NIMBLE_SM_WINDOWS_VERSION_NO=%win_version%.%BUILD_NUMBER%
                       echo NIMBLE_SM_WINDOWS_VERSION_NO=%NIMBLE_SM_WINDOWS_VERSION_NO%
                       REM setting windows version per ticket ESS-3204
                       for /f "tokens=1,2,3,4 delims=. " %%a in ("%VERSION%") do set win_version_comma=%%a,%%b,%%c
                       echo %win_version_comma%
                       set NIMBLE_SM_WINDOWS_VERSION_NO_COMMA=%win_version_comma%,%BUILD_NUMBER%
                       echo NIMBLE_SM_WINDOWS_VERSION_NO_COMMA=%NIMBLE_SM_WINDOWS_VERSION_NO_COMMA%
                       
                       echo "Following is for Releng purposes"
                       cd C:\\var\\workspace\\%JOB_NAME%\\%BRANCH%
                       call "c:\\it_scripts\\creategitbuildinfo.bat"
                       echo jenkins_id=%JOB_NAME%_%BUILD_NUMBER%>>buildinfo.txt
                       echo location=%JOB_NAME%/%BUILD_NUMBER%>>buildinfo.txt
                       echo BUILD_NUMBER=%BUILD_NUMBER%>>buildinfo.txt
                       echo "End of Releng specific"
                       echo "Current Working dir: %CD%"
                       cd "C:\\var\\workspace\\%JOB_NAME%\\%BRANCH%\\nimos\\sm\\src\\sdk"
                       echo "Current Working dir: %CD%"
                       C:\\var\\lib\\workspace\\gradle-5.6\\bin\\gradle.bat -PSMLinuxArtifactDir=%SM_BUILD_NUMBER% copyCppSrcFromArtifact buildWin2013
                       """
                   }
               }
           }
       }
   }
   post {
       success {
           script {
               timestamps {
                   bat """
                   echo "Post Build processing"
                   echo "Current Working dir: %CD%"
                   echo Copying artifacts to samba share
                   xcopy /e /i %CD%\\%BRANCH%\\nimos\\sm\\build\\rest-sdk \\\\cxo-samba-01.eng.nimblestorage.com\\artifactswindows-cxo\\%JOB_NAME%\\%BRANCH%\\%BUILD_NUMBER%\\rest-sdk
                   xcopy /e /i %CD%\\%BRANCH%\\buildinfo.txt \\\\cxo-samba-01.eng.nimblestorage.com\\artifactswindows-cxo\\%JOB_NAME%\\%BRANCH%\\%BUILD_NUMBER%
                   
                   echo Creating latest successful link
                   set artdir=\\\\cxo-samba-01.eng.nimblestorage.com\\artifactswindows-cxo\\%JOB_NAME%\\%BRANCH%
                   set this=%artdir%\\%BUILD_NUMBER%
                   set latest=%artdir%\\latest-successful
                   set PATH=%PATH%;C:\\windows\\system32
                   xcopy.exe /f /v /k /y /e /s /i %this% %this%.dup || exit /b 1
                   if exist %latest% (
                        move /y %latest% %latest%.old || echo "ERROR: mv latest latest.old failed" exit /b 1
                   )
                   move /y %this%.dup %latest% || exit /b 1
                   if exist %latest%.old (
                        rmdir /s /q %latest%.old || exit /b 1
                   )
                   rmdir /s /q \\\\cxo-samba-01.eng.nimblestorage.com\\artifactswindows-cxo\\%JOB_NAME%\\%BRANCH%\\%BUILD_NUMBER%\\nimos
                   echo Trigger remote job
                   """
                   build job: 'windows_addtodbsm_pipe', propagate: false,
                         parameters:[ [$class:'StringParameterValue', name:'parentjobname', value: "${env.JOB_NAME}"], 
                                      [$class:'StringParameterValue', name:'parentjobnumber', value: "${env.BUILD_NUMBER}"],
                                      [$class:'StringParameterValue', name:'branch', value: "${env.BRANCH}"]
                                    ]
                                                  
                   emailext to: 'supraja.muppala@hpe.com',
                   subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} Success",
                   mimeType: 'text/html',
                   body: """<p>BUILD URL: ${env.BUILD_URL}</p> <p>Project: ${env.JOB_NAME}</p> <p>Server: ${env.NODE_NAME}</p> <p>Blue Ocean: ${env.RUN_DISPLAY_URL}</p>""",
                   attachLog: true
               }
           }
       }
       failure {
          script {
              timestamps {
                    emailext to: 'supraja.muppala@hpe.com',
                         subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} Failed",
                         mimeType: 'text/html',
                         body: """<p>BUILD URL: ${env.BUILD_URL}</p> <p>Project: ${env.JOB_NAME}</p> <p>Server: ${env.NODE_NAME}</p> <p>Blue Ocean: ${env.RUN_DISPLAY_URL}</p>""",
                         attachLog: true  
              }
          }
       }   
   }
}
