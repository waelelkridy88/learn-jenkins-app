pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID    = '68e90a47-f0e6-49d2-87d4-dd5bbb0505fe'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION  = "1.0.$BUILD_ID"
    }

    stages {
        stage('Build') {
            agent { docker { image 'node:18-alpine'; reuseNode true } }
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la build
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent { docker { image 'node:18-alpine'; reuseNode true } }
                    steps {
                        sh 'npm test'
                    }
                    post {
                        always { junit 'jest-results/junit.xml' }
                    }
                }

                stage('E2E') {
                    agent { docker { image 'my-playwright'; reuseNode true } }
                    steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false,
                                         reportDir: 'playwright-report', reportFiles: 'index.html',
                                         reportName: 'Local E2E', reportTitles: '',
                                         useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent { docker { image 'my-playwright'; reuseNode true } }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --site=$NETLIFY_SITE_ID --json > deploy-output.json
                    export CI_ENVIRONMENT_URL=$(node-jq -r '.deploy_url' deploy-output.json)
                    echo "Staging URL: $CI_ENVIRONMENT_URL"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false,
                                 reportDir: 'playwright-report', reportFiles: 'index.html',
                                 reportName: 'Staging E2E', reportTitles: '',
                                 useWrapperFileDirectly: true])
                }
            }
        }

        stage('Deploy prod') {
            agent { docker { image 'my-playwright'; reuseNode true } }

            environment {
                CI_ENVIRONMENT_URL = 'https://verdant-zuccutto-09479c.netlify.app'
            }

            steps {
                sh '''
                    netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --site=$NETLIFY_SITE_ID --prod
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false,
                                 reportDir: 'playwright-report', reportFiles: 'index.html',
                                 reportName: 'Prod E2E', reportTitles: '',
                                 useWrapperFileDirectly: true])
                }
            }
        }
    }
}