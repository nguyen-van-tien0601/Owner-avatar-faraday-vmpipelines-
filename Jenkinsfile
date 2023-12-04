pipeline {
	agent none
	stages {
		stage(‘Build’) {
		agent {
		docker {
		image ‘python:3.9.1’
		args ‘-u root:root’
		}
		}
		steps {
		sh ‘pip install -r requirements.txt’
		sh ‘bandit -r . -f xml -o flaskapp_faraday_bandit.xml || true’
		stash name:
		‘flaskapp_faraday_bandit.xml’, includes: ‘flaskapp_faraday_bandit.xml’
		}
		}
	stage(‘Deploy’) {
	agent any
		steps {
		withCredentials([
		string(credentialsId: ‘HEROKU_API_KEY’,
		variable: ‘HEROKU_API_KEY’),
		string(credentialsId: ‘HEROKU_APP_NAME’,
		variable: ‘HEROKU_APP_NAME’),
		]) {
		script {
		try {
		sh ‘git remote add heroku
		https:|/heroku:$HEROKU_API_KEY@git.heroku.com/$HEROKU_APP_NAME.git’
		}
		catch(Exception e) {
		echo ‘Remote heroku already exists’
		}
		}
		sh ‘git push heroku HEAD:master -f’
		}
		  }
	}
	stage(‘Scan’) {
	agent any
		steps {
		withCredentials([
		string(credentialsId: ‘ZAP_SCAN_URL’, variable: ‘ZAP_SCAN_URL’)
		]) {
		sh ‘docker run -v $WORKSPACE:/zap/wrk/:
		rw |-network=host -t owasp/zap2docker-stable zap-baseline.
		py -t $ZAP_SCAN_URL -x flaskapp_faraday_zap.xml || echo 0’
		stash name: ‘flaskapp_faraday_zap.xml’,
		includes: ‘flaskapp_faraday_zap.xml’
		}
		}
	}
	stage(‘Upload’) {
	agent any
		steps {
		withCredentials([
		string(credentialsId: ‘FARADAY_URL’, variable: ‘FARADAY_URL’),
		string(credentialsId:
		‘FARADAY_USERNAME’, variable: ‘FARADAY_USERNAME’),
		string(credentialsId:
		‘FARADAY_PASSWORD’, variable: ‘FARADAY_PASSWORD’)
		]) {
		unstash ‘flaskapp_faraday_bandit.xml’
		unstash ‘flaskapp_faraday_zap.xml’
		script {
		CURRENT_DATE = (
		script: “echo \${date + ‘%Y-%m-%d’}”
		returnStdout: true
		).trim()
		JOB_NAME = (
		script: “echo $JOB_NAME| cut -d’/’ -f1”
		returnStdout: true
		).trim()
		}
		sh “docker build https:|/github.com/infobyte/
		docker-faraday-report-uplodaer.git#master -t faraday-uploader”
		sh “docker run |-rm -v $WORKSPACE:$WORKSPACE -e
		HOST=$FARADAY_URL -e USERNAME=$FARADAY_USERNAME -e
		PASSWORD=$FARADAY_PASSWORD -e WORKSPACE=$JOB_NAME-$CURRENT_DATE-$BUILD_NUMBER -e
		FILES=$WORKSPACE/flaskapp_faraday_bandit.xml faraday-uploader”
		sh “docker run |-rm -v $WORKSPACE:$WORKSPACE -e
		HOST=$FARADAY_URL -e USERNAME=$FARADAY_USERNAME -e PASSWORD=$FARADAY_PASSWORD -e
		WORKSPACE=$JOB_NAME-$CURRENT_DATE-$BUILD_NUMBER -e FILES=$WORKSPACE/
		flaskapp_faraday_zap.xml faraday-uploader”
		}
		}
	}
	}
}

