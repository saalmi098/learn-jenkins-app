pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'd45b949d-ed4c-4192-ab8d-d79b2ba028c6' // coming from Netlify - Site configuraiton
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

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
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
                // if netlify deploy does not have --prod flag, it will automatically create a temporary preview environment
                // and we will use this one as our staging environment
            }
            script {
                env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
            }
        }

        stage ('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.45.1-jammy'
                    reuseNode true
                    //args '-u root:root' // running container as root -> not a good idea!!!
                    // -> better solution: remove "-g" flag in npm install below (serve is not needed as a global dependency) - instead it gets installed as a locale dependency to the project
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright --version
                    npx playwright install
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    // line generated from Jenkins -> Job -> Configure -> Pipeline Syntax (Sample Step: publishHTML)
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E (Staging)', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                // if nothing happens after 15 minutes, the execution will be interrupted (generated with script generator)
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy production') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage ('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.45.1-jammy'
                    reuseNode true
                    //args '-u root:root' // running container as root -> not a good idea!!!
                    // -> better solution: remove "-g" flag in npm install below (serve is not needed as a global dependency) - instead it gets installed as a locale dependency to the project
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://gleeful-dieffenbachia-45e639.netlify.app'
            }

            steps {
                sh '''
                    npx playwright --version
                    npx playwright install
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    // line generated from Jenkins -> Job -> Configure -> Pipeline Syntax (Sample Step: publishHTML)
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E (Prod)', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
