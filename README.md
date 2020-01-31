# Secure Application Deployed to Kubernetes Cluster with App ID Service in IBM Cloud


## Introduction

Application security can be incredibly complicated. For most developers, it's one of the hardest parts of creating an app. How can you be sure that you are protecting your users information? By integrating IBM Cloud™ App ID into your apps, you can secure resources and add authentication; even when you don't have a lot of security experience.

When deploying your application, you can consistently enforce policy-driven security by using the Ingress networking capability in IBM Cloud™ Kubernetes Service or OpenShift. With this approach, you can enable authorization and authentication for all of the applications in your cluster at the same time, **without ever changing your app code**!

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


## Step 1 - Provision the AppID Service and Add Users

To provision an App ID service,

1. Login to IBM Cloud (https://cloud.ibm.com).

1. Navigate to `App ID` service (https://cloud.ibm.com/catalog/services/app-id).

1. Select a `Region`. For example, `Dallas`.

1. Select a `pricing plan`. `Lite plan` provides 1000 monthly events and 1000 authorized users. For `Graduated tier` plan, the first 1000 authentication events and first 1000 authorized users are free each month. \

1. Assign a `Service name`. All lower case in the `Service name` is recommended for consistentcy between this name and `Secure` name in Kubernete cluster.

  > Note: Take a note of `Service name`. You'll need it later.

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

1. Binding App ID to your Kubernetes cluster.

    ```
    $ ibmcloud ks cluster-service-bind --cluster <your K8S cluster> --namespace default --service <AppIDServiceName>

    Binding service instance to namespace...
    OK
    Namespace:    default
    Secret name:  binding-appid
    ```
    
    Replace `<ClusterName>` with the name of the Kubernetes cluster where the application will run and replace `<AppIDServiceName>` with the name of your AppID service.

1. Verify the binding. The command below should return a secret name `binding-<appidservicename>`.

    ```
    $ kubectl get secrets | grep binding

    binding-appid                  Opaque                                1      6d21h
    ```

  
## Step 4 - Build your Container Image

You need to build a container image for your application before deploying it to your Kubernetes cluster.

1. Download this repo in your terminal window. You may also download the repo zipfile.

    ```
    git clone https://github.ibm.com/lee-zhg/appid-intro

    cd appid-intro
    ```

After pulling this repository down into your environment, build the image directly into your IBM Cloud image repository:

```
 ibmcloud cr build -t us.icr.io/<namespace>/appidsample:1 .
```
Where `<namespace>` is one of your container registry namespaces (you many also need to update the registry location, depending on which region your registry is in).

## Step 4 - Create certificates and secrets

AppID only works on web applications locked down with SSL, so you will need to create certificates and a secret.  This repository contains two scripts, `cert.sh` and `secret.sh` (in that order) to help you with that.  

When you create your certificates, make sure you use the fully qualified hostname that includes your ingress subdomain, as per the following example:

```
[root@jumpserver appid-sample]# ./cert.sh
Generating a 2048 bit RSA private key
................................................+++
.....................+++
writing new private key to 'appid-key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CA
State or Province Name (full name) []:ON
Locality Name (eg, city) [Default City]:TO
Organization Name (eg, company) [Default Company Ltd]:IBM
Organizational Unit Name (eg, section) []:Cloud
Common Name (eg, your name or your server's hostname) []:appidsample.clustername.us-east.containers.mybluemix.net 
Email Address []:
[root@jumpserver appid-sample]#
```

You can find your ingress subdomain on your cluster overview page in IBM Cloud:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid3.png)

## Step 5 - Deploy the application

In the `yaml` subdirectory of this repository there is a deployment.yaml file.  Before you deploy the application with it, you will need to make two changes.

On line 20, change the image namespace from `us.icr.io/<namespace>/appidsample:1` to whatever namespace you built the image to.

On line 27, change `binding-appid` to the AppID secret name for you environment.

After you make these changes, deploy the application by typing `kubectl create -f ./deployment.yaml`

Check to make sure the pod is up and running:

```
[root@jumpserver appid-sample]# kubectl get pods | grep appid
appidsample-6c5c4f4d6f-vt42g                1/1     Running   0          32h
[root@jumpserver appid-sample]#
```

## Step 6 - Deploy the service

In the `yaml` subdirectory, create the service with `kubectl create -f ./service.yaml`.  No changes to the yaml file is required.

When you check the service, you will actually see two services:

```
[root@jumpserver appid-sample]# kubectl get services | grep appid
appidsample-insecure            ClusterIP      172.21.39.185    <none>           3000/TCP                        41h
appidsample-secure              ClusterIP      172.21.183.48    <none>           3000/TCP                        41h
[root@jumpserver appid-sample]#
```

## Step 7 - Deploy the ingress

Let's have a look at the `ingress.yaml` file in the `yaml` subdirectory:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: appidsample
  annotations:
    ingress.bluemix.net/appid-auth: bindSecret=binding-appid namespace=default requestType=web serviceName=appidsample-secure
    ingress.bluemix.net/redirect-to-https: "True"
spec:
  rules:
  - host: appidsample.clustername.us-east.containers.mybluemix.net
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
    - appidsample.clustername.us-east.containers.mybluemix.net
    secretName: appidsampl
```

The AppID ingress annotation will intercept all traffic heading to the `appidsample-secure` service, defined as path `/secure`.  All other traffic will be serviced by the `appidsample-insecure` service.  That's why we defined two services.

Before deploying, change lines 10 and 23 to your fully qualified domain name, which must be the same name you used to create the certificate during step 4

After updating the file, deploy the ingress with `kubectl create -f ./ingress.yaml`

Checking your ingress, you should see something like the following:

```
[root@jumpserver appid-sample]# kubectl get ingress | grep appid
appidsample        appidsample.clustername.us-east.containers.mybluemix.net    169.48.5.14   80, 443   41h
[root@jumpserver appid-sample]#
```

## Step 8 - Add the Callback URL to AppID

You need to register your application with your AppID instance.  You do this by adding a unique IKS URL to your AppID config.

Go back to your AppID instance and select Manage Authentication-->Authentication Settings:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid6.png)


Add the following web direct URL to the configuration, substituting in your fully qualfied hostname:

```
https://appidsample.clustername.us-east.containers.mybluemix.nea/secure/appid_callback 
```

## Step 9 - Test the application

At this point you should be able to surf to the application using the ingress host name:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid4.png)

When you press the `Login` button, you should be redirected to the AppID login page.  After providing your credentials, you should be redirected back to the application, with your user information from AppID dumped to the screen:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid5.png)

Pressing the `Logout` button will log you out of the service.


## Notes

If you take a look at the server code, you will see that all secure operations hang off the `/secure` URL:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid7.png)

Everything hanging off `/secure` will be intercepted by AppID and checked to see if you are logged in and have access to the service.  The resulting call on the server side will contain an `Authorization` header in the request bearing the identity token of the logged in user.  The above sample code uses that token to retreive the client details from AppID.

To log out, IKS also supports an AppID logout URL. However, it always forces a redirect back to the AppID login page.  To avoid this, I added a server side logout function to remove the AppID cookies from the browser:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid8.png)

On the client side, the Javascript to perform logout looks like this:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid9.png)

Note the code comments.  If you have just enabled cloud directory, then you are fine.  However, if you have enabled a SAML provider and want to ensure your credentials are cleared there as well, you need to redirect the browser to the supplied `logout.html`.

This file uses a hidden frame to call the SAML provider secretly to remove your credentials:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid10.png)

You will need to change the URL on line 52 to the correct URL for your SAML provider.  The IBM SAML providers can be found here:

https://appid-iks-federated-logout.antona.us-south.containers.appdomain.cloud/test

Good hunting!


