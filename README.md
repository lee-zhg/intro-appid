# Secure Application deployed to Kubernetes Cluster with App ID Service in IBM Cloud


## Introduction

Application security can be incredibly complicated. For most developers, it's one of the hardest parts of creating an app. How can you be sure that you are protecting your users information? By integrating IBM Cloud™ App ID into your apps, you can secure resources and add authentication; even when you don't have a lot of security experience.

When deploying your application, you can consistently enforce policy-driven security by using the Ingress networking capability in IBM Cloud™ Kubernetes Service or OpenShift. With this approach, you can enable authorization and authentication for all of the applications in your cluster at the same time, **without ever changing your app code**!

IBM Cloud App ID is a cloud service that allows developers to easily add authentication and authorization capabilities to their applications while all the operational aspects of the service are handled by the IBM Cloud Platform. App ID is intended for developers that don't need or want to know anything about various security protocols. 

The following diagram to see the authentication flow.

  ![alt text](images/appid_architecture01.png)

1. A user opens your application and triggers a request to the web app or API.

1. For the API flow, the Ingress controller attempts to validate the supplied tokens. If the web flow is used, it kicks off a three-leg OIDC authentication process.

1. App ID begins the authentication process by displaying the Login Widget.

1. The user provides a username or email and password.

1. The Ingress controller obtains access and identity tokens from App ID for authorization.

1. Every request that is validated and forwarded by the Ingress Controller to your apps has an authorization header that contains the tokens.

By binding your instance of App ID to your cluster, you can enforce protection for all of the apps that run in your cluster.

This repository includes a sample application and instructions that you can deploy the application to Kubernetes cluster in IBM Cloud and security it with App ID service.

The instructions were adapted and extended based on repo https://github.ibm.com/robobob/appid-sample.


## Step 1 - Provision the AppID Service and Add Users

To provision an App ID service,

