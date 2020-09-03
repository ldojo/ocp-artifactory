### Spin up a vanilla persistent Artifactory instance in OCP4

```
oc process -f artifactory-template.yml -p NAME=ear -p IMAGE=operator-dev-3rdpartymanual-local.artifactrepo-dev.nam.nsroot.net/artifactory-pro:6.20.1 | oc create -f -
# a Deployment, Service, PVC, and Route will be created, each named "ear"
# The PVC will be 75Gi 
```
### Tear it down
```
oc process -f artifactory-template.yml -p NAME=ear -p IMAGE=operator-dev-3rdpartymanual-local.artifactrepo-dev.nam.nsroot.net/artifactory-pro:6.20.1 | oc delete -f -
```


### Spin up a persistent Artifactory with "pre-baked" configuration

If you need to spin up an instance of Artifactory from an image that would already contain the artifactory configuration you want, this process can be followed:

In an instance of Artifactory that is up and running in OCP, and is already configured with all of the changes you need, export the /opt/jfrog/artifactory/etc/artifactory.config.latest.xml in that artifactory container and save it

You can then build a new Artifactory Docker image with that configuration baked in. Here is the contents of a Dockerfile for such an image:

```
FROM operator-dev-3rdpartymanual-local.artifactrepo-dev.nam.nsroot.net/artifactory-pro:6.20.1 

COPY artifactory.config.xml /artifactory_extra_conf/artifactory.config.xml
```

Assuming the artifactory.config.xml file and Dockerfile are in the current directory, build the image:

```
podman build -t your-repo:port/image:tag .
```

push it to ```your-repo:port``` 

and deploy:

```
oc process -f artifactory-template.yml -p NAME=[name] -p IMAGE=your-repo:port/image:tag | oc create -f -
```

In your new Artifactory deployment, you can then go to the Artifactory Admin UI console, and verify that your configuration is the same. The artifactory.config.xml config file encompasses most of Artifactory's configuration, but not all. For example, licenses are not stored in it. 

### Some tips

Your deployment configuration will survive pod restarts or redeployments, because a PersistentVolumeClaim is bound. However, be careful with Pod starts and stops. Specifically, when starting up a deployment, be careful not to kill it before it has successfully started up. Artifactory does take a minute or two to startup, and if you kill the Pod before that, you're likely to corrupt the data on the Persistent Volume as well, and the next time ARtifactory starts up it may be "broken". 



