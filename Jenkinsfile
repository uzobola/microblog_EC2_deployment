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
                sudo pkill gunicorn || true
                
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
		#cd /home/ubuntu/microblog_EC2_deployment
		
		# This Activates the python virtual environment
                #python3.9 -m venv venv
		#source venv/bin/activate 
                
		# Starts gunicorn in the background
		#nohup gunicorn -b :5000 -w 4 microblog:app > gunicorn.log 2>&1 & 
                gunicorn -b :5000 -w 4 microblog:app &
		
                # Restarts nginx to help resolve any connection issues between nginx and gunicorn
		#sudo systemctl restart nginx

                
        	set -e  # Exit immediately if a command exits with a non-zero status

        	# Change to the application directory
        	cd /home/ubuntu/microblog_EC2_deployment || { echo "Failed to change directory"; exit 1; }

        	# Activate the existing Python virtual environment
        	echo "Activating virtual environment..."
        	source venv/bin/activate

        	# Stop any existing Gunicorn processes to avoid port conflicts
        	echo "Stopping existing Gunicorn processes..."
        	pkill -f gunicorn || echo "No Gunicorn processes found to stop"

        	sleep 2  # Give it a moment to ensure processes have stopped

        	# Start Gunicorn in the background
        	echo "Starting Gunicorn..."
        	nohup gunicorn -b :5000 -w 4 microblog:app > gunicorn.log 2>&1 &

        	# Wait for a moment to allow Gunicorn to start
        	echo "Waiting for Gunicorn to start..."
        	sleep 5

        	# Check if Gunicorn started successfully
        	if pgrep -f gunicorn > /dev/null; then
            	echo "Gunicorn started successfully"
        	else
            	echo "Failed to start Gunicorn"
            	cat gunicorn.log  # Output the log for debugging
            	exit 1
       	  	fi

        	echo "Deployment completed successfully"
                '''
            }
        }
    }
}
