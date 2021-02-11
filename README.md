# devops-demo

A demonstration of Software AG's webMethods API Management and Micro Service runtime, leverage Docker and Jenkins deploymen to show case a CI/CD API hybrid deployment from on-premise to cloud.

## Spin up local containers

Run the docker-compose command to start up the local containers on your machine 

$ docker-compose -f docker-compose.yaml up

This will start up the following containers;
 - Jenkins
 - mysql
 - helloworld (Simple API based on webMethods Micro Service Runtime)
 - webMethods micro gateway (Software AG's policity enforcement runtime for micro serviec side car operation)
 - webMethods API Gateway 10.5 (Software AG's API Mgmt portal)
 - webMethods API Gateway 10.7 
 
## API Gateway setup

Create a test app called "TestApp" under the section "Apps", this is the app that will be auto assigned to any deployed API's.

### Policy enforcement

Go to "Policies" tab and select "**Global policies**", then enable the global policy "Transaction Logging". This will ensure that the micro gateway will log all API transactions via the API Gateway's embedded ELK.

Create a second global policy by clicking on "Create global policy" and naming it "Basic Security", leave the filters and select the "policy" tab. Under the "Identify & Access" section, click "Identify & Authorize". On the right had side of the screen check "API Key" under "Identitfication Type". Click the "Save" button and activate the policy. This will mean that you will have to provide an API key as a minimum in order to invoke any API's.

### Routing & Promotion

Then create a promotion stage called "UAT", which will be used to deploy your API to your SaaS tenant. Click the "profile" pull down menu and select "Promotion Management". Then click "Add Stage".

Name the stage "UAT" and then click on the tab "Technical Information". From here paste the end point for your SaaS API Gateway and appropriate Administrator credentials.

Create a routing alias to your micro service hosted API. Click the "profile" pull download menu and select "Aliases", then click on the "Add Alias" button.
Make sure the name is set to "**API_HOST**", leave type as is and select "technical information" and type "**helloworld:5555**" into the default value, leaving stage as blank. This alias has already been set in the "host" parameter of the original swagger document.

If you want to be able to test the same API from your SaaS tenant then you will need to create an alternative end point for the above alias, ensuring that the host name or IP address refers to your local machine. You can do this from THIS gateway by repeating the above paragraph, but ensuring that you select your UAT stage in the "technical information" tab. Make sure that the alias name IS the same i.e. "**API_HOST**". This will ensure that the correct alias is promoted when deploying the API to your SaaS tenant. 

You can now setup your jenkins pipeline as below to import an API, test and redeploy to your SaaS.


## Jenkins Setup

Complete the following instructions after initial setup password (refer at docker log), creating your admin user and installing all recommended plugins.  
**NOTE:** project specific plugins have already included in the image.

#### API Gateway Credentials

You will also need to declare the credentials for your API Gateway that will receive API's and act to deploy and publish them to various regions. From the jenkins pull download menu under "manage jenkins" choose "manage credentials" then click on the "Jenkins" global domain, then "global credentials". Now you can click on "add credentials"

Enter the login and password associated with the API gateway and specify the id "**wm-apigateway**"

#### Import pipeline into Jenkins

From the jenkins home page, click "New Item", then enter a name and click "**Multibranch pipeline**".
Fill out the form as your wish but you will need to add this repository so that the provided Jenkins file will be used.
Under "**Branch Sources**", click "add source" and select "GitHub". Just paste the url of this repo into "Repository URL", namely "https://github.com/johnpcarter/devops-demo".

No credentials are required as this repository is public.

Click validate to to allow Jenkins to check that the URL is good and then scroll to the bottom of the page and click "save".

Jenkins will then pull and validate the Jenkins file. It may take some time due to github rate limiting, you can stop the indexing build if you wish and then rebuild again if it is taking too long.

**NOTE:** Running the pipeline requires user input in order to override certain parameters such api host address, ports etc and also steps asking whether to continue or not. These steps are included for demo purposes only. The default parameters should work and the pauses will allow you evaluate what each step does via the API Gateway portal.

### In script Permissions

You will get a permission error the first time you run the pipeline. You will need to update the in script permissions before the build will complete. To do click on the the Jenkins pull down and under "manage jenkins" choose "in script proces approval". This option only becomes available after running the pipeline at least once (doh!).

Unfortunately you will not be able to configure these rights in one go and you will have to add them one by one by running the pipeline 6 times!. For reference the required permissions are;

*new java.io.File java.lang.String*  
*method java.io.File getAbsolutePath*  
*new java.io.File java.io.File java.lang.String*  
*method java.io.File list*  
*method groovy.lang.GString getBytes*  
*staticMethod org.codehaus.groovy.runtime.EncodingGroovyMethods encodeBase64 byte[]*  

### Testing your API

**locally**  
You will be able to call the API once you have succeeded in running the pipeline past the deployment and test steps, try the following command from the command line to test, replacing the API key with the one for your app under API Gateway -> Applications -> TestApp.

   ```
   $ curl "http://localhost:7777/gateway/HelloWorld/1/v1/hello/john" \
     -H 'x-Gateway-APIKey: 7723bfc8-05d9-420e-bfd4-dd8ba40a128b' \
     -H 'Accept: application/json'
   ```
You will need to restart your micro-gateway in order to do the same test from your micro-gateway. The current instance will have no idea of your API as it was started before the API was registered with the gateway. Killing and restarting the container by re-running the docker-compose file will solve this.

  ```
  $ curl "http://localhost:4485/gateway/HelloWorld/1/v1/hello/john" \
     -H 'x-Gateway-APIKey: 7723bfc8-05d9-420e-bfd4-dd8ba40a128b' \
     -H 'Accept: application/json'
  ```
  
 **remotely**  
 If your pipeline has completed the promote step successfully they you shold be able to see your API in your SaaS tenant, in which case you can test your API via the SaaS end-point. However you will have to ensure that your local maching where you are hosting your API container is accessible via the alias **API_HOST** that you configured earlier.
