---
title: High-performance model serving with Triton (preview)
titleSuffix: Azure Machine Learning
description: 'Learn to deploy your model with NVIDIA Triton Inference Server in Azure Machine Learning.'
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.date: 05/17/2021
ms.topic: how-to
ms.reviewer: larryfr
ms.custom: deploy, devx-track-azurecli
---

# High-performance serving with Triton Inference Server (Preview) 

Learn how to use [NVIDIA Triton Inference Server](https://aka.ms/nvidia-triton-docs) to improve the performance of the web service used for model inference.

One of the ways to deploy a model for inference is as a web service. For example, a deployment to Azure Kubernetes Service or Azure Container Instances. By default, Azure Machine Learning uses a single-threaded, *general purpose* web framework for web service deployments.

Triton is a framework that is *optimized for inference*. It provides better utilization of GPUs and more cost-effective inference. On the server-side, it batches incoming requests and submits these batches for inference. Batching better utilizes GPU resources, and is a key part of Triton's performance.

> [!IMPORTANT]
> Using Triton for deployment from Azure Machine Learning is currently in __preview__. Preview functionality may not be covered by customer support. For more information, see the [Supplemental terms of use for Microsoft Azure previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/)

> [!TIP]
> The code snippets in this document are for illustrative purposes and may not show a complete solution. For working example code, see the [end-to-end samples of Triton in Azure Machine Learning](https://aka.ms/triton-aml-sample).

> [!NOTE]
> [NVIDIA Triton Inference Server](https://aka.ms/nvidia-triton-docs) is an open-source third-party software that is integrated in Azure Machine Learning.

## Prerequisites

* An **Azure subscription**. If you do not have one, try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/).
* Familiarity with [how and where to deploy a model](how-to-deploy-and-where.md) with Azure Machine Learning.
* The [Azure Machine Learning SDK for Python](/python/api/overview/azure/ml/) **or** the [Azure CLI](/cli/azure/) and [machine learning extension](reference-azure-machine-learning-cli.md).
* A working installation of Docker for local testing. For information on installing and validating Docker, see [Orientation and setup](https://docs.docker.com/get-started/) in the docker documentation.

## Architectural overview

Before attempting to use Triton for your own model, it's important to understand how it works with Azure Machine Learning and how it compares to a default deployment.

**Default deployment without Triton**

* Multiple [Gunicorn](https://gunicorn.org/) workers are started to concurrently handle incoming requests.
* These workers handle pre-processing, calling the model, and post-processing. 
* Clients use the __Azure ML scoring URI__. For example, `https://myservice.azureml.net/score`.

:::image type="content" source="./media/how-to-deploy-with-triton/normal-deploy.png" alt-text="Normal, non-triton, deployment architecture diagram":::

**Deploying with Triton directly**

* Requests go directly to the Triton server.
* Triton processes requests in batches to maximize GPU utilization.
* The client uses the __Triton URI__ to make requests. For example, `https://myservice.azureml.net/v2/models/${MODEL_NAME}/versions/${MODEL_VERSION}/infer`.

:::image type="content" source="./media/how-to-deploy-with-triton/triton-deploy.png" alt-text="Inferenceconfig deployment with Triton only, and no Python middleware":::


## Deploying Triton without Python pre- and post-processing

First, follow the steps below to verify that the Triton Inference Server can serve your model.

### (Optional) Define a model config file

The model configuration file tells Triton how many inputs to expects and of what dimensions those inputs will be. For more information on creating the configuration file, see [Model configuration](https://aka.ms/nvidia-triton-docs) in the NVIDIA documentation.

> [!TIP]
> We use the `--strict-model-config=false` option when starting the Triton Inference Server, which means you do not need to provide a `config.pbtxt` file for ONNX or TensorFlow models.
> 
> For more information on this option, see [Generated model configuration](https://aka.ms/nvidia-triton-docs) in the NVIDIA documentation.

### Use the correct directory structure

When registering a model with Azure Machine Learning, you can register either individual files or a directory structure. To use Triton, the model registration must be for a directory structure that contains a directory named `triton`. The general structure of this directory is:

```bash
models
    - triton
        - model_1
            - model_version
                - model_file
            - config_file
        - model_2
            ...
```

> [!IMPORTANT]
> This directory structure is a Triton Model Repository and is required for your model(s) to work with Triton. For more information, see [Triton Model Repositories](https://aka.ms/nvidia-triton-docs) in the NVIDIA documentation.

### Register your Triton model

# [Azure CLI](#tab/azcli)

```azurecli-interactive
az ml model register -n my_triton_model -p models --model-framework=Multi
```

For more information on `az ml model register`, consult the [reference documentation](/cli/azure/ml/model).

When registering the model in Azure Machine Learning, the value for the `--model-path  -p` parameter must be the name of the parent folder of the Triton Model Repository.
In the example above,  `--model-path` is 'models'.

The value for `--name  -n` parameter, `my_triton_models` in the example, will be the model name known to Azure Machine Learning Workspace. 

# [Python](#tab/python)


[!notebook-python[] (~/Azureml-examples-main/python-sdk/experimental/deploy-triton/1.bidaf-ncd-local.ipynb?name=register-model)]

For more information, see the documentation for the [Model class](/python/api/azureml-core/azureml.core.model.model).

---

### Deploy your model

# [Azure CLI](#tab/azcli)

If you have a GPU-enabled Azure Kubernetes Service cluster called "aks-gpu" created through Azure Machine Learning, you can use the following command to deploy your model.

```azurecli
az ml model deploy -n triton-webservice -m triton_model:1 --dc deploymentconfig.json --compute-target aks-gpu
```

# [Python](#tab/python)

[!notebook-python[] (~/Azureml-examples-main/python-sdk/experimental/deploy-triton/1.bidaf-ncd-local.ipynb?name=deploy-webservice)]

---

See [this documentation for more details on deploying models](how-to-deploy-and-where.md).

[!INCLUDE [endpoints-option](../../includes/machine-learning-endpoints-preview-note.md)]

### Call into your deployed model

First, get your scoring URI and bearer tokens.

# [Azure CLI](#tab/azcli)

```azurecli
az ml service show --name=triton-webservice
```
# [Python](#tab/python)

[!notebook-python[] (~/Azureml-examples-main/python-sdk/experimental/deploy-triton/1.bidaf-ncd-local.ipynb?name=get-keys)]

---

Then, ensure your service is running by doing: 

# [Azure CLI](#tab/azcli)

```azurecli
```{bash}
!curl -v $scoring_uri/v2/health/ready -H 'Authorization: Bearer '"$service_key"''
```

This command returns information similar to the following. Note the `200 OK`; this status means the web server is running.

```{bash}
*   Trying 127.0.0.1:8000...
* Connected to localhost (127.0.0.1) port 8000 (#0)
> GET /v2/health/ready HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.71.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
```
# [Python](#tab/python)

[!notebook-python[] (~/Azureml-examples-main/python-sdk/experimental/deploy-triton/1.bidaf-ncd-local.ipynb?name=query-service)]

---


Once you've performed a health check, you can create a client to send data to Triton for inference. For more information on creating a client, see the [client examples](https://aka.ms/nvidia-client-examples) in the NVIDIA documentation. There are also [Python samples at the Triton GitHub](https://aka.ms/nvidia-triton-docs).

## Clean up resources

If you plan on continuing to use the Azure Machine Learning workspace, but want to get rid of the deployed service, use one of the following options:


# [Azure CLI](#tab/azcli)

```azurecli
az ml service delete -n triton-densenet-onnx
```
# [Python](#tab/python)

[!notebook-python[] (~/Azureml-examples-main/python-sdk/experimental/deploy-triton/1.bidaf-ncd-local.ipynb?name=delete-service)]

---

## How to use Azure Machine Learning Triton Inference Server container image

Learn how to use Azure Machine Learning Triton Inference Server container image with new [CLI(v2)](/cli/azure/ml?view=azure-cli-latest&preserve-view=true). The examples below use [online endpoint and deployments](concept-endpoints.md#what-are-online-endpoints-preview) concept. 

1. [Deploy single Triton model](https://github.com/Azure/azureml-examples/blob/main/cli/deploy-triton-managed-online-endpoint.sh).
1. [Deploy Triton multiple models](https://github.com/Azure/azureml-examples/blob/main/cli/deploy-triton-multiple-models-online-endpoint.sh).
1. [Deploy Triton ensemble model](https://github.com/Azure/azureml-examples/blob/main/cli/deploy-triton-ensemble-managed-online-endpoint.sh).
1. Check out [Triton examples](https://github.com/Azure/azureml-examples/tree/main/cli/endpoints/online/triton).

## Troubleshoot

* [Troubleshoot a failed deployment](how-to-troubleshoot-deployment.md), learn how to troubleshoot and solve, or work around, common errors you may encounter when deploying a model.

* If deployment logs show that **TritonServer failed to start**, please refer to [Nvidia's open source documentation.](https://github.com/triton-inference-server/server)

## Next steps

* [See end-to-end samples of Triton in Azure Machine Learning](https://aka.ms/aml-triton-sample)
* Check out [Triton client examples](https://aka.ms/nvidia-client-examples)
* Read the [Triton Inference Server documentation](https://aka.ms/nvidia-triton-docs)
* [Deploy to Azure Kubernetes Service](how-to-deploy-azure-kubernetes-service.md)
* [Update web service](how-to-deploy-update-web-service.md)
* [Collect data for models in production](how-to-enable-data-collection.md)
