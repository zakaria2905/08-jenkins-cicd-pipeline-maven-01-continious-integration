// PIPELINE STAGES

// Publish Artifact to Repository Manager (Nexus)

pipeline{
    agent    any    
    tools  {
        maven "my-maven-3"
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "host.docker.internal:8081"
        NEXUS_REPOSITORY = "mylocalrepo-snapshots"
        NEXUS_CREDENTIAL_ID = "nexus-user-credentials"
        MOUNTPATH = "//C/DevOps-Workspace/Jenkins CI-CD/08-jenkins-cicd-pipeline-maven-01-continious-integration/src/pt"
    }
    
    stages{
        // Clone from Git
        stage("Clone App from Git"){
            steps{
                echo "====++++  Clone App from Git A++++===="
                git branch:"master", url: "https://github.com/zakaria2905/08-jenkins-cicd-pipeline-maven-01-continious-integration.git"
            }          
        }
        // Build and Unit Test (Maven/JUnit)
        stage("Build and Unit Test (Maven/JUnit)"){
            steps{
                echo "====++++  Build and Unit Test (Maven/JUnit) ++++===="
                sh "mvn clean package"
                junit '**/target/surefire-reports/TEST-*.xml' // Archive the test reports
            }           
        }
        // Static Code Analysis (SonarQube)
        stage("Static Code Analysis (SonarQube)"){
            steps{
                 echo "====++++  Static Code Analysis (SonarQube) ++++===="
                
                 withSonarQubeEnv('my_sonarqube_in_docker') {  
                      sh "mvn clean verify -DskipITs=true sonar:sonar -Dsonar.host.url=http://host.docker.internal:9000   -Dsonar.projectName=08-jenkins-cicd-pipeline-maven-01-continious-integration -Dsonar.projectKey=08-jenkins-cicd-pipeline-maven-01-continious-integration -Dsonar.projectVersion=$BUILD_NUMBER";
                 }  
            }           
        }
        stage("Checking the Quality Gate") {
                  steps {
                       echo "====++++  Checking the returned SonarQube Quality Gate ++++===="
                      timeout(time: 1, unit: 'HOURS') {
                          // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                          // true = set pipeline to UNSTABLE, false = don't
                          waitForQualityGate abortPipeline: true
                      }
                  }
        }
          // Integration Test (Maven/JUnit)
        stage("Integration Test (Maven/JUnit)"){
            steps{
                echo "====++++  Integration Test (Maven/JUnit) ++++===="
                sh "mvn clean verify -Dsurefire.skip=true"    // Do not repeat the unit tests, they have been already done
                junit "**/target/failsafe-reports/*.xml" // Archive the test reports
            }         
        }
        // Publication du snapshot sur nexus
        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
        // Start the Staging Server (Tomcat)
        stage ('Start the application under test ') {
	        
	          steps {
	              sh "docker run --name staging_server -p 9090:8080  -d tomcat:9.0"
	              sh "docker cp $WORKSPACE/target/greetings-0.1-SNAPSHOT.war staging_server:/usr/local/tomcat/webapps/hello.war"
               }
        }
        // Deploy the app on the Staiging Server
        stage('Performance Testing'){
	   	   
	   	    steps {
          
               sh "docker run -d --link staging_server:staging_server  -v ${MOUNTPATH}://jmeter   egaillardon/jmeter -n -t Hello_World_Test_Plan.jmx -l //jmeter/test_report.jtl"
      
         	  }
	    }

        // De performance tests on the app
        stage ('Publish the performance reports') {
	        
	          steps {
	            perfReport sourceDataFiles: "$WORKSPACE/src/pt/test_report.jtl"
	          }
	    }

        // Promote the app on Nexus from the snapshot to release
        stage ('Clean Up : Stop the application under test') {
	        
	          steps {
	                sh "docker stop   myapp_under_test && docker rm  staging_server "
                    echo " STOPPED The Tomcat app Under test"
               }
	    }
    }
   
}

