# DevOps CI/CD pipelines using Git Jenkins Ansible Docker and Kubernetes on AWS

# Step 1: Create EC2 instances for Ansible, Jenkins and Kubernetes
On your AWS console, search for EC2 in the top search bar as shown in the figure below and click on it

<img width="686" alt="Screenshot 2024-01-19 at 09 08 32" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/ac19c1f3-e81d-4fd2-9317-dd3e671fbb29">

On that screen scroll down and click the “Launch Instance” menu button and click “Launch Instance” in sub menu as shown below:
<img width="690" alt="Screenshot 2024-01-19 at 09 08 48" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/8cd981ff-907b-42e3-949d-246a5367dcd0">


On the create instance page, give a temporary name like “ansible-jenkins” and select Ubuntu Server 22.04 LTS, which is marked “free tier eligible” as shown in the image below
<img width="664" alt="Screenshot 2024-01-19 at 09 09 10" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/90c914ae-6c28-45f8-b4e0-961a5e73642b">


Select t2.micro as the instance type (again marked for free tier). Click “Create Key Pair”. In the pop-up window give a name for your login key pair file and click create button
<img width="650" alt="Screenshot 2024-01-19 at 09 09 22" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/dd229582-92ef-4e19-a945-ba3025c60e39">


Create Security Group in the default VPC by giving it a suitable name and allowing “All traffic” from your system’s IP
<img width="682" alt="Screenshot 2024-01-19 at 09 09 37" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/47554441-2748-4b1b-bd00-0edb1dd6c445">


And in the right side, enter 2 as the number of instances and click Launch Instance button in the bottom. Now in EC2 instances page, change the names of each of the instances, name any one of them as “ansible-server” and another one as “jenkins-server”

Now create another instance for Kubernetes cluster of type t2.medium. Give it a suitable name like “web-server”. Select same VPC and Security Group as the ones created for previous two instances. Finally in your EC2 instances page you should have all three instances.

<img width="669" alt="Screenshot 2024-01-19 at 09 09 51" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/a16c3733-bd54-4ad6-bc4b-68f675aea52c">

Now select any one instance and in the details pane below, click the Security tab. Click on the security group name link. A detail page of the security group will be displayed. Click “Edit inbound rules” button. In the edit page, we need to add foloowing rules:
i) A dummy rule, to allow all traffic from itself i.e the Security Group itself
ii) Allow “All traffic” from your IP.
iii) Allow “All traffic” from GitHub webhook. You can find the list of IP addresses from here (the IP addresses mentioned in “hooks” parameter in the link)
iv) Allow SSH protocol from Jenkins server to Ansible server
v) Allow SSH protocol from Ansible server to Kubernetes/Web server
<img width="698" alt="Screenshot 2024-01-19 at 09 10 04" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/205a6f00-23b7-4c2c-9cf5-887a01767fd2">


# Step 2: Setup Password and SSH access without password
Login in to each of the EC2 instances in three separate terminal from your system. On EC2 instances page, put a check against each instances (one at a time) and in the details pane below, copy the public IP.

Login to Jenkins server: ssh -i <download-location-of-login-key-pair-pem-file> ubuntu@<public-ip-of-jenkins-server>

Login to Ansible sever: ssh -i <download-location-of-login-key-pair-pem-file> ubuntu@<public-ip-of-ansible-server>

Login to Kubernetes/Web sever: ssh -i <download-location-of-login-key-pair-pem-file> ubuntu@<public-ip-of-web-server>

On each shell become the root user by running sudo -i and execute the following commands :
i) To set a password for root user run: passwd and enter your password
ii) To set a password for “ubuntu” user: passwd ubuntu

Once password is set for root and ubuntu users on each instances, we need to enable SSH login from Jenkins to Ansible, Jenkins to Web and Ansible to Web Server, so that Jenkins can push the latest code/files on each of those servers and Ansible can deploy the latest app on the Web server.

Now we need to enable SSH access without a password between them. On each of the connected instance shells do the following:

a) Generate SSH keys: ssh-keygen -t rsa In the resulting prompts simply press enter consecutively.

b) Once the keys are generated, open config file for SSH in vim or any favourite CLI editor of your choice. vi /etc/ssh/sshd_config . Search for PermitRootLogin :/PermitRootLogin It would be commented by default. Remove # in front it and set the value as Yes. Do the same for PasswordAuthentication . I know it’s ironic that to enable passwordless connection for SSH you need to enable password based authentication first. Once done, save and quit the edit :wq and restart SSH service: service sshd restart .

