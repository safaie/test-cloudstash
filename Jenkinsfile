pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret')
    }
    stages {
        stage('Install dependacies') {
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'echo install any dependancies..'
                }
            }
        }
        stage('Run VT') {
            environment {
                DEPLOY_FILE = 'cloudstash.csar'
                VT_DOCKER_NAME = 'RadonVT'
                VT_DOCKER_IMAGE = 'marklawimperial/verification-tool'
                VT_FILES_PATH = '{"path":"/tmp/main.cdl"}'
            }
            steps {
                // Pull the latest image of VT
                sh 'docker pull $VT_DOCKER_IMAGE'
                // Unzip the csar
                sh 'unzip $DEPLOY_FILE'
                // Move relevant files to temp folder. 
                sh 'mkdir -p tmp/radon-vt && cp -r _definitions tmp/radon-vt/_definitions && cp radon-vt/main.cdl tmp/radon-vt'
                // Run Verification Tool as Docker image and open port 5000 
                sh 'docker run --name $VT_DOCKER_NAME --rm -d -p 5000:5000 -v $PWD/tmp/radon-vt:/tmp $VT_DOCKER_IMAGE'
                // Wait some sec for the container to spin up.
                sh 'sleep 5'
                // Verify the model with the main.cdl restrictions. - Detect inconsistencies.
                sh 'curl -X POST -H "Content-type: application/json" http://localhost:5000/solve/ -d $VT_FILES_PATH'
                // Correct the model to comply with the main.cdl restrictions. - Propose correction of inconsistencies.
                sh 'curl -X POST -H "Content-type: application/json" http://localhost:5000/correct/ -d $VT_FILES_PATH'
                // Stop the container
                sh 'docker stop $VT_DOCKER_NAME'
            }
        }
        stage('Run DPT') {
            environment {
                DEPLOY_FILE = 'cloudstash.csar'
                DPT_DOCKER_IMAGE = 'radonconsortium/radon-dp'
                GITHUB_REPO = 'AlexSpart/Cloudstash'
            }
            steps {
                // Pull radonconsortium/radon-dp:latest image from Dockerhub
                sh 'docker pull $DPT_DOCKER_IMAGE'
                // Create temporary
                sh 'mkdir -p tmp/radon-dp-volume'
                // Download a suitable model 
                sh 'docker run -v $PWD/tmp/radon-dp-volume:/app $DPT_DOCKER_IMAGE radon-defect-predictor download-model tosca github $GITHUB_REPO'
                // Move CSAR into tmp library. This is colocated with the fetched radondp_model.joblib
                sh 'cp $DEPLOY_FILE tmp/radon-dp-volume'
                // Run predict
                sh 'docker run -v $PWD/tmp/radon-dp-volume:/app $DPT_DOCKER_IMAGE radon-defect-predictor predict tosca $DEPLOY_FILE'
                // Results are available at:
                sh 'cat tmp/radon-dp-volume/radondp_predictions.json'
            }
        }
        stage('Run CTT') {
            environment {
                // Specify the name of the container
                CTT_DOCKER_NAME = 'RadonCTT'
                // The DPT docker image published in Dockerhub 
                CTT_DOCKER_IMAGE = 'radonconsortium/radon-ctt:latest'
                // The path to the config file on jenkins ws
                CTT_CONFIG_FILE_PATH = 'radon-ctt-cli-testing/ctt_config.yaml'
                // URL of CTT API as defined in the docker command
                CTT_SERVER_URL = 'http://localhost:18080/RadonCTT'
            }
            steps {
                // Use the secret file of Jenkins "prq-aws-ssh-key" to Create a variable named as "PRQ_AWS_SSH_KEY" 
                withCredentials([file(credentialsId: "prq-aws-ssh-key", variable: "PRQ_AWS_SSH_KEY")]) {
                                    //Initialize an empty file & copy the SSH key derived from the secret file.
                                    sh 'touch $PWD/tmp/awsec2.pem && cp -r $PRQ_AWS_SSH_KEY $PWD/tmp/awsec2.pem'
                                }
                // Pull the latest version of the CTT docker Image
                sh 'docker pull $CTT_DOCKER_IMAGE'
                // Run CTT server using docker
                sh 'docker run -d --rm  --name $CTT_DOCKER_NAME -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY -e "CTT_FAAS_ENABLED=1" -e "OPERA_SSH_USER=ubuntu" -e "OPERA_SSH_IDENTITY_FILE=/tmp/aws-ec2" -p 18080:18080 -v $PWD/tmp/awsec2.pem:/tmp/aws-ec2 $CTT_DOCKER_IMAGE'
                // Clone the radon-ctt-cli repo, install the requirements, execute the test and print the results
                sh '''
                    git clone https://github.com/radon-h2020/radon-ctt-cli.git
                    python3 -m venv .venv 
                    . .venv/bin/activate 
                    pip install -r radon-ctt-cli/requirements.txt
                    python radon-ctt-cli/ctt_cli.py -u $CTT_SERVER_URL -c $CTT_CONFIG_FILE_PATH
                    unzip /tmp/results.zip
                    cat execution.json
                    '''
                // Stop docker container
                sh 'docker stop $CTT_DOCKER_NAME'
            }
        }
        stage('Fetch blueprint from TL') {
            environment {
                //Template Library credentials. (Stored in Jenkins platform )
                TEMPLATE_LIBRARY_USER = credentials('template-lib-user')
                TEMPLATE_LIBRARY_PASS = credentials('template-lib-pass')
                // The reference name used to store a blueprint in Template Library.
                csar_reference = 'thumbnail-generation'
                // The version used to store a blueprint in Template Library.
                csar_version = '1.0.0'
            }
            steps {
                // Authenticate yourself using the environment variables TEMPLATE_LIBRARY_USER & TEMPLATE_LIBRARY_PASS 
                // GET command to download the blueprint from Template Library. (Assuming the the user has access to the specified blueprint)
                sh '''
                    echo $csar_reference
                    echo $csar_version
                    
                    BEARER_TOKEN=$(curl -X POST https://template-library-radon.xlab.si/api/auth/login -H "accept: */*" -H "Content-Type: application/json" -d "{\"username\":\"$TEMPLATE_LIBRARY_USER\",\"password\":\"$TEMPLATE_LIBRARY_PASS\"}" | python3 -c "import sys, json; print(json.load(sys.stdin)['token'])")
                    curl -X GET https://template-library-radon.xlab.si/api/templates/$csar_reference/versions/$csar_version/files -H "accept: application/octet-stream" -H "Authorization: Bearer $BEARER_TOKEN" --output blueprint
                '''
            }
        }
        stage('Opera deploy') {
            environment {
                // The file previously downloaded from Template Library
                DEPLOY_FILE = 'blueprint'       
            }
             steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {  
                    // install the necessary dependencies as pip packages
                    // unwrap the csar and deploy the file.
                    sh '''
                        pip3 install opera==0.6.2 --user
                        PATH="$(python3 -m site --user-base)/bin:${PATH}"
                        pip3 list
                        opera init $DEPLOY_FILE
                        opera deploy 
                    '''
                }
            }
        }
    }
    post { 
        always { 
            cleanWs()
        }
    }
}   