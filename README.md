# Verdoso's SUSE Cloud Native Foundations Scholarship (SCNFS) Demo
This is an exercise I have implemented to practice what I have learnt at the SUSE Cloud Native Foundations Scholarship (SCNFS).

You can check out this small presentation I did in [Genially](https://www.genial.ly/en) about the project:
[![Genially's Presentation](/helm-chart/img/GeniallysSnapshot.png)](https://view.genial.ly/60e3199ef85d7f0dbcbff023/presentation-verdosos-scnfs-demo)

The project simulates a travel planning web app, where you select number of passengers, the destination where you want to go and the travelling dates, and it returns a list of available hotels. Once you have chosen a hotel, it lets you select a departing airport (from the closest airports to the ip accessing the service) and an arriving airport (from the closest airports to the selected hotel). Once you have selected both airports, you can retrieve information about the cheapest flights to go there.

Bear in mind this is a rough demo. In a real application you would display more information, calculate prices propertly etc. The main goal in my case was to develop a set of microservices that can be deployed using Github Actions, that can be configured with secrets, that communicate with each other. I did not want to implement simply an extended Hello World, so I did this.

All the services, so far, have been implemented using Java 16 / Spring Boot because that's the most familiar framework to me. This way it was easier for me to create all services, externalise the configuration, use secrets

It is implemented following the Microservice architecture, so the service is divided in three pieces:
* [The Travel Planner UI](https://github.com/Verdoso/Travel-Planner-UI), the customer facing interface that calls the other microservices to get the data.
* [The Airport Finder service](https://github.com/Verdoso/Airport-Finder), that returns a list of closest airport to an IP or a given position.
* [The HB Hotel Finder service](https://github.com/Verdoso/HBHotelFinder), that returns the available destinations to choose from and a list of available hotels and their prices given some criteria.
* [The SkyScanner Flight Finder service](https://github.com/Verdoso/skyscanner-flight-finder), that returns the flight information to travel to the selected airport, close to the selected hotel, and back.

All pieces are published as docker images in my [Docker Hub](https://hub.docker.com/) space. The images are created using GitHub Actions on request.

If you want to test it yourself, you'll need to have a couple of free accounts:
* An account for the MaxMind GeoIP2 service - see https://www.maxmind.com/en/geoip2-services-and-databases
* An account for the Apitude API at the Hotelbeds Developer Portal  - see https://developer.hotelbeds.com/
* An account for the Skyscanner Flight Search API at the RapidAPI site - see https://rapidapi.com/skyscanner/api/skyscanner-flight-search/

## Deployment in ArgoCD
![ArgoCD application snapshot](/helm-chart/img/Deployment.png)

## Testing in plain Kubernetes
Once you have vagrant up & running using the Vagrantfile in the home directory and you have k3s installed ([installation instructions here](https://rancher.com/docs/k3s/latest/en/installation/) ), follow these steps:

**Note:** There is a bug in Windows with Hypervisor & K3s so until is is solved, you should install version v1.20.7+k3s1 with `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.7+k3s1 sh`

Move to the `/vagrant/deployment` directory

### Create the **namespace**
`kubectl apply -f namespace.yaml`

### Create the empty secrets holder
`kubectl apply -f scnfs-secrets.yaml`

### Add the real secrets
* MaxMind GeoIP2 secrets: Create a file named *geoip2.account-id* with the id, and another one named *geoip2.license-key* with the license key, of your MaxMind GeoIP2 service account.
* Hotelbed secrets: Create a file named *hotelbeds.api-key* and another one named *hotelbeds.shared-secret* and fill them up with your Apitude license data from the Hotelbeds Developer site.
* SkyScanner secrets: Create a file named *skyscanner.key* and another one named *skyscanner.host* and fill them up with your SkyScanner API authentication data from the RapidAPI site.

No worries, git is configured to ignore those files so they won't be added to the source repository.

### Add the configuration data to the secrets
Run the following command to patch the secrets with theauthentication data:
`kubectl create secret generic scnfs-secrets --save-config --dry-run=client --from-file=./geoip2.account-id --from-file=./geoip2.license-key --from-file=./hotelbeds.api-key --from-file=./hotelbeds.shared-secret --from-file=./skyscanner.key --from-file=./skyscanner.host -o yaml | kubectl apply -f -`

 ### Deploy the airport-finder application
`kubectl apply -f airport-finder-deployment.yaml`

 ### Create the service to support the airport-finder application
`kubectl apply -f airport-finder-service.yaml`

 ### Deploy the hb-hotel-finder application
`kubectl apply -f hb-hotel-finder-deployment.yaml`

 ### Create the service to support the hb-hotel-finder application
`kubectl apply -f hb-hotel-finder-service.yaml`

 ### Deploy the skyscanner-flight-finder application
`kubectl apply -f skyscanner-flight-finder-deployment.yaml`

 ### Create the service to support the skyscanner-flight-finder application
`kubectl apply -f skyscanner-flight-finder-service.yaml`

 ### Deploy the travel-planner application
`kubectl apply -f travel-planner-deployment.yaml`

 ### Create the service to support the travel-planner application
`kubectl apply -f travel-planner-service.yaml`

## Useful commands

### Set up your default context to be 'scnfs-demo'
`kubectl config set-context --current --namespace=scnfs-demo`

### Prepare the script to find the airport-finder pod name to use in commands'
`alias affindpod='POD=$(kubectl get pod -l app=airport-finder -o jsonpath="{.items[0].metadata.name}")'`

### Prepare the script to find the hb-hotel-finder pod name to use in commands'
`alias hbfindpod='POD=$(kubectl get pod -l app=hb-hotel-finder -o jsonpath="{.items[0].metadata.name}")'`

### Prepare the script to find the skyscanner-flight-finder pod name to use in commands'
`alias skfindpod='POD=$(kubectl get pod -l app=skyscanner-flight-finder -o jsonpath="{.items[0].metadata.name}")'`

### Prepare the script to find the travel-planner pod name to use in commands'
`alias tpfindpod='POD=$(kubectl get pod -l app=travel-planner -o jsonpath="{.items[0].metadata.name}")'`

### Make the travel-planner application visible from your machine
`tpfindpod;kubectl port-forward po/$POD --address 0.0.0.0 7777:7777` \
After that you should be able to access the Travel Planner Demo at http://192.168.50.4:7777/ as 192.168.50.4 is the IP of the Vagrant machine.

### Check that the pod and the service are up & running
`kubectl get all`

### Check the airpot-finder pod details
`affindpod;kubectl describe po $POD`

### Check the hb-hotel-finder pod details
`hbfindpod;kubectl describe po $POD`

### Check the skyscanner-flight-finder pod details
`skfindpod;kubectl describe po $POD`

### Check the travel-planner pod details
`tpfindpod;kubectl describe po $POD`

### Check airpot-finder pod logs
`affindpod;kubectl logs --follow $POD`

### Check hb-hotel-finder pod logs
`hbfindpod;kubectl logs --follow $POD`

### Check skyscanner-flight-finder pod logs
`skfindpod;kubectl logs --follow $POD`

### Check travel-planner pod logs
`tpfindpod;kubectl logs --follow $POD`

### Open a terminal in the airpot-finder log, in case you need to check things
`affindpod;kubectl exec --stdin --tty $POD -- /bin/bash`

### Open a terminal in the hb-hotel-finder log, in case you need to check things
`hbfindpod;kubectl exec --stdin --tty $POD -- /bin/bash`

### Open a terminal in the skyscanner-flight-finder log, in case you need to check things
`skfindpod;kubectl exec --stdin --tty $POD -- /bin/bash`

### Open a terminal in the travel-planner log, in case you need to check things
`tpfindpod;kubectl exec --stdin --tty $POD -- /bin/bash`

### To remove the services and deployments
`kubectl delete -f airport-finder-deployment.yaml` \
`kubectl delete -f airport-finder-service.yaml` \
`kubectl delete -f hb-hotel-finder-deployment.yaml` \
`kubectl delete -f hb-hotel-finder-service.yaml` \
`kubectl delete -f skyscanner-flight-finder-deployment.yaml` \
`kubectl delete -f skyscanner-flight-finder-service.yaml` \
`kubectl delete -f travel-planner-deployment.yaml` \
`kubectl delete -f travel-planner-service.yaml` \

## Testing using ArgoCD

You first need to have ArgoCD installed and reachable from your host, as explained in the course material or following the [ArgoCD installation instructions](https://argoproj.github.io/argo-cd/getting_started/).

Assuming you are already inside Vagrant with ssh:

### Move to the project directory:
`cd /vagrant`

### Apply the Helm templates to deploy the cluster
`kubectl apply -f argocd-helm-scnfs-demo.yaml`

### Initialise the namespace and the secret in the cluster
Login to the ArgoCD interface and synchronise just the namespace and the secrets elements

### Add the configuration to the secrets element
`cd deployment`
`kubectl create secret generic scnfs-secrets --save-config --dry-run=client --from-file=./geoip2.account-id --from-file=./geoip2.license-key --from-file=./hotelbeds.api-key --from-file=./hotelbeds.shared-secret --from-file=./skyscanner.key --from-file=./skyscanner.host -o yaml | kubectl apply -f -`

### Initialise the rest of the cluster
In the ArgoCD interface, synchronise the rest of the elements. DO NOT synchronise the secrets again or it will lose the secrets you just configured.