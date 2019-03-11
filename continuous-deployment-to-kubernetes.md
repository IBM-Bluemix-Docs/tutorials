---
copyright:
  years: 2018

lastupdated: "2018-11-14"
---


{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:pre: .pre}
{:tip: .tip}


# Continuous Deployment to Kubernetes
{: #continuous-deployment-to-kubernetes}

This tutorial walks you through the process setting up a continuous integration and delivery pipeline for containerized applications running on the {{site.data.keyword.containershort_notm}}.  You will learn how to set up source control, then build, test and deploy the code to different deployment stages. Next, you will add integrations to other services like security scanners, Slack notifications, and analytics.

{:shortdesc}

## Objectives
{: #objectives}

* Create development and production Kubernetes clusters.
* Create a starter application, run it locally and push it to a Git repository.
* Configure the DevOps delivery pipeline to connect to your Git repository, build and deploy the starter app to dev/prod clusters.
* Explore and integrate the app to use security scanners, Slack notifications, and analytics.

## Services used
{: #services}

This tutorial uses the following {{site.data.keyword.Bluemix_notm}} services:

- [{{site.data.keyword.registrylong_notm}}](https://{DomainName}/containers-kubernetes/launchRegistryView)
- [{{site.data.keyword.containershort_notm}}](https://{DomainName}/containers-kubernetes/catalog/cluster)
- [{{site.data.keyword.contdelivery_short}}](https://{DomainName}/catalog/services/continuous-delivery)
- Slack

**Attention:** This tutorial might incur costs. Use the [Pricing Calculator](https://{DomainName}/pricing/) to generate a cost estimate based on your projected usage.

## Architecture
{: #architecture}

![](images/solution21/Architecture.png)

1. Push code to a private Git repository.
2. Pipeline picks up changes in Git and builds container image.
3. Container image uploaded to registry deployed to a development Kubernetes cluster.
4. Validate changes and deploy to the production cluster.
5. Slack notifications setup for deployment activities.


## Before you begin
{: #prereq}

* [Install {{site.data.keyword.dev_cli_notm}}](https://{DomainName}/docs/cli/idt/setting_up_idt.html#add-cli) - Script to install docker, kubectl, helm, ibmcloud cli and required plug-ins.
* [Set up the {{site.data.keyword.registrylong_notm}} CLI and your registry namespace](https://{DomainName}/docs/services/Registry/registry_setup_cli_namespace.html).
* [Understand the basics of Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/).

## Create development Kubernetes cluster
{: #create_kube_cluster}

{{site.data.keyword.containershort_notm}} delivers powerful tools by combining Docker and Kubernetes technologies, an intuitive user experience, and built-in security and isolation to automate the deployment, operation, scaling, and monitoring of containerized apps in a cluster of compute hosts.

To complete this tutorial you would need to select the **Paid** cluster of type **Standard**. You would be required to setup two clusters, one for development and one for production.
{: shortdesc}

1. Create the first development Kubernetes cluster from the [{{site.data.keyword.Bluemix}} catalog](https://{DomainName}/containers-kubernetes/launch). Later you will be required to repeat these steps and create a production cluster.

   For ease of use, check the configuration details like the number of CPUs, memory and the number of worker nodes you get.
   {:tip}

   ![](images/solution21/KubernetesPaidClusterCreation.png)

2. Select the **Cluster type** and click **Create Cluster** to provision a Kubernetes cluster. The smallest **Machine type** with 2 **CPUs**, 4 **GB RAM**, and 1 **Worker Nodes** is sufficient for this tutorial. All other options can be left to their defaults.
3. Check the status of your **Cluster** and **Worker Nodes** and wait for them to be **ready**.

**Note:** Do not proceed until your workers are ready.

## Create a starter application
{: #create_application}

{{site.data.keyword.containershort_notm}} offers a selection of starter applications, these starter applications can be created using the `ibmcloud dev create` command or the web console. In this tutorial, we are going to use the web console. The starter application greatly cuts down on development time by generating application starters with all the necessary boilerplate, build and configuration code so that you can start coding business logic faster.

1. From the [{{site.data.keyword.cloud_notm}} console](https://{DomainName}), use the left side menu option and select [Web Apps](https://{DomainName}/developer/appservice/dashboard).
2. Under **Start from the Web**, section click on the **Get Started** button.
3. Select the `Node.js web app with Express.js` and then `Create app` to create a Node.js starter application.
4. Enter the **name** `mynodestarter`. Then, click **Create**. 

## Configure DevOps delivery pipeline
{: #create_devops}

1. Now that you successfully created the starter application, under the **Deploy your App**, click on the **Deploy to Cloud** button.
2. Selecting the Kubernetes Cluster deployment method, select the cluster created earlier and then click **Create**. This will create a toolchain and delivery pipeline for you. ![](images/solution21/BindCluster.png)
3. Once the pipeline created, click on **View Toolchain** and then **Delivery Pipeline** to view the pipeline. ![](images/solution21/Delivery-pipeline.png)
4. After the deploy stages complete, click on the **View logs and history** to see the logs.
5. Visit the URL displayed to access the application (`http://worker-public-ip:portnumber/`). ![](images/solution21/Logs.png)
Done, you've used the App Service UI to create the starter applications, and configured the pipeline to build and deploy the application to your cluster.

## Clone, build and run the application locally
{: #cloneandbuildapp}

In this section, you will use the starter app created in the earlier section, clone it to your local machine, modify the code and then build/run it locally.
{: shortdesc}

### Clone the application
1. From the Toolchain overview, select the **Git** tile under **Code**. You will be redirected to your git repository page where you can clone the repo.![HelloWorld](images/solution21/DevOps_Toolchain.png)

2. If you haven't set up SSH keys yet, you should see a notification bar at the top with instructions. Follow the steps by opening the **add an SSH key** link in a new tab or if you want to use HTTPS instead of SSH, follow the steps by clicking **create a personal access token**. Remember to save the key or token for future reference.
3. Select SSH or HTTPS and copy the git URL. Clone the source to your local machine. If you're prompted for a username, provide your git username. For the password, use an existing **SSH key** or **personal access token** or the one created you created in the previous step.

   ```bash
   git clone <your_repo_url>
   cd <name_of_your_app>
   ```
   {: codeblock}

4. Open the cloned repository in an IDE of your choice and navigate to `public/index.html`. Update the code by trying to change "Congratulations!" to something else and the save the file.

### Build the application locally
You can build and run the application as you normally would using `mvn` for java local development or `npm` for node development.  You can also build a docker image and run the application in a container to ensure consistent execution locally and on the cloud. Use the following steps to build your docker image.
{: shortdesc}

1. Ensure your local Docker engine is started, to check run the command below:
   ```
   docker ps
   ```
   {: codeblock}
2. Navigate to the generated project directory cloned.
   ```
   cd <project name>
   ```
   {: codeblock}
3. Build the application locally.
   ```
   ibmcloud dev build
   ```
   {: codeblock}

   This might take a few minutes to run as all the application dependencies are downloaded and a Docker image, which contains your application and all the required environment, is built.

### Run the application locally

1. Run the container.
   ```
   ibmcloud dev run
   ```
   {: codeblock}

   This uses your local Docker engine to run the docker image that you built in the previous step.
2. After your container starts, go to http://localhost:3000/
   ![](images/solution21/node_starter_localhost.png)

## Push application to your Git repository

In this section, you will commit your change to your Git repository. The pipeline will pick up the commit and push the changes to your cluster automatically.
1. In your terminal window, make sure you are inside the repo you cloned.
2. Push the change to your repository with three simple steps: Add, commit, and push.
   ```bash
   git add public/index.html
   git commit -m "my first changes"
   git push origin master
   ```
   {: codeblock}

3. Go to the toolchain you created earlier and click the **Delivery Pipeline** tile.
4. Confirm that you see n **BUILD** and **DEPLOY** stage.
   ![](images/solution21/Delivery-pipeline.png)
5. Wait for the **DEPLOY** stage to complete.
6. Click the application **url** under Last Execution result to view your changes live.

If you don't see your application updating, check the logs of the DEPLOY and BUILD stages of your pipeline.

## Security using Vulnerability Advisor
{: #vulnerability_advisor}

In this step, you will explore the [Vulnerability Advisor](https://{DomainName}/docs/services/va/va_index.html). The vulnerability advisor is used check the security status of container images before deployment, and also it checks the status of running containers.

1. Go to the toolchain you created earlier and click the **Delivery Pipeline** tile.
1. Click on **Add Stage** and change MyStage to **Validate Stage** and then click on the JOBS  > **ADD JOB**.

   1. Select **Test** as the Job Type and Change **Test** to **Vulnerability advisor** in the box.
   1. Under Tester type, select **Vulnerability Advisor**. All the other fields should be populated automatically.
      Container Registry namespace should be same as the one mentioned in **Build Stage** of this toolchain.
      {:tip}
   1. Edit the **Test script** section and replace `SAFE\ to\ deploy` in the last line with `NO\ ISSUES`
   1. Save the stage
1. Drag and move the **Validate Stage** to the middle then click **Run** ![](images/solution21/run.png) on the **Validate Stage**. You will see that the **Validate stage** fails.

   ![](images/solution21/toolchain.png)

1. Click on **View logs and history** to see the vulnerability assessment.The end of the log says:

   ```
   The scan results show that 3 ISSUES were found for the image.

   Configuration Issues Found
   ==========================

   Configuration Issue ID                     Policy Status   Security Practice                                    How to Resolve
   application_configuration:mysql.ssl-ca     Active          A setting in /etc/mysql/my.cnf that specifies the    ssl-ca is not specified in /etc/mysql/my.cnf.
                                                              Certificate Authority (CA) certificate.
   application_configuration:mysql.ssl-cert   Active          A setting in /etc/mysql/my.cnf that specifies the    ssl-cert is not specified in /etc/mysql/my.cnf
                                                              server public key certificate. This certificate      file.
                                                              can be sent to the client and authenticated
                                                              against its CA certificate.
   application_configuration:mysql.ssl-key    Active          A setting in /etc/mysql/my.cnf that identifies the   ssl-key is not specified in /etc/mysql/my.cnf.
                                                              server private key.
   ```

   You can see the detailed vulnerability assessments of all the scanned repositories [here](https://{DomainName}/containers-kubernetes/registry/private)
   {:tip}

   The stage may fail saying the image *has not been scanned* if the scan for vulnerabilities takes more than 3 minutes. This timeout can be changed by editing the job script and increasing the number of iterations to wait for the scan results.
   {:tip}

1. Let's fix the vulnerabilities by following the corrective action. Open the cloned repository in an IDE or select Eclipse Orion web IDE tile, open `Dockerfile` and add the below command after `EXPOSE 3000`
   ```sh
   RUN apt-get remove -y mysql-common \
     && rm -rf /etc/mysql
   ```
   {: codeblock}

1. Commit and Push the changes. This should trigger the toolchain and fix the **Validate Stage**.

   ```
   git add Dockerfile
   git commit -m "Fix Vulnerabilities"
   git push origin master
   ```

   {: codeblock}

## Create production Kubernetes cluster

{: #deploytoproduction}

In this section, you will complete the deployment pipeline by deploying the Kubernetes application to development and production environments respectively. Ideally, we want to set up an automatic deployment for the development environment and a manual deployment for the production environment. Before we do that, let's explore the two ways in which you can deliver this. It's possible to use one cluster for both development and production environment. However, it's recommended to have two separate clusters, one for development and one for production. Let's explore setting up a second cluster for production.
{: shortdesc}

1. Following instructions in [Create development Kubernetes cluster](continuous-deployment-to-kubernetes.html#create_kube_cluster) section, and create a new cluster. Name this cluster `prod-cluster`.
2. Go to the toolchain you created earlier and click the **Delivery Pipeline** tile.
3. Rename the **Deploy Stage** to `Deploy dev`, you can do that by clicking on settings Icon >  **Configure Stage**.
   ![](images/solution21/deploy_stage.png)
4. Clone the **Deploy dev** stage (settings icon > Clone Stage) and name the cloned stage as `Deploy prod`.
5. Change the **stage trigger** to `Run jobs only when this stage is run manually`. ![](images/solution21/prod-stage.png)
6. Under the **Job** tab, change the cluster name to the newly created cluster and then **Save** the stage.
7. You now should have the full deployment setup, to deploy from dev to production, you must manually run the `Deploy prod` stage to deploy to production. ![](images/solution21/full-deploy.png)

Done, you've now created a production cluster and configured the pipeline to push updates to your production cluster manually. This is a simplification process stage over a more advanced scenario where you would include unit tests and integration tests as part of the pipeline.

## Setup Slack notifications
{: #setup_slack}

1. Go back to view the list of [toolchains](https://{DomainName}/devops/toolchains) and select your toolchain, then click on **Add a Tool**.
2. Search for slack in the search box or scroll down to see **Slack**. Click to see the configuration page.
    ![](images/solution21/configure_slack.png)
3. For **Slack webhook**, follow the steps in this [link](https://my.slack.com/services/new/incoming-webhook/). You need to login with your Slack credentials and provide an existing channel name or create a new one.
4. Once the Incoming webhook integration is added, copy the **Webhook URL** and paste the same under **Slack webhook**.
5. The slack channel is the channel name you provided while creating a webhook integration above.
6. **Slack team name** is the team-name(first part) of team-name.slack.com. e.g., kube is the team name in kube.slack.com
7. Click **Create Integration**. A new tile will be added to your toolchain.
    ![](images/solution21/toolchain_slack.png)
8. From now on, whenever your toolchain executes, You should see slack notifications in the channel you configured.
    ![](images/solution21/slack_channel.png)

## Remove resources
{: #removeresources}

In this step, you will clean up the resources to remove what you created above.

- Delete the Git repository.
- Delete the toolchain.
- Delete the two clusters.
- Delete the Slack channel.

## Expand the Tutorial
{: #expandTutorial}

Do you want to learn more? Here are some ideas of what you can do next:

- [Analyze logs and monitor the health of Kubernetes applications using Kibana and Grafana](kubernetes-log-analysis-kibana.html).
- Add a testing environment and deploy it to a 3rd cluster.
- Deploy the production cluster [across multiple locations](multi-region-webapp.html).
- Enhance your pipeline with additional quality controls and analyics using [{{site.data.keyword.DRA_short}}](https://{DomainName}/catalog/services/devops-insights).

## Related Content
{: #related}

* End to end Kubernetes solution guide, [moving VM based apps to Kubernetes](vm-to-containers-and-kubernetes.html).

* [Security](https://{DomainName}/docs/containers/cs_secure.html#cluster) for IBM Cloud Container Service.

* Toolchain [integrations](https://{DomainName}/docs/services/ContinuousDelivery/toolchains_integrations.html#integrations).

* Analyze logs and monitor the health of Kubernetes applications using [Kibana and Grafana](kubernetes-log-analysis-kibana.html).


