def ENABLE_DEPLOY = ''


//derive settings from scm branch name
switch (BRANCH_NAME) {
    case "develop":
        ENABLE_DEPLOY = 'true'
        break
    case "main":
        ENABLE_DEPLOY = 'false'
        break
}

echo $JOB_NAME

if (JOB_NAME == 'job-seeder') {
    node() {
        stage('Checkout SCM') {
            checkout scm
        }
        stage('Update jobs') {
            sh """
                jobfile=$(mktemp)
                cat Jenkinsjobs > ${jobfile}
                echo -e "\n\n" >> ${jobfile}
                cat ./devops/jenkinsjob-template >> ${jobfile}
                jenkins-jobs --conf ./devops/jenkins_jobs.ini update ${jobfile}
            """
        }       
    }
} else {

if (ENABLE_DEPLOY == 'true') {
    node() {
        stage('Checkout SCM') {
            checkout scm
        }
        stage('nothing') {
            echo "${JOB_NAME}"
        }
    }
}

}