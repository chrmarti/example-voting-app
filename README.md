Deploying Cats & Dogs on Azure Container Service
================================================

Using the azure CLI in a container:

```
docker run -it microsoft/azure-cli
```

In the container deploy an Azure Container Service with Swarm ([on SSH keys](https://help.github.com/articles/generating-an-ssh-key/)):

```
azure login
# Use some trial subscription
azure account list
azure account set "Try Out Subscription"
# Replace $sshRSAPublicKey$ with your own SSH public key
azure group create -n mycatsndogs mycatsndogs -l westus --template-uri https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-acs-swarm/azuredeploy.json -p '{ "dnsNamePrefix": { "value": "mycatsndogs" }, "agentCount": { "value": 2 }, "sshRSAPublicKey": { "value": "$sshRSAPublicKey$" } }'
```

Back on the host:

```
# This will only work after the master vm is deployed which takes time
ssh -L 2375:localhost:2375 -f -N azureuser@mycatsndogsmgmt.westus.cloudapp.azure.com -p 2200 -i ~/.ssh/id_rsa
export DOCKER_HOST=:2375

# Wait until this shows two nodes
docker info

git clone https://github.com/chrmarti/example-voting-app.git
cd example-voting-app

docker-compose up -d
# Should show five containers distributed to the two nodes
docker ps
```

Open vote app: [http://mycatsndogsagents.westus.cloudapp.azure.com](http://mycatsndogsagents.westus.cloudapp.azure.com)

Open result app: [http://mycatsndogsagents.westus.cloudapp.azure.com:8080](http://mycatsndogsagents.westus.cloudapp.azure.com:8080)

```
# Work around 'build' only running on one node each time
docker-compose build vote

docker-compose scale vote=2
# Should show an additional 'vote' container on the other node
docker ps
```

Reload the browser tab with the vote app a few times, observe at the bottom on the page that the container id changes, depending on which node the load balancer directs the request to.

Other notes:
```
# Scale agent VMs
azure vmss scale -C 5 mycatsndogs swarm-agent-ACC186E3-vmss

# Tunnel to agent (IPs can be found in virtual network in the Azure Portal)
ssh -L 2375:10.0.0.5:2375 -f -N azureuser@mycatsndogsmgmt.westus.cloudapp.azure.com -p 2200 -i ~/.ssh/id_rsa

# SSH into master
ssh azureuser@mycatsndogsmgmt.westus.cloudapp.azure.com -p 2200

# SSH into agent
ssh -A -t azureuser@mycatsndogsmgmt.westus.cloudapp.azure.com -p 2200 ssh azureuser@10.0.0.5
```
