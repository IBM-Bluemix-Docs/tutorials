---
copyright:
  years: 2018
lastupdated: "2018-02-28"

---

{:java: #java .ph data-hd-programlang='java'}
{:swift: #swift .ph data-hd-programlang='swift'}
{:ios: #ios data-hd-operatingsystem="ios"}
{:android: #android data-hd-operatingsystem="android"}
{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}

# Best practices for organizing users, teams, applications

This tutorial gives an overview of the concepts available in {{site.data.keyword.cloud_notm}} to manage identity and access management and how they can be implemented to support the multiple development stages of an application.

{:shortdesc}

## Objectives
{: #objectives}

* Learn about {{site.data.keyword.iamlong}} and Cloud Foundry access models
* Configure a project with separation between roles and environments
* Setup continuous integration

## Products
{: #products}

This tutorial uses the following products:
* [{{site.data.keyword.iamlong}}](https://console.bluemix.net/docs/iam/index.html)
* [{{site.data.keyword.containershort_notm}}](https://console.bluemix.net/containers-kubernetes/catalog/cluster)
* [{{site.data.keyword.cos_full_notm}}](https://console.bluemix.net/catalog/infrastructure/cloud-object-storage)
* [Cloud Foundry](https://console.bluemix.net/catalog/?category=cf-apps&search=foundry)
* [{{site.data.keyword.cloudantfull}}](https://console.bluemix.net/catalog/services/cloudant-nosql-db)

## Define a project

When building an application, it is very common to define multiple environments reflecting the development lifecycle of a project from a developer committing code to the application code being made available to the end-users. *Sandbox*, *test*, *staging*, *UAT* (user acceptance testing), *pre-production*, *production* are typical names for these environments.

Isolating the underlying resources, implementing governance and access policies, protecting a production workload, validating changes before pushing them to production, are some of the reasons why you would want to create these separate environments.

Let's consider a sample project with the following components:
* several microservices deployed in {{site.data.keyword.containershort_notm}},
* databases,
* file storage buckets.

In this project, we define three environments:
* *Development* - this environment is continuously updated with every commit, unit tests, smoke tests are executed. It gives access to the latest and greatest deployment of the project.
* *Testing* - this environment is built after a stable branch or tag of the code. This is where user acceptance testing is made. Its configuration is similar to the production environment. It is loaded with realistic data (anonymized production data as example).
* *Production* - this environment is updated with the version validated in the previous environment.

**A delivery pipeline manages the progression of a build through the environment.** It can be fully automated or include manual validation gates to promote approved builds between environments - this is really open and should be set up to match the company best practices and workflows.

To support the execution of the build pipeline,  we introduce **a functional user** - a regular {{site.data.keyword.cloud_notm}} user but a team member with no real identity in the physical world. This functional user will own the delivery pipelines and any other cloud resources requiring strong ownership. This approach helps in the case where a team member leaves the company or is moving to another project. The functional user will be dedicated to your project and will not change over the lifetime of the project. The next thing you will want to create is [an API key](https://console.bluemix.net/docs/iam/apikeys.html#manapikey) for this functional user. You will select this API key when you setup the DevOps pipelines, or when you want to run automation scripts, to impersonate the functional user.

When it comes to assigning responsibilities to the project team members, let's define the following roles and related permissions:

|           | Development | Testing | Production |
| --------- | ----------- | ------- | ---------- |
| Developer | <ul><li>contributes code</li><li>can access log files</li><li>can view app and service configuration</li><li>use the deployed applications</li></ul> | <ul><li>can access log files</li><li>can view app and service configuration</li><li>use the deployed applications</li></ul> | <ul><li>no access</li></ul> |
| Tester    | <ul><li>use the deployed applications</li></ul> | <ul><li>use the deployed applications</li></ul> | <ul><li>no access</li></ul> |
| Operator  | <ul><li>can access log files</li><li>can view/set app and service configuration</li></ul> | <ul><li>can access log files</li><li>can view/set app and service configuration</li></ul> | <ul><li>can access log files</li><li>can view/set app and service configuration</li></ul> |
| Pipeline Functional User  | <ul><li>can deploy/undeploy applications</li><li>can view/set app and service configuration</li></ul> | <ul><li>can deploy/undeploy applications</li><li>can view/set app and service configuration</li></ul> | <ul><li>can deploy/undeploy applications</li><li>can view/set app and service configuration</li></ul> |

## Identity and Access Management (IAM)
{: #first_objective}

{{site.data.keyword.iamshort}} (IAM) enables you to securely authenticate users for both platform and infrastructure services and control access to **resources** consistently across the {{site.data.keyword.cloud_notm}} platform. A set of {{site.data.keyword.cloud_notm}} services are enabled to use Cloud IAM for access control and are organized into **resource groups** within your **account** to enable giving **users** quick and easy access to more than one resource at a time. Cloud IAM access **policies** are used to assign users and service IDs access to the resources within your account.

A **policy** assigns a user or service ID one or more **roles** with a combination of attributes that define the scope of access. The policy can provide access to a single service down to the instance level, or the policy can apply to a set of resources organized together in a resource group. Depending on the user roles that you assign, the user or service ID is allowed varying levels of access for completing platform management tasks or accessing a service by using the UI or performing specific types of API calls.

<p style="text-align: center;">
  <img src="./images/solution20-users-teams-applications/iam-model.png" width="50%" />
</p>

At this time, not all services in the {{site.data.keyword.cloud_notm}} catalog can be managed by using IAM. For these services, you can continue to use Cloud Foundry by providing users access to the organization and space to which the instance belongs with a Cloud Foundry role assigned to define the level of access that is allowed.

<p style="text-align: center;">
  <img src="./images/solution20-users-teams-applications/cloudfoundry-model.png" width="50%" />
</p>

## Create the resources for one environment

Although the three environments needed by this sample project require different access rights and may need to be allocated different capacities, they share a common architecture pattern.

<p style="text-align: center;">
  <img src="./images/solution20-users-teams-applications/one-environment.png" width="80%" />
</p>


Let's start by building the Development environment.

1. [Select an {{site.data.keyword.cloud_notm}} region](https://console.bluemix.net/dashboard) where to deploy the environment
1. [Create an organization for the project](https://console.bluemix.net/docs/account/orgs_spaces.html#createorg)
1. [Create a Cloud Foundry space for the environment](https://console.bluemix.net/docs/account/orgs_spaces.html#spaceinfo)
1. [Create a new Kubernetes cluster](https://console.bluemix.net/containers-kubernetes/catalog/cluster) in {{site.data.keyword.containershort_notm}}
1. Select your cluster [in the console](https://console.bluemix.net/containers-kubernetes/clusters)
1. Use *Enable Logging* to redirect the cluster logs to the Cloud Foundry space created in the previous steps. If later you want to change the space where the cluster is sending its logging data, you can use the [logging plugin for the bx command line](https://console.bluemix.net/docs/containers/cs_health.html#log_sources_update).
1. Create the Cloud Foundry services used by the project under the space dedicated to the environment

The following diagram shows where the project resources are created under the account:

<p style="text-align: center;">
  <img src="./images/solution20-users-teams-applications/resources.png" style="height: 400px;" />
</p>

## Assign roles within the environment

1. Invite users to the account
1. Assign Policies to the users to control who can access the {{site.data.keyword.containershort_notm}} instance and their permissions. Refer to the [access policy definition](https://console.bluemix.net/docs/containers/cs_users.html#access_policies) to select the right policy for a user in the environment. 
1. Configure their Cloud Foundry organization and space roles based on their needs within the environment. Refer to the [role definition](https://console.bluemix.net/docs/iam/cfaccess.html#cfaccess) to assign the right roles based on the environment.

Refer to the documentation of services to understand how a service is mapping IAM and Cloud Foundry roles to specific actions. See for example [how the IBM Cloud Monitoring service maps IAM roles to actions](https://console.bluemix.net/docs/services/cloud-monitoring/security_ov.html#iam_roles).

Assigning the right roles to users will require several iterations and refinement. Given permissions can be controlled at the resource group level, for all resources in a group or be fine-grained down to a specific instance of a service, you will discover over time what are the ideal access policies for your project. 

A good practice is to start with the minimum set of permissions then expand carefully as needed. For Kubernetes, you will want to look at its [Role-Based Access Control (RBAC)](https://kubernetes.io/docs/admin/authorization/rbac/) to configure in-cluster authorizations.

For the Development environment, the user responsibilities defined earlier could translate to the following:

|           | IAM Access policies | Cloud Foundry |
| --------- | ----------- | ------- |
| Developer | <ul><li>Resource Group: *Viewer*</li><li>Platform Access Roles in the Resource Group: *Viewer*</li><li>Monitoring: *Administrator, Editor, Viewer*</li></ul> | <ul><li>Organization Role: *Auditor*</li><li>Space Role: *Developer*</li></ul> |
| Tester    | <ul><li>No configuration needed. Tester accesses the deployed application, not the development environments</li></ul> | <ul><li>No configuration needed</li></ul> |
| Operator  | <ul><li>Resource Group: *Viewer*</li><li>Platform Access Roles in the Resource Group: *Operator*, *Viewer*</li><li>Monitoring: *Administrator, Editor, Viewer*</li></ul> | <ul><li>Organization Role: *Auditor*</li><li>Space Role: *Developer*</li></ul> |
| Pipeline Functional User | <ul><li>Resource Group: *Viewer*</li><li>Platform Access Roles in the Resource Group: *Editor*, *Viewer*</li></ul> | <ul><li>Organization Role: *Auditor*</li><li>Space Role: *Developer*</li></ul> |

The IAM access policies and Cloud Foundry roles are defined in the [Identify and Access Management user interface](https://console.bluemix.net/iam/#/users):

<p style="text-align: center;">
  <img title="" src="./images/solution20-users-teams-applications/edit-policy.png" height="400" />
</p>

## Replicate for multiple environments

From there, you can replicate similar steps to build the other environments.

1. Create one Cloud Foundry space per environment
1. Create the required service instances in each space
1. Create one cluster per environment

<p style="text-align: center;">
  <img title="Using separate clusters to isolate environments" src="./images/solution20-users-teams-applications/multiple-environments.png" width="80%" />
</p>

Using a combination of tools like the [{{site.data.keyword.cloud_notm}} `bx` CLI](https://github.com/IBM-Cloud/ibm-cloud-developer-tools), [HashiCorp's `terraform`](https://www.terraform.io/), the [{{site.data.keyword.cloud_notm}} provider for Terraform](https://github.com/IBM-Cloud/terraform-provider-ibm), Kubernetes CLI `kubectl`, you can script and automate the creation of these environments.

Separate Kubernetes clusters for the environments come with good properties:
* no matter the environment, all clusters will tend to look the same;
* it is easier to control who has access to a specific cluster;
* it gives flexibility in the update cycles for deployments and underlying resources; when there is a new Kubernetes version, it gives you the option to update the Development cluster first, validate your application then update the other environment;
* it avoids mixing different workloads that may impact each other such as isolating the production deployment from the others.

Another approach is to use [Kubernetes namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) in conjunction with [Kubernetes resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) to isolate environments and control resource consumption.

<p style="text-align: center;">
  <img title="Using separate namespaces to isolate environments" src="./images/solution20-users-teams-applications/multiple-environments-with-namespaces.png" width="80%" />
</p>

## Setup delivery pipeline

When it comes to deploying to the different environments, your continuous integration / continuous delivery pipeline can be setup to drive the full process:
* continuously update the `Development` environment with the latest and greatest code from the `development` branch, running unit tests and integration tests on the dedicated cluster;
* promote development builds to the `Testing` environment, either automatically if all tests from the previous stages are OK or through a manual promotion process. Some teams will use different branches too here, merging the working development state to a `stable` branch as example;
* Repeat a similar process to move to the `Production` environment.
 
<p style="text-align: center;">
  <img src="./images/solution20-users-teams-applications/cicd.png" />
</p>

When configuring the DevOps pipeline, make sure to use the API key of a functional user. Only the functional user should need to have the required rights to deploy apps to your clusters.

During the build phase, it is important to properly version the Docker images. You can use the Git commit revision as part of the image tag, or a unique identifier provided by your DevOps toolchain; any identifier that will make it easy for you to map the image to the actual build and source code contained in the image.

As you get acquainted with Kubernetes, [Helm](https://helm.sh/), the package manager for Kubernetes, will become a handy tool to version, assemble and deploy your application. [This sample DevOps toolchain](https://github.com/open-toolchain/simple-helm-toolchain) is a good starting point and is preconfigured for continuous delivery to a Kubernetes cluster. As your project grows into multiple microservices, the [Helm umbrella chart](https://github.com/kubernetes/helm/blob/master/docs/charts_tips_and_tricks.md#complex-charts-with-many-dependencies) will provide a good solution to compose your application.

## Related information

* [Getting Started with {{site.data.keyword.iamshort}}](https://console.bluemix.net/docs/iam/quickstart.html#getstarted)
* [Analyze logs and monitor the health of Kubernetes applications](./kubernetes-log-analysis-kibana.html)
* [Continuous Deployment to Kubernetes](./continuous-deployment-to-kubernetes.html)
* [Hello Helm toolchain](https://github.com/open-toolchain/simple-helm-toolchain)
* [Develop a microservices application with Kubernetes and Helm
](https://github.com/open-toolchain/microservices-helm-toolchain)
