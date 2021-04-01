# HL7 and FHIR Demo

## Demo Setup

### Project Setup and Variables

#### Step one

Activate cloud shell.

#### Step two

Export the environment variables needed for the lab by running the following
commands in cloud shell.

```shell
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export PROJECT_NUMBER=$(gcloud projects list --filter=$PROJECT_ID \
  --format='value(PROJECT_NUMBER)')
export COMPUTE_SA=$PROJECT_NUMBER-compute
export LOCATION=us-central1
export DATASET_ID=datastore
export FHIR_STORE_ID=fhirstore
export FHIR_TOPIC=fhirtopic
export HL7_TOPIC=hl7topic
export HL7_SUB=hl7subscription
export HL7_STORE_ID=hl7v2store
export BQ_ID=fhirdata
```

--------------------------------------------------------------------------------

### Pub/Sub Setup

#### Step one

Using the cloud shell, create pub/sub topic and subscription that will pass data
from the HL7 store, as well as the topic to pass data from the FHIR store.

```shell
gcloud pubsub topics create $HL7_TOPIC
gcloud pubsub subscriptions create $HL7_SUB --topic=$HL7_TOPIC
gcloud pubsub topics create $FHIR_TOPIC
```

--------------------------------------------------------------------------------

### BigQuery Setup

#### Step one

Using the cloud shell, create a dataset in BigQuery.

```shell
bq --location=us-east1 mk --dataset --description HCAPI-dataset \
$PROJECT_ID:$BQ_ID
```

--------------------------------------------------------------------------------

### Healthcare API Setup

#### Step one

Ensure the Healthcare API has been enabled in the project. \
Create a dataset for the Healthcare API datastores to be organized under.

NOTE: The datasets for Healthcare API are an organizational construct, similar
to folders in GCS. You can store many stores of different types (HL7, FHIR, and
DICOM) in a single dataset.

```shell
gcloud healthcare datasets create $DATASET_ID \
--location=$LOCATION
```

#### Step two

Create an HL7v2 store.

```shell
gcloud beta healthcare hl7v2-stores create $HL7_STORE_ID \
--dataset=$DATASET_ID \
--location=$LOCATION \
--pubsub-topic=projects/$PROJECT_ID/topics/$HL7_TOPIC
```

#### Step three

Patch HL7 store to parse HL7 to JSON.

```shell
curl -X PATCH \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    -H "Content-Type: application/json; charset=utf-8" \
    --data "{
      'parserConfig': {
        'schema': {
          'schematizedParsingType': 'HARD_FAIL',
          'ignoreMinOccurs' :true,
          'unexpectedSegmentHandling' : 'PARSE'
        }
      }
    }" "https://healthcare.googleapis.com/v1beta1/projects/$PROJECT_ID/locations/$LOCATION/datasets/$DATASET_ID/hl7V2Stores/$HL7_STORE_ID?updateMask=parser_config.schema"
```

#### Step four

Create a FHIR store.

```shell
gcloud healthcare fhir-stores create $FHIR_STORE_ID \
--dataset=$DATASET_ID \
--location=$LOCATION \
--version=R4 \
--pubsub-topic=projects/$PROJECT_ID/topics/$FHIR_TOPIC \
--disable-referential-integrity \
--enable-update-create
```

#### Step five

Set up the appropriate permissions to enable exporting data from your FHIR store
to BigQuery.

```shell
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-healthcare.iam.gserviceaccount.com \
  --role=roles/bigquery.dataEditor
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-healthcare.iam.gserviceaccount.com \
  --role=roles/bigquery.jobUser
```

This may take a few seconds to complete.

#### Step six

Enable streaming from FHIR store to BigQuery

```shell
curl -X PATCH \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    -H "Content-Type: application/json; charset=utf-8" \
    --data "{ 'streamConfigs': [ { 'bigqueryDestination': { 'datasetUri':
'bq://$PROJECT_ID.$BQ_ID', 'schemaConfig': { 'schemaType': 'ANALYTICS' } }
} ] }" \
"https://healthcare.googleapis.com/v1/projects/$PROJECT_ID/locations/$LOCATION/datasets/$DATASET_ID/fhirStores/$FHIR_STORE_ID?updateMask=streamConfigs"
```

--------------------------------------------------------------------------------

### Dataflow Pipeline

#### Step one

Create a bucket to house the HL7v2 to FHIR mappings.

```shell
gsutil mb gs://$PROJECT_ID
```

#### Step two

Copy the mapping files to your newly created bucket.

```shell
gsutil -m cp -r gs://hdc-mapping-files/* gs://$PROJECT_ID/
```

#### Step three

Navigate to the newly created bucket on the Cloud Console UI under Cloud
Storage. \
Select the bucket, then select CREATE FOLDER. \
Name the folder "error". Select save

#### Step four

Give permissions to your compute service account to access the bucket and \
the healthcare API.

