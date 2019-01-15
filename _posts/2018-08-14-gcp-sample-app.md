---
title:  "Deploying a face detection application on Google Cloud"
tags: ['index', 'Google Cloud']
---

This post describes how to use [Alien4Cloud](http://alien4cloud.github.io/) and
the [Ystia Orchestrator](https://github.com/ystia/yorc/) to deploy on Google Cloud
a [sample application detecting faces and text in an image](https://github.com/ystia/yorc-a4c-plugin/tree/develop/tosca-samples/org/ystia/yorc/samples/vision), using Google Cloud Vision API.

You need to have :

* a Google Cloud Platform account,
* a Google Cloud Platform project already created, as described in [Google Documentation](https://cloud.google.com/resource-manager/docs/creating-managing-projects).
* this project must have the access to Google Cloud Vision API enabled as described at [Enable the Vision API](https://cloud.google.com/vision/docs/before-you-begin).

Once done, you can start installing the setup as described in the following sections.

* [Start Alien4Cloud](#startA4C)
* [Start the Ystia Orchestrator](#startYorc)
  * [Prerequisites](#prereq)
    * [Generate SSH keys](#genSSHKeys)
    * [Get Google Service Account keys file](#googleServiceAccount)
  * [Run the Orchestrator](#runYorc)
* [Configure Alien4Cloud to use the orchestrator](#configureA4C)
* [Deploy the application](#deployApp)
  * [Prerequisites on Google Cloud Platform](#prereqGCP)
  * [Create and deploy the application using Alien4Cloud UI](#createDeployAppUI)
    * [Import types and templates definitions](#importUI)
    * [Create the application](#createAppUI)
  * [Create and deploy the application using Alien4Cloud REST API](#createDeployAppREST)
    * [Import types and templates definitions](#importREST)
    * [Create the application](#createAppREST)
* [Use the application](#UseApp)

# Start Alien4Cloud <a name="startA4C"></a>

To perform a standard installation of Alien4Cloud, see [Alien4Cloud Getting started guide](http://alien4cloud.github.io/#/documentation/2.0.0/getting_started/new_getting_started.html).

Here, we will rely on docker to run Alien4Cloud, using this command :

```bash
docker run -d --name a4c -p 8088:8088 alien4cloud/alien4cloud
```

Or if you are using a HTTP proxy, this proxy needs to be known by Alien4Cloud for some operations like importing in Alien4cloud archives from an external web site.

This proxy setting can be defined in Alien4Cloud through the environment variable `JAVA_EXT_OPTIONS`, like this:

```bash
docker run -d --name a4c \
    -p 8088:8088 \
    -e JAVA_EXT_OPTIONS="-Dhttp.proxyHost=10.1.2.3 -Dhttp.proxyPort=8080 -Dhttps.proxyHost=10.1.2.3 -Dhttps.proxyPort=8080 -Dhttp.nonProxyHosts=\"127.0.0.1|10.11.12.13|10.20.*\"" \
    alien4cloud/alien4cloud
```

Logs can be seen running this command :

```bash
docker logs -f a4c
```

Once the log below appear :

```
INFO  Bootstrap:57 - Started Bootstrap in 46.171 seconds (JVM running for 47.79)
```

Alien4Cloud is ready. You can login at [http://localhost:8088](http://localhost:8088)
as admin/admin.

# Start the Ystia Orchestrator <a name="startYorc"></a>

Before starting the orchestrator, the following preliminary steps need to be performed.

## Prerequisites <a name="prereq"></a>

### Generate SSH keys <a name="genSSHKeys"></a>

The orchestrator will need a ssh private key that will be used to ssh on Virtual Machines created on demand.

Generate private/public keys in a directory `$HOME/yorcdir` that will be later mounted on the
orchestrator docker container, running (to keep the configuration simple, don't enter a passphrase here, even if it is supported) :

```bash
$ mkdir $HOME/yorcdir
$ ssh-keygen ed25519 -f $HOME/yorcdir/yorckey
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in yorcdir/yorckey.
Your public key has been saved in yorcdir/yorckey.pub.
...
```

### Get Google Service Account keys file <a name="googleServiceAccount"></a>

The orchestrator will have to authenticate against Google Cloud Platform.

For this, you need to download a service account keys file.
This can be done from Google Cloud console at [https://console.cloud.google.com/apis/credentials/serviceaccountkey](https://console.cloud.google.com/apis/credentials/serviceaccountkey).

Or using [gcloud CLI](https://cloud.google.com/sdk/), you can run the following command to get your default service account :

```bash
$ gcloud iam service-accounts list
NAME                                    EMAIL
Compute Engine default service account  my-user@developer.gserviceaccount.com
```

And then create a keys file `$HOME/yorcdir/gcloudkeys.json` running :

```bash
gcloud iam service-accounts keys create \
    --iam-account my-user.gserviceaccount.com \
    $HOME/yorcdir/gcloudkeys.json
```

This file is created under directory `$HOME/yorcdir` as this directory will be mounted on the orchestrator docker container, so the orchestrator can access it to authenticate against Google Cloud Platform.

## Run the Orchestrator <a name="runYorc"></a>

Here, like previously with Alien4Cloud, we will rely on docker to run the orchestrator.
For a standard installation, see [Ystia Orchestator installation](http://yorc.readthedocs.io/en/latest/install.html).
For an HA installation, see [Ystia Orchestator High Availability installation](http://yorc.readthedocs.io/en/latest/ha.html).

The orchestrator needs to have details on the infrastructures it can use to create Virtual Machines on demand.
For Google Cloud, the orchestrator needs to know :

* the Google Cloud Platform Project ID
* the path to (or content of) Google Cloud service account keys file that we created above.

The project ID is available as well in the service account keys file.
You can get it running :

```bash
$ grep project_id $HOME/yorcdir/gcloudkeys.json
  "project_id": "my-gcloud-project1",
```

The path to `gcloudkeys.json` is the path as seen from the container once
`$HOME/yorcdir` will be mounted on the container.
So if we mount `$HOME/yorcdir` on `/etc/yorc` on the container,
the path to the service account file will be: `/etc/yorc/gcloudkeys.json`.

These values, project id and path to service account file, can be provided 
through the environment variables `YORC_INFRA_GOOGLE_PROJECT` and `YORC_INFRA_GOOGLE_APPLICATION_CREDENTIALS` in the docker run command.

So the orchestrator can be run using this command :

```bash
docker run -d \
    -e YORC_INFRA_GOOGLE_PROJECT=my-gcloud-project1 \
    -e YORC_INFRA_GOOGLE_APPLICATION_CREDENTIALS=/etc/yorc/gcloudkeys.json \
    -v $HOME/yorcdir:/etc/yorc \
    -p 8800:8800 \
    --name yorc \
    ystia/yorc
```

Logs can be seen running this command :

```bash
docker logs -f yorc
```

Finally, to be able to use the ssh private key file, this file has to be owned
by the user used to run the orchestrator, and has to be read-only for this user.

To have the expected ownership and permissions, run the following commands :

```bash
docker exec -it yorc chmod 400 /etc/yorc/yorckey
docker exec -it yorc chown yorc:yorc /etc/yorc/yorckey
```

Update as well the service account file owner and permissions:

```bash
docker exec -it yorc chmod 400 /etc/yorc/gcloudkeys.json
docker exec -it yorc chown yorc:yorc /etc/yorc/gcloudkeys.json
```

# Configure Alien4Cloud to use the orchestrator <a name="configureA4C"></a>

Now that Alien4Cloud and the Orchestrator are up and running, the following
actions need to be performed in Alien4Cloud :

* Upload the Ystia Orchestrator plugin in Alien4Cloud
* Create, configure, enable the orchestrator in Alien4Cloud
* Create a Google Cloud location for this orchestrator in Alien4Cloud
* Create and configure on an on-demand compute resource.

This can be done using Alien4Cloud UI or REST API.
Scripts are available in the Ystia Alien4Cloud Orchestrator plugin repository for each step, for simplicity.
On the host where you installed docker and used `docker run` to start the
Alien4Cloud container, get these scripts running:

```bash
git clone https://github.com/ystia/yorc-a4c-plugin.git
cd yorc-a4c-plugin/alien4cloud-yorc-plugin/src/test/scripts/
chmod a+x *
```

Upload the Ystia Orchestrator plugin running the following script.

This will by default upload the Ystia Orchestrator plugin version 3.0.0 on Alien4Cloud at `http://localhost:8088`, see script usage using subcommand `--help` if you need
to use other values than these default values:

```bash
$ ./upload_yorc_plugin
Downloading the plugin...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   624    0   624    0     0    690      0 --:--:-- --:--:-- --:--:--   690
100  217k  100  217k    0     0    97k      0  0:00:02  0:00:02 --:--:--  236k
Plugin uploaded in Alien4Cloud
```

Create the Ystia Orchestrator in Alien4Cloud:

```bash
$ ./create_orchestrator
Orchestrator created in Alien4Cloud
```

Configure the orchestrator.

You need to pass in argument the URL of the orchestrator, that will be accessed by Alien4Cloud. You cannot pass `http://localhost:8800` as the localhost within the docker container running Alien4Cloud is not the localhost of the Host running the orchestrator container.
You should get your Host IP address running `ifconfig` for example (but you could use instead the IP address of the container running the orchestrator, from this command: `docker exec yorc ip a`).

If for example this host where you used the `docker run` command to run the orchestrator has the IP address `10.1.2.3`,
the following command can be run to configure the orchestrator in Alien4Cloud:

```bash
$ ./configure_orchestrator --yorc-url http://10.1.2.3:8800
Orchestrator configured in Alien4Cloud
```

Enable the orchestrator :

```bash
$ ./enable_orchestrator
Orchestrator enabled in Alien4Cloud
```

Create a Google Cloud location for this orchestrator :

```bash
$ ./create_location
Location Google of type 'Google Cloud' created
```

Create and configure on an on-demand compute resource on this location.

By default, this will configure the use of a `CentOS 7` image for Compute Instances, with a predefined user `yorcuser`.

Mandatory parameters are :

* the Google zone where to create Compute instances,
* the content of the public key to configure on this Compute instance
* the path to private key (accessible from the orchestrator container with the right owner and permissions as set above) used by the orchestrator to ssh on this Compute Instance

```bash
PUBLIC_KEY_CONTENT=`cat $HOME/yorcdir/yorckey.pub`
./create_on_demand_google_compute \
    --zone europe-west1-b \
    --public-key "$PUBLIC_KEY_CONTENT" \
    --private-key-file /etc/yorc/yorckey
```

# Deploy the application <a name="deployApp"></a>

The sample application we will deploy detects faces and text in an image.

This application is using these Google Cloud APIs:

* [Cloud Pub/Sub](https://cloud.google.com/pubsub/) to be notified of new images uploaded in Cloud Storage
* [Cloud Vision](https://cloud.google.com/vision/) to detect faces and text in an image
* [Cloud Storage](https://cloud.google.com/storage/) to download images from an input bucket and upload transformed images to an output bucket

so that when an image is added by the user in a input cloud storage bucket, the application is notified of the upload. The application will then call the Cloud Vision API to request the detection of:

* faces
* likelihood of expressions on these faces
* text
* links to similar images on the Web
* labels

Once the response is provided, the application will create a copy of the image and will update it to surround faces and text according to the coordinates returned by the Vision API.
A result page will be created, stored in an output cloud storage bucket and made publicly available.

This picture describes the flow :

![App flow]({{ site.baseurl }}/assets/images/visionappflow.png)

## Prerequisites on Google Cloud Platform <a name="prereqGCP"></a>

As described above, the application is listening to notifications of new images
uploaded in an input Cloud Storage, and is publishing the result in an output
Cloud Storage.

These entities must be created on Google Cloud before deploying the application.
This can be done from the [Google Cloud Console](https://console.cloud.google.com/) or using Google CLIs [gcloud](https://cloud.google.com/sdk/) and  [gsutil](https://cloud.google.com/storage/docs/gsutil_install).

First, create an input and output storage buckets (you need to provide unique names):

```bash
gsutil mb gs://my-vision-input
gsutil mb gs://my-vision-output
```

A Cloud pub/sub topic and subscription must be created so that the application can be notified
of images uploads in the input storage bucket:

```bash
gsutil notification create -f json -t my-vision-topic gs://my-vision-input
gcloud beta pubsub subscriptions create my-vision-input-sub --topic=my-vision-topic
```

Once this is done, you are ready to deploy the Application.

Up until now, scripts making use of the Alien4Cloud REST API to were run to
perform operations.

Next sections describing how to create and deploy an application, will show both ways

* either using Alien4Cloud UI,
* or using Alien4Cloud REST API.

Next section describes how to create and deploy the application using the UI.
You can directly go to [next section](#createDeployAppREST) to create and deploy the application using scripts.

## Create and deploy the application using Alien4Cloud UI <a name="createDeployAppUI"></a>

We will start by first uploading required types and [topology template](http://alien4cloud.github.io/#/documentation/2.0.0/devops_guide/tosca_grammar/topology_template.html) (description of components making the application, and relationships between these components) in Alien4Cloud.

### Import types and templates definitions <a name="importUI"></a>

The [Alien4Cloud Ystia Orchestrator plugin git repository](https://github.com/ystia/yorc-a4c-plugin) provides types and template definitions for our application in directory `tosca-samples/org/ystia/yorc/samples/vision`.

We will upload these elements in Alien4Cloud using the UI.

Open a web browser on `http://localhost:8088` and login as admin/admin.

This page appears :

![A4C Login]({{ site.baseurl }}/assets/images/a4cLogin.PNG)

Select tab `Catalog`. A page appears showing a list of components already present
in the catalog, brought by Alien4Cloud and the Orchestrator :

![Catalog Components]({{ site.baseurl }}/assets/images/catalogComponents.PNG)

Click on tab `Manage Archives`. This page appears:

![Manage Archives]({{ site.baseurl }}/assets/images/manageArchives.PNG)

Its lists archives provided by Alien4Cloud and the Orchestrator. You can upload your zipped archives
from here, or import a git repository as we'll do below.

Click on `Git import`. The following page appears:

![Git Import]({{ site.baseurl }}/assets/images/gitImport.PNG)

Click on `Git location` to define a git location. A popup appears where you
need to provide the following inputs:

* Repository URL: `https://github.com/ystia/yorc-a4c-plugin.git`
* Branch: `develop`
* Archive(s) to import : `tosca-samples/org/ystia/yorc/samples/vision`

Then click on `+` to add this branch/archive, to have finally this definition (no need to define credentials here):

![Git Location]({{ site.baseurl }}/assets/images/gitLocation.PNG)

Click on `Save`. The following git location appears in the Archives page :

![Git archives]({{ site.baseurl }}/assets/images/gitLocationImport.PNG)

Click on the operation `Import` to perform the import, it will take some time,
and finally provide this result on the Archive page when the import is successful:

![Git archives]({{ site.baseurl }}/assets/images/importSuccess.PNG)

Components and topology template needed fro the application are now uploaded
in the Alien4Cloud catalog.

Clicking on tab `Browse topologies`, you can see the topology template that
was just imported :

![Catalog Topologies]({{ site.baseurl }}/assets/images/catalogTopologies.PNG)

Now that types and topology template `VisionTopology` (version 0.1.0) are
available in Alien4Cloud, we can create an application from this topology template.

### Create the application <a name="createAppUI"></a>

From Alien4Cloud home page at `http://localhost:8088`:

![A4C Login]({{ site.baseurl }}/assets/images/a4cLogin.PNG)

Select tab `Applications` in the upper left corner. This page appears:

![Applications]({{ site.baseurl }}/assets/images/applications.PNG)

Click on `New Application`. A window pops up where you can specify:

* the name of the new application to create, here `ImageDetection`
* Initialize the topology from a Topology Template
* and select the topology template `VisionTopology` that was imported above.

![New Application]({{ site.baseurl }}/assets/images/newApplication.PNG)

Click on `Create`. The following page appears, where you can see the application
created :

![Application created]({{ site.baseurl }}/assets/images/applicationCreated.PNG)

Click on the environment (area surrounded by a green rectangle above). The
following page appears:

![Selected application]({{ site.baseurl }}/assets/images/selectedApplication.PNG)

Select the `Topology` tab to check the application topology :

![Selected application]({{ site.baseurl }}/assets/images/topology.PNG)

We can see above that it consists in a Compute Node `ComputeInstance` that will
be created on demand, and an application `ImageDetection` that will be deployed
on this Compute Node.

Select tab `Inputs`. The following page appears where you need to provide your
inputs for the application :

![Application Inputs]({{ site.baseurl }}/assets/images/inputs.PNG)

Select tab `Locations`, and select the Google Location like below :

![Google Location]({{ site.baseurl }}/assets/images/locations.PNG)

Once done, select tab `Review & deploy`. The following page appears :

![Review and deploy]({{ site.baseurl }}/assets/images/review.PNG)

Click on `Deploy`. The following page showing the deployment in progress appears:

![Deployment progress]({{ site.baseurl }}/assets/images/deploymentInProgress.PNG)

You can click on the `Runtime view` on the left-hand side to see the current status
of components deployment from their color. Here the `ComputeInstance` component is up and running,
while the `ImageDetection` application is being installed:

![Runtime view]({{ site.baseurl }}/assets/images/runtimeview.PNG)

An Alien4Cloud premium version allows to see deployments logs from the UI.

Here, the open source version does not provide this feature. But you can still use
the orchestrator CLI to get deployment logs, running :

```bash
docker exec -ti yorc yorc d logs -b ImageDetection-Environment
```

Back to Alien4Cloud UI, clicking on `Info` on the left-hand side, we go back to the previous page, that will show the status `Deployed` when the application will be deployed, like below :

![Runtime view]({{ site.baseurl }}/assets/images/deployed.PNG)

The application is deployed now, it is listening to notifications of file
uploaded in your input storage bucket.

You can skip next section describing how to create the application through the
REST API, and go to last section on how to [use the application](#UseApp).

## Create and deploy the application using Alien4Cloud REST API <a name="createDeployAppREST"></a>

If you did not follow previous section to create the application from the UI,
this section will allow you to create the application from scripts using the
REST API.

We will start by first uploading required types and [topology template](http://alien4cloud.github.io/#/documentation/2.0.0/devops_guide/tosca_grammar/topology_template.html) (description of components making the application, and relationships between these components) in Alien4Cloud.

### Import types and templates definitions <a name="importREST"></a>

The [Alien4Cloud Ystia Orchestrator plugin git repository](https://github.com/ystia/yorc-a4c-plugin) provides types and template definitions for our application in directory `tosca-samples/org/ystia/yorc/samples/vision`.

Upload these elements in the Alien4Cloud catalog running this command:

```bash
$ ./import_archive \
    --repository https://github.com/ystia/yorc-a4c-plugin.git \
    --path tosca-samples/org/ystia/yorc/samples/vision
Archive created, proceeding to the import...
Archive imported
```

This will import types and a template definition (called Topology template) named
`VisionTopology` (version 0.1.0) in the Alien4Cloud catalog.

### Create the application<a name="createAppREST"></a>

The following scripts will perform the actions of creating, configuring and
deploying an application, using the REST API.

First, create the application:

```bash
$ ./create_application --name ImageDetection \
    --template VisionTopology --version 0.1.0
Application ImageDetection created
```

This application has mandatory inputs that must be provided:

* the Google Cloud Project ID
* the Google Cloud Subscription providing notifications of image uploads
* where to place results
* a path to the keys file needed to authenticate against Google API

The following script can be run to provide input parameters as key=value pairs to the
application:

```bash
$ ./set_application_inputs --application ImageDetection \
    --inputs "project_id=my-gcloud-project1 subscription_id=my-vision-input-sub output_bucket_name=my-vision-output api_keys_file=/etc/yorc/gcloudkeys.json"
Input parameters set on Application ImageDetection
```

Then, you have to select where you want to deploy the application, here this
script will select the Google Cloud location that was created above:

```bash
$ ./select_application_location  --application ImageDetection \
    --location Google
Selected location Google for application ImageDetection
```

The application can now be deployed.

This command will deploy the application,
which will consist first in creating a compute instance, then in deploying the
application on this compute instance created on demand (it should take several 
minutes):

```bash
$ ./deploy_application --application ImageDetection
Application ImageDetection deployment in progress..............
Application ImageDetection deployed
```

The application is deployed now, it is listening to notifications of file
uploaded in your input storage bucket.

The application can now be used to detect faces and text in images that will
be uploaded, as described in next section.

# Use the application <a name="UseApp"></a>

Upload an image containing faces or text in this input storage/bucket,
either through the Google Cloud console, or using the `gsutil cp` CLI, and 30
seconds/1 minute later, you should see in  your output storage bucket a new
subdirectory containing an html result page,  with an associated public link on which you can click to see the detection results.
