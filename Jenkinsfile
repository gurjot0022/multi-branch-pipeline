pipeline {
    agent any

    stages {

        stage('Clean') {
            steps {
                deleteDir()
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'cp -r . /var/www/main/'
                        echo 'Main branch - deployed to main server'
                    } 
                    else if (env.BRANCH_NAME == 'feature') {
                        sh 'cp -r . /var/www/feature/'
                        echo 'Feature branch - deployed to feature server'
                    } 
                    else if (env.BRANCH_NAME == 'prefix') {
                        echo 'Prefix branch - only checking, not deploying'
                    } 
                    else {
                        echo "Unknown branch: ${env.BRANCH_NAME} - skipping"
                    }
                }
            }
        }

    }
}
