# Verdoso's SUSE Cloud Native Foundations Scholarship (SCNFS) Demo
This is an exercise I have implemented to practice what I have learnt at the SUSE Cloud Native Foundations Scholarship (SCNFS).

The same project simulates a travel planning web app, where you select number of passengers, the destination where you want to travel and the dates, and it returns a list of available hotels. Once you have chosen a hotel, it let's you select a departing airport (from the closest airports to the ip accessing the service) and an arriving airport (from the closest airports to the selected hotel). I might implement later on a flight selection if I find a free API to get data from.

It is implemented following the Microservice architecture, so the service is divided in three pieces:
* [The Travel Planner UI](https://github.com/Verdoso/Travel-Planner-UI), the customer facing interface that calls the other microservices to get the data.
* [The Airport Finder service](https://github.com/Verdoso/Airport-Finder), that returns a list of closest airport to an IP or a given position.
* [The HB Hotel Finder service](https://github.com/Verdoso/HBHotelFinder), that returns the available destinations to choose from and a list of available hotels and their prices given some criteria.

All pieces are published as docker images in my [Docker Hub](https://hub.docker.com/) space. The images are created using GitHub Actions on request.

If you want to test it yourself, you'll need to have a couple of free accounts:
* An account for the MaxMind GeoIP2 service - see https://www.maxmind.com/en/geoip2-services-and-databases
* An account for the Apitude API at the Hotelbeds Developer Portal  - see https://developer.hotelbeds.com/
* An account for the Skyscanner Flight Search API at the RapidAPI site - see https://rapidapi.com/skyscanner/api/skyscanner-flight-search/

##Testing in Kubernetes
Once you have vagrant up & running using the Vagrantfile in the home directory and you have k3s installed ([installation instructions here](https://rancher.com/docs/k3s/latest/en/installation/) ):

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

### Add the MaxMind GeoIP2 id and license to the secrets
Run the following command to patch the secrets with the MaxMind GeoIP2 data:
`kubectl create secret generic scnfs-secrets --save-config --dry-run=client --from-file=./geoip2.account-id --from-file=./geoip2.license-key -o yaml | kubectl apply -f -`

### Add the Hotelbeds api-key and shared-secrets to the secrets
Run the following command to patch the secrets with the Hotelbeds Apitude data:
`kubectl create secret generic scnfs-secrets --save-config --dry-run=client --from-file=./hotelbeds.api-key --from-file=./hotelbeds.shared-secret -o yaml | kubectl apply -f -`

### Add the SkyScanner API key and host to the secrets
Run the following command to patch the secrets with the Hotelbeds Apitude data:
`kubectl create secret generic scnfs-secrets --save-config --dry-run=client --from-file=./skyscanner.key --from-file=./skyscanner.host -o yaml | kubectl apply -f -`

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
