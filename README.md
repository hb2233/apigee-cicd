# CICD Demo - Apigee API Management Platform -v2
This repository includes the instructions and pipeline definition for CI/CD using Jenkins, Apigee Lint, Apickli, Cucumber Reports, Slack & Apigee Maven Deploy plugin on Apigee.

Often the most difficult and confusing aspect of application development is figuring out how to build a common framework for creating/deploying new applications. Over time, development teams have started using tools like Maven to automate some of these functions. This repository uses various tools/plugins for deploying Apigee bundles to the Edge platform.
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61174524-03a51d80-a5bf-11e9-8e66-c59da67cabd6.png?raw=true" alt="Pipeline"/>
</p>

<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61175386-170ab580-a5cc-11e9-9d24-ed0f80b7e769.png?raw=true" alt="Flow"/>
</p>

# Introduction
On every pipeline execution, the code goes through the following steps:
1. Develop an API Proxy in `test` environment Apigee Edge UI, Download the proxy and push to Github. 
2. In Jenkins, Apigee Proxy bundle is cloned from `Github`.
3. Code Analysis is done using `Apigee Lint`.
4. Any Javascript files from `apiproxy` directory goes through `Unit Tests` done using `mocha`.
5. Code coverage is done by `Istanbul/nyc` and reports are generated by `Cobertura`.
6. Using `edge.json` configurations is created/updated and a new revision is published in `prod` environment using `Apigee Maven Build Plugin`.
7. The newly deployed `prod` environment goes through Integration tests using `Apickli`.
8. Apickli produced `Cucumber Reports` are displayed in Jenkins.
9. If the test `FAILS`, current revision is `undeployed` and a stable revision is `re-deployed`.
10. Build `Success/Fail` notification along with `Cucumber reports` are sent to `Slack Room`.

# Prerequisites
* [Apigee Edge Account](https://login.apigee.com/login)
* [Slack Account + Jenkins Config](https://wiki.jenkins.io/display/JENKINS/Slack+Plugin)
* [Cucumber-Slack-Notifier](https://wiki.jenkins.io/display/JENKINS/Cucumber+Slack+Notifier+Plugin)
* [Cobertura Jenkins Plugin](https://wiki.jenkins.io/display/JENKINS/Cobertura+Plugin)
* NodeJS & NPM
* Configure Jenkins with Git, Cucumber Reports, JDK, Maven, NodeJS, Slack Plugins

# Demo Guide
1. HR API - A simple API to perform CRUD operations on employee records. For backend I am using Firestore DB.
2. Download `HR-API.zip` proxy bundle from this repo/bundles & deploy to `test` env or create an sample API Proxy.
3. Download `CiCd-Security.zip` proxy bundle from this repo/bundles & deploy to both `prod` & `test` environments.
4. Fork this repo & create a directory structure as per `HR-API` directory & place your `apiproxy` folder.
5. I am using an Parameterzied Build to pass the Apigee `username`, `password` and `base64encoded` string.
6. `ApigeeLint` will go through the `apiproxy` folder,
```node
apigeelint -s HR-API/apiproxy/ -f codeframe.js
```
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61175192-bfb71600-a5c8-11e9-823a-7dabf01bc4af.jpg?raw=true" alt="Apigeelint"/>
</p>

7. Unit test any custom code within the proxy like `Javascript` in our case. But it can be `NodeJS` as well.
```node
npm test test/unit/*.js
npm run coverage test/unit/*.js
```
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61175190-bf1e7f80-a5c8-11e9-9688-22b9deda550f.jpg?raw=true" alt="Mocha"/>
</p>
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61175191-bfb71600-a5c8-11e9-9fc6-33a56f3084d5.jpg?raw=true" alt="Istanbul"/>
</p>

8. Using `Cobertura Plugin` in try-catch-finally block to generate reports in Jenkins.
```
cd coverage && cp cobertura-coverage.xml $WORKSPACE
step([$class: 'CoberturaPublisher', coberturaReportFile: 'cobertura-coverage.xml'])
```
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61174970-6994a380-a5c5-11e9-9e56-6c52ddc1bde3.jpg?raw=true" alt="Cobertura"/>
  
9. Build & Deploy happens through Apigee Maven Plugin (update `pom` and `edge.json` files with appropiate details),
```maven
mvn -f HR-API/pom.xml install -Pprod -Dusername=${apigeeUsername} -Dpassword=${apigeePassword} -Dapigee.config.options=update
```
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61175010-022b2380-a5c6-11e9-9fb2-711c41232850.jpg?raw=true" alt="Maven"/>
  
10. Integration tests happen through Apickli - Cucumber - Gherkin Tests,
```javascript
cd $WORKSPACE/test/integration && npm install
cd $WORKSPACE/test/integration && npm test
```

11. Cucumber Reports plugin in Jenkins will use the `reports.json` file to create HTML Reports & statistics.
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61174977-77e2bf80-a5c5-11e9-833c-2e86f69a0598.jpg?raw=true" alt="Cucumber-Reports"/>
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61174974-774a2900-a5c5-11e9-8b8a-33f3c4668254.jpg?raw=true" alt="Cucumber-Reports"/>

12. If Integration tests fail, then through a `undeploy.sh` shell script I am undoing _Step 8_. Through Jenkins Environment variable I am getting the current deployed revision and storing it as `Stable revision`. Within Shell Script I am using this value to re-deploy in case of Failure.
```javascript
curl -X DELETE --header "Authorization: Basic $base64encoded" "https://api.enterprise.apigee.com/v1/organizations/$org_name/environments/$env_name/apis/$api_name/revisions/$rev_num/deployments"
curl -X DELETE --header "Authorization: Basic $base64encoded" "https://api.enterprise.apigee.com/v1/organizations/$org_name/apis/$api_name/revisions/$rev_num"
curl -X POST --header "Content-Type: application/x-www-form-urlencoded" --header "Authorization: Basic $base64encoded" "https://api.enterprise.apigee.com/v1/organizations/$org_name/environments/$env_name/apis/$api_name/revisions/$stable_revision/deployments"
```
13. To send `cucumber-reports` to `Slack` I used cucumber-slack-notifier, but the pipeline cmd is not working as expected/documented. So for the time being I am running a separate `FreeStyle project >> Build >> Send Cucumber Report to Slack` and point it to the reports.json in this pipeline directory.
```
build job: 'cucumber-report'
```
14. When Build Starts/Ends & At any step if a Failure occurs, a notification is sent to Slack Room along with cucumber reports.
<p align="center">
  <img src="https://user-images.githubusercontent.com/28925814/61175129-d4df7500-a5c7-11e9-8fb1-3ff6fa7307be.jpg?raw=true" alt="Slack"/>

# Things to do in upcoming versions
1. Update Developer Portal.
    * After successful `Integration Test`, we can add another Stage to `Update Developer Portal Docs`. 
    * Currently we have plugin/apis for updating `Apigee Drupal based portal`.
    * We do not have any APIs for updating `Apigee Integrated Developer Portal` as of 14th July 2019.
2. Add `Performace/Load Tests` after `Integrated Tests`
3. Use Git Branches/Projects for `dev >> uat >> prod` environments and use Jenkins to Merge and Commit updates.


# References
1. [Apigee - maven-jenkins-ci-demo](https://github.com/apigee/maven-jenkins-ci-demo)
