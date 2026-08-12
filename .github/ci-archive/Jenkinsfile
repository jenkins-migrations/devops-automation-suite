<![CDATA[
// Source: Custom example with proper licensing
// Test case: Input and approval workflow
pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'production'],
            description: 'Target deployment environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Release version'
        )
    }
    
    stages {
        stage('Build') {
            steps {
                echo "Building version ${params.VERSION}"
                sh 'mvn clean package -DskipTests=${params.SKIP_TESTS}'
            }
        }
        
        stage('Test') {
            when {
                not { params.SKIP_TESTS }
            }
            steps {
                sh 'mvn test'
                publishTestResults testResultsPattern: '**/surefire-reports/*.xml'
            }
        }
        
        stage('Deploy to Dev') {
            when {
                expression { params.ENVIRONMENT == 'dev' }
            }
            steps {
                echo "Deploying to development environment"
                sh 'kubectl apply -f k8s/dev/'
            }
        }
        
        stage('Approval for Staging') {
            when {
                expression { params.ENVIRONMENT == 'staging' }
            }
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        input message: 'Deploy to staging?', 
                              ok: 'Deploy',
                              parameters: [
                                  booleanParam(defaultValue: false, 
                                             description: 'Proceed with deployment', 
                                             name: 'PROCEED')
                              ]
                    }
                }
                echo "Deploying to staging environment"
                sh 'kubectl apply -f k8s/staging/'
            }
        }
        
        stage('Production Approval') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                script {
                    timeout(time: 60, unit: 'MINUTES') {
                        def approvers = input message: 'Deploy to production?',
                                           ok: 'Deploy',
                                           submitterParameter: 'APPROVER',
                                           parameters: [
                                               choice(name: 'DEPLOYMENT_TYPE',
                                                     choices: ['blue-green', 'rolling', 'canary'],
                                                     description: 'Deployment strategy')
                                           ]
                        
                        echo "Approved by: ${approvers.APPROVER}"
                        echo "Deployment type: ${approvers.DEPLOYMENT_TYPE}"
                        env.DEPLOYMENT_TYPE = approvers.DEPLOYMENT_TYPE
                    }
                }
                echo "Deploying to production with ${env.DEPLOYMENT_TYPE} strategy"
                sh "kubectl apply -f k8s/production/${env.DEPLOYMENT_TYPE}/"
            }
        }
        
        stage('Smoke Tests') {
            when {
                anyOf {
                    expression { params.ENVIRONMENT == 'staging' }
                    expression { params.ENVIRONMENT == 'production' }
                }
            }
            steps {
                echo "Running smoke tests against ${params.ENVIRONMENT}"
                sh "curl -f http://${params.ENVIRONMENT}.example.com/health"
            }
        }
    }
    
    post {
        success {
            echo "Deployment to ${params.ENVIRONMENT} completed successfully"
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} failed"
        }
    }
}
    ]]>