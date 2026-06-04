# Implementing a RAG using Juju and Charms

The repository provides the scripts, terraform modules and notebooks to implement a simple RAG system using KServe and OpenSearch. The repository has been tested on testflinger machines to leverage on CPU acceleration. 

## Userguide

### Environment Setup

Start up a testflinger machine

```bash
testflinger submit --poll testflinger.gpu.yaml
```

Note that you need to be on VPN and have installed the `testflinger` snap to be able to spin up a testflinger machine. When choosing the `job_queue`, please consult the [list of machines with an NVIDIA GPU](https://testflinger.canonical.com/queues/nvidia-gpu).

Once the machine is up and running, first start by enabling the CUDA drivers

```bash
source ./setup/setup-gpu.sh
```

Once the drivers are installed, we recommend to reboot the system to make it functional.

After rebooting, you can verify that the GPUs are being recognized by using

```bash
nvidia-smi
```

### Setup K8s and Juju

The setup of K8s and Juju can be done through concierge. The description of the environment is provided in the `setup/concierge.yaml` file.

You can just run the `./setup/setup.sh` file

```bash
source ./setup/setup.sh`
```

that will take care of first installing concierge and terraform to then use `concierge prepare` to setup K8s and Juju.

The script will also take care of installing heml.

Once the installation has completed, make sure that a K8s cluster is correctly running, and that a Juju controller has been spawned in the K8s cluster, e.g.

```bash
kubectl get pod -n controller-concierge-k8s
```

Finally, you can also install the GPU operator to make sure that the K8s and the GPU hardware correctly binds one another

```bash
source ./setup/setup-gpu-operator.sh
```

### Install Kubeflow

In order to install kubeflow, you can use the terraform module that is provided as part of the repository.

First make sure you switch to the K8s Juju controller

```bash
juju switch concierge-k8s:controller
```

And then plan and apply the terraform module

```bash
cd terraform

terraform plan
```

review the resources that will be created before applying

```bash
terraform apply -auto-approve
```

The terraform module comes with a set of variable. Feel free to use them to customize the deployment in evnrionment behind a proxy or to pin specific risks or channels.

After the deployment is completed, make sure that all charms are in `active/idle` state. You can also try to access the dashboard. To do so, first configure the username and password using

```bash
juju config dex-auth static-username=<username>
juju config dex-auth static-password=<password>
```

and then try to login to the dashboard, which should be accessible via the IP provided by

```bash
kubectl get svc -n kubeflow istio-ingressgateway-workload -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### Install OpenSearch

First, make sure that you switch to the VM controller

```bash 
juju switch concierge-lxd:controller
```

After this, you can trigger the setup of opensearch by running the bash script

```bash
source ./setup/setup-opensearch.sh
```

After the deployment is completed, make sure that all charms are in `active/idle` state.

### Integration between Kubeflow and OpenSearch

In order to integrate OpenSearch and Kubeflow, we need to relate one application to another.

First, go to the VM OpenSearch model and expose the OpenSearch offers

```
juju offer opensearch:opensearch-client
```

After this, go to the Kubeflow models and make sure that the offer is correctly exposed:

```
juju switch concierge-k8s:kubeflow

juju find-offers concierge-lxd:
```

You should be able to see something like:

```
Store                 URL                                        Access  Interfaces
conciernge-k8s        admin/opensearch.opensearch-client         admin   opensearch_client:opensearch-client
...
```

You can now consume the offer

```bash 
juju consume concierge-k8s:admin/opensearch.opensearch-client
```

Verify that a new SAAS offer is listed in the juju model:

```bash
juju status
```

At this point, you can relate the `data-kubeflow-integrator` with `opensearch`

```
juju relate general opensearch
```

And after the charms go in `active/idle` state, you can then verify that OpenSearch credentials are correctly injected into the user namespace

```
kubectl get secrets -n <profile-name>
```

that should show opensearch K8s resources, e.g.

```
NAME                         TYPE     DATA  AGE
...
opensearch-secret            Opaque   1     15s
opensearch-tls-secret        Opaque   1     15s
...
```

### Use the platform

You can now use the platform to deploy LLMs and implement a RAG system, leveraging on the LLM deployment.

#### LLM Deployment

To deploy LLMs using KServe, you only need to specify the `InferenceService` K8s resource, that declerative describe the inference services by providing informations such as:
* Model to be served
* Image to be used for deploying the model (e.g. huggingfaceserver or vllm)
* Resources constraints for the pods holding the models
* Extra arguments to be passed to the container entrypoint

In the folder `models/` we provide some examples for the `InferenceService` YAML to be used for a few famous/standard models.

To download some models, you will need to authenticate to HuggingFace, and to do so, you will need to load your credentials/token when deploying the model. To do so, you need to configure the ClusterStorageContainer resource to feed the token when donwloading the model and initializing the inference service pod. We provide an example for its definition under `setup/clusterstoragecontainer.yaml`.

Check that the model is correctly up and running

```
kubectl get isvc -n <profile-name>
```

Note that downloading and starting up the model may take a few minutes.

Once the model is up and running, you can first fetch the IP of the K8s service

```
kubectl get svc -n <profile-name>
```

And fetch the IP that is provided. You can also route the service outside using either NodePort or LoadBalancing services or binding the service with an ingress. Since for the current setup Canonical K8s exposes internal IP on the local machine where it is running, this is generally not required if you stick to the same setup we adopt here.

You can verify that the service is correctly responding to requests by making HTTP calls to the model. In the `notebooks/ExploreLLMs.ipynb` we provide some sample code to query both a language model (e.g. BERT) as well as a conversational generative model (e.g. Qwen 2.5). 

You can also verify that the model has been correctly loaded in the GPU, by using `nvidia-smi` command, and checking that the RAM memory of the device is indeed being used. 

#### Create a RAG

Once that you have deployed a LLM, you can embed this into an RAG system using OpenSearch. To do so, you can start a Jupyter Python notebook in the Kubeflow dashboard.

The notebook should already provide all the code snipped and steps required to deploy the RAG system. The only important custom information to be plugged in the notebook will be the IP for the LLM inference service deployed using KServe.

