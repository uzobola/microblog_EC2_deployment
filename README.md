# Microblog EC2 Deployment
---


---

---
### Clone repo to  GitHub account.
GitHub is the source code management tool of choice. Cloning the repository makes the code available in the developer's local environment. Managing the codebase with SCM tools like GitHub ensures that all changes are tracked and versions are managed appropriately as collaboration happens. This step facilitates integration with automation tools in our CI/CD pipeline, e.g., Jenkins.
---

---
### **Create Jenkins Server**
Create an Ubuntu EC2 instance (t3.micro) named "Jenkins" and install Jenkins onto it. 
A script to install Jenkins was created using the following steps:

**Steps to Install Jenkins**
- #Updates package lists and installs the following
  sudo apt update && sudo apt install fontconfig openjdk-17-jre software-properties-common &&

- #Adds the deadsnakes PPA Repository and installs python 3.7
 sudo add-apt-repository ppa:deadsnakes/ppa && sudo apt install python3.7 python3.7-venv

- #This downloads the Jenkins repository Key
 sudo wget -O /usr/share/keyrings/jenkins-keyring.asc <https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key>

- #This adds the Jenkins repo to the system's sources list 
 echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" <https://pkg.jenkins.io/debian-stable> binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

- #This updates the package lists, installs, starts, and checks the status of the Jenkins service.
 sudo apt-get update sudo apt-get install jenkins sudo systemctl start jenkins sudo systemctl status Jenkins

- #Configure the server by installing 'python3.9', 'python3.9-venv', 'python3-pip', and 'nginx'.
  Steps to Configure the Jenkins Server

- #Install python3.9 
  sudo apt install python3.9 python3.9-venv

- #Install NGINX 
  sudo apt update && sudo apt install nginx
--- 

---
### **Clone your GH repository to the server, create and activate a Python virtual environment**
#Clone repo 
- git clone <https://github.com/uzobola/microblog_EC2_deployment.git>

### **Create and activate Python Virtual Environment in the applications directory**
- cd /path/to/application python3.9 -m venv venv source venv/bin/activate

- In the virtual environment, install the requirements file and required packages
  pip install -r requirements.txt pip install gunicorn mymysql cryptograpy

