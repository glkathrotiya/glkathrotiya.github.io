---


---

<hr>
<p>published: true<br>
title: Continuous deployment to multiple environment using Jenkins for .Net<br>
layout: post<br>
tags: [Jenkins, deployment, CI. CD]</p>
<hr>
<h1 id="continuous-deployment-to-multiple-environment-using-jenkins-for-.net">Continuous deployment to multiple environment using Jenkins for .Net</h1>
<p>One of the most common and repetitive task any agile team do is deploying code to multiple environment at the end of the sprint. So if you haven’t already automated your deployment pipeline then your team is likely spending way more time in deployment ceremony instead of solving actual business problems and also your process is more prone to human errors.</p>
<p>In this post i’m not discussing how to install Jenkins and related plugins from scratch since there is already lot written about that on internet and the process is fairly straight forward.</p>
<h2 id="here-is-what-we-want-to-accomplish">Here is what we want to accomplish,</h2>
<ul>
<li>Restore the nuget dependencies</li>
<li>Get latest from source control</li>
<li>Build and Package web application</li>
<li>Archive the package</li>
<li>Deploy to development server
<ul>
<li>Also Take site offline and show custom nice offline page while deployment is in progress</li>
</ul>
</li>
<li>Deploy to QA environment after the manual approval
<ul>
<li>Deploy with environment specific web config</li>
</ul>
</li>
<li>Send email notification of success or failure of the job</li>
</ul>
<h3 id="requirement">Requirement:</h3>
<p>Based on your application type and .Net framework you’re targeting you should have all the framework/library that your application needs to build successfully should be installed on your build (Jenkins) server.</p>
<p><strong>On Jenkins server,</strong></p>
<ul>
<li>
<p><a href="https://www.microsoft.com/en-us/download/details.aspx?id=48159">Install MSBuild</a></p>
</li>
<li>
<p><a href="https://www.iis.net/downloads/microsoft/web-deploy">Install WebDeploy exe</a></p>
</li>
<li>
<p><a href="https://www.microsoft.com/en-us/download/details.aspx?id=55168">Install .Net targetting pack</a></p>
</li>
</ul>
<p>Create Global crenential to use for SVN or other place where sign on is required</p>
<p><strong>On all web server</strong></p>
<ul>
<li>Enable IIS Management service from Windows feature</li>
<li>Install MS deploy :  select IIS deployment handler and Remote agent</li>
</ul>
<h2 id="jenkins-pipeline-script">Jenkins pipeline script</h2>
<pre class=" language-groovy"><code class="prism  language-groovy">node <span class="token punctuation">{</span>
 <span class="token keyword">def</span> svn_credentialId <span class="token operator">=</span> <span class="token string gstring">"2d7f5f55-6dd2-5t6y-8b98-3a22817b70e4"</span>
 <span class="token keyword">def</span> svn_project_root <span class="token operator">=</span> <span class="token string gstring">"https://example.com/svn/Repo/ToDoApp/trunk"</span>
 <span class="token keyword">def</span> slnPath <span class="token operator">=</span> <span class="token string gstring">"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>WORKSPACE<span class="token punctuation">}</span></span>\\trunk"</span>
 <span class="token keyword">def</span> slnName <span class="token operator">=</span> <span class="token string gstring">"ToDoApp.sln"</span>
 <span class="token keyword">def</span> projPath <span class="token operator">=</span> <span class="token string gstring">"ToDoApp"</span>
 <span class="token keyword">def</span> projName <span class="token operator">=</span> <span class="token string gstring">"ToDoApp.Web.csproj"</span>
 <span class="token keyword">def</span> MsBuildPath <span class="token operator">=</span> <span class="token string gstring">"C:\\Program Files (x86)\\MSBuild\\12.0\\Bin\\MSBuild.exe"</span>
 <span class="token keyword">def</span> MsDeployPath <span class="token operator">=</span> <span class="token string gstring">"C:\\Program Files\\IIS\\Microsoft Web Deploy V3\\msdeploy.exe"</span>
 <span class="token keyword">def</span> packagePath <span class="token operator">=</span> <span class="token string gstring">"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>projPath<span class="token punctuation">}</span></span>\\obj\\Release\\_PublishedWebsites\\ToDoApp.Web_Package\\ToDoApp.Web.zip"</span>
 <span class="token keyword">def</span> IISWebPath <span class="token operator">=</span> <span class="token string gstring">"ToDoApp"</span>
 <span class="token keyword">def</span> server_dev <span class="token operator">=</span> <span class="token string gstring">"DEV_Server"</span>
 <span class="token keyword">def</span> server_qa <span class="token operator">=</span> <span class="token string gstring">"QA_Server"</span>
 <span class="token keyword">def</span> server_admin_userName <span class="token operator">=</span> <span class="token string gstring">"domain\\userName"</span>
 <span class="token keyword">def</span> server_admin_pwd <span class="token operator">=</span> <span class="token string gstring">"SecretPassword"</span>
 <span class="token keyword">def</span> nuget_path <span class="token operator">=</span> <span class="token string gstring">"C:\\Nuget\\nuget.exe"</span>
 <span class="token keyword">def</span> set_param_QA <span class="token operator">=</span> <span class="token string gstring">"C:\\Deployment\\ToDoApp\\QA.ToDoApp.Web.SetParameters.xml"</span>

 <span class="token keyword">try</span> <span class="token punctuation">{</span>
  <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Checkout'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
   <span class="token function">checkout</span><span class="token punctuation">(</span><span class="token punctuation">[</span><span class="token punctuation">$</span><span class="token keyword">class</span><span class="token punctuation">:</span> <span class="token string">'SubversionSCM'</span><span class="token punctuation">,</span>
    additionalCredentials<span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token punctuation">,</span>
    excludedCommitMessages<span class="token punctuation">:</span> <span class="token string">''</span><span class="token punctuation">,</span>
    excludedRegions<span class="token punctuation">:</span> <span class="token string">''</span><span class="token punctuation">,</span>
    excludedRevprop<span class="token punctuation">:</span> <span class="token string">''</span><span class="token punctuation">,</span>
    excludedUsers<span class="token punctuation">:</span> <span class="token string">''</span><span class="token punctuation">,</span>
    filterChangelog<span class="token punctuation">:</span> <span class="token boolean">false</span><span class="token punctuation">,</span>
    ignoreDirPropChanges<span class="token punctuation">:</span> <span class="token boolean">false</span><span class="token punctuation">,</span>
    includedRegions<span class="token punctuation">:</span> <span class="token string">''</span><span class="token punctuation">,</span>
    locations<span class="token punctuation">:</span> <span class="token punctuation">[</span>
     <span class="token punctuation">[</span>credentialsId<span class="token punctuation">:</span> <span class="token string gstring">"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>svn_credentialId<span class="token punctuation">}</span></span>"</span><span class="token punctuation">,</span>
      depthOption<span class="token punctuation">:</span> <span class="token string">'infinity'</span><span class="token punctuation">,</span>
      ignoreExternalsOption<span class="token punctuation">:</span> <span class="token boolean">true</span><span class="token punctuation">,</span>
      remote<span class="token punctuation">:</span> <span class="token string gstring">"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>svn_project_root<span class="token punctuation">}</span></span>"</span>
     <span class="token punctuation">]</span>
    <span class="token punctuation">]</span><span class="token punctuation">,</span>
    workspaceUpdater<span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token punctuation">$</span><span class="token keyword">class</span><span class="token punctuation">:</span> <span class="token string">'UpdateUpdater'</span><span class="token punctuation">]</span>
   <span class="token punctuation">]</span><span class="token punctuation">)</span>
  <span class="token punctuation">}</span>

  <span class="token function">dir</span><span class="token punctuation">(</span>slnPath<span class="token punctuation">)</span> <span class="token punctuation">{</span>
   <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Nuget Restore'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>nuget_path<span class="token punctuation">}</span></span>\" config -set http_proxy=http://proxy.example.com:8080"</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>nuget_path<span class="token punctuation">}</span></span>\" config -set http_proxy.user=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_userName<span class="token punctuation">}</span></span>"</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>nuget_path<span class="token punctuation">}</span></span>\" config -set http_proxy.password=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_pwd<span class="token punctuation">}</span></span>"</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>nuget_path<span class="token punctuation">}</span></span>\" restore \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>slnPath<span class="token punctuation">}</span></span>\\${slnName}\" "</span>
   <span class="token punctuation">}</span>

   <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Build &amp; Package'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>MsBuildPath<span class="token punctuation">}</span></span>\" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>projPath<span class="token punctuation">}</span></span>\\${projName}\" /T:Build;Package /p:Configuration=RELEASE /p:OutputPath=\"obj\\Release\" /p:DeployIIsAppPath=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>IISWebPath<span class="token punctuation">}</span></span>\" /p:VisualStudioVersion=12.0"</span>
   <span class="token punctuation">}</span>

   <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Archive Artifacts'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    archiveArtifacts <span class="token string gstring">"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>packagePath<span class="token punctuation">}</span></span>"</span>
   <span class="token punctuation">}</span>

   <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Deployment to Dev'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>MsDeployPath<span class="token punctuation">}</span></span>\" -verb:sync -source:contentPath=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>slnPath<span class="token punctuation">}</span></span>\\${projPath}\\App_offline-template.htm\" -dest:contentPath=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>IISWebPath<span class="token punctuation">}</span></span>/App_offline.htm\",computerName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_dev<span class="token punctuation">}</span></span>,userName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_userName<span class="token punctuation">}</span></span>,passWord=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_pwd<span class="token punctuation">}</span></span> -allowUntrusted=true"</span>       
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>MsDeployPath<span class="token punctuation">}</span></span>\" -verb:sync -source:package=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>packagePath<span class="token punctuation">}</span></span>\" -dest:auto,computerName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_dev<span class="token punctuation">}</span></span>,userName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_userName<span class="token punctuation">}</span></span>,passWord=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_pwd<span class="token punctuation">}</span></span> -allowUntrusted=true -enablerule:AppOffline"</span>
   <span class="token punctuation">}</span>

   <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Approval - QA Deploy'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    input <span class="token string gstring">"Deploy to QA?"</span>
   <span class="token punctuation">}</span>

   <span class="token function">stage</span><span class="token punctuation">(</span><span class="token string">'Deployment to QA'</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>MsDeployPath<span class="token punctuation">}</span></span>\" -verb:sync -source:contentPath=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>slnPath<span class="token punctuation">}</span></span>\\${projPath}\\App_offline-template.htm\" -dest:contentPath=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>IISWebPath<span class="token punctuation">}</span></span>/App_offline.htm\",computerName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_qa<span class="token punctuation">}</span></span>,userName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_userName<span class="token punctuation">}</span></span>,passWord=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_pwd<span class="token punctuation">}</span></span> -allowUntrusted=true"</span>
    bat <span class="token string gstring">" \"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>MsDeployPath<span class="token punctuation">}</span></span>\" -verb:sync -source:package=\"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>packagePath<span class="token punctuation">}</span></span>\" -dest:auto,computerName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_qa<span class="token punctuation">}</span></span>,userName=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_userName<span class="token punctuation">}</span></span>,passWord=<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>server_admin_pwd<span class="token punctuation">}</span></span> -allowUntrusted=true -enablerule:AppOffline -setParamFile:<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>set_param_QA<span class="token punctuation">}</span></span>"</span>
   <span class="token punctuation">}</span>
  <span class="token punctuation">}</span>
  <span class="token function">notify</span><span class="token punctuation">(</span><span class="token string">'Success'</span><span class="token punctuation">)</span>
 <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">err</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token function">notify</span><span class="token punctuation">(</span><span class="token string gstring">"Error <span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>err<span class="token punctuation">}</span></span>"</span><span class="token punctuation">)</span>
  currentBuild<span class="token operator">.</span>result <span class="token operator">=</span> <span class="token string">'FAILURE'</span>
 <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

