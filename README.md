# Magento-installation
# Overview:
We will set up a single server for Magento 2 Open Source, with these services as shown in the architecture bellow
- :white_circle: nginx
- :white_circle: php-fpm 
- :white_circle: MySQL
- :white_circle: Varnish
- :white_circle: Redis
- :white_circle: OpenSearch

<p align="center"><img src="screenshots/Architecture.png" width="90%" height="90%">
<br><em>Architecture</em>
</p>

* I have created a new user and a new path to work in
* Magento will be ran by a new system user called magento. Let’s create a new system user now, execute this command below.

```shell
$ /usr/sbin/adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Magento user' \
   --group \
   --home /opt/magento \
magento
```
* Then, let’s give the new user a password. You will be prompted to type the password for user ‘magento’ twice, the password will not be shown there in your screen.
```shell
$ passwd magento
```
* Once done, we can give the new user a sudo privilege.
```shell
$ usermod -aG sudo magento
```
* Let’s switch to the new user now. From now on, the commands will be run by the new user.
```shell
$ su - magento
```
### Let’s install PHP 8.3 and its extensions.
```shell
$ sudo apt install php-{bcmath,common,curl,fpm,gd,intl,mbstring,mysql,soap,xml,xsl,zip,cli}
```
* Next, we need to modify the following settings in the php.ini file:
Increase memory_limit to 512M
Set short_open_tag to On
Set upload_max_filesize to 128M
Increase max_execution_time to 3600
Let’s make the changes by executing these commands
```shell
$ sudo sed -i "s/memory_limit = .*/memory_limit = 512M/" /etc/php/8.3/fpm/php.ini
$ sudo sed -i "s/upload_max_filesize = .*/upload_max_filesize = 128M/" /etc/php/8.3/fpm/php.ini
$ sudo sed -i "s/short_open_tag = .*/short_open_tag = On/" /etc/php/8.3/fpm/php.ini
$ sudo sed -i "s/max_execution_time = .*/max_execution_time = 3600/" /etc/php/8.3/fpm/php.ini
```
Then, let’s create a PHP-FPM pool.
```shell
$ sudo vim /etc/php/8.3/fpm/pool.d/magento.conf
```
We need to insert the following into the file.
```shell
[magento]
user = magento
group = magento

listen = /run/php/magento.sock
listen.owner = magento
listen.group = magento
pm = ondemand
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 10
```
Save the file and then exit from the file editor and don’t forget to restart php-fpm service
```shell
$ sudo systemctl restart php8.3-fpm
```

### Install Nginx
```shell
$ sudo apt install nginx -y
```
 we need to create an nginx server block for our Magento website.
```shell
$ sudo vim /etc/nginx/sites-enabled/magento.conf
```
```shell
Insert the following into the configuration file.
upstream fastcgi_backend {
server unix:/run/php/magento.sock;
}

server {
server_name yourdomain.com;
listen 80;
set $MAGE_ROOT /opt/magento/website;
set $MAGE_MODE production;

access_log /var/log/nginx/magento-access.log;
error_log /var/log/nginx/magento-error.log;

include /opt/magento/website/nginx.conf.sample;
}
```
Save the file, then exit.
> [!NOTE]
> Don’t forget to change your server name to your actual name in my case I used localhost
### Install OpenSearch
```shell
$ sudo apt install curl gnupg2
$ curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
$ echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
$ sudo apt update
```
With the repository information added, we can list all available versions of OpenSearch:
```shell
$ sudo apt list -a opensearch
```
The output should be similar to this 
magento@ip-10-101-1-245:~$ sudo apt list -a opensearch
[sudo] password for magento:
Listing... Done
opensearch/stable 2.15.0 amd64 [upgradable from: 2.11.1]
opensearch/stable 2.14.0 amd64
opensearch/stable 2.13.0 amd64
opensearch/stable 2.12.0 amd64
opensearch/stable,now 2.11.1 amd64 [installed,upgradable to: 2.15.0]
opensearch/stable 2.11.0 amd64
opensearch/stable 2.10.0 amd64
opensearch/stable 2.9.0 amd64
opensearch/stable 2.8.0 amd64
opensearch/stable 2.7.0 amd64
opensearch/stable 2.6.0 amd64
opensearch/stable 2.5.0 amd64

```shell

```
```shell

```
```shell

```
```shell

```
```shell

```
```shell

```
```shell

```
```shell

```
```shell

```
```shell

```
```shell

```

<p align="center"><img src="screenshots/Ansible/ansible structure.png" width="90%" height="90%">
<br><em>Ansible structure</em> 
</p>

<p align="center"><img src="screenshots/Ansible/asnible playbook amazon linux run.png" width="90%" height="90%">
<br><em>Ansible-playbook results on Ec2</em> 
</p>

<p align="center"><img src="screenshots/Ansible/ansible playbook in the local inventory.png" width="90%" height="90%">
<br><em>Ansible-playbook results on local machines</em> 
</p>

<p align="center"><img src="screenshots/Ansible/connect to ec2.png" width="90%" height="90%">
<br><em>Connect to your Ec2</em> 
</p>


