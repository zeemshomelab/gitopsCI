pipeline {
    agent  any
    
    environment{
        DOCKERHUB_USERNAME = "zeemlinux"
        APP_NAME = "gitops_argocd_CI"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}" + "/" + "${APP_NAME}"
        REGISTRY_CREDS = 'dockertoken'
        SONAR_LOGIN = credentials('sonartoken')
    }
    stages {
        stage('Clean Workspace') {
            steps {
                script{
                   
                   cleanWs()
                }
            }
        }
        stage ('Check-Git-Secrets') {
		    steps {
                script{
                    sh 'rm trufflesecurity/trufflehog:latest || true'
		     //       sh 'docker pull trufflesecurity/trufflehog:latest'
                    sh 'docker run -t -v "$PWD:/pwd" ghcr.io/trufflesecurity/trufflehog:latest github --repo https://github.com/zeemshomelab/gitopsCI.git --debug > trufflehog'
		            sh 'cat trufflehog'

                }
	         
	       }
        } 
        stage(' Checkout Code From SCM'){
            steps{
                script{
                    git credentialsId: 'github-token',
                    url: 'https://github.com/zeemshomelab/gitopsCI.git',
                    branch: 'main'
                }
            }
        }
        stage('Build Docker Image'){
            steps{
                script{
                    
                    docker_image = docker.build "${IMAGE_NAME}"
                }
            }
        }
        stage ('Source-Composition-Analysis (SCA)') {
		    steps {
		     sh 'rm owasp-* || true'
		     sh 'wget https://raw.githubusercontent.com/devopssecure/webapp/master/owasp-dependency-check.sh'	
		     sh 'chmod +x owasp-dependency-check.sh'
		     sh 'bash owasp-dependency-check.sh'
		     sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
		   }
	   }
       stage ('Network Mapper (Port Scan)') {
		    steps {
			sh 'rm nmap* || true'
			sh 'docker run --rm -v "$(pwd)":/data uzyexe/nmap -sS -sV -oX nmap https://jenkins.zeemshomelab.com'
			sh 'cat nmap'
		    }
	    }
        // stage('SonarQube Analysis') {
            // steps {
            //     script {
            //         withSonarQubeEnv('gitops') {
            //             sh 'sonar-scanner \
            //                 -Dsonar.projectKey=gitops \
            //                 -Dsonar.sources=. \
            //                 -Dsonar.language=py \
            //                 -Dsonar.python.flake8.reportPaths=target/sonar/report.txt \
            //                 -Dsonar.python.coverage.reportPaths=coverage.xml \
            //                 -Dsonar.python.bandit.reportPaths=bandit_report.json \
            //                 -Dsonar.python.pylint.reportPaths=pylint_report.txt'
            //         }
            //     }
            // }
           // }
        stage('Push Docker Image'){
            steps{
                script{
                    docker.withRegistry('',REGISTRY_CREDS){
                        docker_image.push("$BUIlD_NUMBER")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage('Delete Docker Images'){
            steps{
                script{
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
         stage('Updating Kubernetes Deployment File'){
            steps{
                script{
                    sh """
                    cat deployment.yml
                    sed -i 's/${APP_NAME}.*/${APP_NAME}:${IMAGE_TAG}/g' deployment.yml
                    cat deployment.yml

                    """
                }
            }
        }
        stage('Website Vulnerability Scanning (Nikto Scan)'){
            steps{
                script{
                    sh 'rm nikto-output.xml || true'
			        sh 'docker pull secfigo/nikto:latest'
			        sh 'docker run --user $(id -u):$(id -g) --rm -v $(pwd):/report -i secfigo/nikto:latest -h https://jenkins.local.zeemshomelab.com -output /report/nikto-output.xml'
			        sh 'cat nikto-output.xml'
                }
            }
        }
        stage ('Dynamic Application Security Testing (DAST)') {
		  
		    	steps {
                    script{
                       sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t https://jenkins.local.zeemshomelab.com || true'
                    }
			    }
			}    
        stage(' Push the Changed Deployment File Git '){
            steps{
                script{
                    sh """
                     git config --global user.name "zeemshomelab"
                     git config --global user.email zeems.homelab@gmail.com
                     git add deployment.yml
                     git commit -m "updated deployment.yml file"
                    """
                    withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {

                     sh "git push https://github.com/zeemshomelab/gitopsCI.git main"

                   }
                   
                }
            }
       }
       
    }
}   