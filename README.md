## README / Instructions ##

##############################################################################################

## Solution / Assumptions ##

1) This will install a DB and App in one namespace.
2) The configurations are templated and new instance of DB and App can be created just by adding a new namespace and a new values file, which we will see in below steps.
3) Only one instance of App and DB to be deployed in a namespace.
4) This will also create an ingress object (route) to access app from outside the cluster
5) No Dynamic Storage Provisioning. No PV/PVC used for persistent storage. I am using hostPath as I have not setup NFS/GlusterFS or other storage and LocalSettings.php needs to be updated (and stateless pod for mediawiki app pod won't work).
My K8s cluster has just one worker node, so I am going to create a /mediawiki directory on it. If your setup has multiple worker nodes, then you might need NFS and share has to be mounted on all worker nodes to use hostPath or use PV/PVC's for this setup to work. Same /mediawiki directory will be use for multiple instances of mediawiki.
6) DB (Mysql) pod is stateless for this exercise.
7) Have use mediawiki version 1.31.10, as this is the latest stable version.
8) SSL Termination is not done.
9) I am using docker, helm and Kubernetes stack.
10) docker image build is manual.
11) There is no development/customization done to mediawiki. Its Vanilla installation

## Future enhancements ##
1)Use PVC for persistent storage.
2)SSL Termination either at External Load balancer or SSL with ingress.
3)There should be a multibranch Jenkins pipeline which should build and deploy not just new instances (following below explained steps put in a pipeline) and also trigger a build and deployment when code is pushed to repo by developers. 

## Prerequisites on the node from where the installation is done ##

1) Kubernetes Cluster in is place and you can run cluster commands from this node using kubectl.
2) helm version 3 is installed.
3) docker installed to build the images.


###############################################################
###### Installation using docker and helm on Kubernetes  ######
###############################################################

##### Build and Install DB on the cluster  #####

# goto your home dir on the server where you intend to run the installations.

cd ~

# Clone the repo

git clone https://github.com/ShivamMudotia/mediawiki-db.git

# cd to repo dir

cd mediawiki-db

# Build the image

docker build -t mshivam21/mediawiki-db:latest .

# login to docker repository - Here it is my dockerhub account

docker login 

#Push the image to the registry

docker push mshivam21/mediawiki-db:latest

# cd to helm-charts/mediawiki-db to install db through helm.

cd helm-charts/mediawiki-db

# create a namespace on cluster

kubectl create ns dev

# Install the chart. The pod should be up and running in dev namespace.

helm install wiki-db-dev . -f values.dev.yaml --namespace=dev


##### Build and Install App on the cluster, which would use above created DB as its Database. #####

# goto you home dir.

cd ~

# Clone the repo

git clone https://github.com/ShivamMudotia/mediawiki-app.git

# cd to repo dir

cd  mediawiki-app

# Manually copy LocalSettings.php to worker node at /mediawiki/dev (dev is the instance name here in the path - create this dir)

# Build the image

docker build -t mshivam21/mediawiki-app:latest .

# login to docker repository - Here it is my dockerhub account

docker login 

#Push the image to the registry

docker push mshivam21/mediawiki-app:latest

# cd to helm-charts/mediawiki-db to install db through helm.

cd helm-charts/mediawiki-app

# This will be deployed on same namespace as DB, so no need for a new namespace.
# Install the chart. The pod should be up and running in dev namespace and using DB create abobe.

helm install wiki-app-dev . -f values.dev.yaml --namespace=dev
 
# Access the App from outside. Make sure to have appropriate DNS entries or local host entry to External Load balancer/worker node (or wherever you have ingress-controller) as per your setup.

http://dev.mediawiki.mydomain.com

# If you see below error, you might need to fix the installation.

"Fatal exception of type Wikimedia\Rdbms\DBQueryError"

#Access the below link and follow instructions. At the end a LocalSettings.php file will be generated, which need to be uploaded to worker node at /mediawiki/dev


https://dev.mediawiki.mydomain.com/mw-config

# Access the below link again and you should be good now.

http://dev.mediawiki.mydomain.com

# There might/should be a better way to handle this error.
# We are trying to inject some DB specific variables - If we do not inject a LocalSettings.php, then there won't be any errors and it be will a fresh installation screen. 

########################################################################################
#########  deploy another instances using same process on another namspace #############

###  Now there will be fewer steps when creating a new instance.

# Create a new namespace "test"

kubectl create ns test

# Manually copy LocalSettings.php from the repo to worker node at /mediawiki/test (test is the instance name here in the path - create this dir)


## Create DB

cd  ~/mediawiki-db/helm-charts/mediawiki-db
helm install wiki-db-test . -f values.test.yaml --namespace=test

## Create App

cd  ~/mediawiki-app/helm-charts/mediawiki-app
helm install wiki-app-test . -f values.test.yaml --namespace=test

## Access URL # Make sure to have appropriate DNS entries or local host entry
# If you see same error, fix it in same way.

http://test.mediawiki.mydomain.com


### Similary, make more values.<instance>.yaml files for DB and App and use them to create more instance

################################################
#######Now - Let's Scale this test instance #######

## Change number of replicas from 1 to 3 in below file

vi ~/mediawiki-app/helm-charts/mediawiki-app/values.test.yaml

# Run a helm upgrade. This will scale test app pods from 1 to 3.

helm upgrade wiki-app-test . -f values.test.yaml


#########################################################################################

###################################
### Cleanup - Delete all stacks ###
###################################

helm delete wiki-app-dev -n dev
helm delete wiki-db-dev -n dev
helm delete wiki-app-test -n test
helm delete wiki-db-test -n test


##########################################################################################


 