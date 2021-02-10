#!groovy​

// Example template for webMethods API Gateway Devops pipeline
// Developed by John Carter (john.carter@softewareag.com)
// November 2018

// Run command on remote server, requires propagation of jenkins server key to remote server
def ssh(id, server, command) {
	sh("ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no $id@$server $command")
}

// run ant command on remote server
def ant(id, server, command) {
	ssh(id, server, "ant $command")	
}

// format user password into authorization string
def authString(user, password) {

	 def encoded = "${user}:${password}".bytes.encodeBase64().toString()

	 return "Basic $encoded"
}

// check if Integration Server (API Gateway) is running on remote server for given port
def isISRunningInRemoteServer(server, port) {
	
	CONTAINER_RUNNING=true;
	
	try {
		sh "curl -s -o /dev/null -w '%{http_code}' $server:$port"
		
		IS_RUNNING=true
		echo "OK"
	} catch (exc) {

		IS_RUNNING=false
		echo "KO"
	}
	
	return IS_RUNNING
}

// deploy API from one API Gateway to another via Promotion
// Remote gateway is configure via Stage properties.
def promoteAPI(apigwUrl, stage, apis) {

	apiString = null;

	apis.each{ a -> 

		if (apiString == null) {
			apiString = "[\"${a}\""
		} else {		
			apiString = apiString + ",\"${a}\""
		}

		setAPIMaturity(apigwUrl, a, "UAT")
	}

	apiString = apiString + "]"

	def body = """ {
		"description": "Tested APIS ${apis}",
		"name": "CDI-${BUILD_NUMBER}",
		"destinationStages": ["${stage}"],
		"promotedAssets": {
    		"api": ${apiString}
  		}
	}"""

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/promotion", 
				validResponseCodes: '200:201'
}

// publish the API to the given API Portal and community
// Determine id's for stage, portal and communitiy via API
def publishAPI(apigwUrl, stage, id, portalName, communityName) {

	if (stage != null) {

		response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/stages/" + stage, 
				validResponseCodes: '200'


		jsn = readJSON file: '', text: "${response.content}"

		//def url = jsn.stages[0].url;
		//def name = jsn.stages[0].name;

		// publish from API Gateway indicated by staging (NOT master!)
		url = jsn.stage.url;
		auth = jsn.stage.name;

	} else {

		url = apigwUrl;
		auth = "wm-apigateway"
	}

	portalId = getPortalId(url, auth,  portalName)
	communityId = getPortalCommunityId(url, auth, portalId, communityName)

	def body = """ {
		"portalGatewayId": "${portalId}",
		"communities": ["${communityId}"],
		"endpoints": ["${apigwUrl}/rad/${id}"]
	}"""

	println("Publishing API's via "+url+" from "+auth+" to " + portalName)

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: auth, 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${url}/rest/apigateway/apis/" + id + "/publish", 
				validResponseCodes: '200'
}

// Removes given API from API Portal
def unpublishAPI(apigwUrl, apiName, id, portalId, communityId) {

	def body = """ {
		"portalGatewayId": "${portalId}",
		"communities": ["${communityId}"]
	}"""

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/" + id + "/unpublish", 
				validResponseCodes: '200'

}

// Change maturity level for given API.
// maturity labels are set via extended property 'apiMaturityStatePossibleValues'

def setAPIMaturity(apigwUrl, id, maturity) {

   apiWrapper = getAPI(apigwUrl, id);

   println("Setting maturity to " + maturity + " for version " + apiWrapper.apiResponse.api.apiVersion);

    //pain, have to deactivate before update, really need option to only update attribute, NOT apiDefinition
    if (apiWrapper.apiResponse.api.isActive)
    	deactivateAPI(apigwUrl, id);
	
	// fixed in 10.4
	// bug in response, means the type is a string list, whereas it should be a string!!
	//apiWrapper.apiResponse.api.apiDefinition.type = apiWrapper.apiResponse.api.apiDefinition.type[0];

	raw = apiWrapper.apiResponse.api.apiDefinition.toString();

	def body = """{
		"maturityState": "${maturity}",
		"apiVersion": "${apiWrapper.apiResponse.api.apiVersion}",
  		"apiDefinition": ${raw}
	}"""

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/${id}", 
				validResponseCodes: '200:402'

	activateAPI(apigwUrl, id);

	return response.content;
}

