

# Devtron App Creation Guide

---

## 1. Prerequisites

Export the following environment variables:

```bash
export DEVTRON_URL=https://<devtron-url>
export DEVTRON_TOKEN=<your-api-token>
```



---

## 2. Create Application

Run the following command to create a new application.


```bash
curl "$DEVTRON_URL/orchestrator/app" \
  -H "token: $DEVTRON_TOKEN" \
  -X POST \
  -d '{
    "appName": "sample-app",
    "teamId": 1,
    "templateId": 0,
    "labels": [
      {
        "key": "env",
        "value": "dev",
        "propagate": true
      }
    ]
  }'
```

> **Note:**
> - No placeholders to replace in this step. Save the `appId` from the response for later use.


---

## 3. Save Git Repository

Link your Git repository to the application.


```bash
curl "$DEVTRON_URL/orchestrator/app/material" \
  -H "token: $DEVTRON_TOKEN" \
  --data-raw '{
    "appId": <AppId>,
    "material": [
      {
        "url": "https://github.com/badal773/test",
        "checkoutPath": "",
        "gitProviderId": 1,
        "fetchSubmodules": false,
        "filterPattern": []
      }
    ]
  }'
```

> **Note:**
> - Replace `<AppId>` with the application ID from step 2.
> - Save the `gitMaterialId` from the response for later use.


---

## 4. Save Build Configuration

Configure the build pipeline for your application:


```bash
curl "$DEVTRON_URL/orchestrator/app/ci-pipeline" \
  -H "token: $DEVTRON_TOKEN" \
  --data-raw '{
    "id": null,
    "appId": <AppId>,
    "dockerRegistry": "<ContainerRegistryName>",
    "dockerRepository": "devtron-poc",
    "beforeDockerBuild": [],
    "ciBuildConfig": {
      "ciBuildType": "self-dockerfile-build",
      "dockerBuildConfig": {
        "dockerfileRelativePath": "Dockerfile",
        "dockerfilePath": "./Dockerfile",
        "args": {},
        "dockerfileRepository": "test",
        "targetPlatform": ""
      },
      "gitMaterialId": <GitMaterialId>,
      "buildContextGitMaterialId": <GitMaterialId>,
      "useRootBuildContext": true
    },
    "afterDockerBuild": [],
    "appName": ""
  }'
```

> **Note:**
> - Replace `<AppId>` with the application ID from step 2.
> - Replace `<GitMaterialId>` with the git material ID from step 3.
> - Replace `<ContainerRegistryName>` with your container registry name.


---

## 5. Deployment Template

### a. Get Deployment Chart ID

Get the chart reference ID for the deployment:


```bash
curl -s "$DEVTRON_URL/orchestrator/chartref/autocomplete/<AppId>" \
  -H "token: $DEVTRON_TOKEN" \
  | jq '.result.chartRefs[] | select(.name == "Deployment" and .version == "4.21.0")'
```

> **Note:**
> - Replace `<AppId>` with the application ID from step 2.
> - Save the `chartRefId` from the output for later use.


### b. Get Base Deployment Template

Fetch the base deployment template:


```bash
curl -s "$DEVTRON_URL/orchestrator/app/template/<AppId>/<ChartId>" \
  -H "token: $DEVTRON_TOKEN" \
  | jq .result.globalConfig > default-values.json
```

> **Note:**
> - Replace `<AppId>` with the application ID from step 2.
> - Replace `<ChartId>` with the chart reference ID from step 5a.

### c. Edit the JSON

Add required fields to the deployment template:


```bash
jq '. + {resourceName: "BaseDeploymentTemplate"} \
   | . + {valuesOverride: .defaultAppOverride} \
   | . + {appId: <AppId>, chartRefId: <ChartId>}' default-values.json > output.json
```

> **Note:**
> - Replace `<AppId>` with the application ID from step 2.
> - Replace `<ChartId>` with the chart reference ID from step 5a.

### d. Save the Deployment Template

Save the edited deployment template:


```bash
curl "$DEVTRON_URL/orchestrator/app/template" \
  -H "token: $DEVTRON_TOKEN" \
  -X POST \
  --http2 \
  -d @output.json
```

> **Note:**
> - No placeholders to replace in this step. Uses the `output.json` file generated in the previous step.



---

## 6. Create CI Pipelines (CI_JOB)

Create a CI pipeline for your application:


```bash
curl "$DEVTRON_URL/orchestrator/app/ci-pipeline/patch" \
  -H "token: $DEVTRON_TOKEN" \
  --data-raw '{
    "appId": <AppId>,
    "appWorkflowId": 0,
    "action": 0,
    "ciPipeline": {
      "active": true,
      "ciMaterial": [
        {
          "gitMaterialId": <GitMaterialId>,
          "id": 0,
          "source": {
            "type": "SOURCE_TYPE_BRANCH_FIXED",
            "value": "main",
            "regex": ""
          }
        }
      ],
      "dockerArgs": {},
      "externalCiConfig": {},
      "id": 0,
      "isExternal": false,
      "isManual": true,
      "name": "prod-workflow",
      "linkedCount": 0,
      "scanEnabled": false,
      "pipelineType": "CI_JOB",
      "customTag": {
        "tagPattern": "",
        "counterX": 0
      },
      "workflowCacheConfig": {
        "type": "INHERIT",
        "value": true,
        "globalValue": true
      },
      "preBuildStage": {},
      "postBuildStage": {},
      "dockerConfigOverride": {}
    }
  }'
```

> **Note:**
> - Replace `<AppId>` with the application ID from step 2.
> - Replace `<GitMaterialId>` with the git material ID from step 3.

---
