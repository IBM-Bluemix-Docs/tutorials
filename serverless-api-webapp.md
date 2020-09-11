---
subcollection: solution-tutorials
copyright:
  years: 2017, 2019, 2020
lastupdated: "2020-07-21"
lasttested: "2020-06-03"

content-type: tutorial
services: openwhisk, api-gateway, Cloudant
account-plan: paid
completion-time: 2h
---

{:step: data-tutorial-type='step'}
{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}


# Serverless web application and API
{: #serverless-api-webapp}
{: toc-content-type="tutorial"}
{: toc-services="openwhisk, api-gateway, Cloudant"}
{: toc-completion-time="2h"}

This tutorial may incur costs. Use the [Cost Estimator](https://{DomainName}/estimator/review) to generate a cost estimate based on your projected usage.
{: tip}

In this tutorial, you will create a serverless web application by hosting static website content on GitHub Pages and implementing the application backend using {{site.data.keyword.openwhisk}}.
{: shortdesc}

As an event-driven platform, {{site.data.keyword.openwhisk_short}} supports a [variety of use cases](https://{DomainName}/docs/openwhisk?topic=cloud-functions-use_cases). Building web applications and APIs is one of them. With web apps, events are the interactions between the web browsers (or REST clients) and your web app, the HTTP requests. Instead of provisioning a virtual machine, a container or a Cloud Foundry runtime to deploy your backend, you can implement your backend API with a serverless platform. This can be a good solution to avoid paying for idle time and to let the platform scale as needed.

Any action (or function) in {{site.data.keyword.openwhisk_short}} can be turned into a HTTP endpoint ready to be consumed by web clients. When enabled for web, these actions are called *web actions*. Once you have web actions, you can assemble them into a full-featured API with API Gateway. API Gateway is a component of {{site.data.keyword.openwhisk_short}} to expose APIs. It comes with security, OAuth support, rate limiting, custom domain support.

## Objectives
{: #serverless-api-webapp-0}

* Deploy a serverless backend and a database
* Expose a REST API
* Host a static website
* Optional: Use a custom domain for the REST API


The application shown in this tutorial is a simple guestbook website where users can post messages.

<p style="text-align: center;">

   ![Architecture](./images/solution8/Architecture.png)
</p>

1. The user accesses the application hosted in GitHub Pages.
2. The web application calls a backend API.
3. The backend API is defined in API Gateway.
4. API Gateway forwards the request to [{{site.data.keyword.openwhisk_short}}](https://{DomainName}/openwhisk).
5. The {{site.data.keyword.openwhisk_short}} actions use [{{site.data.keyword.cloudant_short_notm}}](https://{DomainName}/catalog/services/cloudantNoSQLDB) to store and retrieve guestbook entries.

## Before you begin
{: #serverless-api-webapp-prereqs}

This guide uses GitHub Pages to host the static website. Make sure you have a public GitHub account.

## Create the Guestbook database
{: #serverless-api-webapp-2}
{: step}

Let's start by creating an {{site.data.keyword.cloudant_short_notm}} service instance. {{site.data.keyword.cloudant_short_notm}} is a fully managed JSON document database. {{site.data.keyword.cloudant_short_notm}} is built upon and compatible with Apache CouchDB.

1. In the [Catalog](https://{DomainName}/catalog/), under **Services**, go to the **Databases** category. Click on the **{{site.data.keyword.cloudant}}** tile. In the new dialog:
   1. Under **Multitenant** select a region.
   1. Under **Configure Cloudant instance** pick a **unique* name for the service, such as `<yourinitials>-guestbook-db`.
   1. Select a resource group.
   2. Select **IAM and legacy credentials** as authentication method.
   3. Make sure the **Lite** plan is selected. If you already have a Lite plan, select another service plan.
   4. Click **Create**.
2. Back in the [{{site.data.keyword.Bluemix_short}} Resource List](https://{DomainName}/resources/), under **Services**, click on the {{site.data.keyword.cloudant}} instance you created to open the instance full details page. Note: You may be required to wait until the status of the service changes to `Provisioned`.
3. Click on **Launch Dashboard** to open the dashboard in a new browser tab.
4. In the upper right, click on **Create Database**. Enter ***guestbook*** as name and select **Non-Partitioned** under **Partitioning**. Click **Create** to create the database.
5. Switch back to the browser tab with the service dashboard page. Go to **Service credentials**, then:
   1. Click **New credential**.
   2. Set the name to **for-guestbook**. Leave the role as **Manager**.
   3. Click **Add** to add the new credentials.
   4. Expand the newly created credentials and review them. We will need these credentials later to allow Cloud Functions actions to read/write to your Cloudant service.

## Create serverless actions
{: #serverless-api-webapp-3}
{: step}

In this section, you will create serverless actions (commonly termed as **Functions**). {{site.data.keyword.openwhisk}} (based on Apache OpenWhisk) is a Function-as-a-Service (FaaS) platform which executes functions in response to incoming events. Serverless functions only incur charges for the execution time.

### Sequence of actions to save the guestbook entry
{: #serverless-api-webapp-4}

You will create a **sequence** which is a chain of actions. In a sequence, the output of one action acts as input to the following action and so on. The first sequence you will create is used to persist a guest message. Provided a name, an emailID and a comment, the sequence will:
   * Create a document to be persisted.
   * Store the document in the {{site.data.keyword.cloudant_short_notm}} database.

Start by creating the first action:

1. In the browser, open a tab and go to [**Functions**](https://{DomainName}/functions).
2. From the namespace drop-down on the top right either select an existing namespace or use **Create Namespace** to create a new one.
2. With a namespace selected, click on **Actions** in the left pane and then **Create** on the right.
3. Under **Create** click on **Action** to open the **Create Action** form.
4. Enter `prepare-entry-for-save` as name, click **Create Package** to create a new package with name `guestbook` and pick a **Node.js** as **Runtime** (Note: Pick the latest version). Click **Create** to create the action.
5. In the new dialog replace the existing code with the code snippet below:
   ```js
   /**
    * Prepare the guestbook entry to be persisted
    */
   function main(params) {
     if (!params.name || !params.comment) {
       return Promise.reject({ error: 'no name or comment'});
     }

     return {
       doc: {
         createdAt: new Date(),
          name: params.name,
          email: params.email,
          comment: params.comment
       }
     };
   }
   ```
   {: codeblock}
6. Thereafter click **Save**.

Then add the action to a sequence:

1. On the left pane, click on **Enclosing Sequences** and then **Add To Sequence**.
1. Under **Create New** set the **Sequence Name** to `save-guestbook-entry-sequence` and choose **guestbook** as package.
1. Then finish by clicking **Create and Add**.

Now, add the second action to that sequence:

1. Click on the entry **save-guestbook-entry-sequence**. It opens sequence details. Then click **Add** on the upper right.
1. Instead of **Create New** select **Use Public**. It loads and displays icons for available integrations. Pick **Cloudant**.
2. Under **Actions** choose **create-document**.
3. Create a **New Binding** and complete the form as follows:
   1. Set **Name** to `binding-for-guestbook`.
   2. For **Instance** select your instance, for the credentials **for-guestbook** as created earlier, and as **Database** pick **guestbook**.
4. Click **Add**, thereafter **Save**.
5. To test the entire sequence, click on **Invoke with parameters** and enter the JSON below
   ```json
      {
        "name": "John Smith",
        "email": "john@smith.com",
        "comment": "this is my comment"
      }
   ```
   {: codeblock}
6. Click **Invoke**.

### Sequence of actions to retrieve entries
{: #serverless-api-webapp-5}

The second sequence is used to retrieve the existing guestbook entries. This sequence will:
   * List all documents from the database.
   * Format the documents and return them.

1. Under [**Functions**](https://{DomainName}/functions), click on **Actions** and then **Create**.
2. Then, after selecting **Action**, use `set-read-input` as name. Again, select **guestbook** as package and a **Node.js** version as runtime. Click **Create**.
3. In the action details, replace the existing code with the code snippet below. This action passes the appropriate parameters to the next action in the sequence.
   ```js
   function main(params) {
     return {
       params: {
         include_docs: true
       }
     };
   }
   ```
   {: codeblock}
4. Click **Save**.

Add the action to a sequence:

1. Similar to earlier, click on **Enclosing Sequences**, **Add to Sequence** and **Create New**.
1. Enter `read-guestbook-entries-sequence` for the **Sequence Name** and click **Create and Add**.

Complete the sequence:

1. Click on **read-guestbook-entries-sequence** sequence and then click **Add** to create and add the second action to get documents from {{site.data.keyword.cloudant_short_notm}}.
1. Under **Use Public**, choose **{{site.data.keyword.cloudant_short_notm}}** and then **list-documents**
1. Under **My Bindings**, choose **binding-for-guestbook** and **Add** to create and add this public action to your sequence.
1. Click **Add** again to create and add the third action which will format the documents from {{site.data.keyword.cloudant_short_notm}}.
1. Under **Create New** enter `format-entries` for name and then click **Create and Add**.
1. Click on **format-entries** and replace the code with below:
   ```js
   const md5 = require('spark-md5');

   function main(params) {
     return {
       entries: params.rows.map((row) => { return {
         name: row.doc.name,
         email: row.doc.email,
         comment: row.doc.comment,
         createdAt: row.doc.createdAt,
         icon: (row.doc.email ? `https://secure.gravatar.com/avatar/${md5.hash(row.doc.email.trim().toLowerCase())}?s=64` : null)
       }})
     };
   }
   ```
   {: codeblock}
1. Click on **Save**.
1. Choose the sequence by clicking on **Actions** and then **read-guestbook-entries-sequence**.
1. Click on **Save** and then **Invoke**.

## Create an API
{: #serverless-api-webapp-6}
{: step}
1. Go to [Actions](https://{DomainName}/functions/actions).
2. Select the **read-guestbook-entries-sequence** sequence. Next to the name, click on **Web Action**, check **Enable as Web Action** and **Save**.
3. Do the same for the **save-guestbook-entry-sequence** sequence.
4. Go to [APIs](https://{DomainName}/functions/apimanagement) and click **Create API** (or **Create Managed API** if you have existing APIs).
5. Set the API name to `guestbook` and, accordingly, the base path to `/guestbook`.
6. Click on **Create operation** and create an operation to retrieve guestbook entries:
   1. Set **path** to `/entries`
   2. Set **verb** to `GET`
   3. Select the **read-guestbook-entries-sequence** action
7. Click on **Create operation** and create an operation to persist a guestbook entry:
   1. Set **path** to `/entries`
   2. Set **verb** to `PUT`
   3. Select the **save-guestbook-entry-sequence** action
8. Scroll to the end of the page to **Create** the API. Make note of the provided route, as you will use it from your web application.

## Deploy the web app
{: #serverless-api-webapp-7}
{: step}

1. Fork the Guestbook user interface repository https://github.com/IBM-Cloud/serverless-guestbook to your public GitHub. You can do this by going to https://github.com/IBM-Cloud/serverless-guestbook in the browser, and then clicking the **Fork** button.
2. In your forked version of the code, modify **docs/guestbook.js** and replace the value of **apiUrl** with the route given by API Gateway. You can do this by navigating to the **docs/guestbook.js** file, and then clicking the edit pencil.
3. Commit the modified file to your forked repository by clicking the **Commit Changes** button at the bottom of the page.
4. In the Settings page of your repository, scroll to **GitHub Pages**, and change the source to **master branch /docs folder**. This setting will be automatically saved.
5. Access the public page for your repository. The link to your public page can be found under the **GitHub Pages** section. Give it a few minutes before it gets activated.
6. You should see the `test` guestbook entry created earlier.
7. Add new entries.

![](images/solution8/Guestbook.png)

## Optional: Use your own domain for the API
{: #serverless-api-webapp-custom-domain}
{: step}

Creating a managed API gives you a default endpoint like `https://1234abcd.us-south.apigw.appdomain.cloud/guestbook`. In this section, you will configure this endpoint to be able to handle requests coming from your custom subdomain.

### Obtain a certificate for the custom domain
{: #serverless-api-webapp-9}

Exposing {{site.data.keyword.openwhisk_short}} actions through a custom domain will require a secure HTTPS connection. You should obtain a SSL certificate for the domain and subdomain you plan to use with the serverless back-end. Assuming a domain like *mydomain.com*, the actions could be hosted at *guestbook-api.mydomain.com*. The certificate will need to be issued for *guestbook-api.mydomain.com* (or **.mydomain.com*).

You can get free SSL certificates from [Let's Encrypt](https://letsencrypt.org/) or directly through [{{site.data.keyword.cloudcerts_short}}](https://{DomainName}/catalog/services/cloudcerts). During the process you may need to configure a DNS record of type TXT in your DNS interface to prove you are the owner of the domain.
{:tip}

Once you have obtained the SSL certificate and private key for your domain make sure to convert them to the [PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) format.

1. To convert a Certificate to PEM format:
   ```
   openssl x509 -in domain-crt.txt -out domain-crt.pem -outform PEM
   ```
1. To convert a Private Key to PEM format:
   ```
   openssl rsa -in domain-key.txt -out domain-key.pem -outform PEM
   ```

### Import the certificate to a central repository
{: #serverless-api-webapp-10}

1. Create a [{{site.data.keyword.cloudcerts_short}}](https://{DomainName}/catalog/services/cloudcerts) instance in a supported location.
1. In the service dashboard, use **Import Certificate**:
   * Set **Name** to the custom subdomain and domain, such as `guestbook-api.mydomain.com`.
   * Browse for the **Certificate file** in PEM format.
   * Browse for the **Private key file** in PEM format.
   * **Import**.

### Configure the custom domain for the managed API
{: #serverless-api-webapp-11}

1. Go to [APIs / Custom domains](https://{DomainName}/apis/domains).
1. In the Region selector, select the location where you deployed the actions.
1. Locate the custom domain linked to the organization and space where you created the actions and the managed API. Click **Change Settings** in the action menu.
1. Make note of the **Default domain / alias** value.
1. Check **Apply custom domain**
   1. Set **Domain name** to the domain you will use such as `guestbook-api.mydomain.com`.
   1. Select the {{site.data.keyword.cloudcerts_short}} instance holding the certificate.
   1. Select the certificate for the domain.
1. Go to your DNS provider and create a new **DNS TXT record** mapping your domain to the API default domain / alias. The DNS TXT record can be removed once the settings have been applied.
1. Save the custom domain settings. The dialog will check for the existence of the DNS TXT record.
1. Finally, return to your DNS provider's settings and create a CNAME record pointing your custom domain (e.g. guestbook-api.mydomain.com) to the Default domain / Alias. This will cause traffic through your custom domain to be routed to your backend API.

Once the DNS changes have been propagated, you will be able to access your guestbook api at https://guestbook-api.mydomain.com/guestbook.

1. Edit **docs/guestbook.js** and update the value of **apiUrl** with https://guestbook-api.mydomain.com/guestbook
1. Commit the modified file.
1. Your application now accesses the API through your custom domain.

## Remove resources
{: #serverless-api-webapp-cleanup}
{: step}

To delete the created {{site.data.keyword.cloudant_short_notm}} service,
1. Navigate to [resource list](https://{DomainName}/resources)
2. Under **Services**, click on the action menu next to `<yourinitials>-guestbook-db` service
3. Click **Delete**

To delete the API and actions from {{site.data.keyword.openwhisk_short}},

1. Navigate to [{{site.data.keyword.openwhisk_short}}](https://{DomainName}/functions/) landing page.
2. On the left pane, click on **APIs**.
3. Click on the **delete** icon in the `guestbook` API row and then **Delete** on the modal window.
4. On the left pane, click on **Actions**.
5. Under the `guestbook` package, delete all the actions by clicking on the **delete** icon in the respective action rows.

## Related content
{: #serverless-api-webapp-13}
* [Serverless Computing](https://www.ibm.com/cloud/learn/serverless)
* [More code patterns on serverless](https://developer.ibm.com/patterns/category/serverless/)
* [Getting started with {{site.data.keyword.openwhisk}}](https://{DomainName}/docs/openwhisk?topic=cloud-functions-getting-started)
* [{{site.data.keyword.openwhisk}} common use cases](https://{DomainName}/docs/openwhisk?topic=cloud-functions-use_cases)
* [Create REST APIs from actions](https://{DomainName}/docs/openwhisk?topic=cloud-functions-apigateway)