def getPortalId(apigwUrl, auth, portalName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: auth, 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/portalGateways", 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.portalGatewayResponse.each { portal -> 
		
		if (portal.gatewayName == portalName) {

			id = portal.id;

			println("found id ${id} for portal name ${portalName}");
		}
	}
	
	return id;
}

def getPortalCommunityId(apigwUrl, auth, portalId, communityName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: auth, 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/portalGateways/communities?portalGatewayId="+portalId, 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.portalGatewayResponse.communities.portalCommunities.each { c -> 
		
		if (c.name == communityName) {

			id = c.id;

			println("found id ${id} for community name ${communityName}");
		}
	}
	
	return id;
}

def getStageId(apigwUrl, stageName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/stages", 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.stages.each { s -> 
		
		if (s.name == stageName) {

			id = s.id;

			println("found id ${id} for stage name ${stageName}");
		}
	}
	
	return id;
}

def getApplicationId(apigwUrl, appName) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/applications", 
				validResponseCodes: '200'

	def jsn = readJSON file: '', text: "${response.content}"

	def id = null;

	jsn.applications.each { a -> 
		
		if (a.name == appName) {

			id = a.id;

			println("found id ${id} for application name ${appName}");
		}
	}
	
	if (id == null)
		println("No app found for ${appName}");

	return id;
}

// Fetch API details from API Gateway
def getAPI(apigwUrl, id) {

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				requestBody: "", 
				url: "${apigwUrl}/rest/apigateway/apis/" + id, 
				validResponseCodes: '200'

	//if (raw) {
	//	return response.content;
	//} else {
		def jsn = readJSON file: '', text: "${response.content}"

		return jsn;
	//}
}

// Activate API for use
def activateAPI(apigwUrl, id) {

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/apis/" + id + "/activate", 
				validResponseCodes: '200'

}

// Deactivate API on API gateway, will no longer be available, i.e. produce 404 error code
def deactivateAPI(apigwUrl, id) {

	httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/apis/" + id + "/deactivate", 
				validResponseCodes: '200'

}

// fetch APIs with given name and maturity status
def queryAPIs(apigwUrl, apiName, maturityState) {

	if (maturityState != "") {
		key = "maturityState"
		value = maturityState
	} else {
		key = "apiName"
		value = apiName
	}

	def body = """ {
	  "types": [
	    "api"
	  ],
	  "condition": "or",
	  "scope": [
	    {
	      "attributeName": "${key}",
	      "keyword": "${value}"
	    }
	  ],
	  "responseFields": [
	    "apiName",
	    "id",
	    "name",
	    "apiVersion"
	  ],
	  "from": 0,
	  "size": -1
	}"""

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/search", 
				validResponseCodes: '200'

	println("query got back: "+response.content);

	def content = readJSON file: '', text: "${response.content}"

	return content;
}

// Link given API with application i.e. Application Key
def linkApiWithApp(apigwUrl, apiRef, appRef) {

	println("linking app with api "+apiRef);

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'GET', 
				ignoreSslErrors: true, 
				url: "${apigwUrl}/rest/apigateway/applications/" + appRef + "/apis", 
				validResponseCodes: '200:400'

	println("got back:" + response.content)

	def reqContent = readJSON file: '', text: "${response.content}"

	reqContent.apiIDs = reqContent.apiIDs << apiRef

	println("will resend "+reqContent.toString())

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: reqContent.toString(), 
				url: "${apigwUrl}/rest/apigateway/applications/" + appRef + "/apis", 
				validResponseCodes: '200'

	println("query got back: "+response.content);

	def content = readJSON file: '', text: "${response.content}"

	return content;
}

