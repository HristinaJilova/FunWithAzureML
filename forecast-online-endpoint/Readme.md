# Forecasting with online-endpoint

[Optional] If you face issues using the [AzureML-AutoML image](https://ml.azure.com/environments/AzureML-AutoML/version/115) in your own environment (e.g. behind vnet) you can clone it.

1. Run `step010_clone_automl_environment.py`.
1. In your custom environments you should have a new environment registered, named `clonedautoml`. If you don't build it, you may get the following error `Message: ResourceNotFound: Deployment failed due to timeout while waiting for Environment Image to become available.` while trying to deploy, in which case you will have to retry.

[Optionally] Build a new machine learning model, to replace the ones in this repo:

1. Use `step020_generate_dataset.py` to generate your own dataset.
1. Run an automl forecasting experiment
1. Download the `model.pkl` file from the trained model's `output` folder and place it in the `model` folder.
1. Download the `scoring_file_v_2_0_0.py` from the trained model's `output` folder and place it in the `score` folder.

Deploy the model:

1. Deploy an endpoint using `az ml online-endpoint create -f endpoint.yml`
   > Uncomment the `public_network_access: disabled` line if you are behind a vnet.
1. Modify the `deployment.yml` to update:
   1. `your-deployment-name` to whatever you want to name your endpoint.
   1. `your-endpoint-name` to match an endpoint that you deployed above
   > If you are behind a vnet:
   >
   > - Uncomment the `egress_public_network_access: disabled` line.
   > - Use the cloned environment `environment: azureml:clonedautoml:1`.
   >
1. Deploy the endpoint locally to test:

   ```bash
   az ml online-deployment create --local -f deployment.yml
   ```

   If this fails you can get logs:

   ```bash
   az ml online-deployment get-logs --local -n your-deployment-name -e your-endpoint-name
   ```

   If you want to run the docker image locally to inspect you can do:

   ```bash
   docker image list | grep your-deployment-name
   docker run -it <IMAGE_ID_FROM_PREVIOUS_CMD> /bin/bash
   ```

   Or you can commit the failed instance and inspect that (which will contain the environment variables):

   ```bash
   docker ps -a | grep your-deployment-name
   docker commit <INSTANCE_ID_FROM_PREVIOUS_CMD> failed:v1
   docker run -it failed:v1 /bin/bash
   ```

1. Deploy the endpoint to a managed online one:

   ```bash
   az ml online-deployment create -f deployment.yml
   ```

   > Note: If you are behind a vnet you need to uncomment the following line in your deployment.yml file:
   >
   > `egress_public_network_access: disabled`
   >
   > See [the azureml examples](https://github.com/Azure/azureml-examples/tree/main/cli/endpoints/online/managed/vnet/sample) on the settings you need to specify for the endpoint and the deployment.

1. Test the endpoint with the following command:

   ```bash
   curl  -H "Content-Type: application/json" -d '{"Inputs": {"data": [{"DateCreated": "2022-07-07T00:00:00.000Z"}]},"GlobalParameters": {"quantiles": [0.025, 0.975]}}' -H "Authorization: Bearer KEY_FROM_ENDPOINT_PORTAL" -H "azureml-model-deployment: your-deployment-name" https://your-endpoint-name.region.inference.ml.azure.com/score
   ```

1. If you want to delete the deployment:

   ```bash
   az ml online-deployment delete -e you-endpoint-name -n your-endpoint-deployment-name
   ```

## Building custom docker images

You can build your own docker images and push them to your container registry using these steps:

1. In a command line navigate in a folder that contains a `Dockerfile`.
1. Build the docker image where you should replace `your-acr-name` with your Azure Container Registry's name (ACR):

   ```bash
   docker build . -t your-acr-name.azurecr.io/repo/customimagename:v1
   ```

1. Login to your ACR with the following command:

   ```bash
   docker login your-acr-name.azurecr.io
   ```

   You will find the username (it's `your-acr-name`) and the password in the `Access keys` section of your ACR.

1. Push the build image to your ACR:

   ```bash
   docker push your-acr-name.azurecr.io/repo/customimagename:v1
   ```

## Deploying with network isolation

Here is a video describing the process of deploying an AutomML generated forecasting model in an managed online endpoint behind a vnet:

[![Deploy AutoML in Managed Online Endpoint](https://img.youtube.com/vi/k8zn0OE2pvw/0.jpg)](https://youtu.be/k8zn0OE2pvw)

**Tldr**: 
1. Install the latest version of the azureml cli and the ml extension (allow `aka.ms` through firewall) in compute instance.
   ```bash
   az upgrade
   az extension add -n ml --upgrade
   ```
2. Create a clone of [AzureML-AutoML environment](https://ml.azure.com/environments/AzureML-AutoML/version/115).
   ```bash
   python step010_clone_automl_environment.py
   ```
3. Uncomment the lines in the `endpoint.yml` and `deployment.yml`, commenting out the overlapping ones.
4. Deploy using `az ml` commands.

## References

- [Deploy managed online-endpoint with network isolation](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-secure-online-endpoint?tabs=model)
- [Update Azure CLI](https://docs.microsoft.com/en-us/cli/azure/update-azure-cli)
- [Install and update AzureML CLI extension](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-configure-cli?tabs=public)
- [Examples for managed online endpoint deployments](https://github.com/Azure/azureml-examples/tree/main/cli/endpoints/online)
