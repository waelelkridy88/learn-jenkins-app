pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID    = '68e90a47-f0e6-49d2-87d4-dd5bbb0505fe'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        stage('Docker') {
            steps {
                sh 'docker build -t my-playwright .'
            }
        }

        stage('Build') {
            agent { docker { image 'node:18-alpine'; reuseNode true } }
            steps {
                sh '''
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
                            wait-on http://localhost:3000 --timeout 30000
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false,
                                         reportDir: 'playwright-report', reportFiles: 'index.html',
                                         reportName: 'Playwright HTML Report', reportTitles: '',
                                         useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            agent { docker { image 'my-playwright'; reuseNode true } }
            steps {
                sh '''
                    netlify --version
                    netlify status
                    netlify deploy --dir=build --prod
                '''
            }
        }
    }
}