# Energy Saving rApp Demo Deployment Guide

This guide explains how to deploy and validate the Energy Saving rApp using the O-RAN SMO rApp platform, with focus on the end-to-end loop through:

- RIC-TaaP / ns-O-RAN-flexric simulator environment
- O1 server (NETCONF control path)
- ns-3 energy-saving scenarios
- rApp onboarding and lifecycle via rApp Manager

The Energy Saving rApp monitors PM data, performs ML-based prediction, and applies energy-saving actions (cell power ON/OFF) using O1.

## What This Guide Covers

- SMO platform installation pointers for Energy Saving flavor
- rApp package generation and preparation
- rApp deployment using Postman and curl
- End-to-end execution with O1 server and ns-3 scenarios
- Validation and logs to confirm actions are applied
- Undeployment, troubleshooting, and cleanup

## Prerequisites

- Kubernetes cluster compatible with O-RAN SMO deployment
- Access to SMO deployment repository and scripts
- Clone of this repository
- Postman (for API workflow option)
- kubectl configured for your target cluster
- Docker (optional, for local image validation)

References:

- SMO install guide:
  - https://gerrit.o-ran-sc.org/r/gitweb?p=it/dep.git;a=blob_plain;f=smo-install/README.md;hb=HEAD
- Flavors details:
  - https://github.com/o-ran-sc/it-dep/blob/master/smo-install/README.md

## 1. SMO Installation (Energy Saving Flavor)

Follow the full SMO installation instructions from the official guide above.

For the Energy Saving demo flavor, use:

- Flavor directory: smo-install/helm-override/ranpm-pynts-es-rapp
- Update onap-flavour-config.yaml under policy-clamp-ac-k8s-ppnt repositories with:

```yaml
- repoName: local
  address: http://172.19.1.81:18080
```

Update the address to match your own environment.

Install the flavor:

```bash
./dep/smo-install/scripts/layer-2/2-install-oran.sh ranpm-pynts-es-rapp dev
```

After core components are ready (wait around 10 minutes), install simulators:

```bash
./dep/smo-install/scripts/layer-2/2-install-simulators.sh ranpm-pynts-es-rapp
```

## 2. Energy Saving rApp Deployment Preparation

1. Clone the rApp Manager repository and create the Energy Saving test branch:

```bash
git clone https://github.com/MinaYonan123/nonrtric-plt-rappmanager.git
cd nonrtric-plt-rappmanager
git checkout -b test-esrapp
```

2. Patch sample rApps with your chart repository endpoint:

```bash
cd scripts/install
./patch-sample-rapps.sh -i IP_ADD -p PORT -r "es-demo-rapp/rapp-energy-saving"
```

3. Move to es-demo-rapp and generate package artifacts:

```bash
./generate.sh rapp-energy-saving
```

This generates the CSAR and Helm chart used for onboarding.

4. Optional local environment override for ES rApp container:

Create src/.env before image build.

```env
O1_SERVER_HOST=<add your machine ip>
O1_SERVER_PORT=8831
CSV_FILE_PATH=/app/.csv
NGINX_EXPORT_URL=<add your nginx deployment address if used>
EXPORT_JSON_FILE=<the desired output json file name>
```

Build and smoke-test container:

```bash
docker build -t ayakamal2000/es-rapp:test_vr6 . -q && docker run --rm ayakamal2000/es-rapp:test_vr6 2>&1 | head -25
```

5. Create namespace for KServe test workloads:

```bash
kubectl create ns kserve-test
```

6. Expose rApp Manager service in nonrtric namespace:

```bash
kubectl expose service rappmanager --type=NodePort --name=rappmanager-exposed -n nonrtric
```

7. Find exposed service IP and port:

```bash
kubectl get svc -A | grep rappmanager
```

Use the ClusterIP of rappmanager-exposed and port 8080 in API calls.

8. Postman collection setup:

- Import rapp-energy-saving.postman_collection.json
- Set collection variables:
  - REMOTE-IP: rappmanager endpoint IP
  - PORT: rappmanager port (typically 8080)
  - rappId: unique rApp package identifier
  - rappInstanceId: auto-populated after instance creation
  - PREIDCT_PORT: prediction port (default 40077)

## 3. End-to-End Loop with O1 Server and ns-3

This is the key sequence for applying energy saving actions in ns-3 through O1.

### 3.1 Required startup order

1. Start O1 server
2. Start ns-3 scenario
3. Deploy Energy Saving rApp

If the sequence changes, O1 action flow may fail.

### 3.2 Start O1 and ns-3

From this repository:

```bash
cd mmwave-LENA-oran/
```

Start O1 server:

```bash
cd O1_sim/config_data
python3 O1_server.py
```

In another terminal, start one scenario from mmwave-LENA-oran/:

```bash
./ns3 run 'scratch/Energy_saving_with_cell_utilization_scenario_O1.cc'
```

Or Lena5G demo scenario:

```bash
./ns3 run 'scratch/opl_nr_mimo_demo.cc'
```

## 4. rApp Deployment via Postman

1. Onboard ES rApp
2. Get Rapps (verify onboard)
3. Prime rApp
4. Get All Rapp Instances (expect none initially)
5. Create Rapp Instance ES
6. Get Rapp Instance (verify create)
7. Deploy Rapp Instance
8. Poll Get Rapp Instance until deployed
9. Watch pods in smo namespace

## 5. rApp Deployment via curl

Use one consistent rApp name for full lifecycle (example: energy-saving-1).

### 5.1 Onboard rApp package

```bash
curl -X POST http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1 \
  -F "file=@/home/oie/nonrtric-plt-rappmanager/sample-rapp-generator/rapp-energy-saving.csar"
```

