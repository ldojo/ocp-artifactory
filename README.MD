### Spin up a vanilla persistent Artifactory instance in OCP4, to be used as a Docker registry

#### Set up certs
first we want to set up our certs so that we can use Artifactory as a docker registry with a secure route

For this example, we'll be spinning up Openshift Service, Deployment, Route, etc.., with the name "ear", so we'll want to set the hostname in the command below for the cert to be the same as the Openshift Route "ear" that we'll generate in a later stage.

```
$ openssl req -new -x509 -sha256 -newkey rsa:2048 -nodes -keyout tls.key -days 365 -out tls.crt
Generating a 2048 bit RSA private key
................................................................+++
............+++
writing new private key to 'tls.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:CO
Locality Name (eg, city) [Default City]:Denver
Organization Name (eg, company) [Default Company Ltd]:Citi
Organizational Unit Name (eg, section) []:Eng
Common Name (eg, your name or your server's hostname) []:ear-operator-pipeline.apps.ilab-ctigtdc21d.ecs.dyn.nsroot.net
Email Address []:
```

In the command above, it is very important, that the "Common Name" or hostname is the same as the Openshift Route that will be generated below. In the example above, because I know my Route will be named "ear", the hostname Openshift will autogenerate will be `ear-[my project name, which is "operator-pipeline"].apps.[cluster name, which is ilab-ctigtdc21d].ecs.dyn.nsroot.net`

The step above will create a tls.key and tls.crt file.

#### create certs secret

create an Openshift Secret with the two files we created above:
```
 oc create secret tls nginx-certs --cert=tls.crt --key=tls.key
```

#### enable cert trust on Docker client OS

On the VM that you'll be using docker to login, you'll want to add the cert to the docker trust store.
On RHEL, this is verified to work:
put the tls.crt file in the docker trust store:
```
cp /path/to/tls.crt /etc/docker/certs.d/[hostname set above when creating the cert]/

#for example:
mkdir  /etc/docker/certs.d/ear-operator-pipeline.apps.ilab-ctigtdc21d.ecs.dyn.nsroot.net/;
cp /path/to/tls.crt /etc/docker/certs.d/ear-operator-pipeline.apps.ilab-ctigtdc21d.ecs.dyn.nsroot.net/
```

It may help to also add the certs to the OS trust store. Copy the tls.key and tls.crt files to `/etc/pki/ca-trust/source/anchors/`, and run `update-ca-trust`

#### Deploy artifactory

```
oc process -f artifactory-template.yml -p NAME=ear -p IMAGE=operator-dev-3rdpartymanual-local.artifactrepo-dev.nam.nsroot.net/artifactory-pro:6.20.1 | oc create -f -
```
A Deployment, PVC(75G), Service, Route, and some ConfigMaps will be created. They will all be named "ear", as that is the NAME parameter we passed above.
This also determines the hostname of the Route that is created. For example:

```
$ oc get route | grep ear
ear                      ear-operator-pipeline.apps.ilab-ctigtdc21d.ecs.dyn.nsroot.net                             ear                      443     passthrough/Redirect   None
```

Note that the hostname is the same as the hostname we set when creating the cert with openssl. This has to match. If it doesn't, either edit the Route manually to set the hostname, or rebuild the certs and cert secret we created above.

Wait for the artifactory container to start up -- it does take a few minutes, so recommend tailing the container log to see and wait until it has started up successfully. Then you can go to the Route above in your browser and set up Artifactory with an admin password and initial settings. Be sure to add an Artifactory license -- without it, docker access won't work.

With all that in place, you should be able to successfully `docker login ear-operator-pipeline.apps.ilab-ctigtdc21d.ecs.dyn.nsroot.net` with the same credentials that you set up in the ARtifactory UI console.

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
It is best to tail the logs of the Pod and ensure it started up successfully before you kill it.
