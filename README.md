# Demo-linuxkit-infrakit (WIP)

Simple playground for linuxkit and infrakit study.

Final goal is to deploy a software forge on Openstack:

* 1 LDAP appliance
* 1 gitea appliance
* 1 jenkins appliance
* 1 jenkins slave appliance

## Demo ldap

### ldap package

ldap package is available in *pkg/ldap*: it contains *build.yml* for package
definition and Dockerfile to build package image

* Go do *pkg* directory
* build and push package (need to be logged in dockerhub and have push rights
  on repo specified by org key in *build.yml*)

  ```
  $ make push
  ```

> **Note**
> On linux, be sure to not use any docker-credentials-helper since the
> tool will look at the base64 encoded user/password in ~/.docker/config.json

ldap package accepts following parameters (as env variables):

* **ROOT_USER**: root username (default: admin)
* **ROOT_PW**: root password (default: password)
* **SUFFIX**: base dn (default: dc=example,dc=org)
* **ORGANISATION_NAME**: organisation name (default: Example.org (c))

ANY env variable has an *_FILE* suffixed alternative that can be used to
specify a file instead (for use with secrets and config in Docker mode swarm or kubernetes)

### ldap appliance

Here an ldap appliane is built as a QCOW2 image, then pushed and tested on an
Openstack Cloud

* Go to *appliances* directory
* Update *ldap/ldap.yml* with last version of the package if necessary
* Build qcow2 image

  ```
  linuxkit build -pull -format qcow2-bios -name linuxkit-ldap-appliance ldap/ldap.yml
  ```

* source Openstack openrc.sh then push image to glance

  ```
  $ linuxkit push openstack -img-name=linuxkit-ldap-appliance -authurl=$OS_AUTH_URL \
    -domain=$OS_USER_DOMAIN -username=$OS_USERNAME -project=$OS_PROJECT_NAME \
    -password=$OS_PASSWORD linuxkit-ldap-appliance.qcow2
  ```

* run the image directly with linuxkit (change any paramater to adapt to your
  Openstack cloud):

  ```
  $ linuxkit run openstack -authurl=$OS_AUTH_URL -domain=$OS_USER_DOMAIN \
    -username=$OS_USERNAME -project=$OS_PROJECT_NAME -password=$OS_PASSWORD \
    -flavor=m1.small -instancename=demo-ldap -keyname=mcottret -sec-groups="openbar" \
    -network=d5c237e1-0376-4d68-87b6-e208f6dc210d linuxkit-ldap-appliance
  ```

To connect to the ldap instance, get instance public ip (or add a floating
if necessary)

To check if the instance is up and running:

```
$ ssh root@LDAP_IP
$ ctr c ls # list containers
$ ctr t ls # list tasks
```

If any task is stopped, logs can be found in */var/log*

To initiate an ldap connexion, use the defaults paramaters:

* **url**: LDAP_IP:389
* **admin dn**: cn=admin,dc=example,dc=org
* **admin passwd**: password

To use custom parameters to configure the instances:

* modify the *ldap.yml*, rebuild and push the new image
* provide a json metadata file at runtime:

  ```
  {
    "ldap_suffix": {
      "content": "dc=wynnlin,dc=lab",
      "perm": "0644"
    },
    "ldap_rootpass": {
      "content": "totolitoto",
      "perm": "0644"
    },
    "ldap_organisation": {
      "content": "Wynnlin Lab",
      "perm": "0644"
    }
  }
  ```