you can access you sonarqube by writing yourIp:9000

> [!NOTE]
> You can not run sonarqube in ec2 t2.micro instance. you need a t2.large.

<p align="center"><img src="screenshots/Ansible/sonarqube login.png" width="90%" height="90%">
<br><em>SonarQube login</em> 
</p>

<p align="center"><img src="screenshots/Ansible/create local project sonar.png" width="90%" height="90%">
<br><em>Create SonarQube project</em> 
</p>

### 4. Centralized Monitoring and Logging with OpenShift
Implemented a centralized monitoring and logging system within the OpenShift cluster as we mentioned in the instructions provided in the reposistory in the instructions folder(Instructions/Instructions-for-setup-for-centralized-loggingTask8.docx). We gain insights into application performance and overall cluster health, thus enhancing operational visibility and intelligence.

### 5. Containerization of the Java Application

Containerizing the Java application is a key part of this project, ensuring environment consistency and ease of deployment. The development of a Dockerfile, detailing all dependencies and configurations, leads to a containerized version of the Java application. 
> [!NOTE]
> Before building the image you need to: chmod +x gradlew

To build your image run the following command

```shell
$ sudo docker build -t nameOfYourApp:tag .
```

To run your image run the following command on a specific port

```shell
$ sudo docker run -d -p 8081:8080 (for example) nameOfYourApp:tag 
```
> [!NOTE]
> Note if the port does not run try to add this port in the firewall


To check your running containers

```shell
$ sudo docker ps
```

<p align="center"><img src="screenshots/docker/docker build .png" width="90%" height="90%">
<br><em>Docker build results</em> 
</p>

<p align="center"><img src="screenshots/docker/docker image run localhost 8081.png" width="90%" height="90%">
<br><em>Accessing Application </em> 
</p>

### 6. CI/CD Automation with Jenkins

Automate the CI/CD process using Jenkins, thereby streamlining the application deployment lifecycle. The development of a Jenkins pipeline script integrates various stages, including code build, testing, and deployment. This results in a seamless, automated pipeline that accelerates the release cycle and reduces manual intervention.

> [!NOTE]
> You need to add your credentials (dockerhub, github, openshift and sonarqube)

<p align="center"><img src="screenshots/jenkins/credentials.png" width="90%" height="90%">
<br><em>Jenkins credentials</em> 
</p>

<p align="center"><img src="screenshots/jenkins/jenkins final pipeline run.png" width="90%" height="90%">
<br><em>Success Pipeline</em> 
</p>

<p align="center"><img src="screenshots/jenkins/test.png" width="90%" height="90%">
<br><em>Unit Test</em> 
</p>

<p align="center"><img src="screenshots/jenkins/pushed to dockerhub.png" width="90%" height="90%">
<br><em>Jenkins compile Pushing to docker hub</em> 
</p>

<p align="center"><img src="screenshots/jenkins/push to docker hub final.png" width="90%" height="90%">
<br><em>Build and Push to docker hub</em> 
</p>

<p align="center"><img src="screenshots/jenkins/deploy to openshift.png" width="90%" height="90%">
<br><em>Deploy to openshift and create the resources </em> 
</p>

<p align="center"><img src="screenshots/jenkins/Deployments.png" width="90%" height="90%">
<br><em>Deployment in the openshift cluster</em> 
</p>

<p align="center"><img src="screenshots/jenkins/Service.png" width="90%" height="90%">
<br><em>Service created </em> 
</p>

<p align="center"><img src="screenshots/jenkins/Route.png" width="90%" height="90%">
<br><em>Route created </em> 
</p>

<p align="center"><img src="screenshots/jenkins/access application.png" width="90%" height="90%">
<br><em>Accessing the Application using route</em> 
</p>

### 7. Jenkins Shared Library
This repository houses a comprehensive Jenkins Shared Library designed to elevate your Continuous Integration and Continuous Deployment (CI/CD) workflows. By centralizing reusable Groovy scripts, this library aims to simplify pipeline definition, enhance code reusability, and streamline your Jenkins pipeline development.

> [!NOTE]
> Before running your shared library you need some configurations. check on allow default > version to be overridden and include @library changes in job recent changes.

<p align="center"><img src="screenshots/jenkins/global shared library.png" width="90%" height="90%">
<br><em>Global Pipeline Libraries</em> 
</p>

> [!NOTE]
> Add your GitHub Repo,credentials and your vars path in the library path 

<p align="center"><img src="screenshots/jenkins/shared library config.png" width="90%" height="90%">
<br><em>Global Pipeline Libraries</em> 
</p>

## Conclusion

This project successfully combines various advanced tools to create an efficient and automated system for deploying and managing a Spring Boot application. By using Jenkins for automation, OpenShift for orchestration, Terraform for setting up AWS infrastructure, Ansible for configuration, and Docker for containerization, we've streamlined the entire deployment process. This approach not only makes deploying applications quicker and more reliable but also ensures consistent performance and easy monitoring. The result is a straightforward, effective, and modern solution that meets the demands of today’s fast-paced software development and deployment needs.
