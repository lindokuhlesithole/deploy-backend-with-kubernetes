<h1 align="center">Deploy Backend with Kubernetes on Amazon EKS</h1>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS EKS" />
  <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes" />
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
  <img src="https://img.shields.io/badge/Amazon%20ECR-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="Amazon ECR" />
  <img src="https://img.shields.io/badge/Python-Flask-000000?style=for-the-badge&logo=flask&logoColor=white" alt="Flask" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS%20Solutions%20Architect-Professional-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS SA Pro" />
  <img src="https://img.shields.io/badge/AWS%20Security-Specialty-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=FF9900" alt="AWS Security" />
</p>

---

<img width="1125" height="580" alt="architecture-done" src="https://github.com/user-attachments/assets/9c8b6c4c-eaae-42d4-920f-dff44febdf1b" />


## Deploy Backend with Kubernetes

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-eks4_6cfb382f2)

---

## Introducing Today's Project!

For this particular project, I am going to launch the backend application using container orchestration on Amazon EKS since I have implemented the whole pipeline of container orchestration in a production-ready manner for scalable deployment.
The process would include the following steps: provisioning of an EC2 instance, retrieving code from GitHub and creating a Docker image of app and dependencies, uploading the image to Amazon ECR with its versioning. Afterwards, I will be using eksctl to initiate an EKS cluster via CloudFormation for the automatic deployment of the control plane, workers, and networking. Finally, I will define the Deployment, Service and Ingress manifests for Kubernetes, which would run on Amazon EKS using kubectl tool.
This is all the process of container orchestration.

### Tools and concepts

The main software used for this project was eksctl, kubectl, Docker, and the AWS CLI. eksctl deployed my EKS cluster and node groups via CloudFormation. Docker created a container image out of my Flask backend. The AWS CLI was responsible for authentication and kubeconfig updates. kubectl was my control plane console for deploying manifests, checking pods, and controlling the life cycle of Deployment and Service.
Containerization, orchestration, and declarative infrastructure were the central concepts of this project. I learned that a container image encapsulates everything necessary for its runtime, and the manifests define the desired state that is enforced by the control plane. In addition, I discovered that NodePort Services provide consistent network access despite transient pods, and IAM integration makes it possible for EKS worker nodes to access ECR without managing credentials manually.

### Project reflection

It took me approximately 5 days to work on this project, and each day was unique in its challenges.
On the first day, I provisioned the EKS cluster using eksctl and waited for CloudFormation. The second day was about working with Docker: I had to install it, make ec2-user part of the docker group, and finally get my first build done. The third day was ECR — creating the repository, authenticating, and pushing the image. The fourth day was the toughest for me. It was about creating the Deployment and Service YAML files and figuring out why kubectl couldn’t find my cluster till I issued the aws eks update-kubeconfig command. Finally, the fifth day was the celebration when I applied my manifests, watched the pods come up, and tested my NodePort.
5 days might seem a lot, but I’ve learned more about container orchestration in this week than in a month of tutorials.

---

## Project Set Up

### Kubernetes cluster

My Kubernetes cluster setup was done using eksctl, which uses CloudFormation to manage EKS deployments.
To start, I used an EC2 instance as my management machine where I setup my tools: eksctl to manage my cluster, kubectl for the Kubernetes API communication, and the AWS CLI for authentication purposes. After configuring my credentials so that eksctl would have access to my AWS account, I executed eksctl create cluster, which deployed everything for me - the EKS control plane, the managed node group of EC2 workers, the VPC, the subnets, security groups, and the IAM roles. 
After deployment, I checked if there was any connectivity by running kubectl get nodes, thus ensuring that the worker nodes were able to schedule workload.

### Backend code

The code was gotten by cloning the GitHub repository of my teammate into my build instance.
I installed Git using sudo dnf install git -y and tested whether it was installed using git --version. My Git credentials were set up to make sure that the environment was good for performing version control operations.
From there, I navigated to the GitHub repository, got the HTTPS clone URL and used the command git clone in my EC2 terminal to download the nextwork-flask-backend repository. Nextwork-flask-backend is a Python based backend developed using Flask framework by my teammate.
Upon doing the git clone, I executed the ls command and found out that the directory has been downloaded. After that, I changed the directory and examined all the files.

### Container image

I built a container image because Kubernetes does not run raw source code — it orchestrates containers spawned from images. My Flask backend exists as Python files on my EC2 instance, but EKS needs a self-contained artifact that includes the interpreter, all dependencies from requirements.txt, and the runtime configuration.
By building the image with Docker, I created a portable blueprint that behaves identically across any environment. Whether Kubernetes schedules it on node one or node fifty, the container starts from the same image and runs the exact same code with the same dependencies. This eliminates the "works on my machine" problem and ensures consistency.
The image is also what I will push to Amazon ECR, making it accessible to my EKS cluster for pulling and scheduling. Without this container image, my cluster has nothing to deploy.

