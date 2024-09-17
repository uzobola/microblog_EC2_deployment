pipeline {
  agent any
    stages {
        stage ('Build') {
            steps {
                sh '''#!/bin/bash
                # This  creates the python  virtual environment
                python3.9 -m venv venv
                
		# This activates the python virtual environment
		source venv/bin/activate

                # This installs any dependencies
                pip install -r requirements.txt
                pip install gunicorn pymysql cryptography

                # Set environment variable
                export FLASK_APP=microblog.py

                # Sets up database
                flask db upgrade

                # Compile translations
                flask translate compile

		# This stops any existing gunicorn processes. ( Avoids potential port conflicts just in case the new instance tries to bind t othe same port)
                pkill gunicorn || true
                
                # Run the application in the background
                gunicorn -b :5000 -w 4 microblog:app &
                '''
            }
        }
        stage ('Test') {
            steps {
                sh '''#!/bin/bash
                #source venv/bin/activate
                #py.test ./tests/unit/ --verbose --junit-xml test-reports/results.xml
		source venv/bin/activate
            	echo "Current directory: $(pwd)"
            	echo "Directory contents: $(ls -la)"
            	echo "Python path before: $PYTHONPATH"
            	export PYTHONPATH=$PYTHONPATH:${WORKSPACE}
            	echo "Python path after: $PYTHONPATH"
            	pytest ./tests/unit/ --verbose --junit-xml test-reports/results.xml
                '''
            }
            post {
                always {
                    junit 'test-reports/results.xml'
                }
            }
        }
      stage ('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
      stage ('Clean') {
            steps {
                sh '''#!/bin/bash
                if [[ $(ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2) != 0 ]]
                then
                ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2 > pid.txt
                kill $(cat pid.txt)
                exit 0
                fi
                '''
            }
        }
      stage ('Deploy') {
            steps {
                sh '''#!/bin/bash
		# This Activates the python virtual environment
                source venv/bin/activate
		
		# This sets the FLask app environment variable
		export FLASK_APP=microblog.py
                
                # This stops any existing gunicorn processes. ( Avoids potential port conflicts just in case the new instance tries to bind t othe same port)
		pkill gunicorn || true

		# Starts gunicorn in the background
		gunicorn -b :5000 -w 4 microblog:app &
		
                # Restarts nginx to help resolve any connection issues between nginx and gunicorn
		sudo systemctl restart nginx
                '''
            }
        }
    }
}