def deployNewAPIToAPIGateway(apigwUrl, apiName, repoUrl, repoUser, repoPassword, version) {

	def body = """{
		"apiName": "${apiName}",
  		"apiVersion": "${version}",
  		"apiGroups": ["Demo"],
		"maturityState": "Beta",
  		"type": "swagger",
  		"url": "${repoUrl}",
  		"authorizationValue": {
    		"keyName": "Authorization",
    		"value": "${authString(repoUser,repoPassword)}",
    		"type": "header"
  		}
	}"""

	print("body: "+body)

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis", 
				validResponseCodes: '200:201'

	return response.content;
}

// maturityState "Beta"
def replaceAPIForAPIGateway(apigwUrl, id, repoUrl, repoUser, repoPassword, version, maturityState) {

	def body = """{
		"apiVersion": "${version}",
		"maturityState": "${maturityState}",
		"apiGroups": ["Demo"],
  		"type": "swagger",
  		"url": "${repoUrl}",
  		"authorizationValue": {
    		"keyName": "Authorization",
    		"value": "${authString(repoUser,repoPassword)}",
    		"type": "header"
  		}
	}"""

	print("body: "+body)

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'PUT', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/${id}", 
				validResponseCodes: '200:201'

	return response.content;
}

// Duplicates existing API with new version reference
def createNewVersionForAPIForAPIGateway(apigwUrl, id, newVersion) {

	def body = """{
  		"newApiVersion": "${newVersion}",
  		"retainApplications": "true"
	}"""

	print("body: "+body)

	response = httpRequest acceptType: 'APPLICATION_JSON', 
				authentication: 'wm-apigateway', 
				contentType: 'APPLICATION_JSON', 
				httpMode: 'POST', 
				ignoreSslErrors: true, 
				requestBody: body, 
				url: "${apigwUrl}/rest/apigateway/apis/${id}/versions", 
				validResponseCodes: '200:201'

	return response.content;
}

// Deploys APIs found in git source directory to API Gateway
def deployAPIsFromGitHubToAPIGateway(apigwUrl, repoAccount, repo, repoUser, repoPassword, directory) {

	return deployAPIsToAPIGateway(apigwUrl, "https://raw.githubusercontent.com/${repoAccount}/${repo}/master/apis/", repoUser, repoPassword, directory)
}

// Deploys each API definition found in sub-directory '{directory}/src/apis' to API Gateway
// assumes that the name of the API is given in the first part of the filename postfixed with '-' e.g. "HelloWorld-api-1.0.swagger" where "HelloWorld" is the name of the API
def deployAPIsToAPIGateway(apigwUrl, swaggerEndPoint, swaggerUser, swaggerPassword, directory) {

	def dir = new File(directory)
	def refs = [];

	println("Will upload API definitions found in :"+dir.getAbsolutePath())

	def files = new File(dir, "src/apis").list()

	files.each { file -> 
		
		content = deployAPIToAPIGateway(apigwUrl, file.split('-')[0], swaggerEndPoint + file, swaggerUser, swaggerPassword, true)

		refs = refs << (content.apiResponse.api.id)
	}

	return refs;
}

// Deploys API at given end-point to API Gateway
def deployAPIToAPIGateway(apigwUrl, apiName, swaggerEndPoint, swaggerUser, swaggerPassword, replaceCurrentVersion) {

	def version = 1
	def newVersion = 1
	def apiRef = ""

	println("Processing API with name "+apiName)

	def results = queryAPIs(apigwUrl, apiName, "")
	
	if (results != null && results.api != null) {

		def lv = -1
		results.api.each { api ->

			def v = api.apiVersion ? api.apiVersion.toInteger() : 1

			if (v > lv) {

				println("current version is " + v)

				if (v == null)
					version = 1
				else
					version = api.apiVersion.toInteger()

				apiRef = api.id
				newVersion = version + 1
				lv = version
			}
		}
	}

	def data = null

	if (newVersion == 1) {

		println("Importing as new API "+apiName+":1 (Beta) via "+swaggerEndPoint)

		data = deployNewAPIToAPIGateway(apigwUrl, apiName, swaggerEndPoint, swaggerUser, swaggerPassword, version)

		def content = readJSON file: '', text: "${data}"

		println("Will now link with app " + API_TEST_APP);

		linkApiWithApp(apigwUrl, content.apiResponse.api.id, getApplicationId(apigwUrl, API_TEST_APP));


	} else if (replaceCurrentVersion) {

		println("Importing as new version of existing API "+apiName+":"+newVersion+" (Beta) - "+apiRef + "via " + swaggerEndPoint)
		
		data = createNewVersionForAPIForAPIGateway(apigwUrl, apiRef, newVersion)

		versionResponse = readJSON file: '', text: "${data}"
		apiRef = versionResponse.apiResponse.api.id

		data = replaceAPIForAPIGateway(apigwUrl, apiRef, swaggerEndPoint, swaggerUser, swaggerPassword, newVersion, "Beta")
	
	} else {

		println("Updating existing version of API "+apiName+":"+version+" (Beta) - "+apiRef)

		data = replaceAPIForAPIGateway(apigwUrl, apiRef, swaggerEndPoint, swaggerUser, swaggerPassword, version, "Beta")
	}

	def content = readJSON file: '', text: "${data}"

	return content;
}

