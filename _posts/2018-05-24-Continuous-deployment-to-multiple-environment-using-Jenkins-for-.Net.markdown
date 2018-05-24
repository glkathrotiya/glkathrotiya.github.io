---
published: true
title: Continuous deployment to multiple environment using Jenkins for .Net
layout: post
tags: [Jenkins, deployment, CI. CD]

---
One of the most common and repetitive task any agile team do is deploying code to multiple environment at the end of the sprint. So if you haven't already automated your deployment pipeline then your team is likely spending way more time in deployment ceremony instead of solving actual business problems and also your process is more prone to human errors.

In this post i'm not discussing how to install Jenkins and related plugins from scratch since there is already lot written about that on internet and the process is fairly straight forward.

## Here is what we want to accomplish,
- Restore the nuget dependencies
- Get latest from source control
- Build and Package web application
- Archive the package
- Deploy to development server
	- Also Take site offline and show custom nice offline page while deployment is in progress
- Deploy to QA environment after the manual approval 	
	- Deploy with environment specific web config
- Send email notification of success or failure of the job	


### Requirement:

Based on your application type and .Net framework you're targeting you should have all the framework/library that your application needs to build successfully should be installed on your build (Jenkins) server.

**On Jenkins server,**
- [Install MSBuild](https://www.microsoft.com/en-us/download/details.aspx?id=48159)

- [Install WebDeploy exe](https://www.iis.net/downloads/microsoft/web-deploy)

- [Install .Net targetting pack](https://www.microsoft.com/en-us/download/details.aspx?id=55168)


Create Global crenential to use for SVN or other place where sign on is required

**On all web server**
- Enable IIS Management service from Windows feature
- Install MS deploy :  select IIS deployment handler and Remote agent

## Jenkins pipeline script
```groovy
node {
 def svn_credentialId = "2d7f5f55-6dd2-5t6y-8b98-3a22817b70e4"
 def svn_project_root = "https://example.com/svn/Repo/ToDoApp/trunk"
 def slnPath = "${WORKSPACE}\\trunk"
 def slnName = "ToDoApp.sln"
 def projPath = "ToDoApp"
 def projName = "ToDoApp.Web.csproj"
 def MsBuildPath = "C:\\Program Files (x86)\\MSBuild\\12.0\\Bin\\MSBuild.exe"
 def MsDeployPath = "C:\\Program Files\\IIS\\Microsoft Web Deploy V3\\msdeploy.exe"
 def packagePath = "${projPath}\\obj\\Release\\_PublishedWebsites\\ToDoApp.Web_Package\\ToDoApp.Web.zip"
 def IISWebPath = "ToDoApp"
 def server_dev = "DEV_Server"
 def server_qa = "QA_Server"
 def server_admin_userName = "domain\\userName"
 def server_admin_pwd = "SecretPassword"
 def nuget_path = "C:\\Nuget\\nuget.exe"
 def set_param_QA = "C:\\Deployment\\ToDoApp\\QA.ToDoApp.Web.SetParameters.xml"

 try {
  stage('Checkout') {
   checkout([$class: 'SubversionSCM',
    additionalCredentials: [],
    excludedCommitMessages: '',
    excludedRegions: '',
    excludedRevprop: '',
    excludedUsers: '',
    filterChangelog: false,
    ignoreDirPropChanges: false,
    includedRegions: '',
    locations: [
     [credentialsId: "${svn_credentialId}",
      depthOption: 'infinity',
      ignoreExternalsOption: true,
      remote: "${svn_project_root}"
     ]
    ],
    workspaceUpdater: [$class: 'UpdateUpdater']
   ])
  }

  dir(slnPath) {
   stage('Nuget Restore') {
    bat " \"${nuget_path}\" config -set http_proxy=http://proxy.example.com:8080"
    bat " \"${nuget_path}\" config -set http_proxy.user=${server_admin_userName}"
    bat " \"${nuget_path}\" config -set http_proxy.password=${server_admin_pwd}"
    bat " \"${nuget_path}\" restore \"${slnPath}\\${slnName}\" "
   }

   stage('Build & Package') {
    bat " \"${MsBuildPath}\" \"${projPath}\\${projName}\" /T:Build;Package /p:Configuration=RELEASE /p:OutputPath=\"obj\\Release\" /p:DeployIIsAppPath=\"${IISWebPath}\" /p:VisualStudioVersion=12.0"
   }

   stage('Archive Artifacts') {
    archiveArtifacts "${packagePath}"
   }

   stage('Deployment to Dev') {
    bat " \"${MsDeployPath}\" -verb:sync -source:contentPath=\"${slnPath}\\${projPath}\\App_offline-template.htm\" -dest:contentPath=\"${IISWebPath}/App_offline.htm\",computerName=${server_dev},userName=${server_admin_userName},passWord=${server_admin_pwd} -allowUntrusted=true"       
    bat " \"${MsDeployPath}\" -verb:sync -source:package=\"${packagePath}\" -dest:auto,computerName=${server_dev},userName=${server_admin_userName},passWord=${server_admin_pwd} -allowUntrusted=true -enablerule:AppOffline"
   }

   stage('Approval - QA Deploy') {
    input "Deploy to QA?"
   }

   stage('Deployment to QA') {
    bat " \"${MsDeployPath}\" -verb:sync -source:contentPath=\"${slnPath}\\${projPath}\\App_offline-template.htm\" -dest:contentPath=\"${IISWebPath}/App_offline.htm\",computerName=${server_qa},userName=${server_admin_userName},passWord=${server_admin_pwd} -allowUntrusted=true"
    bat " \"${MsDeployPath}\" -verb:sync -source:package=\"${packagePath}\" -dest:auto,computerName=${server_qa},userName=${server_admin_userName},passWord=${server_admin_pwd} -allowUntrusted=true -enablerule:AppOffline -setParamFile:${set_param_QA}"
   }
  }
  notify('Success')
 } catch (err) {
  notify("Error ${err}")
  currentBuild.result = 'FAILURE'
 }
}

def notify(status) {
 emailext(to: "gunvant.kathrotiya@bbumail.com",
  subject: "${status}: Deployment job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
  body: "<p>${status}: Deployment Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p> <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>"
  )
}
```
