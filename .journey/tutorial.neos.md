<!--markdownlint-disable MD024 MD033 MD036 MD041 -->
<walkthrough-metadata>
  <meta name="title" content="How to use Ollama as a sidecar with Cloud Run GPUs and Open WebUI as a frontend ingress container" />
  <meta name="description" content="Learn how to use Ollama as a sidecar with Cloud Run GPUs and Open WebUI as a frontend ingress container" />
  <meta name="keywords" content="Cloud Run, GPU, Ollama, Open WebUI, LLM, Gemma" />
</walkthrough-metadata>

# How serve Gemma 3 on Cloud Run and Serverless GPUs


## Let's get started

![Tutorial header image](https://raw.githubusercontent.com/NucleusEngineering/serverless/main/.images/run.jpg)

Cloud Run recently added GPU support. Cloud Run is a container platform on Google Cloud that makes it straightforward to run your code in a container, without requiring you to manage a cluster.

<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>

Estimated time:
<walkthrough-tutorial-duration duration="20"></walkthrough-tutorial-duration>

To get started, click **Start**.

## Introduction
Today, the GPUs we make available are Nvidia L4 GPUs with 24 GB of vRAM. There's one GPU per Cloud Run instance, and Cloud Run auto scaling still applies. That includes scaling out up to 5 instances (with quota increase available), as well as scaling down to zero instances when there are no requests.

One use case for GPUs is running your own open large language models (LLMs). This tutorial walks you through deploying a service that runs a LLM.

In this lab, you'll deploy a container service using an serverless GPU and Ollama to serve Gemma 3 1B model stored in a Google Cloud Storage bucket. That way you have your own and private model available, which we are going to test. 

### **What you'll learn**

* How to create a container service in Cloud Run
* How to deploy Ollama with serverless GPUs serving a Gemma 3 1B model
* How to deploy Open WebUI as a frontend container and connecting it to the Ollama Service. 

## Project Setup

First, make sure you have the correct project selected.

<walkthrough-project-setup billing="true"></walkthrough-project-setup>

Make sure that you use the project with the masterclass billing account.

Next, enable the required Google APIs for this tutorial.

<walkthrough-enable-apis apis="run.googleapis.com,cloudbuild.googleapis.com,storage.googleapis.com,artifactregistry.googleapis.com"></walkthrough-enable-apis>

You can set environment variables that will be used throughout this codelab.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=europe-west1
```

Create a directory for this codelab.

```bash
mkdir ollama-codelab
cd ollama-codelab
```


## Install Ollama and download Gemma 3 1B model

First, you'll install Ollama to download the model. This will download the model to /home/$USER/.ollama/models

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Now run ollama by running

```bash
ollama serve
```

Ollama starts listening on port 11434.

Open a **second terminal window** to pull down the Gemma 3 1B model to 

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=europe-west1
ollama pull gemma3:1b
```

(Optional) You can interact with Gemma from the command line by running

```bash
ollama run gemma3:1b
```

Try out some prompts like: *What is Cloud Run?*

When you are done chatting with Gemma, you can exit by typing in 

```bash
/bye
```


## Create a Storage Bucket

Now that the model has downloaded, you can move the model to your Google Cloud Storage Bucket.

First, create the bucket.

```bash
gcloud storage buckets create gs://$PROJECT_ID-gemma3-1b-codelab
```

Now, move the models folder to the Google Cloud Storage Bucket with *gsutil*, so we can use the model files later in the cloud and mount them directly into the container.

```bash
gcloud storage cp --recursive $HOME/.ollama/models gs://$PROJECT_ID-gemma3-1b-codelab
```

## Deploy a Cloud Run Ollama Service

### Navigate to Cloud Run and Create Service

1.  In the Google Cloud Console, make sure to be at the Cloud Run Overview page.
    <walkthrough-menu-navigation sectionId="SERVERLESS_SECTION"></walkthrough-menu-navigation>

2.  Click **Deploy container**.
    <walkthrough-spotlight-pointer cssSelector="a[href*='/run/create?enableapi=false&deploymentType=container']" validationPath="/run">Show me</walkthrough-spotlight-pointer>

3.  Ensure that **Deploy one revision from an existing container image** is selected.

To configure the service, click **Next**.

## Configure Service Settings

1.  In the **Container image URL** field, enter:
    ```
    ollama/ollama
    ```
    <walkthrough-spotlight-pointer locator="semantic({textbox 'Container image URL'})">Show me</walkthrough-spotlight-pointer>

2.  In the **Service name** field, enter:
    ```
    ollama-gemma3-1b
    ```
    <walkthrough-spotlight-pointer locator="semantic({textbox 'Service name'})">Show me</walkthrough-spotlight-pointer>

3.  Under **Region**, make sure to select **europe-west1**.
    <walkthrough-spotlight-pointer cssSelector="cfc-region-picker">Show me</walkthrough-spotlight-pointer>

4.  For **Authentication**, select the **Allow unauthenticated invocations** radio button.
    <walkthrough-spotlight-pointer locator="semantic({radio 'Allow public access'})">Show me</walkthrough-spotlight-pointer>

5.  For **Billing**, select **Instance-based**.
   
To configure container details, click **Next**.

## Configure Container, Networking, and Security

1.  Expand the **Container(s), volumes, networking, security** section.
    <walkthrough-spotlight-pointer locator="semantic({button 'Containers, Volumes, Networking, Security'})">Show me</walkthrough-spotlight-pointer>

2.  Under the **Container** tab, change **Container port** to `11434`.
    <walkthrough-spotlight-pointer locator="semantic({textbox 'Container port'})">Show me</walkthrough-spotlight-pointer>

3.  Under **Resources**:
    *   Set **Memory** to `32GiB`.
        <walkthrough-spotlight-pointer locator="semantic({combobox 'Memory'})">Show me Memory</walkthrough-spotlight-pointer>
    *   Set **CPU** to `8`.
        <walkthrough-spotlight-pointer locator="semantic({combobox 'CPU'})">Show me CPU</walkthrough-spotlight-pointer>
    *   Select the **GPU** checkbox.
    *   Set the **GPU type** to **NVIDIA L4**.
    *   Click **Done**.

4.  Under **GPU Redundancy**, select the **No Zonal Redundancy** option. Read more about [this option here](https://cloud.google.com/run/docs/configuring/services/gpu#zonal-redundancy)
    <walkthrough-spotlight-pointer locator="semantic({radio 'No Zonal Redundancy'})">Show me</walkthrough-spotlight-pointer>

To configure scaling, click **Next**.

## Configure Revision Scaling

1.  Under **Revision Scaling**, make sure that the **Maximum number of instances** is `3`.
    <walkthrough-spotlight-pointer locator="semantic({textbox 'Maximum number of instances'})">Show me</walkthrough-spotlight-pointer>

To add a volume, click **Next**.

## Add Cloud Storage Volume

1.  Navigate back to the **Container(s), volumes, networking, security** section and click the **Volumes** tab.
    <walkthrough-spotlight-pointer locator="semantic({tab 'Volumes'})">Show me Volumes Tab</walkthrough-spotlight-pointer>

2.  Click **Add Volume**.
    <walkthrough-spotlight-pointer locator="semantic({button 'add volume'})">Show me</walkthrough-spotlight-pointer>

3.  Select the volume type as **Cloud Storage bucket**.

4.  Set the volume name to `gcs-1`.

5.  Click **Browse**. Select the bucket name provided at the start of the lab, and click **Select**.

6.  Click **Done**.

To mount the volume, click **Next**.

## Mount Volume to Container

1.  Click the **Container(s)** tab.
    <walkthrough-spotlight-pointer locator="semantic({tab 'Containers'})">Show me Container(s) Tab</walkthrough-spotlight-pointer>

2.  Click on the container row to expand its settings.

3.  Click the **Volume Mounts** tab.
    <walkthrough-spotlight-pointer locator="semantic({tab 'Volume Mounts'})">Show me Volume Mounts Tab</walkthrough-spotlight-pointer>

4.  Click **Mount volume**. 

5.  For **Name 1**, select `gcs-1`.

6.  For **Mount path 1**, enter:
    ```
    root/.ollama
    ```

7.  Click **Done**.

To create the service, click **Next**.

## Deploy the Service

1.  Review the settings.

2.  Click **Create** to deploy the service.
    <walkthrough-spotlight-pointer cssSelector="button[type='submit']">Show me Create</walkthrough-spotlight-pointer>

Wait for the service to deploy. You can view the status on the Cloud Run service details page.

## Test your own Ollama endpoint

Get your Service URL. You will see it in the console
<walkthrough-spotlight-pointer locator="semantic({button 'Copy to clipboard:*'})">Show me the URL</walkthrough-spotlight-pointer>. But you can also use a gcloud command to fetch your public service URL.

```bash
export OLLAMA_SERVICE_URL=$(gcloud run services describe ollama-gemma3-1b --region=europe-west1 --format="value(status.url)")
```

Now use this Service URL to connect Ollama to your remote infirence


```bash
OLLAMA_HOST=$OLLAMA_SERVICE_URL ollama run gemma3:1b
```

You might experience a cold start but after the serverless GPU is provisioned, you will see much faster responses compared to running gemma3 with ollama on the cloud shell's CPU alone. 

Play and test your deployment.

Congratulations. You deployed your first open model with a Cloud Run Serverless GPU.

You can exit by typing in 

```bash
/bye
```

When you are ready to go further, click **Next**.

## Create the Open WebUI frontend Service

Now this step is more open and it your turn to find out the steps.

Create another Cloud Run Service via the UI or using a gcloud command in Cloud Shell with the following config values:

* Container Image: dyrnq/open-webui
* CPU: 8
* Memory: 32Gi
* Allow Unauthenticated invocations


  
and also  add open-webui specific environment variables:

* OLLAMA_BASE_URL and set this value to be equal to your Cloud Run Ollama Service URL 
* WEBUI_AUTH=false

Can you managed to to that? 

Open the open-webui service URL in your browser and you will see the result. 

## Test the Cloud Run service

Now open the Service URL in your web browser.

Once the UI has completed loading, under **Select a model**, choose **Gemma 3 1B**.

Now ask Gemma a question, e.g. "Why is the sky blue?"

Note, that you will see Ollama3:1B as the selected model, which is running and do the infirence privatly on your Ollama Service.


## Congratulations!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

Congratulations for completing the masterclass!

We recommend reviewing the documentation on [Cloud Run GPUs](https://cloud.google.com/run/docs/configuring/services/gpu-best-practices). 


### **What we've covered** 

* How to create a container service in Cloud Run
* How to deploy Ollama with serverless GPUs serving a Gemma 3 1B model
* How to deploy Open WebUI as a frontend container and connecting it to the Ollama Service. 
  
## Summary

You now know how to use Cloud Run GPUs for an Ollama inference server using the
open LLM Gemma3.

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

You have completed the tutorial, well done! Please take a moment and let us know
what you think.

<walkthrough-inline-feedback></walkthrough-inline-feedback>



## Clean up

To avoid inadvertent charges, (for example, if the Cloud Run services are inadvertently invoked more times than  [your monthly Cloud Run invokement allocation in the free tier](https://cloud.google.com/free/docs/gcp-free-tier#free-tier-usage-limits)), you can either delete the Cloud Run or delete the project.

To delete the Cloud Run Services, go to the Cloud Run Cloud Console at  https://console.cloud.google.com/run and delete the all services.

If you choose to delete the entire project, you can go to  [https://console.cloud.google.com/cloud-resource-manager](https://console.cloud.google.com/cloud-resource-manager), select the project you created in Step 2, and choose Delete. If you delete the project, you'll need to change projects in your Cloud SDK. You can view the list of all available projects by running `gcloud projects list`.