// repoUrl - e.g. https://github.com/johnpcarter/api-deployment.git
// check out the APIs from github, will then reference this to determine what APIs to deploy to API Gateway
def checkoutAPIs(repoAccount, repo) {

	println("Checking out from ${repoAccount} and repo ${repo}")

	def repoUrl = "https://github.com/${repoAccount}/${repo}.git"

	checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'dev']], submoduleCfg: [], userRemoteConfigs: [[url: repoUrl]]])

	//checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'src']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "git-apis", url: repoUrl]]])
}

// Run test stun for given API, test stub needs to comply with name-space presented by ${TST_NAMESPACE}/${apiName}/${TST_POSTFIX}
def testAPI(apigwServer, testServer, apiRef) {

	apiWrapper = getAPI(apigwServer, apiRef)

	def api = apiWrapper.apiResponse.api.apiName.toLowerCase()
	def gatewayEndpoint = java.net.URLEncoder.encode(apiWrapper.apiResponse.gatewayEndPoints[0], "UTF-8")

	if (TST_POSTFIX != null) 
		testStub = "${testServer}${TST_NAMESPACE}${api}${TST_POSTFIX}"
	else 
		testStub = "${testServer}${TST_NAMESPACE}${api}"

	def url = "${testStub}/${gatewayEndpoint}"

	println("Using test stub at "+url)

	httpRequest acceptType: 'APPLICATION_JSON', 
					authentication: 'test-server', 
					contentType: 'APPLICATION_JSON', 
					httpMode: 'GET', 
					customHeaders: [[maskValue: false, name: 'api-key', value: API_TEST_APP]],
					ignoreSslErrors: true, 
					url: url, 
					validResponseCodes: '200'
}

