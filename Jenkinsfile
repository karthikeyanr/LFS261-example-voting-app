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
		dir('worker'){
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
		dir('worker'){
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
		dir('worker'){
		 sh 'mvn package -DskipTests'
     		 archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
		}
            }
        }

        stage('result-build') {
	    when {
		changeset "**/result/**"
	    }
            steps {
                echo 'Compiling result app'
		dir('worker'){
		  sh 'npm install'
		}
            }
        }

        stage('result-test') {
	    when {
		changeset "**/result/**"
	    }
            steps {
                echo 'Running Unit Tests on result app'
		dir('result'){
		  sh 'npm install'
		  sh 'npm test'
		}
            }
        }

        stage('vote-build') {
	    when {
		changeset "**/vote/**"
	    }
            steps {
                echo 'Running vote app'
		dir('vote'){
		  sh 'pip install -r requirements.txt'
		  sh 'gunicorn app:app -b 0.0.0.0:80 --log-file - --access-logfile - --workers 4 --keep-alive 0'
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