<span class="token keyword">def</span> <span class="token function">notify</span><span class="token punctuation">(</span>status<span class="token punctuation">)</span> <span class="token punctuation">{</span>
 <span class="token function">emailext</span><span class="token punctuation">(</span>to<span class="token punctuation">:</span> <span class="token string gstring">"gunvant.kathrotiya@bbumail.com"</span><span class="token punctuation">,</span>
  subject<span class="token punctuation">:</span> <span class="token string gstring">"<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>status<span class="token punctuation">}</span></span>: Deployment job '<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>JOB_NAME<span class="token punctuation">}</span></span> [<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>BUILD_NUMBER<span class="token punctuation">}</span></span>]'"</span><span class="token punctuation">,</span>
  body<span class="token punctuation">:</span> <span class="token string gstring">"&lt;p&gt;<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>status<span class="token punctuation">}</span></span>: Deployment Job '<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>JOB_NAME<span class="token punctuation">}</span></span> [<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>BUILD_NUMBER<span class="token punctuation">}</span></span>]':&lt;/p&gt; &lt;p&gt;Check console output at &lt;a href='<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>BUILD_URL<span class="token punctuation">}</span></span>'&gt;<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>JOB_NAME<span class="token punctuation">}</span></span> [<span class="token expression"><span class="token punctuation">$</span><span class="token punctuation">{</span>env<span class="token operator">.</span>BUILD_NUMBER<span class="token punctuation">}</span></span>]&lt;/a&gt;&lt;/p&gt;"</span>
  <span class="token punctuation">)</span>
<span class="token punctuation">}</span>
</code></pre>

