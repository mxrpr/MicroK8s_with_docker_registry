# Guide to install Microk8s with multiple nodes with your own docker repository  

## micro8ks installation
### Install MicroK8s
```
sudo snap install microk8s --classic
```
if you are using multiple nodes, MicroK8s has to be installed on all nodes
### Check status on main node
```
microk8s status --wait-ready
```

### Enable common add-ons
run this command on the main node
```
microk8s enable dns dashboard ingress registry
```
After installation, we run check the dashboard functionality

### Create alias for easier command usage 
```bash
alias k='microk8s kubectl'
```
### Add your user to microk8s group 
```bash
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
```
With these steps, the installation is done on the main node. Run the following commands to check the state of the installation:
```bash
microk8s status #get the status
k get nodes     #get nodes
k get pods --all-namespaces #get pods
```
## Add another node 
On the main node, run the following command:
```bash
microk8s add-node
```
This will print out the command you have to execute on the other (new) node.
After the command is executed successfully, it will print out all the details.  
Check the available nodes on main node:
```bash
k get nodes
```
If you want to have more details, run the following command on main node:
```bash
k get nodes -o wide
```

## Check dashbard:
On the main node, run the following command:
```bash
microk8s dashboard-proxy
```
This command will print out the token you have to use to log in, also the url.
This feature requires the add-on to be enabled (microk8s enable dashboard)

After all these steps you have a 2 nodes cluster setup with dashboards.


## use your own docker registry - (https://github.com/Joxit/docker-registry-ui)

### Installation
I used this docker registry on another linux machine
```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  registry:2
```
### Push a docker image to registry
```bash
docker push <IP>:8080/<app-name>:<version>
```
### Get content of the registry with curl
```bash
curl http://<REGISTRY_HOST>:<PORT>/v2/_catalog
```
You can also use UI for registry, check the documentation.

### how to set your microk8s to use this newly installed docker registry to download images
On all nodes got to directory: "/var/snap/microk8s/current/args/certs.d"
Create a directory with the name of the registry server. In my case '192.168.0.127:8080'.
In this new diretory create a new file named 'hosts.toml', and add the following content:
```declarative
server = "http://192.168.0.127:8080"

[host."http://192.168.0.127:8080"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
```
Modify the IP address to your address and the port to the port you are using.

After all these changes you can push your app the registry and use in yaml files.
Example:
```bash
docker push 192.168.0.127:8080/my-java-app:1.1
```
```declarative
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-my-java-app-deployment
  labels:
    app: my-java-app
spec:
  replicas: 3 
  selector:
    matchLabels:
      app: my-java-app
  template:
    metadata:
      labels:
        app: my-java-app
    spec:
      containers:
      - name: my-java-app-container
        image: 192.168.0.127:8080/my-java-app:1.1
        #imagePullPolicy: Never
        ports:
        - containerPort: 8080 
```

### Enable metrics server
```bash
microk8s enable metrics-server
```
Run the following commands:
```bash
microk8s kubectl top nodes
microk8s kubectl top pods
```