pipeline {
	agent any
	environment {
		
		WORKSPACE='dev'
		GIT_ACCOUNT='johnpcarter'
		GIT_REPO='devops-demo'

		APIGW_SERVER='http://host.docker.internal:7777'
		
		TEST_SERVER='http://host.docker.internal:5555'
		TST_NAMESPACE= "/rest/jc/api/"
		TST_POSTFIX="_/test"

		APIPORTAL="default"
		APIPORTAL_COMMUNITY="Public Community"
		API_TEST_APP="TestApp"
		API_STAGE="UAT"
	}
	stages {
		stage('Prepare') {
			steps {
				script {
					
					def userInput = input(
						id: 'apiInput', message: 'Git Repository', parameters: [
							[$class: 'TextParameterDefinition', defaultValue: GIT_ACCOUNT, description: 'GIT Account', name: 'apiAccount'],
							[$class: 'TextParameterDefinition', defaultValue: GIT_REPO, description: 'GIT Repository', name: 'apiRepo'],
						])

					GIT_REPO=userInput['apiRepo']
					GIT_ACCOUNT=userInput['apiAccount']
					
					def esbInput = input(
						id: 'esbInput', message: 'API & Service Containers', parameters: [
							[$class: 'TextParameterDefinition', defaultValue: TEST_SERVER, description: 'API Runtime Container', name: 'esbServer'],
							[$class: 'TextParameterDefinition', defaultValue: APIGW_SERVER, description: 'webMethods API Gateway', name: 'apiServer'],
							[$class: 'TextParameterDefinition', defaultValue: API_STAGE, description: 'API Gatway Stage (optional)', name: 'apiStage'],
						])

					ESB_TEST_SERVER=esbInput['esbServer']
					APIGW_SERVER=esbInput['apiServer']
					API_STAGE=esbInput['apiStage'];

					def apiInput = input(
						id: 'apiInput', message: 'API Details', parameters: [
							[$class: 'TextParameterDefinition', defaultValue: API_TEST_APP, description: 'Test App Name', name: 'apiApp'],
							[$class: 'TextParameterDefinition', defaultValue: APIPORTAL, description: 'API Portal to Deploy to', name: 'apiPortal'],
							[$class: 'TextParameterDefinition', defaultValue: APIPORTAL_COMMUNITY, description: 'API Community', name: 'apiCommunity']
						])

					API_TEST_APP=apiInput['apiApp']
					APIPORTAL=apiInput['apiPortal']
					APIPORTAL_COMMUNITY=apiInput['apCommunity']

					cleanWs()

					checkoutAPIs(GIT_ACCOUNT, GIT_REPO)

					println("GIT ACCOUNT IS " + GIT_ACCOUNT);

				}
			}
		}
		stage('Deploy') {
			environment {
				REPO_CREDS = credentials('git-apis')
			}
			steps {
				script {
					
					TST_API_IDS = deployAPIsFromGitHubToAPIGateway(APIGW_SERVER, GIT_ACCOUNT, GIT_REPO, REPO_CREDS_USR, REPO_CREDS_PSW, "${WORKSPACE}")
				}
			}
		}
		stage('Test') {
			steps {
				input("Deployment Completed, Ready to Test?")
				script {

					PROD_API_IDS = []
					FAILED_API_IDS = []

					TST_API_IDS.each { ref ->

						try {
							println("Testing API "+ref)
							activateAPI(APIGW_SERVER, ref)
							
							// testAPI(APIGW_SERVER, TEST_SERVER, ref);
							
							println("Test Successful, setting maturity level to Test")
							setAPIMaturity(APIGW_SERVER, ref, "Test");

							PROD_API_IDS = PROD_API_IDS << ref;
					
						} catch (err) {

							println("Test for API "+ref+" failed with error: "+err)
							deactivateAPI(APIGW_SERVER, ref)

							FAILED_API_IDS = FAILED_API_IDS << ref
						}
					}
				}
			}
		}
		stage('Rollback') {
			when {
				expression { FAILED_API_IDS.size() > 0 }
			}
			steps {
				script {

					FAILED_API_IDS.each { ref -> 

						setAPIMaturity(APIGW_SERVER, ref, "Failed");

						println("Tests failed for API, will restore previous version: "+ref)

					}
				}
			}
		}
		stage('Promote') {
			when {
				expression { PROD_API_IDS.size() > 0 && API_STAGE != ""}
			}
			steps {
				input("Tests Completed, Ready to Promote to ${API_STAGE}?")
				script {
					print("Promoting Tested API's to UAT platform")

					promoteAPI(APIGW_SERVER, getStageId(APIGW_SERVER, API_STAGE), PROD_API_IDS)
				}
			}
		}
		stage('Publish') {
			when {
				expression { PROD_API_IDS.size() > 0 }
			}
			steps {
				input("UAT Promotion Completed, Ready to Publish?")
				script {
					print("Publishing API to API Portal")

					PROD_API_IDS.each{apiRef ->
						println("publi")
						if (API_STAGE != "") {
							publishAPI(APIGW_SERVER, getStageId(APIGW_SERVER, API_STAGE), apiRef, APIPORTAL, APIPORTAL_COMMUNITY)
						} else {
							publishAPI(APIGW_SERVER, null, apiRef, APIPORTAL, APIPORTAL_COMMUNITY)
						}
					}
				}
			}
		}
	}
}