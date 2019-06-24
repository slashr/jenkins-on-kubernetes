# jenkins-on-kubernetes
1. Replace all instances of <account-id> with your AWS account ID.
2. Feel free to add or remove variables defined in the ```environment {}``` block inside Jenkinsfile
3. Jenkinsfile is just a template and can be customised as per your application/environment. It's a very fundamental blueprint of a CI/CD pipeline
4. This project uses AWS ECR as the image repository. If you happen to use some other repository, simply replace the link in ```spec.containers.image``` with your docker registry link inside the .yaml files.
5. Create an EBS volume and replace the volume ID inside ```persistent-volume.yaml```
6. Replace the registry URL inside ```docker-registry-config.yaml```. For GCR, you need an additional "authentication string" inside this file.
