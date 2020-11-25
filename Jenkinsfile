def git_repo = "HelloWorld"

//====do not make any changes below this line====
pipeline {
     environment {
        NEWRELIC_API_KEY = credentials('newrelic-api-key')
    }
	options {
		buildDiscarder(logRotator(numToKeepStr: '$builds'))
			}
       agent {label 'master'}
       stages {

	  stage('Code Qulity Scan') {
            steps {
             withSonarQubeEnv('SonarQube') {	

              sh '''/var/lib/jenkins/.dotnet/tools/dotnet-sonarscanner begin /k:"Kona-Members-Service" /v:1.0 /d:sonar.cs.vstest.reportsPaths="**/*.trx" /d:sonar.exclusions="viewModels/*.cs,Models/*.cs,Entity/*.cs,Migrations/*.cs,Mappers/*.cs,SeedData/*.cs,Program.cs,Startup.cs,appsettings.json,Dockerfile,ServiceConfiguration.cs,AggregatorModels/*,*.Designer.cs,*.resx,Repository/Context/*.cs,Tests/*.cs" /d:sonar.cs.opencover.reportsPaths="**/lcov.opencover.xml"'''
              sh '''dotnet build ./API/API.csproj'''
              sh '''/var/lib/jenkins/.dotnet/tools/dotnet-sonarscanner end'''   
	     }
           }
       }   
       
       /* stage ('Code Quality Check') {
        steps {
                timeout(time: 10, unit: 'MINUTES') { // Just in case something goes wrong, pipeline will be killed after a timeout
                script {
                    def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }   	
       }   */ 
	
         stage('Build Docker Image') {
            steps {
		script {
		if (env.GIT_BRANCH == 'origin/develop') 
				  {		
		     
                      		app = docker.build("${proj}", '--build-arg NEWRELIC_LICENSEKEY=${NEWRELIC_API_KEY} --no-cache .')
				  }
                      }
                  }
		 }		 
         stage("Image Securiy Scan"){
	        steps {	
		      script {
			 if (env.GIT_BRANCH == 'origin/develop') 
				  {     
		         
				sh '''
					if [ ! -e Kona-AutomationScripts ] 
					 then
					   git clone git@github.com:brierley/Kona-AutomationScripts.git
					fi
					cd Kona-AutomationScripts && chmod 755 clair-scan.sh && sh clair-scan.sh memberservice;
				'''
				  }
			  }		
		    }	
		  } 
        stage('Push Docker Image to ECR') {
           steps {
            script {
		if (env.GIT_BRANCH == 'origin/develop') 
				  {		
			  docker.withRegistry("https://${ECR}", "ecr:$AWS_REGION:awscreds") {
                   			app.push("${env.BUILD_NUMBER}")
                    }
				  }		  	
                }
            }
          }
  	stage('Update Build Number') {
           steps {
		     script {
			if (env.GIT_BRANCH == 'origin/develop') 
				  {	
					sh (""" 
					sed -i  "s/latest/$currentBuild.number/1" ${deployfile}
						""")
				  }		
                 	    }
         	} 
	}	
	stage('Configmap Preperation') {
     	 steps {
    		script {    
		if (env.GIT_BRANCH == 'origin/develop') 
				  {
    		withCredentials([string(credentialsId: 'members-key',
                            variable: 'secretText')]) {
     		secret_key = "${secretText}"
		
    		}
    	    }
	    	sh """sed -i  "s/xxxxxxxx/${secret_key}/g" ${configfile}"""
   		  	}
			}
     	}
	
  	 stage('Deploy to Kubernetes') {
         steps {
		script {
			if (env.GIT_BRANCH == 'origin/develop') 
				  {
         withKubeConfig([credentialsId: "${kube_key}"]) {
	
         sh 'kubectl config set-context $(kubectl config current-context) --namespace=brierley-kona'
	 sh """kubectl apply -f $configfile"""
         sh """kubectl apply -f $deployfile"""
	 sh "echo $currentBuild.number > version.txt"
	 		
							 }
				 }
            } 
        }
     }
     
     stage('Archive Version') {
         steps {
     	     archiveArtifacts artifacts: 'version.txt'
	     }
	}     
      stage('Tagging') {
           steps {
		      script {
		      try {
			if (env.GIT_BRANCH == 'origin/develop') 
				  {	
					sh("git tag $BUILD_NUMBER")
					sh("git push origin $BUILD_NUMBER")
				  }
			   }	  
			 catch(all) {
				sh 'exit 0'
        			echo "Tag already exist"
    				    } 	 	 	  
		      	     	
	   		  }
		  }
		  	 	
	}	
		  	 	
}
       post {
       always {
        	 office365ConnectorSend webhookUrl:'$TEAMS_NOTIFY'      
     }	 
      }
}
