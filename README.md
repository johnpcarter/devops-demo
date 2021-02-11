# devops-demo

## Import into Jenkins

After running and performing basic setup tasks (make sure to install all recommended plugins), click on "New Item".
Choose a name for your project and select "Multi-branch" pipeline to create your project.

Fill out the form as your wish but you will need to add this repository so that the provided Jenkins file will be used.
Under "**Branch Sources**", click "add source" and select "GitHub". Just paste the url of this repo into "Repository URL", namely "https://github.com/johnpcarter/devops-demo".

No credentials are required as this repository is public.

Click validate to to allow Jenkins to check that the URL is good and then scroll to the bottom of the page and click "save".

Jenkins will then pull and validate the Jenkins file.

### Build Setup

**In script Permissions**  

You will have to run 6 builds before success due to in script security alerts. On each failure click on the Jenkins pull down and under "manage jenkins" choose "in script proces approval"

You can also explicitly add them beforehand as below

*new java.io.File java.lang.String*  
*method java.io.File getAbsolutePath*  
*new java.io.File java.io.File java.lang.String*  
*method java.io.File list*  
*method groovy.lang.GString getBytes*  
*staticMethod org.codehaus.groovy.runtime.EncodingGroovyMethods encodeBase64 byte[]*  

**API Gateway Credentials**

You will also need to declare the credentials for your API Gateway that will receive API's and act to deploy and publish them to various regions. From the jenkins pull download menu under "manage jenkins" choose "manage credentials" then click on the "Jenkins" global domain, then "global credentials". Now you can click on "add credentials"

Enter the login and password associated with the API gateway and specify the id "**wm-apigateway**"

### API Gateway setup

Go to "Policies" tab and select "**Global policies**", then enable the global policy "Transaction Logging". This will ensure that the micro gateway will log all API transactions via the API Gateway's embedded ELK.

Create a test app called "TestApp" under the section "Apps", this is the app that will be auto assigned to any deployed API's.

Then create a promotion stage called "UAT", which will be used to deploy your API to your SaaS tenant. Click the "profile" pull down menu and select "Promotion Management". Then click "Add Stage".

Name the stage "UAT" and then click on the tab "Technical Information". From here paste the end point for your SaaS API Gateway and appropriate Administrator credentials.

Create a routing alias to your micro service hosted API. Click the "profile" pull download menu and select "Aliases", then click on the "Add Alias" button.
Make sure the name is set to "**API_HOST**", leave type as is and select "technical information" and type "**helloworld:5555**" into the default value, leaving stage as blank. This alias has already been set in the "host" parameter of the original swagger document.


You can now run your jenkins build to import an API, test and redeploy to your SaaS.

