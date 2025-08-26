
## 1. Prerequisites

Export the following environment variables:

```bash
export DEVTRON_URL=https://<devtron-url>
export DEVTRON_TOKEN=<your-api-token>
````


## 2. Create Application

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

Save the **`appId`** from the response.

## 3. Save Git Repository

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

Save the **`gitMaterialId`** from the response.

## 4. Save Build Configuration

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

## 5. Deployment Template

#### a. Get Deployment Chart ID

```bash
curl -s "$DEVTRON_URL/orchestrator/chartref/autocomplete/<AppId>" \
  -H "token: $DEVTRON_TOKEN" \
  | jq '.result.chartRefs[] | select(.name == "Deployment" and .version == "4.21.0")'
```

Save the **`chartRefId`**.

#### b. Get Base Deployment Template

```bash
curl -s "$DEVTRON_URL/orchestrator/app/template/<AppId>/<ChartId>" \
  -H "token: $DEVTRON_TOKEN" \
  | jq .result.globalConfig > default-values.json
```

#### c. Edit The JSON
```bash
jq '. + {resourceName: "BaseDeploymentTemplate"} 
   | . + {valuesOverride: .defaultAppOverride} 
   | . + {appId: <AppId>, chartRefId: <ChartId>}' default-values.json > output.json

```
#### d. Save the Deployment Template

```bash
curl "$DEVTRON_URL/orchestrator/app/template" \
  -H "token: $DEVTRON_TOKEN" \
  -X POST \
  --http2 \
  -d @output.json
```


## 6. Create CI Pipelines (CI_JOB)

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

---
