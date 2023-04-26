pipeline {
    agent none

    stages {
        stage('worker-build') {
            when {
                changeset "**/worker/**"
            }
	    agent {
	      docker{
	       image 'maven:3.6.1-jdk-8-slim'
	       args '-v $HOME/.m2:/root/.m2'
	      }
	    }
            steps {
                echo 'building worker app'
		dir(path: 'worker'){
		  sh 'mvn compile'
		}
            }
        }
        stage('worker-test') {
            when {
                changeset "**/worker/**"
            }
	    agent {
	      docker{
	       image 'maven:3.6.1-jdk-8-slim'
	       args '-v $HOME/.m2:/root/.m2'
	      }
	    }
            steps {
                echo 'Running Unit Tests on worker app'
		dir(path: 'worker'){
		  sh 'mvn clean test'
		}
            }
        }
        stage('worker-package') {
            when {
                changeset "**/worker/**"
		branch 'master'
            }
	    agent {
	      docker{
	       image 'maven:3.6.1-jdk-8-slim'
	       args '-v $HOME/.m2:/root/.m2'
	      }
	    }
            steps {
                echo 'Packaging worker app into a jar file'
		dir(path: 'worker'){
		 sh 'mvn package -DskipTests'
     		 archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
		}
            }
        }


    stage('worker-docker-package') {
      agent any
      when {
        changeset '**/worker/**'
        branch 'master'
      }
      environment{
        def BRANCH_NAME = "${env.BRANCH_NAME.replaceFirst(/\//,'.'}"
      }
      steps {
        echo 'Packaging worker app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def workerImage = docker.build("karthikeyanr/worker:v${env.BUILD_ID}", './worker')
            workerImage.push()
            workerImage.push("${BRANCH_NAME}")
            workerImage.push('latest')
          }
        }

      }
    }


   stage('result-build') {
      agent {
        docker {
          image 'node:8.16.0-alpine'
        }

      }
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Compiling result app..'
        dir(path: 'result') {
          sh 'npm install'
        }

      }
    }

    stage('result-test') {
      agent {
        docker {
          image 'node:8.16.0-alpine'
        }

      }
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Running Unit Tests on result app..'
        dir(path: 'result') {
          sh 'npm install'
          sh 'npm test'
        }

      }
    }

    stage('result-docker-package') {
      agent any
      when {
        changeset '**/result/**'
        branch 'master'
      }
      environment{
        def BRANCH_NAME = "${env.BRANCH_NAME.replaceFirst(/\//,'.'}"
      }
      steps {
        echo 'Packaging result app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def resultImage = docker.build("karthikeyanr/result:v${env.BUILD_ID}", './result')
            resultImage.push()
            resultImage.push("${BRANCH_NAME}")
            resultImage.push('latest')
          }
        }
      }
    }



    stage('vote-build') {
      agent {
        docker {
          image 'python:2.7.16-slim'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Compiling vote app.'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
        }

      }
    }

    stage('vote-test') {
      agent {
        docker {
          image 'python:2.7.16-slim'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Running Unit Tests on vote app.'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
          sh 'nosetests -v'
        }

      }
    }

    stage('vote integration'){ 
    agent any 
    when{ 
      changeset "**/vote/**" 
      branch 'master' 
    } 
    steps{ 
      echo 'Running Integration Tests on vote app' 
      dir('vote'){ 
        sh 'sh integration_test.sh' 
      } 
    } 
} 


    stage('vote-docker-package') {
      agent any
      environment{
        def BRANCH_NAME = "${env.BRANCH_NAME.replaceFirst(/\//,'.'}"
      }
      steps {
        echo 'Packaging vote app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
            def voteImage = docker.build("karthikeyanr/vote:${env.GIT_COMMIT}", "./vote")
            voteImage.push()
            voteImage.push("${BRANCH_NAME}")
            voteImage.push("latest")
          }
        }

      }
    }

    stage('Sonarqube') {
      agent any
      when{
        branch 'master'
      }
      // tools {
       // jdk "JDK11" // the name you have given the JDK installation in Global Tool Configuration
     // }

      environment{
        sonarpath = tool 'SonarScanner'
      }

      steps {
            echo 'Running Sonarqube Analysis..'
            withSonarQubeEnv('sonar-instavote') {
              sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
            }
      }
    }


    stage("Quality Gate") {
        steps {
            timeout(time: 1, unit: 'HOURS') {
                // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                // true = set pipeline to UNSTABLE, false = don't
                waitForQualityGate abortPipeline: true
            }
        }
    }


   stage('deploy to dev'){
          agent any
          when{
            branch 'master'
          }
          steps{
            echo 'Deploy instavote app with docker compose'
            sh 'docker-compose up -d'
          }
      }


 stage('Trigger deployment') {
      agent any
      environment{
        def GIT_COMMIT = "${env.GIT_COMMIT}"
      }
      steps{
        echo "${GIT_COMMIT}"
        echo "triggering deployment"
        // passing variables to job deployment run by vote-deploy repository Jenkinsfile
        build job: 'deployment', parameters: [string(name: 'DOCKERTAG', value: GIT_COMMIT)]
      }    
   }

    }

   post {
    always {
     echo 'Building multibranch pipeline for worker is completed..' 
    }
    failure{
      slackSend (channel: "instavote-cd", message: "Build Failed - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)") 
    }
    success{
      slackSend (channel: "instavote-cd", message: "Build Succeeded - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)") 
    }
   }

}
