def podlabel = "jenkins-${UUID.randomUUID().toString()}"
pipeline { 
 agent { 
   kubernetes {
     label podlabel
     yaml """
kind: Pod
metadata:
  name: jenkins-slave
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: aws-secret
        mountPath: /root/.aws/
      - name: docker-registry-config
        mountPath: /kaniko/.docker
  - name: kubectl
    image: <account-id>.dkr.ecr.eu-central-1.amazonaws.com/jenkins/deploy-agent:latest
    imagePullPolicy: Always
    tty: true
    volumeMounts:
      - name: kube-config
        mountPath: /.kube/config
  restartPolicy: Never
  volumes:
    - name: kube-config
      secret:
        secretName: kube-cluster-config
    - name: aws-secret
      secret:
        secretName: aws-secret
    - name: docker-registry-config
      configMap:
        name: docker-registry-config
"""
   }  
  }

 environment {
  NPM_TOKEN = credentials('NPM_TOKEN')
  NODE_VERSION = "v8.10.0"
  
  BUILD_ENVIRONMENT = "test"
  DEPLOY_ENVIRONMENT = "no-environment-specified"

  //Test credentials
  DATABASE_PASSWORD = credentials('TEST_DB_PASS')
  AWS_SECRET_ACCESS_KEY = credentials('TEST_AWS_SECRET_ACCESS_KEY')

  //Commit values
  repo = checkout scm
  AUTHOR = sh(script: "git log -1 --pretty=%an ${repo.GIT_COMMIT}", returnStdout: true).trim()
  COMMIT_MESSAGE = sh(script: "git log -1 --pretty=%B ${repo.GIT_COMMIT}", returnStdout: true).trim()
  DEPLOYMENT_TAG = "Commit by (${AUTHOR}): ${COMMIT_MESSAGE}"
  COMMIT_TAG = "${repo.GIT_COMMIT[0..6]}"
 }

 stages {

  stage('Branch: Develop') {
    when { branch 'develop' }
    steps {script { BUILD_ENVIRONMENT = "test"
                    DEPLOY_ENVIRONMENT = "test-app" 
  }}}

  stage('Build') {
   environment {
     PATH = "/busybox:/kaniko:$PATH"
   }
   steps {
     
     container(name: 'kaniko', shell: '/busybox/sh') {

        sh """#!/busybox/sh
       
        echo " \n *** Building Nginx *** \n"
        /kaniko/executor -c `pwd` -f `pwd`/nginx/Dockerfile --cleanup --cache=true --destination=<account-id>.dkr.ecr.eu-central-1.amazonaws.com/org/nginx:${BUILD_ENVIRONMENT}-latest --destination=<account-id>.dkr.ecr.eu-central-1.amazonaws.com/org/nginx:${BUILD_ENVIRONMENT}-${COMMIT_TAG}
  

        echo " \n *** Building FPM *** \n"
        /kaniko/executor -c `pwd` -f `pwd` /php/Dockerfile --build-arg npm_token=${NPM_TOKEN} --build-arg node_version=${NODE_VERSION} --cache=true --destination=<account-id>.dkr.ecr.eu-central-1.amazonaws.com/org/php:${BUILD_ENVIRONMENT}-latest --destination=<account-id>.dkr.ecr.eu-central-1.amazonaws.com/org/php:${BUILD_ENVIRONMENT}-${COMMIT_TAG}

      """
     }
    }
   }
  
   stage('Deploy') {

        //Deploy only if develop or release branch
         when { branch 'develop'; }

          steps {

                  container(name: 'kubectl') {
                  sh """

                  echo 'Updating ConfigMap (run time env variables)'
                  kubectl create configmap runTimeVariables --from-env-file=deployment/${BUILD_ENVIRONMENT}/env.dist.${BUILD_ENVIRONMENT} --dry-run -o yaml | kubectl replace -f - --namespace=${DEPLOY_ENVIRONMENT}

                  echo 'Applying changes to pods'
                  kubectl apply -f ./deployment/${BUILD_ENVIRONMENT}/org-${BUILD_ENVIRONMENT}-deploy.yml --namespace=${DEPLOY_ENVIRONMENT}

                  echo 'Updating FPM pod image'
                  kubectl set image deployment/org php=<account-id>.dkr.ecr.eu-central-1.amazonaws.com/org/php:${BUILD_ENVIRONMENT}-${COMMIT_TAG} --namespace=${DEPLOY_ENVIRONMENT}

                  echo 'Updating Nginx pod image'
                  kubectl set image deployment/org nginx=<account-id>.dkr.ecr.eu-central-1.amazonaws.com/org/nginx:${BUILD_ENVIRONMENT}-${COMMIT_TAG} --namespace=${DEPLOY_ENVIRONMENT}

                  echo 'Tagging deployment with commit ID and commit message'
                  kubectl annotate deployment/org kubernetes.io/change-cause='${DEPLOYMENT_TAG}' --namespace=${DEPLOY_ENVIRONMENT}

                  kubectl rollout history deployment -n ${DEPLOY_ENVIRONMENT}
                  """
          }
        }
      }
    }
    post {

       failure {
          mail to:"coder@org.com",
          subject:"Build ${currentBuild.fullDisplayName} failed",
          body: "Uh ow! Someone messed up. ${env.BUILD_URL} ${currentBuild.fullDisplayName}"
       }
    } 
  }
