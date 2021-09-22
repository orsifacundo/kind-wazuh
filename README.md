`kind` is a suite of tools for running local Kubernetes clusters using Docker container “nodes”. It was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

`Wazuh` is a free, open source and enterprise-ready security monitoring solution for threat detection, integrity monitoring, incident response and compliance.

`MetalLB` provides a network load-balancer implementation. In short, it allows you to create Kubernetes services of type LoadBalancer in clusters that don’t run on a cloud provider, and thus cannot simply hook into paid products to provide load balancers.

`nginx` is a free, open-source, high-performance HTTP server, load-balancer and reverse proxy. It's known for its high performance, stability, rich feature set, simple configuration, and low resource consumption.

## Objective

The idea of this project is to create a `kind` cluster (with 1 control-plane node and 2 worker nodes) where we are going to deploy a `Wazuh` cluster with 4 Wazuh nodes (1 master node and 3 worker nodes) + 1 Elasticsearch node + 1 Kibana node.

In order to have external access to the LoadBalancer services of the cluster then we are going to use MetalLB (network load balancer implementation that integrates with standard network equipment). 

Finally, to make the Wazuh deployment accesible from the LAN, a `nginx` service can be installed on the host.

Each of the folders (will) have a README section with instructions on how to install the respective packages/services.

For `Wazuh` and `nginx` self-signed certificates are provided for testing purposes and they should be replaced.

## Prerequisites

- **go (1.11+)**: https://golang.org/doc/install (Tested with `go version go1.17 linux/amd64`): **kind** is written in GoLang.

- **docker**: https://docs.docker.com/engine/install/ (Tested on: `Docker version 20.10.7`): this is where the K8s cluster is going to be deployed.

- **kubectl**: https://kubernetes.io/docs/tasks/tools/ (Tested with: `v1.22.1`): to run commands against the K8s cluster.

- **kind**: https://kind.sigs.k8s.io/docs/user/quick-start/ (Tested with: `v0.11.1`): for running local Kubernetes clusters using Docker container “nodes”

## Steps

### Create `kind` cluster:

- Create the `kind` cluster:

  `kind create cluster --name wazuh --config kind/kind-config.yaml`

- Check that the nodes are ready:

  `kubectl get nodes --context kind-wazuh`

### Deploy `Wazuh`:

- **OPTIONAL**: by default Wazuh cluster certificates are provided on this repo but in case that you want to regenerate them:

  `chmod +x wazuh-kubernetes/wazuh/certs/odfe_cluster/generate_certs.sh && wazuh-kubernetes/wazuh/certs/odfe_cluster/generate_certs.sh`

  `chmod +x wazuh-kubernetes/wazuh/certs/kibana_http/generate_certs.sh && wazuh-kubernetes/wazuh/certs/kibana_http/generate_certs.sh`

- Apply Wazuh manifests:

  `kubectl apply -k wazuh-kubernetes/envs/local-env/`

- Check that the pods are fully started (STATUS=Running):

  `kubectl get pods --namespace wazuh`

### Setup `MetalLB`:

- Create the `metallb` namespace:

  `kubectl apply -f metalLB/namespace.yaml`

- Create the memberlist secrets (encrypt the communication between speakers):

  `kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`

- Apply `metallb` manifest:

  `kubectl apply -f metalLB/metallb.yaml`

- Check that the pods are fully started (STATUS=Running):

  `kubectl get pods -n metallb-system`

- Obtain the `docker` kind network address space in order to let MetalLB assign External IPs to the LoadBalancers based on this range:

  `docker network inspect -f '{{.IPAM.Config}}' kind`

- Based on this result modify the ConfigMap object located on `metalLB/metallb-configmap.yaml` replacing the range on the `addresses:` parameter.

- Apply the ConfigMap manifest:

  `kubectl apply -f metalLB/metallb-configmap.yaml`

### Setup `nginx`:

- Get the EXTERNAL-IP (assigned by MetalLB) of each of the K8s LoadBalancer Services:

  `kubectl get services -o wide -n wazuh`

- Modify the `nginx` configuration file (`nginx/nginx.conf`), replacing the obatined EXTERNAL-IP with the proper service:

  **<EXTERNAL-IP_KIBANA_SERVICE>** = `kibana` LB service EXTERNAL-IP

  **<EXTERNAL-IP_WAZUH-WORKERS_SERVICE>** = `wazuh-workers` LB service EXTERNAL-IP

  **<EXTERNAL-IP_WAZUH_SERVICE>** = `wazuh` LB service EXTERNAL-IP

- Copy the `nginx/nginx.conf` file and certificates into the appropriate path:

 `mkdir /etc/nginx/certs && cp nginx/nginx.conf /etc/nginx/nginx.conf && cp nginx/certs/nginx.* /etc/nginx/certs/`

- Start the `nginx` service:

  `systecmtl start nginx`

- That's all, you should be able to:

  - Access Kibana on `https://<HOST_IP>`

  - Register Wazuh agents on: <HOST_IP> port `1515`

  - Wazuh server/agent communication: <HOST_IP> port `1514`
  
## References

- Base documentation: https://kind.sigs.k8s.io/

- Design: https://kind.sigs.k8s.io/docs/design/initial/

- Wazuh deployment over Kubernets (local environment): https://documentation.wazuh.com/current/deploying-with-kubernetes/kubernetes-local-env.html

- MetalLB: https://metallb.universe.tf/

- Kubectl cheatsheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

## ToDo

- Troubleshooting & Common errors section.
- Installation instructions for each of the services (on the README md of each folder)
- Step-by-step on how to regenerate certificates.
