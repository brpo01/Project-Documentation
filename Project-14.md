# Experience Continuous Integration with Jenkins | Ansible | Artifactory | SonarQube | PHP

## Step 1: Simulating a typical CI/CD Pipeline for a PHP Based application
    This is a continuation of Project 11 through Project 13.
    Note: Create servers required for an environment you are working with at the moment only. For example, when doing deployments for development, do not create servers for integration, pentest, or production yet).

### Step 1.1: Set Up
#### Ansible inventory should look like this:
    ├── ci
    ├── dev
    ├── pentest
    ├── preprod
    ├── prod
    ├── sit
    └── uat
#### ci inventory
    [jenkins]
    <Jenkins-Private-IP-Address>

    [nginx]
    <Nginx-Private-IP-Address>

    [sonarqube]
    <SonarQube-Private-IP-Address>

    [artifact_repository]
    <Artifact_repository-Private-IP-Address>
#### dev 
    [tooling]
    <Tooling-Web-Server-Private-IP-Address>

    [todo]
    <Todo-Web-Server-Private-IP-Address>

    [nginx]
    <Nginx-Private-IP-Address>

    [db:vars]
    ansible_user=ec2-user
    ansible_python_interpreter=/usr/bin/python

    [db]
    <DB-Server-Private-IP-Address>
#### pentest
    [pentest:children]
    pentest-todo
    pentest-tooling

    [pentest-todo]
    <Pentest-for-Todo-Private-IP-Address>

    [pentest-tooling]
    <Pentest-for-Tooling-Private-IP-Address>

### Step 1.2: Add two more roles to your Ansible playbook

- Sonarqube: Sonarqube is an open-source platform developed by SonarSource for continuous inspection of code quality to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities.
  
