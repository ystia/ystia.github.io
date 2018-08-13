---
title:  "Deploying an image detection application on Google Cloud Platform"
tags: ['index', 'Google Cloud', 'howto']
---

This post described how to use [Alien4Cloud](http://alien4cloud.github.io/) and 
the [Ystia Orchestrator](https://github.com/ystia/yorc/) to deploy on Google Cloud
a [sample application](https://github.com/ystia/yorc-a4c-plugin/tree/develop/tosca-samples/org/ystia/yorc/samples/vision) detecting faces and text in an image, using the Google Cloud Vision API.

You need to have :
  * a Google Cloud Platform account,
  * a Google Cloud Platform project already created, as described in [Google Documentation](https://cloud.google.com/resource-manager/docs/creating-managing-projects).
  * this project must have the access to Google Cloud Vision API enabled as described at [Enable the Vision API](https://cloud.google.com/vision/docs/before-you-begin).

Once done, you can start the setup as described in the following sections. 

# Start Alien4Cloud

To perform a standard installation of Alien4Cloud, see [Alien4Cloud Getting started guide](http://alien4cloud.github.io/#/documentation/2.0.0/getting_started/new_getting_started.html). 

Here, we will rely on docker to run Alien4Cloud, using this command :
```
docker run -d --name a4c -p 8088 laurentg/docker-alien4cloud
```

Or if you are using a HTTP proxy, this proxy needs to be known by Alien4Cloud for some operations like importing in Alien4cloud archives from an external web site.
This proxy setting can be defined in Alien4Cloud through the environment variable JAVA_EXT_OPTIONS, like this:
```
docker run -d --name a4c \
           -p 8088 \
           -e JAVA_EXT_OPTIONS="-Dhttp.proxyHost=10.1.2.3 -Dhttp.proxyPort=8080 -Dhttp.nonProxyHosts=\"127.0.0.1|10.11.12.13|10.20.*\"" \
           laurentg/docker-alien4cloud:2.0.0
```

Logs can be seen running this command :
```
docker logs -f a4c
```
Once the log below appear :
```
INFO  Bootstrap:57 - Started Bootstrap in 46.171 seconds (JVM running for 47.79)
```
Alien4Cloud is ready, and you can login at http://localhost:8088 as admin/admin.


# Start the Ystia Orchestrator

Before starting the orchestrator, the following preliminary steps need to be performed.

## Prerequisites

### Generate SSH keys

The orchestrator will need a ssh private key that will be used to ssh on Virtual Machines created on demand.
Generate private:public keys in a directory `$HOME/yorcdir` that will be later mounted on the 
orchestrator docker container, running :
```
$ mkdir $HOME/yorcdir
$ ssh-keygen ed25519 -f $HOME/yorcdir/yorckey
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in yorcdir/yorckey.
Your public key has been saved in yorcdir/yorckey.pub.
...
```

### Get Google Service Account keys file

The orchestrator will have to authenticate against Google Cloud Platform.
For this, you need to download a service account keys file.
This can be done from Google Cloud console at https://console.cloud.google.com/apis/credentials/serviceaccountkey

Or using [gcloud CLI](https://cloud.google.com/sdk/), you can run the following command to get your default service account :
```
$ gcloud iam service-accounts list
NAME                                    EMAIL
Compute Engine default service account  my-user@developer.gserviceaccount.com
```

And then create a keys file `$HOME/yorcdir/gcloudkeys.json` running :
```
$ gcloud iam service-accounts keys create \
         --iam-account my-user.gserviceaccount.com \
         $HOME/yorcdir/gcloudkeys.json
```

This file is created under directory `$HOME/yorcdir` as this directory will be mounted on the orchestrator docker container, so the orchestrator can access it to authenticate against Google Cloud Platform.

## Run the Orchestrator

Here, like previously with Alien4Cloud, we will rely on docker to run the orchestrator.
For a standard installation, see [Ystia Orchestator installation](http://yorc.readthedocs.io/en/latest/install.html)
For an HA installation, see [Ystia Orchestator High Availability installation](http://yorc.readthedocs.io/en/latest/ha.html).


The orchestrator needs to have details on the infrastructures it can use to create Virtual Machines on demand.
For Google Cloud, the orchestrator needs to know :
  * the Google Cloud Platform Project ID
  * the path to (or content of) the Google Cloud serivce account keys file that we created above.

The project ID is available as well is the service account keys file.
You can get it running :

$ grep project_id $HOME/yorcdir/gcloudkeys.json
  "project_id": "my-gcloud-project1",

The path to gcloudkeys.json, is the path as seen from the container once $HOME/yorcdir will be mounted on the container. So if we mount $HOME/yorcdir on /etc/yorc on the container,
the path to the service account file will be: /etc/yorc/gcloudkeys.json.

These values, project id and path to service account file, can be provided 
through the environment variables YORC_INFRA_GOOGLE_PROJECT and YORC_INFRA_GOOGLE_APPLICATION_CREDENTIALS in the docker run command.

So the orchestrator can be run using this command :
```
$ docker run -d \
	-e YORC_INFRA_GOOGLE_PROJECT=my_project_1 \
	-e YORC_INFRA_GOOGLE_APPLICATION_CREDENTIALS=/etc/yorc/gcloudkeys.json \
    -v $HOME/yorcdir:/etc/yorc \
    -p 8800:8800 \
	--name yorc \
	ystia/yorc:3.0.0
```

Logs can be seen running this command :
```
$ docker logs -f yorc
```

Finally, to be able to use the ssh private key, it has to be owned by the user
used to run the orchestrator, and has to be read-only for this user.
For this, run the following commands :
```
$ docker exec -it yorc chmod 400 /etc/yorc/yorckey
$ docker exec -it yorc chown yorc:yorc /etc/yorc/yorckey
```
Update as well the service account file owner and permissions:
```
$ docker exec -it yorc chmod 400 /etc/yorc/gcloudkeys.json
$ docker exec -it yorc chown yorc:yorc /etc/yorc/gcloudkeys.json
```

# Configure Alien4Cloud to use the orchestrator

Now that Alien4Cloud and the Orchestrator are up and running, the following
actions need to be performed in Alien4Cloud :
  * Upload the Ystia Orchestrator plugin in Alien4Cloud
  * Create, configure, enable the orchestrator in Alien4Cloud
  * Create a Google Cloud location for this orchestator in Alien4Cloud
  * Create and configure on an on-demand compute resource.

This can be done using Alien4Cloud UI or REST API.
Scripts are available in the Ystia Alien4Cloud Orchestrator plugin repository for each step.
Get these scripts running :
```
$ git clone https://github.com/ystia/yorc-a4c-plugin.git
$ cd yorc-a4c-plugin/alien4cloud-yorc-plugin/src/test/scripts/
$ chmod a+x *
```

Upload the Ystia Orchestrator plugin, this will by default upload the Ystia Orchestrator plugin version 3.0.0 on Alien4Cloud at http://localhost:8088, see script usage using subcommand --help if you need to user other values thant these default values):
```
$ ./upload_yorc_plugin
Downloading the plugin...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   624    0   624    0     0    690      0 --:--:-- --:--:-- --:--:--   690
100  217k  100  217k    0     0    97k      0  0:00:02  0:00:02 --:--:--  236k
Plugin uploaded in Alien4Cloud
```

Create This Ystia Orchestrator in Alien4Cloud:
```
$ ./create_orchestrator
Orchestrator created in Alien4Cloud
```
Configure the orchestrator. You need to pass in argument the URL of the orchestrator, that will be accessed by Alien4Cloud. You cannot pass http://localhost:8800 as the localhost within the docker container running Alien4Cloud is not the localhost of the Host running the orchestrator container.
You should get your Host IP address running `ifconfig` for example.
Assuming this host where used the 'docker run' command to run the orchestrator has the IP address 10.1.2.3,
we will run the following command to configure the orchestrator in Alien4Cloud :
```
$ ./configure_orchestrator --yorc-url http://10.1.2.3:8800
Orchestrator configured in Alien4Cloud
```

Enable the orchestrator :
```
$ ./enable_orchestrator
Orchestrator enabled in Alien4Cloud
```

Create a Google Cloud location for this orchestrator :
```
$ ./create_location
Location Google of type 'Google Cloud' created
```

Create and configure on an on-demand compute resource on this location.
By default, this will configure the use of a CentOS 7 image for Compute Instances, with a predefined user yorcuser. Mandatory parameters are :
  * the Google zone where to create Compute instances,
  * the content of the public key to configure on this Compute instance
  * the path to private key (accessible from the orchestrator container with the right owner and permissions as set above) used by the orchestrator to ssh on this Compute Instance :
```
PUBLIC_KEY_CONTENT=`cat $HOME/yorcdir/yorckey.pub`
./create_on_demand_google_compute \
    --zone europe-west1-b \
    --public-key "$PUBLIC_KEY_CONTENT" \
	--private-key-file /etc/yorc/yorckey
```

# Deploy the application
