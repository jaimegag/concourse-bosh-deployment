# Lite Concourse deployment with Bosh Director

A "lite" Concourse deployment will co-locate all Concourse components together.
These instructions and configuration deploy Concourse as a separate deployment
in an existing Bosh Director.

This is a single VM deployment, making it easy to get your feet wet, but it
won't help you scale later if you need it.

This approach can be useful when you want to leverage an existing Bosh environment
but you don't need a full fledged Concourse deployment. A good combination with
Bosh BootLoader to facilitate the creation of the Jumpbox and Bosh Director.

## Using Bosh BootLoader

The `bbl` project maintains documentation for standing up Bosh and deploying Concourse
 quickly and easily across a few supported IaaSes. Consult their
[`concourse.md`](https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/concourse.md)
docs for general information. Keep in mind those instructions point to the Concourse
Cluster deployment approach.

For convenience we provide here end-2-end steps to deploy BBL with Concourse on GCP with
single VM deployment.

### Deploying on GCP

#### Create a Service Account

In order for `bbl` to interact with GCP, a service account must be created.
```
gcloud iam service-accounts create <service account name>

gcloud iam service-accounts keys create --iam-account='<service account name>@<project id>.iam.gserviceaccount.com' <service account name>.key.json

gcloud projects add-iam-policy-binding <project id> --member='serviceAccount:<service account name>@<project id>.iam.gserviceaccount.com' --role='roles/editor'
```

#### Pave Infrastructure, Create a Jumpbox, and Create a BOSH Director for Concourse use

1. Export environment variables.
    ```
    export BBL_IAAS=gcp
    export BBL_GCP_REGION=
    export BBL_GCP_SERVICE_ACCOUNT_KEY=
    ```
    Service Account Key variable is the path + filename.

1. Create infrastructure, jumpbox, and bosh director.
    ```
    bbl plan —lb-type concourse
    bbl up
    ```

#### Prepare for Concourse Deployment

```
eval "$(bbl print-env)"

export EXTERNAL_HOST="$(bbl outputs | grep concourse_lb_ip | cut -d ' ' -f2)"
export STEMCELL_URL="https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-xenial-go_agent"

bosh upload-stemcell "${STEMCELL_URL}"
```

#### Customize & Deploy

```
git clone https://github.com/concourse/concourse-bosh-deployment.git

pushd concourse-bosh-deployment/lite-with-director

bosh deploy -d concourse concourse.yml \
    -l ../versions.yml \
    -v external_url="https://${EXTERNAL_HOST}" \
    -v external_host="${EXTERNAL_HOST}" \
    -v concourse_password=concourse \
    -v concourse_vm_type=small \
    -v concourse_persistent_disk_type=100GB \
    -v network_name=default \
    -v web_network_vm_extension=lb \
    -v deployment_name=concourse \
    -o operations/web-network-extension.yml \
    -o operations/privileged-http.yml \
    -o operations/privileged-https.yml \
    -o operations/tls.yml \
    -o operations/tls-vars.yml
```

#### Check it out
Once it's successfully done, we can simply check it out.

View the BOSH VMs' status:

```
bosh  vms

Using environment 'https://10.0.0.6:25555' as client 'admin'

Task 31. Done

Deployment 'concourse'

Instance                                        Process State  AZ  IPs       VM CID                                   VM Type  Active
concourse/7d33e60d-17ce-4e77-bd2b-160c6a715567  running        z1  10.0.1.0  vm-2a598437-4f8a-4836-76b7-8c6eb5b40133  small    true

1 vms

Succeeded
```

Open Concourse in Browser:

Access Concourse using the `$EXTERNAL_HOST`
```
open https://${EXTERNAL_HOST}/
```

Login via with username/password from below output:

```
fly -t ci login -c https://${EXTERNAL_HOST}/ -k
```
Credentials are: concourse / concourse