To connect Jenkins to Ansible, on the shell connected to Jenkins server run ssh-copy-id ubuntu@<ec2-private-ip-ansible-server> to copy the public key of ubuntu user to Ansible server. You’ll be prompted to enter the password of ubuntu user on Ansible server. Upon entering the correct password you would be logged in to the Ansible server from the Jenkins instance.

Now from each Jenkins and Ansible instance shell, do the same to connect to Kubernetes/Web server ssh-copy-id ubuntu@<ec2-private-ip-web-server>

To test run ssh connect from Jenkins to Ansible, Jenkins to Web server and Ansible to Web server. And you should be able to do it without the password prompt.

# Step 3: Install Jenkins
To install Jenkins perform the following steps in the SSH connected shell:
a) First, add the repository key to the system:

wget -q -O - http://pkg.jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
b) Append the Debian package repository address to the server’s sources.list:

sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
c) Run update so that the apt will use the new repository: sudo apt update

d) Install OpenJDK: sudo apt install openjdk-11-jdk

e) Finally, install Jenkins and its dependencies: sudo apt install jenkins
and check it’s status after installation: systemctl status jenkins

# Step 4: Set Jenkins Panel and Plugin
Now it’s time to configure Jenkins as a CI/CD tool. Head over to your browser and enter <public-ip-of-jenkins-server>:8080 in the address bar. It will mention that the default password is located in /var/lib/jenkins/secrets/initialAdminPassword Copy the password from this file and paste in the input text box.

<img width="667" alt="Screenshot 2024-01-19 at 09 10 49" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/73d3c6b5-c24c-4da6-a2d6-ef0291e46f0e">

Click install selected plugins. Once done, create admin user


Now we need to add SSH Agent plugin for Jenkins, so that this server can perform SSH to other two servers during the pipeline operation. Go to plugins from “Manage Jenkins” menu in sidebar

<img width="700" alt="Screenshot 2024-01-19 at 09 11 04" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/c4e04b58-1121-4ef6-95cd-b12261626e67">

Search for “ssh agent” in the “Available Plugins” from left side menu. Select it from the list item and click install without restart
<img width="713" alt="Screenshot 2024-01-19 at 09 11 14" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/c22ab44c-70b8-4262-b845-962143eb2395">


# Step 5: Set Ansible Server
We need to setup Docker and Ansible on this EC2 instance.
So login to the instance via SSH. This time create a shell script file to setup both the tools. Enter vi ansible-server-setup.sh and enter the following content

#!/bin/sh
#install ansible
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt update -y
sudo apt install ansible -y

#now install docker
sudo apt-get update
sudo apt-get install -y  \
    ca-certificates \
    curl \
    gnupg

sudo install -y -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo rm -rf /etc/docker/daemon.json
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -a -G docker ubuntu 
sudo systemctl start docker
Make this file executable but running chmod +x ansible-server-setup.sh And then execute the file begin installation: ./ansible-server-setup.sh Now configure the Web server as a host for Ansible. Edit /etc/ansible/hosts file and at the end the file, enter the following :

[webnode]
<private-ip-of-web-server>
To test simply use the ping module of ansible: ansible -m ping webnode
If you see success message in json format then our Ansible server setup is ready for action.

# Step 6: Set Kubernetes/Web Server
On this server we need to install Docker, Kubectl and Minikube.
So like the previous step we will keep the commands in a shell script file and execute it to install the required tools

#!/bin/sh
#install docker
sudo apt-get update
sudo apt-get install -y  \
    ca-certificates \
    curl \
    gnupg

sudo install -y -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo rm -rf /etc/docker/daemon.json
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -a -G docker ubuntu 
sudo systemctl start docker

#install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# Step 7: Setting up Jenkins Pipeline
First we need to create a project. So in the Jenkin’s panel in the browser, click New Item in the left menu
<img width="678" alt="Screenshot 2024-01-19 at 09 11 37" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/72572d69-b9b2-4369-b18f-b1116a7cf5b4">


In the new screen, enter a name and select Pipeline from the displayed list and click Ok button.
<img width="677" alt="Screenshot 2024-01-19 at 09 11 53" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/615297fa-5fbc-4083-8543-223e4fe1daf6">


In the next screen select “GitHub hook trigger for GITScm polling”. Then click Apply and Save button
<img width="699" alt="Screenshot 2024-01-19 at 09 12 10" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/4d8ca44a-ddea-4be9-9cc7-b5f80f9293f3">


Next we need to generate a token in Jenkin’s panel that will be used in the GitHub webhook for triggering Jenkin’s pipeline on code push. So click the admin name/icon in the top right corner and click Configure from left menu. In the new screen scroll to API Token section, click Add new Token and click Generate button. Copy and save this generated token somewhere in your notepad

