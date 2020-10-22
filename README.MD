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
Organization Name (eg, company) [Default Company Ltd]:ORG
Organizational Unit Name (eg, section) []:Eng
Common Name (eg, your name or your server's hostname) []:ear.yourdomain.net
Email Address []:
```

The step above will create a tls.key and tls.crt file.

#### create certs secret

create an Openshift Secret with the two files we created above (be sure to name it 'nginx-certs', as the deployment below will look for it):
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
mkdir  /etc/docker/certs.d/ear.yourdomain.net/;
cp /path/to/tls.crt /etc/docker/certs.d/ear.yourdomain.net/
```

It may help to also add the certs to the OS trust store. Copy the tls.key and tls.crt files to `/etc/pki/ca-trust/source/anchors/`, and run `update-ca-trust`

#### Deploy artifactory

The artifactory container will run as uid 1030 and attempt to write to the PVC, and you may want to set your project's security context settings accordingly if it is not already. You can do so by yadjusting the scc settings like so:

```
oc adm policy add-scc-to-user anyuid -z default
```

```
oc process -f artifactory-template.yml -p NAME=ear -p IMAGE=docker.bintray.io/jfrog/artifactory-pro:6.20.1 | oc create -f -
```
A Deployment, PVC(75G), Service, Route, and some ConfigMaps will be created. They will all be named "ear", as that is the NAME parameter we passed above.
**Note that the hostname is the same as the hostname we set when creating the cert with openssl. This has to match.** If it doesn't, either edit the Route manually to set the hostname, or rebuild the certs and cert secret we created above.



### Giving Openshift ability to pull from the artifactory instance

With your own Artifactory instance running in place and providing a docker registry you can do docker pull/push against, you may also want to give Openshift the ability to pull images from this docker registry so that it can run Pods from those images. To do this, similarly to how you added the cert to your client host's ca cert trust, you need to tell Openshift to trust your cert as well. You must be a cluster-admin on the openshift cluster to run these commands:
```
#substitute the name of the configmap, route, cert file location, etc., in the below commands with those relevant to your use case.
#create a configmap in the 'openshift-config' namespace that contains your .crt file
oc create configmap ear-cas -n openshift-config --from-file=eae.yourdomain.net=/etc/docker/certs.d/ear.yourdomain.net/tls.crt 
#add the configMap to the cluster configuration. Note, this will overwrite the "additionalTrustedCA" section of image.config.openshift.io/cluster with just the configmap below that is listed. Recommended to check first if someone else might have already added data in that section, and to do an `oc edit image.config.openshift.io/cluster` instead of a patch. 
oc patch image.config.openshift.io/cluster --patch '{"spec":{"additionalTrustedCA":{"name":"ear-cas"}}}' --type=merge
```


