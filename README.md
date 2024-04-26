# Local K8s cluster with container registry and custom DNS

## Requirements

```shell
brew install dnsmasq kind kubectl docker 
```

## Get local DNS working for *.hack tld

```shell
export LOCAL_CUSTOM_TLD="hack"
    
# register .hack TLD locally
echo "address=/.${LOCAL_CUSTOM_TLD}/127.0.0.1" >> $(brew --prefix)/etc/dnsmasq.conf
    
# configure the local resolver
sudo mkdir /etc/resolver/
cat <<EOF | sudo tee /etc/resolver/${LOCAL_CUSTOM_TLD}
nameserver 127.0.0.1
EOF
    
# restart local dnsmasq service
sudo brew services restart dnsmasq
    
# restart mDNSResponder
sudo killall -HUP mDNSResponder
    
# verify the new resolver was picked up
scutil --dns
```

## Configure local cluster using KinD

Six node cluster with 4 workers, 2 control plans in HA and a container registry

1. Create the cluster from the cluster definition file, with correct network configuration

```shell
kind create cluster --config cluster.yaml --name k8s-playground 
```

2. Run the container registry patch shell script

This is a modified script from:  https://kind.sigs.k8s.io/docs/user/local-registry/ but instead of creating a new cluster, patches the existing nodes with the registry configuration settings

```shell
./kind-registry.sh
```

3. Wait for the registry to be ready, can check the status using `docker ps`

4. (Optional) Add the following zsh alias

```shell
alias kube-local="kubectl --context kind-k8s-playground"
```

5. Once ready, deploy the nginx ingress controller

```shell
kube-local apply -f  apache-ingress.yaml
```

6. Deploy the sample services and their ingress

```shell
kube-local apply -f services.yaml \
        -f ingress.yaml
```

7. Test the services by navigating to `foo.hack` and `bar.hack` in your browser

## Using The Registry

1. Pull an image

```shell
docker pull gcr.io/google-samples/hello-app:1.0
```

2. Tag the image to use the local registry

```shell
docker tag gcr.io/google-samples/hello-app:1.0 localhost:5001/hello-app:1.0
```

3. Push it to the registry

```shell
docker push localhost:5001/hello-app:1.0
```

4. Deploy the service from the local registry using a deployment

```shell
kube-local apply -f deployment.yaml \
        -f deployment-ingress.yaml
```