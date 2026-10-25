pipeline {
  agent any

  parameters {
    choice(
      name: 'SERVICE',
      choices: ['frontend', 'cartservice', 'both'],
      description: 'Which service to build'
    )
    booleanParam(
      name: 'DEPLOY',
      defaultValue: false,
      description: 'Deploy to EKS (cluster must exist)'
    )
    string(
      name: 'IMAGE_TAG',
      defaultValue: '',
      description: 'Optional: deploy this existing tag instead of the new build (leave empty for normal runs)'
    )
  }

  environment {
    AWS_REGION = 'ap-southeast-1'
    REGISTRY = '<AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com'
    CLUSTER = 'eks-lab'
    NAMESPACE = 'boutique'
    TAG = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('ECR Login') {
      steps {
        sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $REGISTRY'
      }
    }

    stage('Build and Push') {
      steps {
        script {
          def services = (params.SERVICE == 'both') ?
            ['frontend', 'cartservice'] : [params.SERVICE]

          services.each { svc ->
            def ctx = (svc == 'cartservice') ?
              'cartservice/src' : svc
            def image = "${env.REGISTRY}/boutique/${svc}"

            sh "docker build -t ${image}:${env.TAG} -t ${image}:latest ${ctx}"
            sh "docker push ${image}:${env.TAG}"
            sh "docker push ${image}:latest"
          }
        }
      }
    }

    stage('Deploy to EKS') {
      when {
        expression { params.DEPLOY }
      }
      steps {
        script {
          sh 'aws eks update-kubeconfig --name $CLUSTER --region $AWS_REGION'

          def services = (params.SERVICE == 'both') ?
            ['frontend', 'cartservice'] : [params.SERVICE]
          def deployTag = params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : env.TAG

          services.each { svc ->
            def image = "${env.REGISTRY}/boutique/${svc}:${deployTag}"

            sh "kubectl -n ${env.NAMESPACE} set image deployment/${svc} server=${image}"

            def rc = sh(
              script: "kubectl -n ${env.NAMESPACE} rollout status deployment/${svc} --timeout=180s",
              returnStatus: true
            )

            if (rc != 0) {
              echo "Rollout of ${svc} (${deployTag}) failed - rolling back to the previous revision"
              sh "kubectl -n ${env.NAMESPACE} rollout undo deployment/${svc}"
              sh "kubectl -n ${env.NAMESPACE} rollout status deployment/${svc} --timeout=180s"
              error("Deploy of ${svc}:${deployTag} failed and was rolled back")
            }
          }

          sh "kubectl -n ${env.NAMESPACE} get deployments frontend cartservice"
        }
      }
    }
  }

  post {
    always {
      sh 'docker image prune -af --filter "until=24h" || true'
    }
  }
}
