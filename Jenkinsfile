pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        
        stage('Tests') {
            parallel {
                // these stages get executed in parallel

                stage ('Unit Tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            test -f "build/index.html"
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage ('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.45.1-jammy'
                            reuseNode true
                            //args '-u root:root' // running container as root -> not a good idea!!!
                            // -> better solution: remove "-g" flag in npm install below (serve is not needed as a global dependency) - instead it gets installed as a locale dependency to the project
                        }
                    }
                    steps {
                        sh '''
                            npx playwright --version # Verify that Playwright is installed
                            npx playwright install # Verify that the browsers are installed
                            npm install serve
                            # serve -s build                    # ... first version (when using -g at npm install -> serve as global dependency)
                            node_modules/.bin/serve -s build &  # serve as local dependency
                                                                # the & character at the end will start the server in the background (so that the jenkins job does not run infinitely)
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            // line generated from Jenkins -> Job -> Configure -> Pipeline Syntax (Sample Step: publishHTML)
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli -g
                    netlify --version
                '''
            }
        }
    }
}
