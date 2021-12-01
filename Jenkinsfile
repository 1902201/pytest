pipeline {
    agent any
    environment {
        // CI set to true to allow it to run in "non-watch" (i.e. non-interactive) mode
        CI = 'true'
//         HOST_IP = "${HOST_IP}".0
//         HOST_PORT = "${HOST_PORT}"
    }
    stages {
        stage('Build') { 
            steps {
                script {
                    try { sh 'yes | docker image prune' }
                    catch (Exception e) { echo "no dangling images deleted" }
                    try { sh 'yes | docker image prune -a' }
                    catch (Exception e) { echo "no images w containers deleted" }
                    try { sh 'yes | docker container prune' }
                    catch (Exception e) { echo "no unused containers deleted" }
                }
                // ensure latest image is being build
                sh 'docker build -t theimg:latest .'
            }
        }
        /* OWASP Dependency Check */
        stage('OWASP-DC') {
            agent { 
                docker {
                    image 'theimg:latest'
                }
            }
            steps {
                dependencyCheck additionalArguments: '--format HTML --format XML', odcInstallation: 'OWASP-DC'
                
                // --suppression suppression.xml  // suppress warnings xml 
                // --enableExperimental // to test on python files
                // --log odc.log // generate log file
            }
            post {
                always {
                    dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    // untested codes of x08 below, do not uncomment unless you know what u doing. 
                    // recordIssues enabledForFailure: true, tool: analysisParser(pattern: "dependency-check-report.xml", id: "owasp-dependency-check")
                    // recordIssues enabledForFailure: true, tool: pyLint(pattern: 'pylint.log')
                
                }
            }
        }
        /* Selenium portion */
        stage('unit/sel test') {
            parallel {
                stage('Deploy') {
                    agent {
                        docker { image 'theimg:latest' }
                    }
                    steps {
                        script {
                            try {sh 'yes | docker stop thecon'}
                            catch (Exception e) {echo "no container to stop"}

                            try {sh 'yes | docker rm thecon'}
                            catch (Exception e) {echo "no container to remove"}
                        }

                        sh """docker run -u root -d --rm -p 5000:5000 --name thecon \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v "$HOME":/home \
                        -e VIRTUAL_PORT=5000 \
                        theimg"""
                    }
                }
                stage('Headless Browser Test') {
                    agent {
                        docker { image 'theimg:latest' }
                    }
                    steps {
                        sh 'nohup flask run & sleep 1'
                        // generate x07 pytest
                        sh 'pytest -s -rA --junitxml=test-report.xml'

                        // generate x09 report
                        script {
                            def scannerHome = tool 'SonarQube';
                            withSonarQubeEnv('SonarQube') {
                                // rmb to change the "projectKey=your_project_name"
                                sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=test -Dsonar.sources=."
                            }
                        }
                        
                        input message: 'Finished using the web site? (Click "Proceed" to continue)'
                        
                        script {
                            try {sh 'pkill -f flask'}
                            catch (Exception e) {echo "no process to kill"}

                            try {sh 'yes | docker stop thecon'}
                            catch (Exception e) {echo "no container to stop"}

                            try {sh 'yes | docker rm thecon'}
                            catch (Exception e) {echo "no container to remove"}
                        }
                    }
                    post {
                        always {
                            junit testResults: 'test-report.xml'
                            recordIssues enabledForFailure: true, tool: sonarQube()	
                        }
                    }
                }
            }
        }

        /* Warnings X08 is not well done, pls refer to notes for doc
        it is too dynamic
        */ 
        // stage('warnings') {
            
        //     agent {
        //         docker { image 'theimg:latest' }
        //     }
        //     steps {
        //         sh 'nohup flask run & sleep 1'
        //         sh 'pytest -s -rA --junitxml=warn-report.xml'
        //         echo "hello"

        //     }
        //     post {
        //         always {
                    
        //             // recordIssues enabledForFailure: true, tools: [mavenConsole(), java(), javaDoc()]
                
        //             recordIssues enabledForFailure: true, tool: codeAnalysis()	
        //             recordIssues enabledForFailure: true, tool: codeChecker()
        //             recordIssues enabledForFailure: true, tool: dockerLint()
        //         }
        //     }
        // }

        /* X09 SonarQube */ 
         stage('SonarQube') {
             agent {
                 docker { image 'theimg:latest' }
             }
             steps {
                 script {
                     def scannerHome = tool 'SonarQube';
                     withSonarQubeEnv('SonarQube') {
                         sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=test -Dsonar.sources=."
                     }
                 }
             }
             post {
                 always {
                     recordIssues enabledForFailure: true, tool: sonarQube()	
                 }
             }
         }
    }
}
