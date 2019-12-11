pipeline
{
    options
    {
        buildDiscarder(logRotator(numToKeepStr: '3'))
    }
    agent any
    environment 
    {
        VERSION = 'latest'
        PROJECT = 'dev-pro'
        IMAGE = "$PROJECT:$VERSION"
        ECRURL = 'http://021344489861.dkr.ecr.us-west-2.amazonaws.com'
        ECRCRED = 'ecr:us-west-2:dev-pro-ecr'
        TASKDEF = 'file://aws/task-definition-${IMAGE}.json'
    }
    stages
    {
        stage("Checkout") {
            steps
            {
                script
                {
                    checkout scm
                }

            }
        }

        stage('Build preparations')
        {
            steps
            {
                script 
                {
                    // calculate GIT lastest commit short-hash
                    gitCommitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                    shortCommitHash = gitCommitHash.take(7)
                    // calculate a sample version tag
                    VERSION = shortCommitHash
                    // set the build display name
                    currentBuild.displayName = "#${BUILD_ID}-${VERSION}"
                    IMAGE = "$PROJECT:$VERSION"
                }
            }
        }
        stage('Docker build')
        {
            steps
            {
                script
                {
                    // Build the docker image using a Dockerfile
                    docker.build("$IMAGE",".")
                }
            }
        }
        stage('Docker push')
        {
            steps
            {
                script
                {
                    // login to ECR - for now it seems that that the ECR Jenkins plugin is not performing the login as expected. I hope it will in the future.
                    sh("eval \$(aws ecr get-login --no-include-email | sed 's|https://||')")
                    // Push the Docker image to ECR
                    docker.withRegistry(ECRURL, ECRCRED)
                    {
                        docker.image(IMAGE).push()
                    }
                }
            }
        }
        /*
        / These steps to create new revision of the TaskDefinition, then -
        / update the servie with the new TaskDefinition revision to deploy the image
        */
        stage("Deploy") 
        {
            steps
            {
            // Replace BUILD_TAG placeholder in the task-definition file -
            // with the IMAGE (imageTag-BUILD_NUMBER)
            sh  "                                                                     \
            sed -e  's;%BUILD_TAG%;${IMAGE};g'                             \
                aws/task-definition.json >                                      \
                aws/task-definition-${IMAGE}.json                      \
            "

        // Get current [TaskDefinition#revision-number]
            def currTaskDef = sh (
                returnStdout: true,
                script:  "                                                              \
                    aws ecs describe-task-definition  --task-definition ${taskFamily}     \
                                                      | egrep 'revision'                  \
                                                      | tr ',' ' '                        \
                                                      | awk '{print \$2}'                 \
                    "
            ).trim()

            def currentTask = sh (
                returnStdout: true,
                script:  "                                                              \
                    aws ecs list-tasks  --cluster ${clusterName}                          \
                                        --family ${taskFamily}                            \
                                        --output text                                     \
                                        | egrep 'TASKARNS'                                \
                                        | awk '{print \$2}'                               \
                    "
            ).trim()

        /*
        / Scale down the service
        /   Note: specifiying desired-count of a task-definition in a service -
        /   should be fine for scaling down the service, and starting a new task,
        /   but due to the limited resources (Only one VM instance) is running
        /   there will be a problem where one container is already running/VM,
        /   and using a port(80/443), then when trying to update the service -
        /   with a new task, it will complaine as the port is already being used,
        /   as long as scaling down the service/starting new task run simulatenously
        /   and it is very likely that starting task will run before the scaling down service finish
        /   so.. we need to manually stop the task via aws ecs stop-task.
        */
            if(currTaskDef) {
                sh  "                                                                   \
                    aws ecs update-service  --cluster ${clusterName}                      \
                                            --service ${serviceName}                      \
                                            --task-definition ${taskFamily}:${currTaskDef}\
                                            --desired-count 0                             \
                    "
            }
            if (currentTask) {
                sh "aws ecs stop-task --cluster ${clusterName} --task ${currentTask}"
            }   

        // Register the new [TaskDefinition]
            sh  "                                                                     \
                aws ecs register-task-definition    --family ${taskFamily}             \
                                                    --cli-input-json ${TASKDEF}        \
            "

        // Get the last registered [TaskDefinition#revision]
            def taskRevision = sh (
                returnStdout: true,
                script:  "                                                              \
                aws ecs describe-task-definition    --task-definition ${taskFamily}     \
                                                    | egrep 'revision'                  \
                                                    | tr ',' ' '                        \
                                                    | awk '{print \$2}'                 \
                "
            ).trim()

        // ECS update service to use the newly registered [TaskDefinition#revision]
        //
            sh  "                                                                     \
                aws ecs update-service  --cluster ${clusterName}                        \
                                        --service ${serviceName}                        \
                                        --task-definition ${taskFamily}:${taskRevision} \
                                        --desired-count 1                               \
                "
            }
        }
    }
    
    post
    {
        always
        {
            // make sure that the Docker image is removed
            sh "docker rmi $IMAGE | true"
        }
    }
}