Kubernetes CRD for yadage workflows

[![Build Status](https://travis-ci.com/yadage/yadage-crd.svg?branch=master)](https://travis-ci.com/yadage/yadage-crd)

# Instructions to build RECAST cluster at CERN

## 1. Create openstack kubernetes cluster

Follow the instructions in the [Container Quickstart Guide](https://clouddocs.web.cern.ch/containers/quickstart.html) to create a kubernetes cluster at CERN. Specifically:

1. If you don't yet have one, follow [these instructions](https://clouddocs.web.cern.ch/tutorial_using_a_browser/subscribe_to_the_cloud_service.html) to request a personal CERN openstack project. Once it's created, you should be able to access your openstack project at https://openstack.cern.ch/project/. Follow [these instructions](https://clouddocs.web.cern.ch/tutorial/create_your_openstack_profile.html#create-your-openstack-keypair) to add your lxplus public key as an openstack keypair, which will allow you to ssh onto openstack nodes in your project from lxplus.

2. Follow [these instructions](https://clouddocs.web.cern.ch/using_openstack/environment_options.html#download-from-the-dashboard) to download the openstack RC file for your openstack project. Then, copy the file onto lxplus: `scp my-openstack-rc.sh damacdon@lxplus.cern.ch:` (note: you should replace `my-openstack-rc.sh` with the actual name of your openstack rc file).

3. Log onto lxplus-cloud, and source your openstack RC file to get access to your personal openstack project via the openstack API:

```bash
ssh damacdon@lxplus.cern.ch
source my-openstack-rc.sh
```

4. Create a kubernetes cluster on your openstack project following the [quickstart instructions](https://clouddocs.web.cern.ch/containers/quickstart.html):

```bash
openstack coe cluster create recast-cluster --keypair lxplus --cluster-template kubernetes-1.18.6-3 --node-count 2

# ...Wait for cluster creaton to complete...

# Get the kube config file to access the k8s cluster
$(openstack coe cluster config mykubcluster)
```

## 2. Clone yadage-crd and build `crdctrl` image

Log onto the master node of your cluster using the IP address shown on openstack at https://openstack.cern.ch/project/instances/ (you can also get the master node IP directly with `openstack coe cluster show recast-cluster | grep master_addresses`). Clone this git repo

```bash
ssh core@188.185.86.174
git clone --recursive https://github.com/yadage/yadage-crd.git
cd yadage-crd
```

Build the `crdctrl` image and push it to your personal docker hub:

```bash
sudo docker login
sudo docker build -t danikam/crdctrl:latest -f images/crdctrl/Dockerfile images/crdctrl
sudo docker push danikam/crdctrl:latest
```

## 3. Set up yadage

```bash
# Install `kubectl` and helm
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | sudo bash

# Set up clusterrolebinding
kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts
```
Modify the `private_token` in `helm/yadage/values.yaml` to a private gitlab token ([instructions for creating gitlab tokens](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html#creating-a-personal-access-token)) for the account you want to use to access images from the gitlab container registry and input files from private gitlab projects. You can also modify the `filebrowser_user` and `filebrowser_password` fields if you want to set passwords for accessing workflow outputs via http using `wget`.

Example (mods made to `helm/yadage/values.yaml`
`private_token: ''` --> `private_token: 'somePhonyGitlabToken1234'`
`filebrowser_user: ''` --> `filebrowser_user: 'recast'`
`filebrowser_password: ''` --> `filebrowser_password: 'recast'`

Now you can finish setting up yadage

```bash
# Install helm releases for yadage and metacontroller
helm init --service-account default --wait --upgrade
kubectl create namespace yadage
cd helm
helm install metacontroller
helm install yadage --set crdctrl_image=danikam/crdctrl:latest  # replace danikam/crdctrl:latest with the image you pushed to docker hub
cd ..

# Check that the controller pod is running
kubectl get pods -n yadage

# Create a PVC for CERN storage
kubectl create -f crd/pvc_cern.yml 
```

## 4. Add secrets

Secrets need to be added to provide the recast client authentication credentials to pull images from private gitlab registries and download files from `eos` storage.

Edit the `data: [...]` field in `crd/hepauth_secret.yml` with a base64-encoded kinit initialization of the form `echo 'BibuSabi4'|kinit recast@CERN.CH`. To do so, replace `BibuSabi4` and `recast` with the password and username, respectively, of the CERN account you want to use for access, and base-64 encode the whole thing as follows:

```bash
printf "echo 'phonypassword'|kinit recast@CERN.CH"|base64
```
which will output 

```
ZWNobyAncGhvbnlwYXNzd29yZCd8a2luaXQgcmVjYXN0QGNlcm4uY2g=
```

so the `data: [...]` line would become `data: {"getkrb.sh": "ZWNobyAncGhvbnlwYXNzd29yZCd8a2luaXQgcmVjYXN0QGNlcm4uY2g="}`.

Now, create the `hepauth` and `hepimgcred` secrets for access to `eos` storage and gitlab registry images, respectively:

```bash
kubectl create -f crd/hepauth_secret.yml
kubectl create secret docker-registry hepimgcred --docker-server=gitlab-registry.cern.ch  --docker-username=recast --docker-password=phonypassword --docker-email='none'
```

where `recast` in the second line is replaced by the name of the CERN account you want to use for access to images from the gitlab registry and `phonypassword` is the **personal access token** used earlier for the `private_token` field in `helm/yadage/values.yaml`.

## 5. Test sample workflow

Test out some of the sample workflows located in the `examples` directory, eg.

```bash
kubectl create -f examples/wflow_mg.yml
```


