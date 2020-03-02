# Developer Experience Starter Kit Monitoring Toolchain

This repository provides an [IBM Cloud DevOps](https://cloud.ibm.com/devops) [open toolchain](https://github.com/open-toolchain) template for monitoring Developer Experience starter kits. It uses deployment and scripting assets available in [this repository](https://github.com/IBM/devex-skit-assets) to streamline verification testing for [published starter kits](https://cloud.ibm.com/developer/appservice/starter-kits).

IBM internally uses instances of this template to ensure successful out-of-the-box functionality of IBM-provided starter kits. However, the template can be used as-is to create similar toolchain instances in a developer's own IBM Cloud account or as a starting point for developing custom monitoring pipelines.

## DevOps Service Integrations

This template provides integration with a public GitHub repository to track code changes as well as Delivery Pipelines for various deployment targets.

### GitHub
The toolchain can connect to a public GitHub repository and monitor one of its branches for changes (by default, the master branch is selected). This integration can react to commits on the specified branch or on pull requests. After creating the toolchain instance using the template, modify the input of the `Build` stage in each pipeline to change the configuration.

### Continuous Delivery Pipelines
Pipelines are provided to deploy to Cloud Foundry environments or Kubernetes clusters using Helm and/or Knative. Each pipeline contains common `Build` and `Deploy` stages as well as an additional `Verification` stage that verifies the app is functioning properly after deployment. This is accomplished by executing an `experience_test.sh` script contained in the app's repository which executes UI or service tests to ensure app endpoints are working as expected.

Each stage has one or more jobs whose scripts define the actions that take place when the stage is running. The scripts in the template largely delegate to scripts found in the [starter kit assets repository](https://github.com/IBM/devex-skit-assets/tree/master/scripts) but can be modified entirely or in part to meet the application's needs.

### Slack
Delivery Pipelines can send [Slack](https://slack.com/) messages to a specific channel using [Slack bots](https://api.slack.com/bot-users). By providing a webhook for the bot and the channel name, the pipelines can notify users in the channel when various stages fail or if the `Verification` stage passes or fails.

### PagerDuty
By providing a [PagerDuty](https://www.pagerduty.com/) API token and service integration key, the included pipelines can create an alert to notify on-call developers of breaking changes to application code during the `Verification` stage. Other stages can incorporate this integration by modifying their scripts. Alerts are created using the [Events API v2](https://v2.developer.pagerduty.com/docs/events-api-v2).

## Creating toolchain instances

There are two ways of creating toolchain instances based on this template.

### DevOps Console
To get started in the DevOps console, click this button:
[![Create toolchain](https://cloud.ibm.com/devops/graphics/create_toolchain_button.png)](https://cloud.ibm.com/devops/setup/deploy?repository=https%3A%2F%2Fgithub.com%2FIBM%2Fdevex-skit-monitor-toolchain&env_id=ibm:yp:us-south)

Fill out the required information for the GitHub and Delivery Pipeline services. Note that for successful deployment, the application's deployment assets should be included with the application code. Alternatively, modify the scripts in the Build stages of each pipeline in order to download deployment assets before the application is built and sent to subsequent stages. After creating an instance of the toolchain, click on the gear icon of each stage to modify the scripts of its jobs.

For example, [this script](https://github.com/IBM/devex-skit-assets/blob/master/scripts/build_cr.sh) is used in the [Knative pipeline](https://github.com/IBM/devex-skit-monitor-toolchain/blob/master/.bluemix/pipeline_knative.yml) in the template to download assets before building the container image used during deployment. You can copy the contents of this script, paste them into the Knative pipeline's Build stage editor, and modify accordingly.

### Headless 
This toolchain template also supports headless creation using `curl` commands against the DevOps REST APIs. See the [Toolchain Creation Page documentation](https://github.com/open-toolchain/sdk/wiki/Toolchain-Creation-page-parameters) for details.

For example, the following script can be used to create an instance of the toolchain by passing in appropriate argument values and modifying other environment variables like the `PROD_CLUSTER_NAME` to match the developer's environment. You must log into the IBM Cloud using the [IBM Cloud Developer Tools](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started) before executing this script.

```
#!/bin/bash
# set -x

SKIT_NAME=$1
ENABLE_CF=$2
ENABLE_HELM=$3
ENABLE_KNATIVE=$4
SLACK_WEBHOOK=$5
OWNER_SLACK_CHANNEL=$6
PAGERDUTY_API_TOKEN=$7
PAGERDUTY_SVC_NAME=$8

TEMPLATE_GIT_URL="https%3A%2F%2Fgithub.com%2FIBM%2Fdevex-skit-monitor-toolchain"
TEMPLATE_GIT_BRANCH="master"
DEVX_SKIT_ASSETS_GIT_BRANCH="master"
SKIT_REPO_URL="https%3A%2F%2Fgithub.com%2FIBM%2F$SKIT_NAME"
TOOLCHAIN_NAME=$SKIT_NAME-code-monitor
RUN_PIPELINES=true
# TODO the query param enablePDAlerts isn't getting passed down when creating the toolchain
ENABLE_PD_ALERTS=true

CONTAINER_REGISTRY_REGION="us-south"
CONTAINER_REGISTRY_NAMESPACE="my_namespace"
PROD_CLUSTER_NAME="my-kube-cluster"
PROD_CLUSTER_NAMESPACE="default"

IBM_CLOUD_REGION=$(ibmcloud target |grep Region | sed 's/Region\: [ ]*//g')
IBM_CLOUD_REGION=$(echo -e "${IBM_CLOUD_REGION}" | tr -d '[:space:]')
PROD_REGION_ID="ibm:yp:$IBM_CLOUD_REGION"

ORG=$(ibmcloud target |grep Org | sed 's/Org\:[ ]*//g')
ORG=$(echo -e "${ORG}" | tr -d '[:space:]')
ORG_GUID=$(ibmcloud --quiet cf org $ORG --guid)

SPACE=$(ibmcloud target |grep Space | sed 's/Space\: [ ]*//g')
SPACE=$(echo -e "${SPACE}" | tr -d '[:space:]')
SPACE_GUID=$(ibmcloud --quiet cf space $SPACE --guid)

RESOURCE_GROUP=$(ibmcloud target |grep "Resource group" | sed 's/Resource group\:  [ ]*//g')
RESOURCE_GROUP=$(echo -e "${RESOURCE_GROUP}" | tr -d '[:space:]')
RESOURCE_GROUP_ID=$(ibmcloud resource group $RESOURCE_GROUP --id)
RESOURCE_GROUP_ID=$(echo -e "${RESOURCE_GROUP_ID}" | tr -d '[:space:]')

echo "Checking for existing toolchain named $TOOLCHAIN_NAME..."
toolchain_get_count=$(ibmcloud dev toolchain-get $TOOLCHAIN_NAME --json | jq '.items|length')
if [ "0" == $toolchain_get_count ]; then
    echo "Existing toolchain not found, creating a new one..."
else
    echo "Existing toolchain found, deleting it..."
    ibmcloud dev toolchain-delete $TOOLCHAIN_NAME --force
fi

echo "Creating new toolchain $TOOLCHAIN_NAME..."
curl -X POST -H "Authorization: $IAM_TOKEN" -H "Accept: application/json" \
  -d "repository=$TEMPLATE_GIT_URL&branch=$TEMPLATE_GIT_BRANCH&autocreate=true&apiKey=$IAM_KEY&env_Id=ibm:yp:$IBM_CLOUD_REGION&resourceGroupId=$RESOURCE_GROUP_ID&toolchainName=$TOOLCHAIN_NAME&sourceRepoUrl=$SKIT_REPO_URL&skitAssetsBranch=$DEVX_SKIT_ASSETS_GIT_BRANCH&prodRegion=$PROD_REGION_ID&prodResourceGroup=$RESOURCE_GROUP&prodClusterName=$PROD_CLUSTER_NAME&prodClusterNamespace=$PROD_CLUSTER_NAMESPACE&prodOrganization=$ORG&prodSpace=$SPACE&registryRegion=ibm:yp:$CONTAINER_REGISTRY_REGION&registryNamespace=$CONTAINER_REGISTRY_NAMESPACE&enableCF=$ENABLE_CF&enableHelm=$ENABLE_HELM&enableKnative=$ENABLE_KNATIVE&slackWebhookURL=$SLACK_WEBHOOK&slackOwnersChannel=$OWNER_SLACK_CHANNEL&pagerDutyAPIToken=$PAGERDUTY_API_TOKEN&pagerDutySvcName=$PAGERDUTY_SVC_NAME&enablePDAlerts=$ENABLE_PD_ALERTS" \
  -k https://cloud.ibm.com/devops/setup/deploy?env_id=ibm:yp:us-south
echo "Toolchain $TOOLCHAIN_NAME created successfully!"
```
