pipeline {
    agent any

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'feature') {
                        sh '''
                            sudo rm -rf /var/www/feature/*
                            sudo cp -r . /var/www/feature/
                            sudo chown -R www-data:www-data /var/www/feature/
                            echo "Feature deployed successfully!"
                        '''
                    }
                    else if (env.BRANCH_NAME == 'main') {
                        sh '''
                            sudo rm -rf /var/www/main/*
                            sudo cp -r . /var/www/main/
                            sudo chown -R www-data:www-data /var/www/main/
                            echo "Main deployed successfully!"
                        '''
                    }
                    else if (env.BRANCH_NAME == 'prefix') {
                        echo 'Prefix branch - only checking'
                    }
                }
            }
        }

    }
}
