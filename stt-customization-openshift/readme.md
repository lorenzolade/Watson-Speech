# Deploy Customizable STT on OpenShift

IBM Watson Speech to Text (STT) Library for Embed transcribes written text from spoken audio in a number of languages. The service leverages machine learning to combine knowledge of grammar, language structure, and the composition of audio and voice signals to accurately transcribe the human voice.

The STT service can be deployed either with or without customization. A customizable deployment allows users to update the service to understand how to transcribe words. While a non-customizable deployment requires only a single container image and can easily be deployed with a container runtime like Docker or Podman, a customizable deployment requires multiple images and a PostgreSQL database. It is recommended to use the Helm chart to install on an OpenShift cluster.

This tutorial walks you through the steps install a customizable STT service in OpenShift. We will use [this](https://github.com/IBM/ibm-watson-embed-charts/tree/main/charts/ibm-watson-stt-embed) helm chart to deploy the service.

## Reference Architecture

![Diagram](architecture-stt.png)

## Prerequisites

- Install [Helm 3](https://helm.sh/docs/intro/install/).
- Ensure you have an [entitlement key](https://myibm.ibm.com/products-services/containerlibrary). You may need to create one. This key is required to access [images](https://www.ibm.com/docs/en/watson-libraries?topic=i-accessing-files) used in this tutorial.
- Set an environment variable.

  ```sh
  export IBM_ENTITLEMENT_KEY=<Set the entitlement key>
  ``` 

- S3 compatible storage. Below we give instructions on setting this up in IBM Cloud. For other clouds, use the instructions from the cloud provider.
- PostgreSQL Database is required to manage metadata related to customization
- An OpenShift Cluster on which you will deploy the service.

## S3 Compatible Storage

For customization, you will need an S3 compatible storage service that supports HMAC (access key and secret key) credentials. Watson Speech requires a bucket that it can read and write to. The bucket will be populated with stock models at install time and will also store customization artifacts, including training data and trained models.

### Create an S3 Bucket on IBM Cloud

Here are the steps to obtain IBM Cloud S3 bucket HMAC credentials and endpoint. If you are using a different cloud, then use the instructions given by the cloud provider to set it up.

1. Log in to [IBM Cloud](https://cloud.ibm.com/login).
2. From the IBM Cloud Dashboard, click the Cloud Object Storage service instance that you want to work with.
3. To get the Bucket name, click `Buckets` in the left pane and select your preferred bucket name and make a note.
4. To get the endpoint URLs, click `Endpoint` in the left pane and select your preferred location or region.
   1. Copy your preferred public `endpoint` (for example, s3.us-east.cloud-object-storage.appdomain.cloud) and use it as the value for the `Endpoint URL` field (or `endpointUrl` parameter).
   2. Copy your preferred location or `region` (for example, us-east) and use it as the value for the Region field (or region parameter).
5. To view the service credentials, click Service credentials in the left pane, and then click View credentials. (If you want to define new credentials, click New credential, click Advanced options, then select Include HMAC Credential.)
6. Copy the `cos_hmac_keys/secret_access_key` value and use it as the value for the `Secret access key` field (or `secretAccessKey parameter`).
7. Copy the `cos_hmac_keys/access_key_id` value and use it as the value for the `Access key ID` field (or `accessKeyId parameter`).

### Set S3 Information in Environment Variables

Set the S3 crededentials and information into the following environment variables. These variables will be used when deploying the STT Helm chart.

```sh
export S3_BUCKET_NAME=<Bucket name you found in step 3 >
export S3_REGION=<Region can be found in step 4.1 when you select your bucket>
export S3_ENPOINT_URL=<Endpoint URL you found in step 4>
export S3_SECRET_KEY=<secretAccessKey you found in step 6>
export S3_ACCESS_KEY=<accessKeyId you found in step 7>
```

The commands you use to set the variables should look similar to the following.

```sh
export S3_BUCKET_NAME=speech-embed
export S3_REGION=us-east
export S3_ENPOINT_URL=https://s3.us-east.cloud-object-storage.appdomain.cloud
export S3_SECRET_KEY=12a3bcd4567890ef123g4567890hij12k1m3n4567o8901p2
export S3_ACCESS_KEY=1a2dfbc3d45678901ef2g3h45678i90jkl
```

## Set PostgreSQL information in Environment Variables

A PostgreSQL database is required to manage metadata related to customization. The customization container uses TLS to Postgres, but always sets up the connection with a NonValidatingFactory which does not do certificate validation. 

Set the Database connection information into the following environment variables. These variables will be used when deploying the STT Helm chart.

```sh
export POSTGRES_HOST=<Postgresql hostname>
export POSTGRES_USER=<Postgresql username>
export POSTGRES_PASSWORD=<Postgresql password>
```

## Install Speech to Text Helm Chart

Clone the Helm chart Github repository.

```sh
git clone https://github.com/IBM/ibm-watson-embed-charts.git
cd ibm-watson-embed-charts/charts
```

The containers deployed by this chart come from the IBM Entitled Registry. You must create a Secret with credentials to pull from this registry. The default name is `ibm-entitlement-key`, but this can be changed by updating the value of `imagePullSecrets`.

An example command to create the pull secret:

```sh
 oc create secret docker-registry ibm-entitlement-key \
  --docker-server=cp.icr.io \
  --docker-username=cp \
  --docker-password=$IBM_ENTITLEMENT_KEY \
  --docker-email=<your-email>
```

By default, the models that are enabled are `en-US_Multimedia` and `en-US_Telephony` with defaultModel set to `en-US_Multimedia`.

Helm charts have configurable values that can be set at install time. To configure, e.g. enable additional models, refer to the base `values.yaml`. Values can be changed using `--set` or using YAML files specified with `-f/--values`. Below we set values using `--set` parameter.

```sh
helm install stt-release ./ibm-watson-stt-embed \
--set license=true \
--set nameOverride=stt \
--set models.enUSTelephony.enabled=false \
--set postgres.host=${POSTGRES_HOST} \
--set postgres.user=${POSTGRES_USER} \
--set postgres.password=${POSTGRES_PASSWORD} \
--set objectStorage.endpoint=${S3_ENPOINT_URL} \
--set objectStorage.region=${S3_REGION} \
--set objectStorage.bucket=${S3_BUCKET_NAME} \
--set objectStorage.accessKey=${S3_ACCESS_KEY} \
--set objectStorage.secretKey=${S3_SECRET_KEY}
```

## Verify the Chart

See the instruction (from NOTES.txt within chart) after the Helm installation completes for chart verification. The instruction can also be viewed by running the following command.

```sh
helm status stt-release
```

For basic usage of customization, see the [documentation](https://www.ibm.com/docs/en/watson-libraries?topic=containers-customization-example).

The complete API reference for Watson Speech to Text can be found [here](https://cloud.ibm.com/apidocs/speech-to-text).

## Test the Service

In one terminal, create a proxy through the service.

```sh
oc proxy
```

Download an example audio file as `example.flac`:

```sh
curl --url https://github.com/watson-developer-cloud/doc-tutorial-downloads/raw/master/speech-to-text/0001.flac \
      -sLo example.flac
```

Send a `recognize` request using the downloaded file:

```sh
curl --url "http://localhost:8001/api/v1/namespaces/stt-demo/services/https:stt-release-runtime:https/proxy/speech-to-text/api/v1/recognize?model=en-US_Multimedia" \
      --header "Content-Type: audio/flac" \
      --data-binary @example.flac
```