#!groovy

// Run this pipeline on the custom Maven Slave ('maven-appdev')
// Maven Slaves have JDK and Maven already installed
// 'maven-appdev' has skopeo installed as well.
node('maven-appdev') {
  // Define Maven Command. Make sure it points to the correct
  // settings for our Nexus installation (use the service to
  // bypass the router). The file nexus_openshift_settings.xml
  // needs to be in the Source Code repository.
  def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

  // Checkout Source Code
  stage('Checkout Source') {
    git url: "https://github.com/marthenlt/openshift-tasks.git"
  }

  // The following variables need to be defined at the top level
  // and not inside the scope of a stage - otherwise they would not
  // be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  def version    = getVersionFromPom("pom.xml")

  // Set the tag for the development image: version + build number
  def devTag  = "${artifactId}-${version}-DEV"
  // Set the tag for the production image: version
  def prodTag = "${artifactId}-${version}"

  // Using Maven build the war file
  // Do not run tests in this step
  stage('Compiling the source and building war file') {
    echo "Building version ${devTag}"
    sh "${mvnCmd} clean package -DskipTests=true"
  }

  // Using Maven run the unit tests
  stage('Unit Tests') {
    echo "Running Unit Tests"
    sh "${mvnCmd} test"
  }

  // Using Maven call SonarQube for Code Analysis
  stage('Code Analysis') {
    echo "Running Code Analysis"
    sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-marthen-sonarqube.apps.na311.openshift.opentlc.com -Dsonar.login=c613ff09dbdf5aec220f4632e35e93fd831a29cf -Dsonar.projectName=${JOB_BASE_NAME}-${devTag}"
  }

  // Publish the built war file to Nexus
  stage('Publish to Nexus') {
    echo "Publish to Nexus"
    sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.marthen-nexus.svc.cluster.local:8081/repository/releases"
  }

  // Build the OpenShift Image in OpenShift and tag it.
  stage('Build and Tag OpenShift Image') {
    echo "Building OpenShift container image tasks:${devTag}"
    sh "oc start-build tasks --follow --from-file=./target/openshift-tasks.war -n marthen-tasks-dev"
    
    // Tag the image using the devTag
    openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: 'marthen-tasks-dev', namespace: 'marthen-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  // Deploy the built image to the Development Environment.
  stage('Deploy to Dev') {
    echo "Deploying container image to Development Project"
    
    // Update the Image on the Development Deployment Config
    sh "oc set image dc/tasks tasks=docker-registry.default.svc:5000/marthen-tasks-dev/tasks:${devTag} -n marthen-tasks-dev"
    
    // Update the Config Map which contains the users for the Tasks application
    sh "oc delete configmap tasks-config -n marthen-tasks-dev --ignore-not-found=true"
    sh "oc create configmap tasks-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n marthen-tasks-dev"
    
    openshiftDeploy depCfg: 'tasks', namespace: 'marthen-tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: 'tasks', namespace: 'marthen-tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'marthen-tasks-dev', svcName: 'tasks', verbose: 'false'
  }

  // Run Integration Tests in the Development Environment.
  stage('Integration Tests') {
    echo "Running Integration Tests"
    sh "oc project marthen-tasks-dev"
    echo "Adding tasks"
    //sh "curl --verbose -u tasks:redhat1 -H 'Content-Length: 0' -X POST http://$(oc get route tasks --template='{{ .spec.host }}')/ws/tasks/newtask123"
    sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.marthen-tasks-dev.svc.cluster.local:8080/ws/tasks/newtask123"
    
    echo "Retrieve tasks"
    //sh "curl --verbose -u tasks:redhat1 -H 'Content-Length: 0' -X GET http://$(oc get route tasks --template='{{ .spec.host }}')/ws/tasks/1"
    sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks.marthen-tasks-dev.svc.cluster.local:8080/ws/tasks/1"

    echo "Delete tasks"
    //sh "curl --verbose -u tasks:redhat1 -H 'Content-Length: 0' -X DELETE http://$(oc get route tasks --template='{{ .spec.host }}')/ws/tasks/1"
    sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.marthen-tasks-dev.svc.cluster.local:8080/ws/tasks/1"
  }

  // Copy Image to Nexus Docker Registry
  stage('Copy Image to Nexus Docker Registry') {
    echo "Copy image to Nexus Docker Registry"
    sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/marthen-tasks-dev/tasks:${devTag} docker://nexus-registry.marthen-nexus.svc.cluster.local:5000/tasks:${devTag}"

    // Tag the built image with the production tag.
    echo "Tag the build image with production tag"
    openshiftTag alias: 'false', destStream: 'tasks', destTag: prodTag, destinationNamespace: 'marthen-tasks-dev', namespace: 'marthen-tasks-dev', srcStream: 'tasks', srcTag: devTag, verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  // Do not activate the new version yet.
  def destApp   = "tasks-green"
  def activeApp = ""

  stage('Blue/Green Production Deployment') {
    activeApp = sh(returnStdout: true, script: "oc get route tasks -n marthen-tasks-prod -o jsonpath='{ .spec.to.name }'").trim()
    if (activeApp == "tasks-green") {
      destApp = "tasks-blue"
    }
    echo "Active Application:      " + activeApp
    echo "Destination Application: " + destApp

    // Update the Image on the Production Deployment Config
    sh "oc set image dc/${destApp} ${destApp}=docker-registry.default.svc:5000/marthen-tasks-dev/tasks:${prodTag} -n marthen-tasks-prod"

    // Update the Config Map which contains the users for the Tasks application
    sh "oc delete configmap ${destApp}-config -n marthen-tasks-prod --ignore-not-found=true"
    sh "oc create configmap ${destApp}-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n marthen-tasks-prod"

    // Deploy the inactive application.
    // Replace xyz-tasks-prod with the name of your production project
    openshiftDeploy depCfg: destApp, namespace: 'marthen-tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: destApp, namespace: 'marthen-tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'marthen-tasks-prod', svcName: destApp, verbose: 'false'
  }

  stage('Switch over to new Version') {
    // TBD
    echo "Switching Production application to ${destApp}."
    sh 'oc patch route tasks -n marthen-tasks-prod -p \'{"spec":{"to":{"name":"' + destApp + '"}}}\''
  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}

