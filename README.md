# kas-installer

KAS Installer allows the deployment and configuration of Managed Kafka Service
in a single K8s cluster.

## Contents
- [Prerequisites](#prerequisites)
- [Description](#description)
- [Usage](#usage)
- [Installation Modes](#installation-modes)
  - [Standalone Mode](#standalone)
  - [OCM Mode](#ocm)
- [Fleet Manager Parameter Customization](#fleet-manager-parameter-customization)
  - [Instance Types](#instance-types)
- [SSO Providers](#sso-providers)
- [Custom Components](#custom-components)
- [Using rhoas CLI](#using-rhoas-cli)
- [Running the User Interface](#running-the-user-interface)
- [Custom TLS](#custom-tls)
- [Legacy Scripts](#legacy-scripts)
- [Running E2E Test Suite (experimental)](#running-e2e-test-suite-experimental)

## Prerequisites

* [jq][jq]
* [curl][curl]
* [OpenShift][openshift]. In the future there are plans to make it compatible
  with native K8s. Currently an OpenShift dedicated based environment is needed
  (Currently needs to be a multi-zone cluster if you want to create a Kafka
  instance through the fleet manager by using `managed_kafka.sh`).
* [git][git_tool]
* [opm][opm] required to build custom kas-fleetshard OLM bundle from source (see [Custom Components](#custom-components))
* oc
* kubectl
* openssl CLI tool
* rhoas CLI (https://github.com/redhat-developer/app-services-cli)
* A user with administrative privileges in the OpenShift cluster and is logged in using `oc` or `kubectl`
* [yq][yq] if `kas-fleet-manager-service-template-params` is provided
* OSD Cluster with the following specs. Clusters with fewer/smaller compute nodes _may_ work, but have not been verified with kas-installer.
   * Plan `developer.x1`
      * 6 compute nodes
      * Size: m5.2xlarge
      * MultiAz: N/A
   * Plan `standard.x1`
      * 9 compute nodes (3 per zone)
      * Size: m5.2xlarge
      * MultiAz: True
   * Plan `standard.x2`
      * 12 compute nodes (4 per zone)
      * Size: m5.2xlarge
      * MultiAz: True

On Mac install:

* brew gsed
* brew coreutils
* brew openssl

## Description

KAS Installer deploys and configures the following components that are part of
Managed Kafka Service:
* MAS SSO
* KAS Fleet Manager
* Observability Operator (via KAS Fleet Manager)
* sharded-nlb IngressController
* KAS Fleet Shard and Strimzi Operators (via KAS Fleet Manager)

It deploys and configures the components to the cluster set in
the user's kubeconfig file.

Additionally, a single Data Plane cluster is configured ready to be used, in the
same cluster set in the user's kubeconfig file.

## Usage

### Deploy Managed Kafka Service
1. Create and fill the KAS installer configuration file `kas-installer.env`. Minimally, the
   values identified as [required] in [kas-installer-defaults.env](kas-installer-defaults.env) must be configured.
1. make sure you have run `oc login --server=<api cluster url|https://api.xxx.openshiftapps.com:6443>` to your target OSD cluster. You will be asked a password or a token
1. Run the KAS installer `kas-installer.sh` to deploy and configure Managed
   Kafka Service
1. Run `uninstall.sh` to remove KAS from the cluster.  You should remove any deployed Kafkas before runnig this script.

**Troubleshooting:** If the installer crashed due to configuration error in `kas-installer.env`, you often can rerun the installer after fixing the config issue.
It is not necessary to run uninstall before retrying.

## Installation Modes

### Standalone

Deploying a cluster with kas-fleet-manager in `standalone` is the default for kas-installer (when `OCM_SERVICE_TOKEN` is not defined).
In this mode, the fleet manager deploys the data plane components from OLM bundles.

**NOTE:**
In standalone mode, predefined bundles are used for Strimzi and KAS Fleetshard operators. To use a different bundle
you'll need to build a dev bundle and set either `STRIMZI_OLM_INDEX_IMAGE` or `KAS_FLEETSHARD_OLM_INDEX_IMAGE` environment variables.

### OCM

Installation with OCM mode allows kas-fleet-manager to deploy the data plane components as OCM addons. This mode can be
used by setting `OCM_SERVICE_TOKEN` to your [OCM offline token](https://console.redhat.com/openshift/token) and also setting
`OCM_CLUSTER_ID` to the idenfier for the OSD cluster used in deployment.

**NOTE:**
In OCM mode, it may take up to 10 minutes after the `kas-installer.sh` completes before the addon installations are ready. Until ready,
the fleet-manager API will reject new Kafka requests.

## Fleet Manager Parameter Customization

### Service Customization

The `kas-installer.sh` process will check for the presence of an executable named `kas-fleet-manager-service-template-params` in
the project root. When available, it will be executed with the expection that key/value pairs will be output to stdout. The output
will be used when processing the kas-fleet-manager's [service-template.yml](https://github.com/bf2fc6cc711aee1a0c2a/kas-fleet-manager/blob/main/templates/service-template.yml).
The name of the executable is intentially missing an extension to indicate that any language may be used that is known to the user
to be supported in their own environment.

Note that `oc process` requires that individual parameters are specified on a single line.

For example, to provide a custom `KAS_FLEETSHARD_OPERATOR_SUBSCRIPTION_CONFIG` parameter value to the fleet manager template,
something like the following may be used for a `kas-fleet-manager-service-template-params` executable:
```shell
#!/bin/bash

# Declare the subscription config using multi-line YAML
MY_CUSTOM_SUBSCRIPTION_CONFIG='---
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "500m"
env:
- name: SSO_ENABLED
  value: "true"
- name: MANAGEDKAFKA_ADMINSERVER_EDGE_TLS_ENABLED
  value: "true"
- name: STANDARD_KAFKA_CONTAINER_CPU
  value: 500m
- name: STANDARD_KAFKA_CONTAINER_MEMORY
  value: 4Gi
- name: STANDARD_KAFKA_JVM_XMS
  value: 1G
- name: STANDARD_ZOOKEEPER_CONTAINER_CPU
  value: 500m
'

# Quota override for my organization
MY_CUSTOM_ORG_QUOTA='
- id: <YOUR ORG HERE>
  max_allowed_instances: 10
  any_user: true
  registered_users: []
'

# Serialize to a single line as JSON (subset of YAML) to standard output
echo "KAS_FLEETSHARD_OPERATOR_SUBSCRIPTION_CONFIG='$(echo "${MY_CUSTOM_SUBSCRIPTION_CONFIG}" | yq e -o=json -I=0)'"

# Disable organization quotas (allows deployment of developer instances)
echo "REGISTERED_USERS_PER_ORGANISATION='[]'"
# Custom organization quota (allows deployment of standard instances), disabled/commented here
#echo "REGISTERED_USERS_PER_ORGANISATION='$(echo "${MY_CUSTOM_ORG_QUOTA}" | yq e -o=json -I=0)'"

```

### Secret Customization
Similar to the `kas-fleet-manager-service-template-params` previously described, the `kas-installer.sh` process will check for the presence of an executable named `kas-fleet-manager-secrets-template-params` in
the project root. When available, it will be executed with the expection that key/value pairs will be output to stdout. The output
will be used when processing the kas-fleet-manager's [secrets-template.yml](https://github.com/bf2fc6cc711aee1a0c2a/kas-fleet-manager/blob/main/templates/secrets-template.yml).

### Instance Types
The default kas-fleet-manager configuration enables the deployment of two instance types, `standard` and `developer`. Permission
to create an instance of a particular type depends on the organization's quota for the user creating the instance.

1. Creating a `developer` instance type requires that the user creating the instance does not have an instance quota. Use [parameter customization](#fleet-manager-parameter-customization) to set the `REGISTERED_USERS_PER_ORGANISATION` property to an empty array `[]`.
1. Creating a `standard` instance type requires that the user creating the instance _does_ have an instance quota. Use [parameter customization](#fleet-manager-parameter-customization) to set the `REGISTERED_USERS_PER_ORGANISATION` property to an array containing the user's org. See the `MY_CUSTOM_ORG_QUOTA` variable in the sample script for an example.

To create a `standard.x2` (or other non-default types) instance type, the `plan` must be provided to the Kafka create request. If using the `managed_kafka.sh` script, the `--plan` argument may be used:
```shell
managed_kafka.sh --create mykafka --plan standard.x2
```

### Deploying to multiple clusters/zones
In order to allow clients to deploy Kafka clusters to multiple clusters and/or zones, the following parameters must be customized:

   * KUBE_CONFIG in `kas-fleet-manager-secrets-template-params` (standalone only)
   * SUPPORTED_CLOUD_PROVIDERS in `kas-fleet-manager-service-template-params`
   * CLUSTER_LIST in `kas-fleet-manager-service-template-params`

The following sections describe the contents of these parameters in detail.

#### SUPPORTED_CLOUD_PROVIDERS

The SUPPORTED_CLOUD_PROVIDERS parameter contains a list of supported cloud providers in a yaml format.  See [deploying-kas-fleet-manager-to-openshift.md](https://github.com/bf2fc6cc711aee1a0c2a/kas-fleet-manager/blob/main/docs/deploying-kas-fleet-manager-to-openshift.md) for more details.


#### CLUSTER_LIST

The CLUSTER_LIST parameter contains a list clusters for kas fleet manager.  See [data-plane-osd-cluster-options.md](https://github.com/bf2fc6cc711aee1a0c2a/kas-fleet-manager/blob/main/docs/data-plane-osd-cluster-options.md) for more details.

#### KUBE_CONFIG (standalone only)

When running in standalone mode, the KUBE_CONFIG contains cluster connection information for each cluster to which Kafka can be deployed.  There should be one entry in KUBE_CONFIG for each entry in CLUSTER_LIST.  For example, to generate a KUBE_CONFIG for two clusters:

```shell
oc login -u kubeadmin -p <first_cluster_password> <first_cluster_url>
oc config view --minify --raw > /tmp/c1.yaml
oc login -u kubeadmin -p <second_cluster_password> <second_cluster_url>
oc config view --minify --raw > /tmp/c2.yaml
(KUBECONFIG=/tmp/c1.yaml:/tmp/c2.yaml oc config view --raw)
```

The KUBE_CONFIG value must be base64 encoded.  For example, the following may be used to encode the value in the `kas-fleet-manager-secrets-template-params`:

```shell
echo "KUBE_CONFIG='$(echo "${KUBE_CONFIG}"  | yq -o=json -I=0 |  ${BASE64} -w0)'"
```

### Custom Domain for the Data Plane

Custom domain name registration for the data plane routes may be enabled using the following configurations in the kas-fleet-manager
[customization](#fleet-manager-parameter-customization) scripts. If `KAFKA_DOMAIN_NAME` is not specified, the default value of in the
KFM template will be used.
1. `kas-fleet-manager-service-template-params`
    ```shell
    echo "ENABLE_KAFKA_CNAME_REGISTRATION='true'"
    echo "KAFKA_DOMAIN_NAME='<your domain here>'"
    ```
1. `kas-fleet-manager-secrets-template-params`
   ```shell
    echo "ROUTE53_ACCESS_KEY='<your AWS Route53 access key>'"
    echo "ROUTE53_SECRET_ACCESS_KEY='<your AWS Route53 secret access key>'"
    ```
With this configuration, you will likely also want to provide a certificate and key, either by generating one using
the [custom TLS instructions](#custom-tls), or by directly providing values for `KAFKA_TLS_CERT` and `KAFKA_TLS_KEY`
in the `kas-installer.env` file or environment.

## SSO Providers

Configuration of kas-fleet-manager's SSO providers is done by setting the `SSO_PROVIDER_TYPE` configuration variable. When not set, the default provider is `mas_sso`. To use RH SSO,
the variable may be set to `redhat_sso` and additional configuration can be provided for `REDHAT_SSO_HOSTNAME` (default sso.stage.redhat.com), `REDHAT_SSO_REALM` (default `redhat-external`),
`REDHAT_SSO_CLIENT_ID` (required), and `REDHAT_SSO_CLIENT_SECRET` (required). See the description for each variable in the [kas-installer-defaults.env](kas-installer-defaults.env)
file for more information.

### MAS SSO Resources

MAS SSO (must always available to support kas-fleet-manager admin operations) may optionally be installed with modified
resource requests and limits. The two environment variable should be in JSON format on a single line.

Configuration of the SSO operator can be done using the `MAS_SSO_OPERATOR_SUBSCRIPTION_CONFIG`
variable, the format of which must conform to the [Subscription Config object](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/subscription-config.md).
Example with custom CPU and memory requests:
```shell
MAS_SSO_OPERATOR_SUBSCRIPTION_CONFIG='{"resources": { "requests": { "cpu": "500m", "memory": "512Mi" }, "limits": { "cpu": "500m", "memory": "512Mi" }}}'
```

The MAS SSO Keycloak instance may be configured using the `MAS_SSO_KEYCLOAK_RESOURCES` variable. The format is the same
as the Kubernetes `resources` object for other types. See `mas-sso/keycloak.yaml` for the defaults.
Example custom CPU and memory requests:
```shell
MAS_SSO_KEYCLOAK_RESOURCES='{ "requests": { "cpu": "500m", "memory": "768Mi" }, "limits": { "cpu": "500m", "memory": "768Mi" }}'
```

## Custom Components

Custom-built components are supported for kas-fleet-manager and kas-fleetshard.

1. kas-fleet-manager: see the documentation for `KAS_FLEET_MANAGER_IMAGE_BUILD` in the [kas-installer-defaults.env](kas-installer-defaults.env) file.
1. kas-fleetshard: prior to running `kas-installer.sh`, execute `operators/generate-kas-fleetshard-olm-bundle.sh`, passing the required
   parameters (see `operators/generate-kas-fleetshard-olm-bundle.sh --help` for details). The script will build kas-fleetshard from source and an OLM bundle index,
   optionally updating the `kas-installer.env` file with the configuration. Running `kas-installer.sh` will deploy the custom-built operator.

## Using rhoas CLI

Use `./rhoas_login.sh` as a short cut to login to the CLI.  Login using the username you specified as RH_USERNAME in the env file.  The password is the same as the RH_USERNAME value.
Alternatively (and likely preferably), use `./login-all.sh` which makes sure you are also logged to your OpenShift Dedicated cluster.

There are a couple of things that are expected not to work when using the RHOAS CLI with a kas-installer installed instance.  These are noted below.

### Service Account Maintenace

1. To create an account, run `rhoas service-account create --short-description foo --file-format properties`.
1. To list existing service accounts, run `rhoas service-account list`.
1. To remove an existing service account, run `rhoas service-account delete --id=<ID of service account>`.

### Kafka Instance Maintenance

1. To create a cluster, run `rhoas kafka create --bypass-checks --provider aws --region us-east-1 --name <clustername>`.  Note that `--bypass-checks` is required as the T&Cs endpoint will not
   exist in your environment. The provider and region must be passed on the command line.
1. To list existing clusters, run `rhoas kafka list`
1. To remove an existing cluster, run `rhoas kafka delete --name <clustername>`.

#### Kafka topics / consumergroups / ACL

To use these cli featurs, you must set `MANAGEDKAFKA_ADMINSERVER_EDGE_TLS_ENABLED=true` in your `kas-installer.env` so that the admin-server will run over TLS (edge terminated).

1. To create a topic `rhoas kafka topic create --name=foo`
1. To grant access `rhoas kafka acl grant-access  --topic=foo --all-accounts --producer`
etc.

## Running the User Interface

*Only with `SSO_PROVIDER_TYPE=redhat_sso` and `REDHAT_SSO_HOSTNAME=sso.redhat.com` (production)*

See the [Running the UI wiki page](https://github.com/bf2fc6cc711aee1a0c2a/kas-installer/wiki/Running-the-UI) for more
detailed instructions.

Local instances of the [app-services-ui](https://github.com/redhat-developer/app-services-ui),
[kas-ui](https://github.com/bf2fc6cc711aee1a0c2a/kas-ui), and [kafka-ui](https://github.com/bf2fc6cc711aee1a0c2a/kafka-ui)
may be run locally by using the `ui/install.sh` script. Running the UI installation will start three containers using
`podman` or `docker` (auto-detected, but can be forced by setting `CONTAINER_CLI` to `docker` or `podman`)
with a main entrypoint of `https://127.0.0.1:1337`. The IP must be configured with name `prod.foo.redhat.com` in the user's
local `/etc/hosts` file.
```
...
127.0.0.1 prod.foo.redhat.com
...
```

The repository and branch (or tag/commit) maybe configured in the `kas-installer.env` file using the following variables
- `APP_SERVICES_UI_GIT_URL`
- `APP_SERVICES_UI_GIT_REF`
- `KAS_UI_GIT_URL`
- `KAS_UI_GIT_REF`
- `KAFKA_UI_GIT_URL`
- `KAFKA_UI_GIT_REF`

*Note*, when navigating to `https://prod.foo.redhat.com:1337/`, you may be prompted to login to `sso.redhat.com` as well
as the local MAS-SSO instance. The credentials for MAS-SSO should be the value of the `RH_USERNAME` variable for both
username and password.

## Custom TLS

Users may provide custom-generated TLS certificates using the `gen-certs.sh` script. The output is placed in the `certs`
directory (ignored by git) and includes a CA certificate, CA key, server certificate, and server key. Each time the script
is run, the server files will be replaced, but the CA certificate and key will be retained. The server certificate is issued
specifically for the current session's domain name, as determined by the `K8S_CLUSTER_DOMAIN` variable. To configure the
certificates for a Kafka instance, set the following variables in `kas-installer.env`, where `KAS_INSTALLER_HOME` is the
path to the project root:
```shell
KAFKA_TLS_CERT="$(cat ${KAS_INSTALLER_HOME}/certs/server-cert.pem)"
KAFKA_TLS_KEY="$(cat ${KAS_INSTALLER_HOME}/certs/server-key.pem)"
```

The CA certificate may be (for example) imported to your browser, enabling the generated server certificates used by
the admin API endpoint and the UI to be trusted during testing without the needing to trust them individually.

## Legacy scripts

Please favour using the rhoas command line.  These scripts will be remove at some point soon.

### Service Account Maintenance

The `service_account.sh` script supports creating, listing, and deleting service accounts.

1. To create an account, run `service_account.sh --create`. The new service account information will be printed to the console. Be sure to retain the `clientID` and `clientSecret` values to use when generating an access token or for connecting to Kafka directly.
1. To list existing service accounts, run `service_account.sh --list`.
1. To remove an existing service account, run `service_account.sh --delete <ID of service account>`.

### Generate an Access Token
1. Run `get_access_token.sh` using the `clientID` and `clientSecret` as the first and second arguments. The generated access token and its expiration date and time will be printed to the console.

### Kafka Instance Maintenance

The `managed_kafka.sh` script supports creating, listing, and deleting Kafka clusters.

1. To create a cluster, run `managed_kafka.sh --create <cluster name>`. Progress will be printed as the cluster is prepared and provisioned.
1. To list existing clusters, run `managed_kafka.sh --list`.
1. To remove an existing cluster, run `managed_kafka.sh --delete <cluster ID>`.
1. To patch an existing cluster (for instance changing a strimzi version), run ` managed_kafka.sh --admin --patch  <cluster ID> '{ "strimzi_version": "strimzi-cluster-operator.v0.23.0-3" }'`
1. To use kafka bin scripts against pre existing kafka cluster, run `managed_kafka.sh --certgen <kafka id> <Service_Account_ID> <Service_Account_Secret>`. If you do not pass the <Service_Account_ID> <Service_Account_Secret> arguments, the script will attempt to create a Service_Account for you. The cert generation is already performed at the end of `--create`. Point the `--command-config flag` to the generated app-services.properties in the working directory.
* If there is already 2 service accounts pre-existing you must delete 1 of them for this script to work

### Access the Kafka Cluster using command line tools

To use the Kafka Cluster that is created with the `managed_kafka.sh` script with command line tools like `kafka-topics.sh` or `kafka-console-consumer.sh` do the following.

1. run `tool_access.sh <cluster name>`.  This will generate the certificate / truststore and `app-services.properties` file. 
   It will also create a service account and grant GROUP and TOPIC permissions.  The bootstrap host will be displayed upon completion, and is also in the properties file as bootstrap.servers.
1. Execute your tool like `kafka-topics.sh --bootstrap-server <bootstrap-host>:443 --command-config app-services.properties --topic foo --create --partitions 9`
1. If you want to use a different service account, you may edit the `app-services.properties` file and update the username and password with `clientID` and `clientSecret`

### Running E2E Test Suite (experimental)

1. Install all cluster components using `kas-installer.sh`
1. Clone the [e2e-test-suite][e2e_test_suite] repository locally and change directory to the test suite project root
1. Generate the test suite configuration with `${KAS_INSTALLER_DIR}/e2e-test-config.sh > config.json`. When using `kas-installer.sh`
   with the [`redhat_sso` provider type](#sso-providers), the user accounts to be used by the E2E tests must be preconfigured. The
   `e2e-test-config.sh` script will set all variables named like `E2E_USER_` in the test suite's `config.json` file. The `E2E_USER_`
   prefix is not included in the JSON configuration.

   For example, an environment variable `E2E_USER_PRIMARY_USERNAME='my-primary-user'` will be added to `config.json` as key/value
   pair `"PRIMARY_USERNAME": "my-primary-user"`.
1. Execute individual test classes:
   - `./hack/testrunner.sh test KafkaAdminPermissionTest`
   - `./hack/testrunner.sh test KafkaInstanceAPITest`
   - `./hack/testrunner.sh test KafkaCLITest`

[git_tool]:https://git-scm.com/downloads
[jq]:https://stedolan.github.io/jq/
[openshift]:https://www.openshift.com/
[curl]:https://curl.se/
[e2e_test_suite]:https://github.com/bf2fc6cc711aee1a0c2a/e2e-test-suite
[opm]:https://github.com/operator-framework/operator-registry
[yq]:https://github.com/mikefarah/yq