### 5.2 Prime rApp

```bash
curl -X PUT http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1 \
  -H "Content-Type: application/json" \
  -d '{"primeOrder": "PRIME"}'
```

### 5.3 Check package status

```bash
curl -X GET http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1
```

Wait for status PRIMED.

### 5.4 Create rApp instance

```bash
curl -X POST http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1/instance \
  -H "Content-Type: application/json" \
  -d '{
    "acm": {
      "instance": "es-instance"
    },
    "sme": {
      "providerFunction": "es-model-provider-function",
      "serviceApis": "api-set-kserve-predictor",
      "invokers": "invoker-app1"
    }
  }'
```

Save the returned rappInstanceId as <INSTANCE_ID>.

### 5.5 Deploy instance

```bash
curl -X PUT http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1/instance/<INSTANCE_ID> \
  -H "Content-Type: application/json" \
  -d '{"deployOrder": "DEPLOY"}'
```

### 5.6 Check instance status

```bash
curl -X GET http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1/instance/<INSTANCE_ID>
```

Monitor runtime pods:

```bash
kubectl get pods -n smo | grep energy-saving
```

## 6. Confirm Energy Saving Actions Are Applied

1. Watch rApp logs:

```bash
kubectl logs -f -l app.kubernetes.io/name=energy-saving-rapp -n smo
```

2. Confirm expected behaviors:

- Cell ON/OFF control decisions based on predicted load
- Prediction results used in action selection
- NETCONF/O1 responses confirming applied cell state changes

3. Cross-check with O1 server terminal output and ns-3 scenario behavior to verify closed-loop action execution.

## 7. Undeployment

Use reverse lifecycle order.

### 7.1 Undeploy via Postman

1. Undeploy Rapp Instance
2. Get Rapp Instance (verify undeployed)
3. Delete Rapp Instance
4. Get All Rapp Instances (verify empty)
5. Deprime rApp
6. Delete ES Rapp

### 7.2 Undeploy via curl

Undeploy instance:

```bash
curl -X PUT http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1/instance/<INSTANCE_ID> \
  -H "Content-Type: application/json" \
  -d '{"deployOrder": "UNDEPLOY"}'
```

Delete instance:

```bash
curl -X DELETE http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1/instance/<INSTANCE_ID>
```

Deprime rApp:

```bash
curl -X PUT http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1 \
  -H "Content-Type: application/json" \
  -d '{"primeOrder": "DEPRIME"}'
```

Delete package:

```bash
curl -X DELETE http://<RAPPMANAGER_IP>:8080/rapps/energy-saving-1
```

## 8. rApp Package Structure (Reference)

| Path | Description |
|---|---|
| Artifacts/Deployment/HELM/energy-saving-chart/ | Helm chart for rApp deployment |
| Definitions/asd.yaml | ASD referencing Helm chart |
| Files/Acm/definition/compositions.json | ACM composition definition |
| Files/Acm/instances/es-instance.json | ACM instance configuration |
| Files/Sme/providers/ | SME provider function registrations |
| Files/Sme/invokers/ | SME invoker registrations |
| Files/Sme/serviceapis/ | SME service API definitions |

## 9. Troubleshooting

### 9.1 Deployment stuck in ACM

- Check ACM and kubernetes participant logs in onap namespace
- Check whether Helm install failed
- Check deployment status for related workloads

### 9.2 SME communication issues

- Check servicemanager logs in nonrtric namespace
- Verify communication path between Service Manager and rApp Manager

## 10. Cleanup (When API Cleanup Fails)

### 10.1 ACM cleanup

Use Postman collection Cleanup requests in order:

1. Get All Templates ACM-Direct
2. Get Template ACM-Direct
3. Get All Instances ACM-Direct
4. Get Instance ACM-Direct
5. Undeploy Instance ACM-Direct
6. Delete Instance ACM-Direct
7. Delete Template ACM-Direct

### 10.2 SME cleanup

Clear Kong services/routes from servicemanager pod:

```bash
SERVICEMANAGER_POD=$(kubectl get pods -o custom-columns=NAME:.metadata.name -l app.kubernetes.io/name=servicemanager --no-headers -n nonrtric)
if [[ -n $SERVICEMANAGER_POD ]]; then
  kubectl exec $SERVICEMANAGER_POD -n nonrtric -- ./kongclearup
else
  echo "Error - Servicemanager pod not found, didn't delete Kong routes and services for ServiceManager."
fi
```

Restart key pods:

```bash
kubectl delete pod -l app.kubernetes.io/name=servicemanager -n nonrtric
kubectl delete pod -l app.kubernetes.io/name=rappmanager -n nonrtric
kubectl delete pod -l app.kubernetes.io/name=capifcore -n nonrtric
```

Then re-preload Service Manager configurations from it/dep:

```bash
cd nonrtric/servicemanager-preload
./servicemanager-preload.sh config-nonrtric.yaml
./servicemanager-preload.sh config-smo.yaml
```

## Notes

- This guide focuses on deployment and operations. For full infrastructure details, always use the official SMO installation documentation.
- You can export this Markdown to PDF from your editor if a PDF version is needed.

## Acknowledgements

This demo builds on the OSC Non-RT RIC Energy Saving rAPP work:

- https://github.com/bmw-ece-ntust/nonrtric-rapp-energysaving

Citation:

Lan, Y., Zhang, H., & Bimo, F. A. (2025). nonrtric-rapp-energysaving (Version 1.0.0) [Computer software]. https://github.com/bmw-ece-ntust/nonrtric-rapp-energysaving