```shell
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$COMPUTE_SA@developer.gserviceaccount.com \
  --role=roles/pubsub.subscriber
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$COMPUTE_SA@developer.gserviceaccount.com \
  --role=roles/healthcare.hl7V2Consumer
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$COMPUTE_SA@developer.gserviceaccount.com \
  --role=roles/healthcare.fhirResourceEditor
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$COMPUTE_SA@developer.gserviceaccount.com \
  --role=roles/storage.objectAdmin
```

#### Step three

Create a new VM by navigate to Compute Engine. \
Select create instance. Name your instance **dataflow-vm.** \
Under machine type select the E2 series and an **e2-standard-4 type.** \
Under boot disk select **change.** \
Leave the OS and version as Debian GNU/Linux 10. \
Change the Boot disk size to be **500 GB.** Click select to finish. \
Check Allow full access to all Cloud APIs. Check **Allow HTTP and HTTPS
traffic.** \
Select create.

#### Step four

Once the VM has initialized, under connect SSH into the VM.

## STOP

Go to the VM setup file to setup the pipeline VM.

**Return back to this page when complete.**

### Check the Dataflow Pipeline

Navigate to Dataflow on the GCP Console, view your recently created pipeline.

### HL7 to FHIR Streaming

#### Step one

Copy hl7 from GCS to your local cloud shell.

```shell
gsutil cp gs://$PROJECT_ID/hl7_sample ./
```

#### Step two

Send the HL7 message to the HL7 store.

```shell
echo "$(cat ./hl7_sample)" | base64 | tr -d \\n | cat <(echo -n "{\"message\": {\"data\" : \"") <(cat -) <(echo -n "\" }}") | curl -X POST -H "Content-Type: application/json" --data-binary @-   https://healthcare.googleapis.com/v1beta1/projects/$PROJECT_ID/locations/$LOCATION/datasets/$DATASET_ID/hl7V2Stores/$HL7_STORE_ID/messages?access_token=$(gcloud auth print-access-token)
```

#### Step three

Validate the message has reached the store by envoking a GET call on your HL7
store.

```shell
curl -X GET \
     -H "Authorization: Bearer "$(gcloud auth print-access-token) \
     -H "Content-Type: application/json; charset=utf-8" \
     "https://healthcare.googleapis.com/v1beta1/projects/$PROJECT_ID/locations/$LOCATION/datasets/$DATASET_ID/hl7V2Stores/$HL7_STORE_ID/messages"
```

#### Step three

Navigate to the Healthcare API product in the GCP console, select FHIR viewer. \
Select **datastore,** then **fhirtore,** to see resources that were sent \
from the HL7 message and mapped to FHIR. Select a resource to view its contents.

#### Step four

Navigate to BigQuery, under your project drop-down select **fhirdata.** \
Tables should be populated based on the resources that were created.

## COMPLETE!

## OPTIONAL: Test using synthetic streaming data

--------------------------------------------------------------------------------

### MLLP Set Up

Follow the instructions to set up an MLLP adapter on kubernetes engine here: (https://cloud.google.com/healthcare/docs/how-tos/mllp-adapter?hl=en#deploying_the_mllp_adapter_to_google_kubernetes_engine

In the setup the set "--receiver_ip=0.0.0.0" You will use the “LoadBalancer
Ingress:2575” in a step downstream.

--------------------------------------------------------------------------------

### Create Synthetic HL7 Data via Simulated Hospital

Create a VM using UI - ensure full access to APIs are selected and HTTP/S
traffic is selected.

SSH into the VM.

```shell
sudo apt-get update
sudo apt-get install docker git curl gnupg
```

```shell
sudo apt install curl gnupg
curl -fsSL https://bazel.build/bazel-release.pub.gpg | gpg --dearmor > bazel.gpg
sudo mv bazel.gpg /etc/apt/trusted.gpg.d/
echo "deb [arch=amd64] https://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
```

```shell
sudo apt update && sudo apt install bazel
```

```shell
mkdir github
cd github/
git clone https://github.com/google/simhospital.git
cd simhospital/
LOCAL_DIR=$(pwd)
sudo apt-get install python-pip
bazel build //...
bazel test //...
```

This step will take a significant amount of time.

```shell
bazel run //cmd/simulator:simulator -- -local_path=${LOCAL_DIR} --output=mllp -mllp_destination=10.128.15.211:2575
# Note the above address 10.128.15.211:2575 comes from the MLLP install steps. It may be a different IP during set up.
```

```shell
curl -X GET \
     -H "Authorization: Bearer "$(gcloud auth print-access-token) \
     -H "Content-Type: application/json; charset=utf-8" \
     "https://healthcare.googleapis.com/v1/projects/$PROJECT_ID/locations/$LOCATION/datasets/$DATASET_ID/hl7V2Stores/$HL7_STORE_ID/messages"
```

--------------------------------------------------------------------------------