1. Login to IBM Cloud (https://cloud.ibm.com).

1. Navigate to `App ID` service (https://cloud.ibm.com/catalog/services/app-id).

1. Select a `Region`. For example, `Dallas`.

1. Select a `pricing plan`. `Lite plan` provides 1000 monthly events and 1000 authorized users. For `Graduated tier` plan, the first 1000 authentication events and first 1000 authorized users are free each month. \

1. Assign a `Service name`. All lower case in the `Service name` is recommended for consistentcy between this name and `Secure` name in Kubernete cluster.

  > Note: Take a note of `Service name`. You'll need it soon.

1. Accept the the `resource group` or select a different one.

1. Click `Create`.


## Step 2 - Configure the AppID Service and Add Users

Once provisioned, initiate the service and enable the cloud directory.

1. Select `Manage Authentication` in the left pane.

1. Make sure `Cloud Directory` feature is enabled.

    ![alt text](images/appid1.png)

1. Expand `Cloud Directory` in the left pane.

1. Select `Users` in the left pane.

1. Click `Create User` button in the right window.

1. Enter 

    * `First Name`
    * `Last Name`
    * `Email`
    * `Password` twice

1. Click `Save`.

1. Add a few more users.

    ![alt text](images/appid2.png)


## Step 3 - Bind App ID Service to your Kubernetes cluster

This step is not strictly neccessary to secure the application. However, the sample application in this repo does make a call to the AppID service. Therefore, we need to pull the credentials of AppID service into the Kubernetes cluster:

1. Open a new terminal ort command window.

1. Login to the IBM Cloud. When prompted, enter user name and password. 

    ```sh
    $ ibmcloud login
    ```

1. List the clusters.

    ```sh
    $ ibmcloud ks clusters
    ```

1. Connect to your Kubernetes cluster.

    ```sh
    $ eval $(ibmcloud ks cluster-config --cluster <your K8S cluster> --export | tee -a ~/.bash_profile) 
    ```
    >If the command has a syntax error in your terminal (e.g. windows cmd shell), you may instead run the command `ibmcloud ks cluster-config --cluster <your K8S cluster>`. Then, copy the output and execute it in the same terminal.

1. You should be able to use kubectl command to list kubernetes resources. Try getting the list of pods (there should be none yet)

    ```sh
    $ kubectl get pods
 
    No resources found.
    ```

1. Login to the registry service.

    ```sh
    ibmcloud cr login
    ```

. Optionally, you may add a new `name space` in registry to store your docker image if you need one.

  	```
	  $ ibmcloud  cr  namespace-add  appid_[your initial]

  	$ export CRNAMESPACE=appid_[your initial]
	  ```

1. Bind App ID to your Kubernetes cluster.

    ```
    $ ibmcloud ks cluster-service-bind --cluster <your K8S cluster> --namespace default --service <AppIDServiceName>

    Binding service instance to namespace...
    OK
    Namespace:    default
    Secret name:  binding-appid
    ```
    
    Replace `<ClusterName>` with the name of the Kubernetes cluster where the application will run and replace `<AppIDServiceName>` with the name of your AppID service.

1. Verify the App ID binding. The command below should return a secret name `binding-<appidservicename>`.

    ```
    $ kubectl get secrets | grep binding

    binding-appid                  Opaque                                1      6d21h
    ```

1. Take note of your scret name for App ID binding. You'll need it soon. In the above example, it is `binding-appid`. Yours will be slightly different.

  
## Step 4 - Build your Container Image

You need to build a container image for your application before deploying it to your Kubernetes cluster.

1. Download this repo in your terminal window. You may also download the repo zipfile.

    ```
    $ git clone https://github.com/lee-zhg/intro-appid

    $ cd intro-appid
    ```

1. Build the docker image and store in your IBM Cloud image registry.

    ```
    $ ibmcloud cr build -t us.icr.io/<namespace>/appidsample:1 .
    ```
    
    Where `<namespace>` is one of your container registry namespaces (you many use a different registry location, depending on which region your registry is in).


## Step 5 - Deploy the application

In the `yaml` subdirectory of this repository, there is a deployment.yaml file.  Before you deploy the application with it, you will need to make two changes.

1. In the terminal window, go to the directory where this repo was downloaded.

1. Navigate to yaml/ subfolder.

1. Open file deployment.yaml in your favor file editor.

  * Replace `<registry name space>` with your registry name space.

  * Replace `<binding secret of appid service>` with your binding secret name.

  * Save the changes.

1. Deploy the sample application to your Kubernetes cluster.

    ```
    $ kubectl create -f ./deployment.yaml
    ```

1. Verify the deployment.

    ```
    $ kubectl get pods | grep appid

    appidsample-6c5c4f4d6f-vt42g                1/1     Running   0          32h
    ```


## Step 6 - Deploy the service

To create the required service for the sample application. No changes to the yaml file is necessary.

1. In the `yaml` subdirectory, execute the command below to create the service 

    ```
    $ kubectl create -f ./service.yaml
    ```

1. Verify the service. Two services should be created.

  * appidsample-secure
  * appidsample-insecure

    ```
    $ kubectl get services | grep appid

    appidsample-insecure            ClusterIP      172.21.39.185    <none>           3000/TCP                        41h
    appidsample-secure              ClusterIP      172.21.183.48    <none>           3000/TCP                        41h
    ```


## Step 7 - Determine hostname for your sample application

AppID only works on web applications locked down with SSL and a fully qualified domain name. IKS and OpenShift clusters on IBM Cloud have a default wildcard certificate and secret you can use, as well as an Ingress subdomain to add to your application host name.

* If you are running an IKS cluster, type the following CLI command:

    ```
    ibmcloud ks cluster get --cluster <clustername> | grep Ingress
    ```

* Alternatively, if you are running an OpenShift cluster, type the following:

    ```
    ibmcloud oc cluster get --cluster <clustername> | grep Ingress
    ```

*   In both cases you will see something like the following:

    ```
    Ingress Subdomain:              <YOUR INGRESS SUBDOMAIN>
    Ingress Secret:                 <YOUR INGRESS SECRET>
    ```

* The `hostname` is the combination of string 

    * `appidsample.`
    * `Ingress subdomain`

    For example, `appidsample.lz-mycluster-paid-a08dc5e3ad7d218fff67b7b2d9460c79-0000.us-south.containers.appdomain.cloud`.


## Step 8 - Deploy the ingress

File `ingress.yaml` is provided in the `yaml` subdirectory. You can use the file to configure the `Ingress` of your Kubernetes cluster. The AppID ingress annotation will intercept all traffic heading to the `appidsample-secure` service, defined as path `/secure`.  All other traffic will be served by the `appidsample-insecure` service.  If you recall, you defined two services in the previous step.

You need to modify `ingress.yaml` file before deploying the `Ingress`.

1. Open `ingress.yaml` file in a file editor.

    ```
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: appidsample
      annotations:
        ingress.bluemix.net/appid-auth: bindSecret=<binding-appid-secret> namespace=default requestType=web     serviceName=appidsample-secure
        ingress.bluemix.net/redirect-to-https: "True"
    spec:
      rules:
      - host: <hostname>
        http:
          paths:
          - backend:
              serviceName: appidsample-secure
              servicePort: 3000
            path: /secure
          - backend:
              serviceName: appidsample-insecure
              servicePort: 3000
            path: /
      tls:
      - hosts:
        - <hostname>
          secretName: appidsample
    ```

1. Replace `<binding-appid-secret>` with the secret name of your App ID binding. If you forgot writting down the secret name, run command `kubectl get secrets | grep binding` in terminal window.

1. Replace `<hostname>` with the hostname value that you identied in the previous step.

1. Save the changes.

1. Deploy the ingress.

    ```
    $ kubectl create -f ./ingress.yaml

    ingress.extensions/appidsample created
    ```

1. Verify the `Ingress` deployment.

    ```
    $ kubectl get ingress | grep appid

    appidsample        appidsample.clustername.us-east.containers.mybluemix.net    169.48.5.14   80, 443   41h
    ```


## Step 9 - Add the Callback URL to App ID Service

To secure your sample application with App ID service, you need to register your sample application with your App ID instance. You do this by adding a unique IKS URL to your App ID configuration.

1. Go back to your App ID service instance in IBM Cloud.

1. Navigate to `Manage Authentication` --> `Authentication Settings`.

    ![alt text](images/appid6.png)

1. Your unique `Callback URL` is the combination of

    * String `https//`
    * `hostname` identified in the previous steps.
    * String `/secure/appid_callback`

    For example, `https://appidsample.lz-mycluster-paid-a08dc5e3ad7d218fff67b7b2d9460c79-0000.us-south.containers.appdomain.cloud/secure/appid_callback`.

1. Enter your `Callback URL` in the `Add web redirect URLs` field.

1. Click `+` icon at the end of the line.


## Step 10 - Test the sample application

Your sample application is deployed in your Kubernetes cluster in IBM Cloud and secured by your App ID service instance.

1. Access your sample application by entering `hostname` identified in previous steps in a browser. For example, `appidsample.lz-mycluster-paid-a08dc5e3ad7d218fff67b7b2d9460c79-0000.us-south.containers.appdomain.cloud`.

    > Note: You may not be able to test this sample application in `Chrome` browser.

1. The sample application home page is loaded. There is a `Login` button at the top-right corner.

    ![alt text](images/appid4.png)

1. Click the `Login` button, you should be redirected to the AppID login page.  

    ![alt text](images/appid_login.png)

1. After providing your credentials, you should be redirected back to the application, with your user information from AppID dumped to the screen.

    ![alt text](images/appid5.png)

1. Pressing the `Logout` button will log you out of the service.


## Step 11 - IBM Cloud Identity

IBM `Cloud Identity` provides identity-as-a-service for your users, including SSO, multifactor authentication, and user lifecycle management.

In the previous sections, you used the internal `Cloud Directory` included within the `IBM Cloud App ID` as a user repository. The `IBM Cloud App ID` authenticates your sample application based on user information stored in internal the `Cloud Directory`.

For the remaining sections of this repo, you are going to store user login information in `IBM Cloud Identity`. The `IBM Cloud App ID` will authenticate your sample application based on user information stored in `IBM Cloud Identity` system.

If you don't have access to a `IBM Cloud Identity` system, you may sign up a free trial.

1. Navigate to `IBM Marketplace` at https://www.ibm.com/us-en/marketplace.

1. Search for `Cloud Identity Connect`.

1. `IBM Cloud Identity - Overview - United States` (https://www.ibm.com/us-en/marketplace/cloud-identity?mhsrc=ibmsearch_a&mhq=Cloud%20Identity%20Connect) should appear at the top of the search result.

1. Click the `IBM Cloud Identity - Overview - United States` link to visit the `IBM Cloud Identity Overview` homepage.

    ![alt text](images/cloud_identity_overview.png)

1. Click the `Try free edition`.

1. Follow the online instruction to sign up a free trial of `IBM Cloud Identity`.

1. The Screen shot below is the `IBM Cloud Identity` dashboard.

    ![alt text](images/cloud_identity_dashboard.png)


## Step 12 - Add Users to IBM Cloud Identity User Repository

After you completed the signing up a free trial of `IBM Cloud Identity`, you can create testing user(s) in its `user repository`.

1. Click the main manu (three horizental lines) at the top-left corner and select `Users & groups`.

1. Click `Add user` button.

    ![alt text](images/cloud_identity_newuser01.png)

1. Enter required information for the new user.

    * `Identity Source`: `Cloud Directory`
    * `User name`: `a_user01`
    * `Given name`: `a_user01`
    * `Surname`: `Smith`
    * `Email`: <your email>

1. Click `Save`.s

1. The new user appears on the user list.

1. Select the `Action menu` (three dots) of the new user `a_user01` and choose `Reset password`.

1. Confirm the `Reset password` action when prompted. This should trigger an out-going email containing username, temporary password and a link to access your `IBM Cloud Identity` system.

1. Login to your email account and open the email.

1. Click the `IBM Cloud Identity` link.

1. After login with the temporary password, you should be prompted to enter new password twice.

1. The new user is ready by now.

    ![alt text](images/cloud_identity_newuser02.png)


## Step 13 - Download SAML Metadata File from IBM App ID

Now, you have an existing user repository in `IBM Cloud Identity`, and you want to authenticate your application via `IBM App ID`. This offers the integrated IBM Cloud experience.

You can set up App ID to use your enterprise identity provider, such as `IBM Cloud Identity` and `Microsoft Active Directory`, by using SAML2.0. By configuring App ID to work with your SAML IdP, your users are able to sign in by using enterprise credentials that they already know. To learn more about SAML 2.0 and see specific SAML identity provider examples. see [docs](https://cloud.ibm.com/docs/services/appid?topic=appid-enterprise#enterprise).

1. Navigate to your `IBM App ID` dashboard.

    ![alt text](images/cloud_identity_dashboard.png)

1. Expand the `Identity Providers` in the left pane and select `SAML 2.0 Federation`.

    ![alt text](images/appid_saml01.png)

1. Specify the name you'd like to use for the provider. For example, `IBM Cloud Identity`.

1. Click `Download SAML Metadata file` button.

1. Open the downloaded file and take note of the following information.

    * the `entityID` property under **EntityDescriptor** element.
    * the `Location` property under **AssertionConsumerService** element.

    ```
    <?xml version="1.0"?>
    <EntityDescriptor xmlns="urn:oasis:names:tc:SAML:2.0:metadata" entityID="urn:ibm:cloud:services:appid:8ba69d56-33f4-430a-8db6-a6bf090f1cf7">
        <SPSSODescriptor WantAssertionsSigned="true" protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
            <NameIDFormat>urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress</NameIDFormat>
            <AssertionConsumerService index="1" isDefault="true" Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://us-south.appid.cloud.ibm.com/saml2/v1/8ba69d56-33f4-430a-8db6-a6bf090f1cf7/login-acs"/>
        </SPSSODescriptor>
    </EntityDescriptor>
    ```


## Step 14 - Configure IBM Cloud Identity

To configure your `IBM Cloud Identity` system as the user repository of `IBM App ID`,

1. Navigate  to the `IBM Cloud Identity` dashboard.

1. Make sure your Cloud Identity instance has at least one user you'll be able to sign in with.

1. Click the main menu (three vertical lines at the top-left corner) and select `Applications`.

1. Click `Add application`.

1. Select `Custom Application` type.

1. Click `Add application` button at the bottom-right corner.

    ![alt text](images/clooud_identity_add_app01.png)

1. In the `General` tab, enter a name. For Example, `IBM App ID`.

1. Enter `IBM` in the `Company Name` field.

    ![alt text](images/clooud_identity_add_app02.png)

1. Go to the `Sign-on` tab.

1. Copy the entityID value from the `SAML Metadata file` in the previous section to the `Provider ID` filed.

1. Copy the `Location` value from the `SAML Metadata file` in the previous section to the `Assertion Consumer Service URL (HTTP-POST)` field.

1. Copy the `Location` value from the `SAML Metadata file` in the previous section to the `Server Provider SSO URL` field. Scroll down to locate the `Server Provider SSO URL` field.

    ![alt text](images/clooud_identity_add_app03.png)

1. Save your configuration.

1. Select checkbox of `All users are entitled to this application`.

    ![alt text](images/clooud_identity_add_app04.png)

1. Save your configuration again.

1. Switch back to the Sign-on tab. There are information on the right side of the screen.

    ![alt text](images/clooud_identity_add_app05.png)

1. In the next section, you will need the following information.

    * the `Provider ID` value on the right side of the screen.
    * the `Login URL` on the right side of the screen.
    * the `Signing Certificate` on the right side of the screen.


## - Step 15 Configure IBM App ID

Complete the steps below to configure `IBM App ID`.

1. Navigate to `IBM App ID` dashboard.

1. Expand the `Identity Providers` in the left pane and select `SAML 2.0 Federation`.

    ![alt text](images/clooud_identity_add_app06.png)

1. Copy the `Provider ID` value of `IBM Cloud Identity` (gathered at the end of the previous section) to the `entityID` field in the `IBM App ID` window. The `entityID` field is under `2. Provide metadata from SAML IdP` section on the right.

1. Copy the `Login URL` value of `IBM Cloud Identity` (gathered at the end of the previous section) to the `Sign-in` URL box in the `IBM App ID` window. The `Sign-in` field is under `2. Provide metadata from SAML IdP` section on the right.

1. Copy the `Signing Certificate` value of `IBM Cloud Identity` (gathered at the end of the previous section) to the `Primary Certificate` box in the `IBM App ID` window. The `Sign-in` field is under `2. Provide metadata from SAML IdP` section on the right.

    ![alt text](images/clooud_identity_add_app07.png)

1. Save your changes.

1. Click the `Test` button to see everything is working together.

    ![alt text](images/clooud_identity_add_app08.png)


## - Step 16 Verification

As you have tested before switching to `IBM Cloud Identity SaaS` for your user repository, your sample application was deployed to your Kubernetes cluster in IBM Cloud. Now, it's secured by your `IBM App ID` service and your `IBM Cloud Identity SaaS` instance.

1. Access your sample application by entering `hostname` identified in previous steps in a browser. For example, `appidsample.lz-mycluster-paid-a08dc5e3ad7d218fff67b7b2d9460c79-0000.us-south.containers.appdomain.cloud`.

    > Note: You may not be able to test this sample application in `Chrome` browser.

1. The sample application home page is loaded. There is a `Login` button at the top-right corner.

    ![alt text](images/appid4.png)

1. Click the `Login` button, you should be redirected to the AppID login page.  

    ![alt text](images/appid_login.png)

1. After providing your credentials from your `IBM Cloud Identity SaaS`, you should be redirected back to the application, with your user information from `IBM Cloud Identity SaaS` via `IBM AppID` dumped to the screen.

    ![alt text](images/appid5.png)

1. Pressing the `Logout` button will log you out of the service.


## Setting up IBM Cloud App ID with your Azure Active Directory

Information on how to use `Azure Active Directory` as user repository is available at https://www.ibm.com/cloud/blog/setting-ibm-cloud-app-id-azure-active-directory


## Additioal Notes on the Source Code

If you take a look at the server code, you will see that all secure operations hang off the `/secure` URL:

![alt text](images/appid7.png)

Everything hanging off `/secure` will be intercepted by AppID and checked to see if you are logged in and have access to the service.  The resulting call on the server side will contain an `Authorization` header in the request bearing the identity token of the logged in user.  The above sample code uses that token to retreive the client details from AppID.

To log out, IKS also supports an AppID logout URL. However, it always forces a redirect back to the AppID login page.  To avoid this, I added a server side logout function to remove the AppID cookies from the browser:

![alt text](images/appid8.png)

On the client side, the Javascript to perform logout looks like this:

![alt text](images/appid9.png)

Note the code comments.  If you have just enabled cloud directory, then you are fine.  However, if you have enabled a SAML provider and want to ensure your credentials are cleared there as well, you need to redirect the browser to the supplied `logout.html`.

This file uses a hidden frame to call the SAML provider secretly to remove your credentials:

![alt text](images/appid10.png)

You will need to change the URL on line 52 to the correct URL for your SAML provider.  The IBM SAML providers can be found here:

https://appid-iks-federated-logout.antona.us-south.containers.appdomain.cloud/test

Good hunting!


