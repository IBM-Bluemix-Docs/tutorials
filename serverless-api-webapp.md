---
copyright:
  years: 2017, 2018
lastupdated: "2017-10-27"

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}


# Serverless web application and API

In this tutorial, you will create a serverless web application by hosting static website content on GitHub Pages and  implementing the application backend using Functions on IBM Cloud.

The application shown in this tutorial is a simple guestbook website where users can post messages.

## Objectives

* Deploy a serverless backend and database
* Expose a REST API
* Host a static website
* Optional: Use a custom domain for the REST API

## Products

This tutorial uses the following products:
   * [Cloudant NoSQL DB](https://console.bluemix.net/catalog/services/cloudantNoSQLDB)
   * [Cloud Functions](https://console.bluemix.net/openwhisk)

   ![](./images/solution8/Architecture.png)

1. The user access the application hosted in GitHub Pages.
2. The web application calls a backend API.
3. The backend API is defined in API Gateway.
4. API Gateway forwards the request to [Cloud Functions](https://console.bluemix.net/openwhisk).
5. The Cloud Functions actions use [Cloudant](https://console.bluemix.net/catalog/services/cloudantNoSQLDB) to store and retrieve guestbook entries.

## Before you begin
{: #prereqs}

This guide uses GitHub Pages to host the static website. Make sure you have a public GitHub account.

## Create the Guestbook database

Let's start by creating a Cloudant NoSQL Database. Cloudant NoSQL DB is a fully managed data layer designed for modern web and mobile applications that leverages a flexible JSON schema. Cloudant is built upon and compatible with Apache CouchDB and accessible through a secure HTTPS API, which scales as your application grows.

![](images/solution8/Catalog_Cloudant.png)

1. In the Catalog, under **Data & Analytics**, select **Cloudant NoSQL DB**.
2. Set the service name to **guestbook-db** and click **Create**.
3. Under **Service Credentials**, create **New credential** and click **Add**.
4. Click on **Manage** on the left and then **Launch** the Cloudant service console.
5. Create a database named **guestbook**.
   ![](images/solution8/Create_Database.png)

## Create Cloud Functions actions

In this section, you will create serverless actions (commonly termed as Functions). IBM Cloud Functions (based on Apache OpenWhisk) is a Function-as-a-Service (FaaS) platform which executes functions in response to incoming events and costs nothing when not in use.

![](images/solution8/Functions.png)

### Sequence of actions to save the guestbook entry

You will create a **sequence** which is a chain of actions where output of one action acts as an input to the following action and so on. The first sequence you will create is used to persist a guest message. Provided a name, an emailID and a comment, the sequence will:
   * Create a document to be persisted.
   * Store the document in the Cloudant NoSQL database.

Start by creating the first action, adding the action to a sequence and then adding the second action to the sequence.

1. Switch to **Functions** https://console.bluemix.net/openwhisk.
2. On the left pane, click on **Actions** and then **Create**.
3. **Create Action** with name `prepare-entry-for-save` and select **Node.js 6** as the Runtime.
4. Replace the existing code with the code snippet below and **Save**
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
5. Click on **Enclosing Sequences** and then **Add To Sequence**.
6. For Action name, enter `save-guestbook-entry-sequence` and then click **Create and Add**.
7. Click on **save-guestbook-entry-sequence** and then click **Add**.
8. Select **Use Public**, **Cloudant** and then choose **create-document** under **Actions**
9. Create **New Binding** and set name to `binding-for-guestbook`.
10. Select the **guestbook-db** Cloudant instance and the **guestbook** database and **Add** and then **Save**.
11. To test it, click on **Change Input** and enter the JSON below
    ```json
    {
    "doc":{
      "name": "John Smith",
      "email": "john@smith.com",
      "comment": "this is my comment"
    }
    }
    ```
    {: codeblock}
12. **Apply** and then **Invoke**.
    ![](images/solution8/Save_Entry_Invoke.png)

### Sequence of actions to retrieve entries

The second sequence is used to retrieve the existing guestbook entries. This sequence will:
   * List all documents from the database.
   * Format the documents and returning them.

1. Under **Functions**, **Create** a new Node.js action and name it **set-read-input**.
2. Replace the existing code with the code snippet below. This action passes the appropriate parameters to the next action.
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
3. Click on **Enclosing Sequences**, **Add to Sequence** and **Create New**
4. Enter `read-guestbook-entries-sequence` for the **Action Name** and click **Create and Add**.
5. Click on **read-guestbook-entries-sequence** sequence and then click **Add** to create and add the second action to get documents from Cloudant.
6. Under **Use Public**, choose **Cloudant** and then **list-documents**
7. Choose **binding-for-guestbook** and **Add** to create and add this public action to your sequence.
8. Click **Add** again to create and add the third action which will format the documents from Cloudant.
9. Under **Create New** enter `format-entries` for name and then click **Create and Add**.
10. Click on **format-entries** and replace the code with below and **Save**
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
11. Choose the sequence by clicking on **Actions** and then **read-guestbook-entries-sequence**.
12. Click on **Save** and then **Invoke**. The output should look like the following:
   ![](images/solution8/Read_Entries_Invoke.png)

## Create an API

![](images/solution8/Cloud_Functions_API.png)

1. Go to Actions https://console.bluemix.net/openwhisk/actions.
2. Select the **read-guestbook-entries-sequence** sequence. Under **Endpoints**, check **Enable Web Action** and **Save**.
3. Do the same for the **save-guestbook-entry-sequence** sequence.
4. Go to APIs https://console.bluemix.net/openwhisk/apimanagement and **Create a Cloud Functions API**
5. Set name to **guestbook** and base path to **/guestbook**
6. Create an operation to retrieve guestbook entries:
   1. Set **path** to **/entries**
   2. Set **verb** to **GET**
   3. Select the **read-guestbook-entries-sequence** action
7. Create an operation to persist a guestbook entry:
   1. Set **path** to **/entries**
   2. Set **verb** to **PUT**
   3. Select the **save-guestbook-entry-sequence** action
8. Save and expose the API.

## Deploy the web app

1. Fork the Guestbook user interface repository https://github.com/IBM-Cloud/serverless-guestbook to your public GitHub.
2. Modify **docs/guestbook.js** and replace the value of **apiUrl** with the route given by API Connect.
3. Commit the modified file to your forked repository.
4. In the Settings page of your repository, scroll to **GitHub Pages**, change the source to **master branch /docs folder** and Save.
5. Access the public page for your repository.
6. You should see the "test" guestbook entry created earlier.
7. Add new entries.

![](images/solution8/Guestbook.png)

## Optional: Use your own domain for the API

1. Create your domain under your organization https://console.bluemix.net/docs/admin/manageorg.html#managedomains.
2. Upload a SSL certificate for your domain and the subdomain you will use for the API.
3. Go to the Cloud Functions dashboard, select **APIs** and the Guestbook API.
4. Switch to **Definition**.
5. Set the **Domain for API** to the domain you added to your organization.
6. Set the **Subdomain for API** to **guestbook-api**
7. At this stage, you need to configure your DNS to create a CNAME mapping this subdomain to the IBM Cloud servers. Create a CNAME record for the domain targeting one of the following secure endpoints depending on which region hosts the target API:
   * US South: secure.us-south.bluemix.net.
   * United Kingdom: secure.eu-gb.bluemix.net.
   * Frankfurt: secure.eu-de.bluemix.net.
   * Sydney: secure.au-syd.bluemix.net.

   > Refer to https://console.bluemix.net/docs/apis/management/manage_apis.html#custom_domains for additional information
8. Save the API.
9. Wait for DNS to propagate and you will be able to access your guestbook api at https://guestbook-api.mydomain.com/guestbook.
10. Edit **docs/guestbook.js** and update the value of **apiUrl** with https://guestbook-api.mydomain.com/guestbook
11. Commit the modified file.
12. Your application now access the API through your custom domain.

## Related Content
* [More guides and samples on serverless](https://developer.ibm.com/code/journey/category/serverless/)
* [Getting started with Cloud Functions](https://console.bluemix.net/docs/openwhisk/index.html#getting-started-with-openwhisk)
* [Cloud Functions common use cases](https://console.bluemix.net/docs/openwhisk/openwhisk_use_cases.html#openwhisk_common_use_cases)
* [Create APIs from Cloud Functions actions](https://console.bluemix.net/docs/apis/management/manage_openwhisk_apis.html#manage_openwhisk_apis)