<img width="715" alt="Screenshot 2024-01-19 at 09 12 22" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/2f47d39a-fb4a-4d7a-9dfe-1dfb880aafa6">

Now head over to your GitHub repository containing application/code files. Then click on Settings and in the new window click Webhooks from the left menu and click Add Webhook button in the content pane

<img width="671" alt="Screenshot 2024-01-19 at 09 12 39" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/ff1b7de9-a5d7-4482-b1c9-5390b4ac78df">

In the payload url, enter the public IP url of your Jenkins server and append it with /github-webhook/ . Note the trailing forward slash. In the Secret input text field paste the token copied from the Jenkin’s panel. Keep the rest of the fields in the form unchanged and hit Add Webhook

<img width="671" alt="Screenshot 2024-01-19 at 09 12 54" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/5e23c9ce-7e66-463a-86f3-c3a831a595d8">

It will check the connectivity as shown in the above image as a grey circle icon besides the webhook url. You can hit refresh button of your browser, if it is taking time. That icon should turn into a green check mark

<img width="670" alt="Screenshot 2024-01-19 at 09 13 03" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/375b48b7-5420-40c1-a67a-23c374286f6c">

We should have the following content in our repo:

i) Dockerfile

FROM centos:latest
RUN cd /etc/yum.repos.d/
RUN sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
RUN sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
RUN yum install httpd wget zip unzip -y
ADD https://www.tooplate.com/zip-templates/2121_wave_cafe.zip /var/www/html
WORKDIR /var/www/html
RUN unzip -o 2121_wave_cafe.zip
RUN cp -r 2121_wave_cafe/* .
RUN rm -rf 2121_wave_cafe 2121_wave_cafe.zip
CMD ["/usr/sbin/httpd","-D","FOREGROUND"]
EXPOSE 80 30000
ii) Deployment.yml file

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myfirstdevopsappdeployment
spec:
  replicas: 5
  selector:
    matchLabels:
      name: myapp
  template:
    metadata:
      labels:
        name: myapp
    spec:
      containers:
        - name: myapp
          image: kubemubin/devops-project-one
          ports:
            - containerPort: 80
iii) Service.yml file

kind: Service
apiVersion: v1
metadata:
  name: myfirstdevopsservice
spec:
  selector:
    name: myapp
  ports:
    - protocol: TCP
      # Port accessible inside cluster
      port: 80
      # Port to forward to inside the pod
      targetPort: 80
      # Port accessible outside cluster
      nodePort: 30000
  type: NodePort
iv) Ansible playbook file

- hosts: all
  become: true
  become_user: root
  tasks:
    - name: delete old deployment
      command: kubectl delete -f /home/ubuntu/Deployment.yml --kubeconfig=/home/ubuntu/.kube/config
      ignore_errors: true
    - name: delete old service
      command: kubectl delete -f /home/ubuntu/Service.yml --kubeconfig=/home/ubuntu/.kube/config
      ignore_errors: true
    - name: create new deployment
      command: kubectl apply -f /home/ubuntu/Deployment.yml --kubeconfig=/home/ubuntu/.kube/config
    - name: create new service
      command: kubectl apply -f /home/ubuntu/Service.yml --force --kubeconfig=/home/ubuntu/.kube/config
Now back to the Jenkin’s panel, click on your project name and then click Configure from the left menu. In the content pane, scroll down to the Pipeline section, select “Pipeline Script” from the drop down. Below the textarea field there is a link to the Pipeline syntax. Click on it to setup certain global variables and credeentials for our pipeline script. In the new window do the following:

a) Setup ansible-server and kubernetes-server SSH Agent variables.
Select sshagent option from the Sample Step dropdown and click Add button and click Jenkins icon from it.

<img width="686" alt="Screenshot 2024-01-19 at 09 13 27" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/664cd0a0-e99a-4f86-a43a-ad4e1a2ff101">

In new pop-up window, select “SSH username with private key”. Then enter ID and description as ansible-server. Then set the username as “ubuntu”. Then select the Enter directly option in Private key and click add. In the resulting input field paste the content of .pem file which you downloaded while creating login key pair. Click Add.
<img width="698" alt="Screenshot 2024-01-19 at 09 13 37" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/45ab1d1b-a7e9-4bb7-9a0d-e84d5512d4e2">


Do the same for Kubernetes/Web server and generate kubernetes-server variable.

b) Generate variable for DockerHub automatic login from Ansible Server
Register on DockerHub then on the Pipeline Syntax page. Select withCredentials in the Sample Step dropdown field. Click Add button under Binding section and select “secret text” and enter variable name, like, “dockerhub_passwd”. Then under credentials sub-section, click add, click Jenkins icon. In the pop-up window, select secret text as kind and enter the ID same variable name as variable name you used. Click add to close pop-up window

Enter the following content in the Script textarea field

ansible_server_private_ip="<your-ansible-server-private-ip>"
kubernetes_server_private_ip="<your-web-server-private-ip>"

node{
    stage('Git checkout'){
        //replace with your github repo url
        git branch: 'main', url: 'https://github.com/khalifemubin/devops-project-one.git'
    }
    
     //all below sshagent variables created using Pipeline syntax
    stage('Sending Dockerfile to Ansible server'){
        sshagent(['ansible-server']) {
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip}"
            sh "scp /var/lib/jenkins/workspace/devops-project-one/* ubuntu@${ansible_server_private_ip}:/home/ubuntu"
        }
    }
    
    stage('Docker build image'){
        sshagent(['ansible-server']) {
         //building docker image starts
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} cd /home/ubuntu/"
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image build -t $JOB_NAME:v-$BUILD_ID ."
         //building docker image ends
         //Tagging docker image starts
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image tag $JOB_NAME:v-$BUILD_ID kubemubin/$JOB_NAME:v-$BUILD_ID"
         sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image tag $JOB_NAME:v-$BUILD_ID kubemubin/$JOB_NAME:latest"
         //Tagging docker image ends
        }
    }
    
    stage('push docker images to dockerhub'){
     sshagent(['ansible-server']) {
      withCredentials([string(credentialsId:'dockerhub_passwd', variable: 'dockerhub_passwd')]){
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker login -u kubemubin -p ${dockerhub_passwd}"
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image push kubemubin/$JOB_NAME:v-$BUILD_ID"
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image push kubemubin/$JOB_NAME:latest"
       
       //also delete old docker images
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image rm kubemubin/$JOB_NAME:v-$BUILD_ID kubemubin/$JOB_NAME:latest $JOB_NAME:v-$BUILD_ID"
      }
        }
    }
    
    stage('Copy files from jenkins to kubernetes server'){
     sshagent(['kubernetes-server']) {
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${kubernetes_server_private_ip} cd /home/ubuntu/"
      sh "scp /var/lib/jenkins/workspace/devops-project-one/* ubuntu@${kubernetes_server_private_ip}:/home/ubuntu"
     }
    }
 
    stage('Kubernetes deployment using ansible'){
     sshagent(['ansible-server']) {
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} cd /home/ubuntu/"
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} ansible -m ping ${kubernetes_server_private_ip}"
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} ansible-playbook ansible-playbook.yml"
     } 
    }
 
}
Let’s go through an overview of the pipeline script:

The first stage is to do git checkout of the repository. So whenever the changes are pushed to the main branch of the repo, this is the first thing pipeline will do.

Second step is to send everything (in the /var/lib/jenkins/workspace/<your-jenkins-projectname>/*) to the Ansible server at the home directory of ubuntu user.

Third stage is to build a Docker image with $JOB_NAME:v-$BUILD_ID format and tag the image.

Fourth stage is to push this built docker image to Dockerhub and delete the old image from the Ansible server.

Penultimate stage is to copy files from the checked out git repo to Kubernetes/Web server to the home directory of the ubuntu user.

Final stage is for Ansible server to run the playbook on Kubernetes/Web server.

# Step 8: Start Minikube and See CI/CD in action
On the Kubernetes/Web server run minikube start . Since docker is already installed on this server it is similar to running minikube start — driver=docker . See the kubernetes cluster details kubectl get all . Now since we have nothing, lets add a dummy READMe.md file to our repo and push. Open up your project screen on Jenkin’s panel and you should see a the job getting built automatically
<img width="706" alt="Screenshot 2024-01-19 at 09 14 00" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/a4c6f377-dfa2-4c1c-a7d2-4a88d051261d">


Now in the browser go to the public ip of web server and append node port to it: http://<public-ip-of-web-server>:30000

But there seems to be a problem. The web page is not rendering. Let’s do a small fix. Let’s forward the port of our service by running:

kubectl port-forward - address 0.0.0.0 svc/myfirstdevopsservice 30000:80 &
Et voilà !


<img width="674" alt="Screenshot 2024-01-19 at 09 14 12" src="https://github.com/redjules/DevOps-CI-CD-pipelines-using-Git-Jenkins-Ansible-Docker-and-Kubernetes-on-AWS/assets/106017493/a3a37094-83a9-4ded-8628-23a6e807d755">
