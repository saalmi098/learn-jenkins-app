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
        
        stage ('Test') {
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
                    npm install serve
                    # serve -s build                    # ... first version (when using -g at npm install -> serve as global dependency)
                    node_modules/.bin/serve -s build &  # serve as local dependency
                                                        # the & character at the end will start the server in the background (so that the jenkins job does not run infinitely)
                    sleep 10
                    npx playwright test
                '''
            }
        }
    }

    post {
        always {
            junit 'jest-results/junit.xml'
        }
    }
}
