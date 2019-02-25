import groovy.json.JsonOutput

// Do not resolve JIRA email by the user name:
// hudson.plugins.jira.JiraMailAddressResolver.disabled=true

// A new JIRA issue template
def newJiraIssue(summary, description){
    def testIssue = [fields: [ project: [key: 'CMSTZDEV'],
                       summary: summary,
                       description: description,
                       issuetype: [name: 'Task']]]  
                       
    response = jiraNewIssue issue: testIssue, site: 'CMSTZ'
    JIRA_ISSUE_ID = response.data.id
    assignResponse = jiraAssignIssue site: 'CMSTZ', idOrKey: JIRA_ISSUE_ID, userName: 'cmst0'
    addWatcher1 = jiraAddWatcher site: 'CMSTZ', idOrKey: JIRA_ISSUE_ID, userName: 'vjankaus'
    // addWatcher2 = jiraAddWatcher site: 'CMSTZ', idOrKey: JIRA_ISSUE_ID, userName: 'anquinte'
}

// A new JIRA comment template
def addJiraComment(text){
    response = jiraAddComment site: 'CMSTZ', idOrKey: JIRA_ISSUE_ID, comment: text
    echo response.data.toString()
}

node('t0ReplayNode') {
    environment {
        JIRA_ISSUE_ID = ""
        SHELL_OUTPUT = ""
        GIT_DIR=""
    }
    // Clean up the agent before a new deployment
    stage('CleanupBefore') {
        checkout scm
        sh '''
            #switch to working dir
            cd /data/tier0/
            #stop the agent
            ./00_stop_agent.sh
            #remove all the previous jobs from condor q
            condor_rm -all
            #sleep 10
        '''
        JiraIssueMessage = "Configuration for the replay is available at: ${ghprbPullLink}"
        script{
            newJiraIssue("Tier0_REPLAY v${BUILD_NUMBER} on ${NODE_NAME}. ${ghprbPullTitle}", "${JiraIssueMessage}")
            echo JIRA_ISSUE_ID
            echo 'Cleaning up the agent before the replay'
        }
    }
    // Deploy a new T0 WMAgent
    stage('DeployTheAgent') {
        sh '''
            gitdir=$(pwd)
            cd /data/tier0/
            ./00_software.sh
            ##deploy them
            ./00_deploy_replay.sh
            #copy the necessary scripts and configs:
            cp ${gitdir}/etc/ReplayOfflineConfiguration.py /data/tier0/admin/ReplayOfflineConfiguration.py
            cp ${gitdir}/bin/message.py /data/tier0/jenkins/message.py
            cp ${gitdir}/bin/compile.py /data/tier0/jenkins/compile.py
            cp ${gitdir}/bin/replayWorkflowStatus.py /data/tier0/jenkins/replayWorkflowStatus.py
        '''
        script{
            SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/message.py /data/tier0/jenkins/compile.py /data/tier0/admin/ReplayOfflineConfiguration.py')
            SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/message.py /data/tier0/jenkins/compile.py /data/tier0/admin/ReplayOfflineConfiguration.py 1')
            echo 'Tier0 WMAgent was deployed'
        }
    }
    // Start the agent
    stage('StartTheAgent') {
        sh '''
        cd /data/tier0/
        export T0_PROCESSING_VERSION=$BUILD_NUMBER
        #start the agent
        ./00_start_agent.sh
        #echo "Tier0 processing version is: $T0_PROCESSING_VERSION"
        sleep 150
        '''
        script{
            echo 'Starting T0 WMAgent'
               addJiraComment('*Replay has started. The progress of it will be reported here.*')
        }
    }
    // Watch and report the replay progress
    stage('ReplayChecks') {
        parallel (
            'PauseProgress': 
                { stage('PauseProgress'){
                    script{
                        echo 'Passing Checking the Pause status.'
                        SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/message.py /data/tier0/jenkins/compile.py /data/tier0/jenkins/replayWorkflowStatus.py')
                        SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/compile.py /data/tier0/jenkins/replayWorkflowStatus.py 1')
                        def status = sh(returnStatus: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Paused')
                        // echo SHELL_OUTPUT
                        echo status.toString()
                        if  (status == 0) {
                            // echo "status zero"
                            addJiraComment("*There were NO paused jobs in the replay.*")
                        } else if (status == 1) {
                            // echo "status one"
                            addJiraComment("*There are some paused jobs in the replay.*")
                        } else {
                            // echo "status other"
                            addJiraComment("*There were some issues as the paused job checker script failed with exit code ${status}*")
                        }
                    }
                }},
            'RepackProgress': 
                { stage('RepackProgress'){
                    script{   
                        SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Repack')
                        def replayStatus = sh(returnStatus: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Repack')
                        echo SHELL_OUTPUT
                        if  (replayStatus == 0) {
                            // echo "status zero"
                            addJiraComment('All Repack workflows were processed.')
                        } else {
                            // echo "status non zero"
                            addJiraComment("*Something went wrong with checking existing Replay workflows. Please check the replay manually*")
                        }
                    }
                }},
            'ExpressProgress': { 
                stage('ExpressProgress'){
                    script{
                        
                        SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Express')
                        def expressStatus = sh(returnStatus: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Express')
                        echo SHELL_OUTPUT
                        if  (expressStatus == 0) {
                            // echo expressStatus.toString()
                            addJiraComment('All Express workflows were processed')
                        } else {
                            // echo expressStatus.toString()
                            addJiraComment("*Something went wrong with checking existing Express workflows. Please check the replay manually*")
                        }   
                    }
                }
            },
            'FilesetProgress': {
                stage('FilesetProgress'){
                    script{
                        SHELL_OUTPUT = sh(returnStdout: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Fileset')
                        def filesetStatus = sh(returnStatus: true, script: 'python /data/tier0/jenkins/replayWorkflowStatus.py Fileset')
                        echo SHELL_OUTPUT
                        if  (filesetStatus == 0) {
                            // echo filesetStatus.toString()
                            addJiraComment('All filesets were closed')
                        } else {
                            // echo filesetStatus.toString()
                            addJiraComment("*Something went wrong with checking existing filesets. Please check the replay manually*")
                        }
                    }
                }
            }
        )
        // The replay was successful if all above steps exited with exite code 0.
        echo "The replay has finished successfully."
        addJiraComment('*The replay has finished successfully.*')
    }
}
