# Install and login to the `az` CLI

```
# assuming you have az cli installed on the newly created dev machine
#if not, then run the following:
# sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev
# curl -L https://aka.ms/InstallAzureCli | bash 
```

When `az login` does not detect a local web browser to open, it will use device login which invokes
the full 2FA flow and only lasts a few hours. Logging in with OAuth authorization codes lasts much
longer but requires a web browser. VSCode's Remote SSH plugin provides a wrapper to open the local
browser from its built-in terminal:

1. Connect to the VM from VSCode.

2. From VSCode's built-in terminal, run `az login`.

3. After authorizing, you will be redirected to a `localhost` URL that will fail
   to connect. Use VSCode to forward the port in the URL then refresh the page.
   You should now be logged in.


# Bootsrap an acs-engine Kubernetes Cluster Using Your Hyperkube Image

1. Get acs-engine and build it

```
current_dir="$(pwd)"
mkdir -p ~/go/src/github.com/Azure
cd ~/go/src/github.com/Azure

git clone https://github.com/Azure/acs-engine.git
cd acs-engine
make

cd $current_dir
```


2. Create ARM Templates
```
# output directory
mkdir -p funkycluster/out

#cluster keys
ssh-keygen -t rsa -b 2048 -C "funkydev@funky.com" -f ./funkycluster/cluster

# Copy the source file
cp ~/go/src/github.com/Azure/acs-engine/examples/disks-managed/kubernetes-vmas.json ./funkycluster/in.json

# ****** modify ./funckycluster/in.json with values needed -- You use MSI and avoid putting clientid/client secret in api-model file

# Generate the ARM template 
./go/src/github.com/Azure/acs-engine/bin/acs-engine generate --api-model ./funkycluster/in.json --output-directory ./funkycluster/out

# Modify the template to use your hyperkube image
sed -i "s/gcrio\.azureedge\.net\/google_containers\/hyperkube-amd64:v1\.6\.6/<<REGISTRY>>\/hyperkube-amd64:<<VERSION>>/g" ./funkycluster/out/azuredeploy.parameters.json


```

3. Deploy ARM Template 

```
# Create the group  
az group create --name funkycluster --location centralus

# Deploy ARM template

az group deployment create --name one --resource-group funkycluster --template-file ./funkycluster/out/azuredeploy.json --parameters @./funkycluster/out/azuredeploy.parameters.json

# the newly created cluster now runs using your custom image. 
```


# Working with kubectl that matches your image

It is always a good idea to use kubectl that matches your image. Run the following to sync versions

> This was needed on older ACS engine clusters. Currently a systemd unit at /etc/systemd/system/kubelet-extract.service, performs pretty much the below

```
ssh -i ./funkycluster/cluster azureuser@funkyk8scluster.centralus.cloudapp.azure.com

mkdir _kubectl

sudo docker run -v /tmp:/mnt/h khenidak/hyperkube-amd64:test001 bash -c "cp /hyperkube /mnt/h"
cp /tmp/hyperkube ~/_kubectl/hyperkube

echo 'alias kubectl="~/_kubectl/hyperkube kubectl"' >> ~/.bash_aliases
echo "source ~/.bash_aliases" >> ~/.bashrc
source .bashrc
#confirm
type kubectl # should point to the alias
kubectl get no
```


# Replace hyperkube image on running cluster. 

*This is **brute-force** approach but clean and leaves no space for errors*

1. Create a new build

```
cd ~/go/src/k8s.io/kubernetes
export REGISTRY=<<YOUR DOCKER HUB USER>>
export VERSION=<<NEW VERSION>>
```

2. Copy the key the cluster ssh key to master 

```
scp -i ./funkycluster/cluster ./funkycluster/cluster azureuser@funkyk8scluster.centralus.cloudapp.azure.com:~/clusterkey
```

3. Configure Master

```
# Connect
ssh -i ./funkycluster/cluster azureuser@funkyk8scluster.centralus.cloudapp.azure.com

# Update 
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y jq pssh

# Get nodes 
cat << EOF >>  ~/.bashrc
get_cluster_nodes()
{
  # negate https://github.com/Azure/acs-engine/issues/1083 by redirect stderr duplicate proto reg log lines
  # we want the nodes only
  kubectl get node 2> /dev/null | sed '1d' | sed  '/master/d' | awk '{print $1}'
}
EOF

cat << EOF >>  ~/.bashrc
allnodes()
{
  get_cluster_nodes > /tmp/nodes
  parallel-ssh --inline  --hosts /tmp/nodes --user $(whoami) -x '-i ~/clusterkey -o StrictHostKeyChecking=no'
}
EOF
```

4. Swap hyperkube images on nodes

```
#on master: 
# Now every time you need to change image on the cluster
allnodes "sudo sed -i 's/KUBELET_IMAGE=.*\KUBELET_IMAGE=<<YOUR DOCKER HUB USER>>/hyperkube-amd64:<<YOUR TAG>>/g' /etc/default/kubelet" # this replaces kubelet image on all nodes
allnodes "sudo systemctl stop kubelet.service && sleep 3 && sudo systemtl start kubelet.service" # restart kubelet, again systemctl restart does not seem to be reliable.
```

5. Swap hyperkube image on master

```
# At minimum you will need to swap for api-server and controller manager
# on master

# Kubelet keeps a file watch on this file and will automatically reset api-server container to the new image 
sudo sed -i 's/<<DOCKER HUB USER>>\/hyperkube-amd64:<<CURRENT TAG>>/<<DOCKER HUB USER>>\/hyperkube-amd64:<<TAG>>/g' /etc/kubernetes/manifests/kube-apiserver.yaml

# Also kubelet keeps a file watch on this file.
sudo sed -i 's/<<DOCKER HUB USER>>\/hyperkube-amd64:<<CURRENT TAG>>/<<DOCKER HUB USER>>\/hyperkube-amd64:<<TAG>>/g' /etc/kubernetes/manifests/kube-controller-manager.yaml

# You generally don't need to update kubelet image since on master its mainly used to start other control plan components. but should you do

# Replace image in systemd uni'ts env file 
sudo sed -i 's/KUBELET_IMAGE=.*/KUBELET_IMAGE=<<DOCKER HUB USER>>\/hyperkube-amd64:<<TAG>>/g' /etc/default/kubelet

# Restart kubelet on master
sudo systemctl stop kubelet.service && sleep 3 && sudo systemctl start kubelet.service
```