The Microblog application runs on FLASK. Setting the environment variable to [microblog.py](http://microblog.py) points flask to main file that contains the application. Setting this variable correctly allows Flask commands to be run from any directory within the project.
- #Set Environment Variable
  FLASK_APP=[microblog.py](http://microblog.py)

- #Compile translation files into binary files to be used by the application
  flask translate compile

- #Upgrade the database to the latest required version 
 flask db upgrade
---

---
### **Set up Nginx as a Reverse Proxy**
The '/etc/nginx/sites-enabled/default file' is the default server configuration file for nginx. The location block values below set up nginx as a proxy server for forwarding of all incoming requests to the application running on port 5000. This allows Nginx to handle tasks like SSL termination, load balancing, and generating static files.

- #Edit Nginx configuration file 
  vim /etc/nginx/sites-enabled/default

- #Configure the following location block values
  location / { proxy_pass <http://127.0.0.1:5000>; proxy_set_header Host $host; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; }
---  

---
### Run the Application
Running the command below starts Gunicorn ( a WSGI HTTP server for Python applications), binds the server to port 5000, and specifies that Gunicorn starts 4 worker processes. It also informs Gunicorn to load the app object from the microblog module.

- #Start Gunicorn
  gunicorn -b :5000 -w 4 microblog:app

In the background, the browser sends an HTTP request to the server's IP on port 5000 where unicorn is listening. Once the request is received, Gunicorn passes it to one of the 4 workers processes which executes the Flask application code. Once the request is successfully processed, an HTTP response is generated and sent back to the browser. The browser receives the response and renders the webpage for the user to see.
---

---
### **Pipeline Automation with Jenkins**
Important components of the Jenkins pipeline Script.
**The Build Stage:** 
Includes all of the commands required to prepare the environment for the application. This includes creating the virtual environment and installing all the dependencies, setting variables, and setting up the databases.

stage ('Build') { steps { sh '''#!/bin/bash

This creates the python virtual environment

python3.9 -m venv venv

```
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

            # Run the application in the background
            gunicorn -b :5000 -w 4 microblog:app &

```

The Test Stage: The test stage will run pytest to run a unit test of the application source code.

```} stage ('Test') { steps { sh '''#!/bin/bash source venv/bin/activate py.test ./tests/unit/ --verbose --junit-xml test-reports/results.xml source venv/bin/activate echo "Current directory: $(pwd)" echo "Directory contents: $(ls -la)" echo "Python path before: $PYTHONPATH" export PYTHONPATH=$PYTHONPATH:${WORKSPACE} echo "Python path after: $PYTHONPATH" pytest ./tests/unit/ --verbose --junit-xml test-reports/results.xml ''' } post { always { junit 'test-reports/results.xml' } }```

![Test Successful](https://github.com/user-attachments/assets/579f32d2-2767-4967-a80b-844dd39f0bc2)




The Deploy Stage:
Runs the command required to deploy the application publicly
```
stage ('Deploy') {
steps {
sh '''#!/bin/bash

# Restarts nginx to help resolve any connection issues between nginx and gunicorn
sudo systemctl restart nginx
sleep 3


# Activate the existing Python virtual environment
echo "Activating virtual environment..."
source venv/bin/activate

# Created a systemd service to start gunicorn service
sudo systemctl start gunicorn.service
'''
        }
    }

```


---
### **OWASP FS Scan**

The OWASP Dependency-Check plugin for Jenkins is a security tool that automates the process of identifying vulnerabilities in project dependencies. This plugin:
1.  Scans all project dependencies thoroughly
2.  Cross-references findings with the National Vulnerability Database
3.  Generates comprehensive reports that include, identified vulnerable libraries, the severity ratings of vulnerabilities, any associated Common Vulnerabilities and Exposures (CVEs) as well as actionable remediation guidance

**Benefits of including this in the Jenkins pipeline**
- Helps identify security risks early in the development process
- Provides actionable guidance for remediation of the vulnerabilities
- Integrates security early into the build process

  
![Dependency Check](https://github.com/user-attachments/assets/f4c31749-c45f-48a8-ab2d-ba8ffe4d52b6)

---


### **Create a MultiBranch Pipeline**

Steps to create the 
Log into Jenkins

a. Enter the initial admin password

b. Install suggested plugins

c. Create the first admin user

1. Create a Multi-Branch pipeline
    
    a. Click on “New Item” in the menu on the left of the page
    
    b. Enter a name for your pipeline
    
    c. Select “Multibranch Pipeline”
    
    d. Under “Branch Sources”, click “Add source” and select “GitHub”
    
    e. Click “+ Add” and select “Jenkins”
    
    f. Make sure “Kind” reads “Username and password”
    
    g. Under “Username”, enter your GitHub username
    
    h. Under “Password”, enter your GitHub personal access token
    

1. Connect GitHub repository to Jenkins
    
    a. Enter the repository HTTPS URL and click "Validate"
    
    b. Make sure that the "Build Configuration" section says "Mode: by Jenkinsfile" and "Script Path: Jenkinsfile"
    
    c. Click "Save" and a build should start automatically
    
![Successful Build](https://github.com/user-attachments/assets/775f55dd-ce39-4c9e-ba0c-d6970b469f5e)


---
### Monitoring

Prometheus and Grafana are open-source monitoring tools that  when used together provide powerful infrastructure and application resource monitoring capabilities. Prometheus collects and stores metric data while Grafana provides customizable visualization dashboards.
![Prometheus Configured](https://github.com/user-attachments/assets/4b41e029-e6d5-4cfb-afe6-4a929a1717e9)
![Garafana Configured Metrics](https://github.com/user-attachments/assets/9fe6c14f-3575-4bfb-abd9-548e9d2cb591)




---
