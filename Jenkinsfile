pipeline {
    agent any
    stages {

        stage('Clean') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def branch = env.GIT_BRANCH?.replaceAll('origin/', '').trim()
                    
                    echo "Branch hai: ${branch}"

                    if (branch == 'feature') {
                        sh '''
                            sudo rm -rf /var/www/feature/*
                            sudo cp -r * /var/www/feature/
                        '''
                        echo 'Feature deployed!'
                    }
                    else if (branch == 'main') {
                        sh '''
                            sudo rm -rf /var/www/main/*
                            sudo cp -r * /var/www/main/
                        '''
                        echo 'Main deployed!'
                    }
                    else if (branch == 'prefix') {
                        echo 'Prefix - only checking'
                    }
                }
            }
        }

    }
}