- Artifactory: JFrog Artifactory is a universal DevOps solution providing end-to-end automation and management of binaries and artifacts through the application delivery process that improves productivity across your development ecosystem. (source: https://www.jfrog.com/confluence/display/JFROG/JFrog+Artifactory)

### Step 1.3: Configure Ansible for Jenkins development
In previous projects, you have been launching Ansible commands manually from a CLI. Now, with Jenkins, we will start running Ansible from Jenkins UI.

- Navigate to Jenkins UI and install Blue Ocean plugin
- Click Open Blue Ocean from the left pane
- Once you're in the Blue Ocean UI, click Create Pipeline
- Select GitHub
- On the Connect to GitHub step, click 'Create an access token here' to create your access token
- Type in the token name of your choice, leave everything as is and click Generate token
- Copy the generated token and paste in the field provided in Blue Ocean and click connect
- Select your organization (typically your GitHub repository name)
- Select the repository to create the pipeline from
- Blue Ocean would take you to where to create a pipeline, since we are not doing this now, click Administration from the top bar to exit Blue Ocean.
- Create Jenkinsfile
  - Inside the Ansible project, create a deploy folder
  - In the deploy folder, create a file named **Jenkinsfile**
  - Add the following snippet into the newly created file named **Jenkinsfile**
    ```
    pipeline {
    agent any
        stages {
            stage('Build') {
                steps {
                    script {
                        sh 'echo "Building Stage"'
                    }
                }
            }
        }
    }
    ```
    The above pipeline job has only one stage (Build) and the stage contains only one step which runs a shell script to echo "Building stage"
- Go back to Ansible project in Jenkins and click Configure
- Scroll down to Build Configuration and for script path, enter the path to the Jenkinsfile (deploy/Jenkinsfile in our case)
- Go back to the pipeline and click Build Now
- Open Blue Ocean again to see the build in action
- You could trigger the build again by clicking the play button against the branch, then click the branch.
  
  Since our pipeline is multibranch, we could build all the branches in the repo independently. To see this in action, 
  - Create a new git branch and name it features/jenkinspipeline-stages
  - Add a new build stage "Test"

    ```
    pipeline {
    agent any
      stages {
        stage('Build') {
          steps {
            script {
              sh 'echo "Building Stage"'
            }
          }
        }

        stage('Test') {
          steps {
            script {
              sh 'echo "Testing Stage"'
            }
          }
        }
        }
    }
    ```
    
    

  - To make the new branch show in Jenkins UI, click Administration to exit Blue Ocean, click the project and click *Scan Repository Now* from the left pane
  - Refresh the page and you should see the new branch.
  - Open Blue Ocean and you should see the new branch building (or has finished building)
  
  ![{6FD8E835-A19D-4429-865E-354B9E996FFF} png](https://user-images.githubusercontent.com/76074379/119834508-e036a380-beb4-11eb-8110-0ca43e16163d.jpg)
  
    ```
    Quick task:
    1. Create a pull request to merge the latest code into the `main branch`
    2. After merging the `PR`, go back into your terminal and switch into the `main` branch.
    3. Pull the latest change.
    4. Create a new branch, add more stages into the Jenkins file to simulate below phases. (Just add an `echo` command like we have in `build` and `test` stages)
       1. Package 
       2. Deploy 
       3. Clean up
    5. Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch
    6. Eventually, your main branch should have a successful pipeline like this in blue ocean
    ```
    Your final pipeline should look like this:
  ![{034E93DC-F506-410F-8C70-EB490FC3DAA2} png](https://user-images.githubusercontent.com/76074379/119834964-44596780-beb5-11eb-8497-0b51ad3538db.jpg)

### Step 1.4: Running Ansible Playbook from Jenkins
- Install Ansible on your Jenkins server
  ```
  sudo apt install ansible -y
  ```
- Install Ansible plugin in Jenkins UI
  - Go to Manage Jenkins
  - Click Manage Plugins
  - Click Available tab and in the search bar, enter Ansible and click the check box next to the plugin
  - Scroll down and click 'Install without restart'
- After the ansible plugin has been installed. Go to *Manage Jenkins*
  - Click on *Global Configuration Tools*
  - Scroll down to Ansible and click on it, in the 'Name' box, type *ansible* and in the 'Path to ansible Executables directory' box, type the path to ansible(you can check this using command "which ansible" on the Linux terminal)
- Create Jenkinsfile from scratch (delete all the current stages in the file)
  - Add a new stage to clone the GitHub repository
    ```
    stage('SCM Checkout') {
      steps {
        git(branch: 'main', url: 'https://github.com/TheCountt/config-ansible-mgt.git')
      }
    }
    ```
    
  - Add the next stage, to run the playbook. For this, we need to click on *Pipeline Syntax* at bottom left of Jenkins UI and input appropriate value and generate a pipeline script. (https://www.youtube.com/watch?v=PRpEbFZi7nI&feature=youtu.be)
 
    ```
    stage('Execute Ansible') {
        steps {
            ansiblePlaybook colorized: true, credentialsId: 'privateKEY', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/${inventory_file}', playbook: 'playbooks/site.yml'}
        }
      }
    }
    ```
    This build stage requires a credentails file (privateKEY) which can be created by following these steps:
    - Click Manage Jenkins and scroll down a bit to Manage Credentials
    - Under Stores scoped to Jenkins on the right, click on Jenkins
    - Under System, click the Global credentials (unrestricted)
    - Click Add Credentials on the left
    - For credentials kind, choose SSH username with private key
    - Enter the ID as privateKEY
    - Enter the username Jenkins would use (ubuntu/ec2-user)
    - Enter the secret key (the contents of your <\private_key>.pem file from AWS)
    - Click OK to save
  - You could add a stage that cleans your workspace after every build
    ```
    stage('Clean up') {
      steps {
        cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
      }
    }
    ```
  - Commit and push your changes
  - Go to Blue Ocean and trigger a build against the branch. If everything is configured properly, you should see something like this:
  
    ![{73E0A380-8DB1-4859-9B94-16D0C0DCD0A8} png](https://user-images.githubusercontent.com/76074379/119836654-c1391100-beb6-11eb-89e7-6fd649578185.jpg)
    
    ![{A9351339-EDBE-4F06-BB0F-3C35320D7ABE} png](https://user-images.githubusercontent.com/76074379/119837149-2b51b600-beb7-11eb-8a81-7def45aae7e7.jpg)
    

### Step 1.5: Parameterizing Jenkinsfile for Ansible Development
To deploy to other environments, we have to use parameters
- Update SIT environment inventory file with new servers
- Update Jenkinsfile to add parameters
  ```
  pipeline{
   agent any
   environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }
     parameters {
      string(name: 'inventory_file', defaultValue: '${inventory_file}', description: 'selecting the environment')
          }
  ```
  
  Overall, the Jenkinsfile in the deploy folder(deploy/Jenkinsfile) should like this
  
  ```
  pipeline{
   agent any
   environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }
     parameters {
      string(name: 'inventory_file', defaultValue: '${inventory_file}', description: 'selecting the environment')
          }
   stages{
      stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }
      stage('SCM Checkout') {
         steps{
            git branch: 'main', url: 'https://github.com/TheCountt/config-mgt-ansible.git'
         }
       }
      stage('Prepare Ansible For Execution') {
        steps {
          sh 'echo ${WORKSPACE}'
        }
     }
      stage('Execute Ansible Playbook') {
        steps {
            ansiblePlaybook colorized: true, credentialsId: 'privateKEY', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/${inventory_file}', playbook: 'playbooks/site.yml', skippedTags: 'skipped', tags: 'run'
        }
      }
      stage('Clean Workspace after build'){
        steps{
          cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
        }
      }
   }
  }
  ```

## Step 2: CI/CD Pipeline for a TODO Application

### Step 2.1: Configure Artifactory

- Create an Ansible role to install Artifactory(You may install manually first on artifactory server: https://www.howtoforge.com/tutorial/ubuntu-jfrog/)

- Run the role against the Artifactory server

### Step 2.2: Prepare Jenkins
- Fork the php-todo repository (https://github.com/darey-devops/php-todo.git) into Bastion(Jenkins) Server
- On you Jenkins server, install PHP, its dependencies(Feel free to do this manually at first, then update your Ansible accordingly later)

  ```
  sudo apt install -y zip libapache2-mod-php php7.4-fpm phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip,xdebug}
  
  ```
  Install PHP Composer manually:
 
 ```
    php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
    
    php -r "if (hash_file('sha384', 'composer-setup.php') === '756890a4488ce9024fc62c56153228907f1545c228516cbf63f885e036d37e9a59d27d63f46af1d4d07ee0f76181c7d3') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
    
    php composer-setup.php
    
    mv composer.phar /usr/local/bin/composer
  ```


- Configure the php.ini file, you can get the path to the file by running
  ```
  php --ini | grep xdebug
  ```
 After getting the path to the file, enter the file and paste in:
  ```
  xdebug.mode=coverage

```
- Install nodejs and npm

```
sudo apt-get update -y
sudo apt-get install nodejs -y
sudo apt install npm -y
```

- Install typescript using node package manager(npm). Only root user can install typecript.

```
su
npm install -g typescript

```

Install Tthe following plugins on Jenkins UI
  - Plot plugin: to display tests reports and code coverage information
  - Artifactory plugin: to easily deploy artifacts to Artifactory server
  - Go to the artifactory URL(http://<artifactory-server-ip>:8082) and create a local generic repository named *php-todo*(Default username and password is admin. After logging in, change the password)
Configure Artifactory in Jenkins UI
  - Click Manage Jenkins, click Configure System
  - Scroll down to JFrog, click Add Artifactory Server
  - Enter the Server ID
  - Enter the URL as:
    
```
http://<artifactory-server-ip>:8082/artifactory
```
    
  - Enter the Default Deployer Credentials(the username and the changed password of artifactory)

   ![{760F9B42-2325-411C-AF6B-C8FD7E6C7922} png](https://user-images.githubusercontent.com/76074379/119845043-cbaad900-bebd-11eb-98cb-c49c07cfefdf.jpg)
    
### Step 2.3: Integrate Artifactory repository with Jenkins
- On Jenkins server, install mysql-client
- Create a dummy Jenkinsfile in php-todo repo
- In Blue Ocean, create multibranch php-todo pipeline(follow the previous steps earlier)
- Spin up an instance for a database, install and configure mysql-server( This is where data pertaining to the artifactory repository will be stored). 
- Create a database and user on the database server
  ```
  CREATE DATABASE homestead;
  CREATE USER 'homestead'@'%' IDENTIFIED BY 'sePret^i';
  GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';
  FLUSH PRIVILEGES;
  ```
- Check if the database created on the database server can be reached from the Jenkins server. On the jenkins server, run command:
    
    ```
    mysql -u <DB_user> -h <DB-private-ip-address> -p
    ```
- Update the .env.sample file with your db connectivity details
- Update Jenkinsfile with proper configuration
  ```
  pipeline {
    agent any

    stages {

     stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }
  
    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/darey-devops/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
          }
        }
      }
    }
    ```
   ![{C5F61134-7A4F-4B9B-8352-E53FF0FF2751} png](https://user-images.githubusercontent.com/76074379/119847341-c058ad00-bebf-11eb-9354-6e291fbb92e8.jpg)

- Update Jenkinsfile to include unit tests
  ```
  stage('Execute Unit Tests') {
      steps {
             sh './vendor/bin/phpunit'
      }
  ```
    
  ![{5676EDD4-F9E3-452C-8BE3-F43492EE790A} png](https://user-images.githubusercontent.com/76074379/119847660-031a8500-bec0-11eb-910a-e692cdd00d75.jpg)
    
   
### Step 2.4: Code Quality Analysis
Most commonly used tool for php code quality analysis is **phploc**
    
- Add the code analysis step in Jenkinsfile
    
  ```
      stage('Code Analysis') {
      steps {
            sh 'phploc app/ --log-csv build/logs/phploc.csv'

      }
    }
  ```
 
 ![{9C2C664A-094C-4DCF-8ABA-3D8A2B70619B} png](https://user-images.githubusercontent.com/76074379/119847999-483eb700-bec0-11eb-9e08-9124fd3eeb92.jpg)
    
    
Plot the data using plot Jenkins plugin.
    
This plugin provides generic plotting (or graphing) capabilities in Jenkins. It will plot one or more single values variations across builds in one or more plots. Plots for a particular job (or project) are configured in the job configuration screen, where each field has additional help information. Each plot can have one or more lines (called data series). After each build completes the plots’ data series latest values are pulled from the CSV file generated by phploc.
    
```
      stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'
          }
        }
  ```
  
 CLick on the Plots button on the left pane. If everything was configured properly, you should see something like this:
 
![{67576C97-320F-452D-B744-8A382A161985} png](https://user-images.githubusercontent.com/76074379/119848413-a66b9a00-bec0-11eb-9f9f-57a31beab6ca.jpg)
    
    
- Package the artifacts 
 
  ```
  stage ('Package Artifact') {
    steps {
            sh 'zip -qr ${WORKSPACE}/php-todo.zip ${WORKSPACE}/*'
    }
  ```
![{B87A718F-1594-4C0E-A4C0-1BBA3E1A83A7} png](https://user-images.githubusercontent.com/76074379/119848671-dadf5600-bec0-11eb-8f87-090517fc9916.jpg)
    
    
- Publish packaged artifact into Artifactory
    
  ```
  stage ('Deploy Artifact') {
    steps {
            script { 
                 def server = Artifactory.server 'artifactory-server'
                 def uploadSpec = """{
                    "files": [{
                       "pattern": "php-todo.zip",
                       "target": "php-todo"
                    }]
                 }"""

                 server.upload(uploadSpec) 
               }
    }
  }
  ```
![{C3BA016B-91AE-468C-B348-328842460557} png](https://user-images.githubusercontent.com/76074379/119848951-17ab4d00-bec1-11eb-863a-a6df0f911d4c.jpg)
    
    
 For this to work, you have to create a local repository on your Artifactory with package type as Generic and Repository Key as the name of the repo (php-todo in this case) which we already did as instructed up there
 

- Deploy application to dev environment by launching the Ansible pipeline 
  ```
  stage ('Deploy to Dev Environment') {
    steps {
    build job: 'ansible-config-mgt/main', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']], propagate: false, wait: true
    }
  }
  ```
```
**Note**: The stage above will not deploy to "dev" environment if you do not specify it in the String ParameterValue at the beginning of the script. The "defaultValue" should be changed from "${inventory_file}" to "dev". That way the application will be automatically deployed to Dev environment.
```
    
 This build job tells Jenkins to trigger a job in the ansible-config-mgt pipeline. Since the ansible-config-mgt pipeline requires some parameters, we pass them using the 'parameters' value.(I provisioned servers for SIT,DEV,CI environments only)
    
![{73E0A380-8DB1-4859-9B94-16D0C0DCD0A8} png](https://user-images.githubusercontent.com/76074379/120633085-969f0900-c41e-11eb-91df-49809554e025.jpg)

## Step 3: Install and Configure SonarQube on Ubuntu 20.04 With PostgreSQL as a Backend Database
    
Software Quality - The degree to which a software component, system or process meets specified requirements based on user needs and expectations.

Software Quality Gates - Quality gates are basically acceptance criteria which are usually presented as a set of predefined quality criteria that a software development project must meet in order to proceed from one stage of its lifecycle to the next one.
    
We will make some Linux Kernel configuration changes to ensure optimal performance of the tool - we will increase vm.max_map_count, file discriptor and ulimit(Check Sonarqube Documentation for details)

### Step 3.1: Tune Linux Kernel
 
On the sonarqube server, log into to the root user and run the following commands
    ```
      su
  sysctl -w vm.max_map_count=262144
  sysctl -w fs.file-max=65536
  ulimit -n 65536
  ulimit -u 4096 
    ```
    

Open the /etc/security/limits.conf file and append the following.
    
  ```
  sonarqube   -   nofile   65536
  sonarqube   -   nproc    4096
  ```
To enable persistence after reboot, open the /etc/sysctl.conf and append the following.
    
  ```
  vm.max_map_count=262144
  fs.file-max=65536 
  ```

### Step 3.2: Update system packages and Install Java(sonarqube is java-based) and other required packages
- Update and upgrade packages
  ```
  sudo apt-get update -y
  sudo apt-get upgrade -y
  ```
- Install wget and unzip packages
  ```
  sudo apt-get install wget unzip -y
  ```
- Install OpenJDK and Java Runtime Environment (JRE)
  ```
  sudo apt-get install openjdk-11-jdk -y
  sudo apt-get install openjdk-11-jre -y
  ```
  - To set default JDK or switch to OpenJDK, run the command,
    ```
    sudo update-alternatives --config java
    ```
  - Verify JAVA is installed by running:
    ```
    java -version
    ```
### Step 3.3: Install and Setup PostgreSQL 10 Database for SonarQube
    
- Go to the security group and open port 5432 for postgreSQL on Sonarqube Instance
    
- Add PostgreSQL to repo list
  ```
  sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
  ```
- Download PostgreSQL software
  ```
  wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
  ```
- Install PostgreSQL
  ```
  sudo apt-get -y install postgresql postgresql-contrib
  ```
- Start and enable PostgreSQL Database service
  ```
  sudo systemctl start postgresql
  sudo systemctl enable postgresql
  ```
- Change the default password for postgres user (to any password you can easily remember)
  ```
  sudo passwd postgres
  ```
  Then switch to postgres user
  ```
  su - postgres
  ```
- Create a new user for SonarQube
  ```
  createuser sonar
  ```
- Create Sonar DB and user
  - Switch to PostgreSQL shell
    ```
    psql
    ```
  - Set up encrypted password for newly created user
    ```
    ALTER USER sonar WITH ENCRYPTED password 'sonar';
    ```
  - Create a DB for SonarQube
    ```
    CREATE DATABASE sonarqube OWNER sonar;
    ```
  - Grant required privileges
    ```
    grant all privileges on DATABASE sonarqube to sonar;
    ```
  - Exit the shell
    ```
    \q
    ```
- Switch back to initial user
  ```
  exit
  ```

### Step 3.4: Install and configure SonarQube
- Download temporary files to /tmp folder
  ```
  cd /tmp && sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
  ```
- Unzip archive to the /opt directory and move to /opt/sonarqube
  ```
  sudo unzip sonarqube-7.9.3.zip -d /opt
  sudo mv /opt/sonarqube-7.9.3 /opt/sonarqube
  ```
- Configure SonarQube
Since Sonarqube cannot be run as root user, we have to create a **sonar** user to start the service with.
  - Create a group sonar
    ```
    sudo groupadd sonar
    ```
  - Add a **sonar** user with control over /opt/sonarqube
    ```
    sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar 
    sudo chown sonar:sonar /opt/sonarqube -R
    ```
  - Open and edit SonarQube configuration file
    
    ```
    sudo vim /opt/sonarqube/conf/sonar.properties
    ```
    
    Then find and edit the following lines
  
    ```
    sonar.jdbc.username=sonar
    sonar.jdbc.password=sonar
    ```
    Append this line
    ```
    sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
    ```
  - enter the sonar script file. Find and set RUN_AS_USER to be equals to sonar
    
    ```
    sudo nano /opt/sonarqube/bin/linux-x86-64/sonar.sh
    ```
    ```
    # If specified, the Wrapper will be run as the specified user.

    # IMPORTANT - Make sure that the user has the required privileges to write

    #  the PID file and wrapper.log files.  Failure to be able to write the log

    #  file will cause the Wrapper to exit without any way to write out an error

    #  message.

   # NOTE - This will set the user which is used to run the Wrapper as well as

   #  the JVM and is not useful in situations where a privileged resource or

   #  port needs to be allocated prior to the user being changed.

  RUN_AS_USER=sonar
    
  ```
    
  - Add sonar user to sudoers file
    - First set the user's password to something you can easily remember
      ```
      sudo passwd sonar
      ```
    - Then run the command to add sonar to sudo group
      ```
      sudo usermod -a -G sudo sonar
      ```
    - To check if it has been added, run command
    ```
    groups sonar
    ```
    
  - Switch to the sonar user, switch to script directory and start the script
    ```
    su - sonar
    cd /opt/sonarqube/bin/linux-x86-64/
    ./sonar.sh start
    ```
    Output:
    ```
    Starting SonarQube...

    Started SonarQube
    ```
  - Check SonarQube logs
    ```
    tail /opt/sonarqube/logs/sonar.log
    ```
  - Configure systemd to manage SonarQube service
    - Stop the sonar service
      ```
      cd /opt/sonarqube/bin/linux-x86-64/
      ./sonar.sh stop
      ```
    - Create systemd service file
      ```
      sudo vim /etc/systemd/system/sonar.service
      ```
      Paste in the following lines:
      ```
      [Unit]
      Description=SonarQube service
      After=syslog.target network.target

      [Service]
      Type=forking

      ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
      ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

      User=sonar
      Group=sonar
      Restart=always

      LimitNOFILE=65536
      LimitNPROC=4096

      [Install]
      WantedBy=multi-user.target
      ```
      Save and exit
    - Use systemd to manage the service
      ```
      sudo systemctl start sonar
      sudo systemctl enable sonar
      sudo systemctl status sonar
      ```
### Step 3.5: Access Sonarqube
- Access the SonarQube by going to http://sonarqube-public-ip-address:9000, use **admin** as your username and password
    
 **NOTE**: Do not forget to open ports 9000 and 5432 on the security group.
    
![{C3EC02C7-4DFF-47E8-9C50-92BC0A873486} png](https://user-images.githubusercontent.com/76074379/120636060-06fb5980-c422-11eb-8199-eb54b6fbd3d0.jpg)

### Step 3.6: Configure SonarQube and Jenkins for Quality Gate
- Install SonarQube plugin in Jenkins UI
- Navigate to Manage Jenkins > Configure System and add a SonarQube server
  - In the name field, enter **sonarqube**
  - For server URL, enter the IP of your SonarQube instance
- Generate authentication tokens to use in the Jenkins UI
  - In SonarQube UI, navigate to User > My Account > Security > Generate tokens
  - Type in the name of the token and click Generate

- Configure SonarQube webhook for Jenkins
  -  Navigate to Administration > Configuration > Webhooks > Create
  - Enter the name
  - Enter URL as http://<jenkins-server-ip>:8080/sonar-webhook/

- Setup SonarScanner for Jenkins
  - In Jenkins UI, go to Manage Jenkins > Global Tool Configuration
  - Look for SonarQube Scanner
  - Click 'Add SonarQube Scanner' and enter the scanner name as 'SonarQubeScanner'
  - Check the 'Install automatically' box
  - Select 'Install from Maven Central'

- Add the following build stage for Quality Gate
  ```
  stage('SonarQube Quality Gate') {
      environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
  ```
  Save and run the pipeline to install the scanner. An error will be generated but it will also create "/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/sonar-scanner.properties" directory

- Edit sonar-scanner.properties file
  ```
  sudo vim /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/sonar-scanner.properties
  ```
  Paste in the following lines:
  ```
  sonar.host.url=http://<SonarQube-Server-IP-address>:9000
  sonar.projectKey=php-todo
  #----- Default source code encoding
  sonar.sourceEncoding=UTF-8
  sonar.php.exclusions=**/vendor/**
  sonar.php.coverage.reportPaths=build/coverage/phploc.csv
  sonar.php.tests.reportPath=build/coverage.xml
  ```
    
- If everything was configured properly, you should see something like this: ![](imgs/endtoend.png)
- Navigate to your php-todo dashboard on SonarQube UI ![](imgs/qualitygate.png)

### Step 3.7: Conditionally Deploy to Higher Environments
- Include a **when** condition to execute the Quality Gate stage only when the running branch is develop, hotfix, release, main or master
  ```
  when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
  ```
- Add a timeout step to abort to Quality Gate stage after 1 minute (or any time you like)
  ```
      timeout(time: 1, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }
  ```
  Quality Gate stage snippet should look like this:
  ```
  stage('SonarQube Quality Gate') {
      when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
            }
            timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
  ```
  You should get the following when you run the pipeline:
  
  When running a non-specified branch, SonarQube Quality Gate stage is skipped
  ![](imgs/branch-testing.png)

## Step 4: Configure Jenkins slave servers
- Spin up a new EC2 Instance(like the master(bastion) server)
  - Install all the neccessary software packages just like you did with the master(bastion) serve
  
- On the main Jenkins server
  - Navigate to Manage Jenkins > Manage Nodes
  - Click New Node
  - Enter name of the node and click the 'Permanent Agent' button and click the OK button
  - Fill in the remote root directory as /home/ubuntu/jenkins
  - Set 'Host' value as the IP of the slave node
  - For Launch Method, select Launch Agents via SSH
  - Add SSH with username and private key credentials with username as ubuntu and private key as the private key you copied from the node
  - For Host Key Verification Strategy, select Manually trusted key validation strategy
  - Click Save

- Repeat the above steps to add more servers
    
  

## Step 5: Configure GitHub WebHook for Automatic Build of Pushed Code

## Step 6: Deploy to all Environments


## Credits

https://www.jenkins.io/doc/book/blueocean/getting-started/

https://en.wikipedia.org/wiki/SonarQube

https://www.howtoforge.com/tutorial/ubuntu-jfrog/

https://www.jfrog.com/confluence/display/JFROG/JFrog+Artifactory

https://www.jenkins.io/doc/book/blueocean/getting-started/#accessing-blue-ocean

https://dev.to/nicoavila/how-to-validate-your-jenkinsfile-locally-before-committing-334l

https://matthiasnoback.nl/2019/09/using-phploc-for-quick-code-quality-estimation-part-1/

https://plugins.jenkins.io/sonar/

https://getcomposer.org/doc/00-intro.md#globally

https://code.visualstudio.com/docs/setup/linux#_visual-studio-code-is-unable-to-watch-for-file-changes-in-this-large-workspace-error-enospc
    
https://stackoverflow.com/questions/60269790/jenkins-set-a-parameter-defaultvalue-dynamically
