# Fabric8 online

This project generates the distribution of the [fabric8 online platform](https://fabric8.io/)

 <p align="center">
   <a href="http://fabric8.io/">
    <img src="https://raw.githubusercontent.com/fabric8io/fabric8/master/docs/images/cover/cover_small.png" alt="fabric8 logo"/>
   </a>
 </p>

## What's included?

The distribution currently builds two packages

### fabric8-dsaas - SaaS tools 

  - fabric8-ui
  - planner
  - forge SaaS - not yet included
  - bayesian - not yet included
  - elasticsearch
  - kibana
  - exposecontroller
  - configmapcontroller

### fabric8-team - developer tools

  - jenkins
  - che
  - content repository

## Running

### Remote 

To deploy fabric8 online to a remote cluster ensure your oc client is connnected to your remote cluster.

### Local 

To run fabric8 online locally we recommend using minishift
```
minishift start --vm-driver=xhyve --memory=6096 --cpus=2
```
If you know the IP address your minishift VM will use (you can get this after the VM has started with `minishift ip`) you can switch to use the more reliable nip.io for magic DNS
```
minishift start --vm-driver=xhyve --memory=6096 --cpus=2 --routing-suffix=192.168.64.151.nip.io
```
## Deploy
The fabric8 online distribution is versioned and released to maven central.  Using the scripts and commands below we will deploy the latest version in a new project called openshift-tennant, change the `$PRJ_NAME` value if you want a different project:
```
export PRJ_NAME=online-tennant
export ONLINE_VERSION=$(curl -L http://central.maven.org/maven2/io/fabric8/online/packages/fabric8-online-team/maven-metadata.xml | grep '<latest' | cut -f2 -d">"|cut -f1 -d"<")

oc new-project $PRJ_NAME
oc apply -f http://central.maven.org/maven2/io/fabric8/online/packages/fabric8-online-team/$ONLINE_VERSION/fabric8-online-team-$ONLINE_VERSION-openshift.yml
```
for now we have to update the Che configmap to use the che hostname so we can connect to the workspace:
```
oc get route che
```
edit the configmap
```
oc edit cm che
```
and replace `hostname-http:` value with the Che external hostname from the previous step

update the `externalName:` value for the Bayesian external SaaS service endpoint:
```
oc edit svc bayesian-link
```
Until we figure out the right roles for the pielines to create and edit environments run:
```
oc new-project $PRJ_NAME-staging
oc new-project $PRJ_NAME-production
oc adm policy add-cluster-role-to-user edit system:serviceaccount:$PRJ_NAME:jenkins --namespace $PRJ_NAME-staging
oc adm policy add-cluster-role-to-user view system:serviceaccount:$PRJ_NAME:jenkins --namespace $PRJ_NAME-production
oc adm policy add-cluster-role-to-user edit system:serviceaccount:$PRJ_NAME:jenkins --namespace $PRJ_NAME-staging
oc adm policy add-cluster-role-to-user view system:serviceaccount:$PRJ_NAME:jenkins --namespace $PRJ_NAME-production
oc project $PRJ_NAME
```
__minishift ONLY__

now use gofabric8 to change the PVCs to use the minishift VM host path to persist data and set extra permissions for Che
```
oc login -u system:admin
oc adm policy add-scc-to-user privileged -z che
gofabric8 volumes
oc login -u developer
```
retry the Che deployment as it will have failed first time without the SCC added:
```
oc deploy --retry che
```

### Wait for applications to run running
wait for the pods to start:
```
oc get pods -w
```
get the URLs to access an application:
```
oc get route
```

### Run an example quickstart:

Add a sample quickstart build config using the piepline strategy and start the build:
```
oc apply -f https://gist.githubusercontent.com/rawlingsj/eff84421af56b6825c6ff38c1646382e/raw/49bcf50b6872268665e9fe9279e8888a7b1ab8ab/spring-boot-webmvc-build-config.yml
oc start-build spring-boot-webmvc-jr
```
View the pipeline in Jenkins or the OpenShift console