The reason why I have pushed the container image to the ECR service is that Kubernetes clusters require a centralized registry where containers can be pulled to run the pods.
Although the container image was created locally on the EC2, I needed the cluster’s nodes to have access to the specific version of the image on demand. The ECR offers centralized storage services together with its integration into AWS, and since my nodes are authenticated using IAM roles, there is no need for me to worry about credentials management or image copying manually.
In addition, the ECR also manages the versioning by using image tagging. When I am tagging my container with the latest label, then I will refer to this tag in my Kubernetes deployment. Otherwise, without ECR, I would be required to manually pre-load all of the nodes and make updates independently, which is not practical at all.
ECR is the key of my pipeline.

---

## Manifest files

The manifest of Kubernetes is a declarative configuration file, normally written in YAML format. It dictates precisely how the Kubernetes cluster should go about deploying the app.
The desired state is specified by declaring which image to download, how many replicas of a pod to create, exposed ports, resource limits, environment variables, and network specifications. Upon applying a manifest through kubectl command, the control plane of Kubernetes reads it and keeps on working towards reaching the stated desired state. 
Without manifests, I will be left with the task of manually creating pods and assigning them to nodes and networking each time I want to deploy – an error-prone and impractical process on a large scale.

The Deployment manifest file will provide Kubernetes information regarding how my app should be deployed and scaled.
It describes the desired state: what container image is supposed to be used (ECR image), number of replicas, their labels, and which ports have to be exposed. Upon applying it to Kubernetes, the cluster control plane will create the corresponding Deployment object that will make sure that the actual state is always aligned with the declared one. In the event that a pod fails, another pod will automatically be created by the Deployment object, and if I update the image version, it will rollout an update.
In my specific example, it will tell Kubernetes to use nextwork-flask-backend image, create three replicas, and expose port 8080. The Deployment object will take care of the pods creation and distribution among worker nodes without any additional manual effort.

Kubernetes Service is a reliable network target that directs traffic to the group of pods.
Pods are dynamic in nature and constantly being created, killed, and scheduled again by the Deployment, hence their IPs keep changing. To address this problem, the Service provides one consistent IP address and uses labels for redirecting traffic to the right pods. This way, the Service works as a load balancer directing traffic to all healthy replicas without letting the client know to which specific pod it has been redirected.
In my case, the type of the Service is NodePort, which means that the app becomes available at a certain port in each worker node. Traffic that comes to any NodePort of any node is redirected by the Service to port 8080 in the appropriate pods.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-eks4_b01876554)

---

## Backend Deployment!

My backend deployment involved configuring kubectl to access my EKS cluster and deploying my Kubernetes YAML files.
At first, I installed kubectl on my EC2 machine and set it up as an executable program. Upon running kubectl version, the command was unsuccessful since kubectl could not figure out the location of my cluster – it was searching for localhost:8080 while my cluster resides in AWS.
The solution entailed running aws eks update-kubeconfig command that created the .kubeconfig file located in ~/.kube/config. This file contains the information about the cluster endpoint, certificate authority, and tokens needed for kubectl to access the EKS API server.
After getting authenticated and pointing kubectl to my cluster, I executed the kubectl apply -f on my two YAML files – flask-deployment.yaml and flask-service.yaml. It instructed the control plane to create a Deployment and schedule the pods in three copies and activate the NodePort service. My backend is live.

### kubectl

The kubectl tool is the CLI for the Kubernetes API and is the primary means of provisioning and managing the resources within a running Kubernetes cluster.
Whereas eksctl takes care of the management of the EKS cluster itself – setting up and tearing down infrastructure – the kubectl tool is responsible for provisioning resources within the application layer. It is the bridge from my YAML manifests into the API calls to set up Deployments, schedule pods, configure Services, and manage configuration.
When I use kubectl apply -f, I'm telling the cluster what I want the state of my application to be. Then the Kubernetes controller ensures that it achieves that state – it provisions containers and distributes them among the nodes while ensuring the replica count I specified.
In summary: eksctl creates the platform. kubectl runs the applications on the platform.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-eks4_6cfb382f2)

---
## Author

**Lindokuhle Sithole** - *Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist*

Based in Bremen, Germany. BSc Mathematical Science from the University of the Witwatersrand. 5x AWS Certified (Solutions Architect Professional, Security Specialty, CloudOps Engineer Associate, Solutions Architect Associate, Cloud Practitioner) plus CompTIA Security+.

- **LinkedIn:** [linkedin.com/in/lindokuhle-sithole-bb701b19a](https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a)
- **GitHub:** [github.com/lindokuhlesithole](https://github.com/lindokuhlesithole)
- **Email:** sitholelindokuhle371@gmail.com

---
