# Energy Saving rApp Demo

This guide deploys the Energy Saving (ES) rApp on the O-RAN SMO platform and connects it to a RIC-TaaP simulation. It is an end-to-end demonstration: the rApp observes performance-management (PM) data, predicts traffic load, and requests cell power-state changes through the O1/NETCONF control path.

RIC-TaaP supplies the reproducible RAN-side test environment. It runs the ns-3 scenario, produces the traffic and cell-load conditions to be evaluated, and — when using the `O1_test` branch of the `mmwave-LENA-oran` submodule, which carries the O1 server and O1-enabled scenarios — receives the resulting O1 actions. The SMO platform supplies rApp lifecycle management and the services on which the rApp depends; it is not installed by this repository.

The instructions below describe how to:

- Deploy the demo in a Kubernetes cluster.
- Create and deploy the Energy Saving rApp using the rApp Manager REST API (curl commands) and Postman.
- Confirm that the Energy Saving rApp is running and managing energy consumption in the network.
- Undeploy the Energy Saving rApp.
- Troubleshoot any issues that may arise during the deployment or undeployment process.

## Closed-loop architecture

```text
RIC-TaaP / ns-3 scenario ── PM data ──> InfluxDB ──> ES rApp
       ^                                                   |
       |                                             prediction
       |                                                   v
       └── cell ON/OFF <── O1 / NETCONF <── control decision
```

| Component | Responsibility |
|---|---|
| RIC-TaaP (`ns-O-RAN-flexric`) | Runs the ns-3 energy-saving scenario and provides the RAN-side validation environment. |
| O1 server | Terminates NETCONF requests and applies/reports the requested cell state. It ships on the `O1_test` branch of the `mmwave-LENA-oran` submodule, not on the default checkout. |
| O-RAN SMO / Non-RT RIC | Hosts rApp Manager, ACM, Service Manager, KServe, and related services. |
| Energy Saving rApp | Reads PM data, invokes the predictor, and selects ON/OFF actions. |

## Prerequisites

- A clone of this repository (`https://github.com/Orange-OpenSource/ns-O-RAN-flexric.git`). Inside it, `cd mmwave-LENA-oran/` and check out the `O1_test` branch for the O1 server and O1-enabled ns-3 scenarios.
- The Energy Saving rApp source/package, including `rapp-energy-saving.csar` and the Postman collection.
- Postman, if you use the Postman workflow.
- Docker, only when you build a custom rApp image locally.
- A Kubernetes cluster (Kubernetes 1.30+)
- **SMO Prerequisites:**
  - VM: 64GB Memory, 20VCPU, 100GB disk
  - Helm 3.12.0+ (< 4.0.0)
  - Helm deploy/undeploy plugin
  - Helm cm-push plugin
  - yq
  - jq

## Deployment Steps

### SMO Installation

#### Dev Mode Installation
Builds charts from source.

**Steps:**

1. Clone the repository:
   ```bash
   git clone --recursive "https://gerrit.o-ran-sc.org/r/it/dep"
   ```

2. Prepare the Kubernetes cluster using the helper script:
   ```bash
   ./dep/tools/setup_k8s/scripts/setup_k8s.sh
   ```
   This script can be used to setup the Kubernetes cluster using kubeadm.

3. Setup chartmuseum and Helm:
   ```bash
   ./dep/smo-install/scripts/layer-0/0-setup-charts-museum.sh
   ./dep/smo-install/scripts/layer-0/0-setup-helm3.sh
   ```

4. Build charts:
   ```bash
   ./dep/smo-install/scripts/layer-1/1-build-all-charts.sh
   ```

As an additional note, a special flavour of the SMO installation is available for the Energy Saving rApp demo.
This flavour is located in the `smo-install/helm-override/ranpm-pynts-es-rapp` directory.
You also need to modify `onap-flavour-config.yaml`. Under `policy-clamp-ac-k8s-ppnt`, add the chart repository entry below to the repository list:

```yaml
- repoName: local
  address: http://<chart-repository-host>:<chart-repository-port>
```

The address above must be updated to match your deployment environment.

There is some detail on flavours [here](https://github.com/o-ran-sc/it-dep/blob/master/smo-install/README.md).
This flavour is designed to install the SMO components required for the Energy Saving rApp demo.
After all the other steps in the SMO installation guide are completed, you can run the following command to
install the Energy Saving rApp flavour:

```bash
./dep/smo-install/scripts/layer-2/2-install-oran.sh ranpm-pynts-es-rapp dev
```

Once all the components are installed and ready, you can proceed with the install of the DU simulators.
These will start a flow of sample data through the system, which will be used by the Energy Saving rApp demo.
Wait around ten minutes after all the components are installed before proceeding with the next simulator install.

```bash
./dep/smo-install/scripts/layer-2/2-install-simulators.sh ranpm-pynts-es-rapp
```

```text
NOTE: The installation is just pointed at with the above commands. For the full installation and details of flavours, go to the smo installation docs.
```
See the [SMO installation guide](https://github.com/o-ran-sc/it-dep/blob/master/smo-install/README.md) for the authoritative instructions.

### Energy Saving rApp Deployment Preparation
1. The rApp needs to know the address and port of your chart repository. If you need to run a local chart repository, you can refer to the instructions [here](https://github.com/o-ran-sc/it-dep/blob/master/smo-install/helm-override/rappmanager/README.md#helm-repository-configuration).
   Replace `<IP_ADD>` and `<chart-repository-port>` in the command below with the address and port of your chart repository.
   ```bash
   cd nonrtric-plt-rappmanager/scripts/install
   ./patch-sample-rapps.sh -i <IP_ADD> -p <chart-repository-port> \
     -r "es-demo-rapp/rapp-energy-saving"
   ```
2. Navigate to the `sample-rapp-generator` directory in the `nonrtric-plt-rappmanager` repository.
3. To generate the rApp csar and helm chart, run the following command:
   ```bash
   ./generate.sh es-demo-rapp/rapp-energy-saving
   ```
4. Note that the `src` directory for this demo is in `es-demo-rapp`.
5. If you run the ES rApp with a local environment-specific configuration, add a `.env` file in the `es-demo-rapp/src/` directory before deployment and rebuild the image. Example:
   ```env
   O1_SERVER_HOST=<add your machine IP>
   O1_SERVER_PORT=8831

   # Path to the CSV file to load into InfluxDB.
   # Default: /app/*.csv (set the CSV file name if you want to use your custom data)
   CSV_FILE_PATH=/app/<input-file>.csv

   # nginx export URL for exporting InfluxDB data
   NGINX_EXPORT_URL=<optional-export-endpoint>

   # Optional: Output file path for exported JSON (default: output.json)
   EXPORT_JSON_FILE=<optional-output-file>.json
   ```
   After adding the `.env` file, ensure Docker is installed on your machine, then navigate to the `es-demo-rapp` directory and build the image. The best approach is to build the image and load it directly into your Kubernetes cluster without pushing to a registry:

   ```bash
   cd es-demo-rapp
   docker build -t <registry>/es-rapp:<tag> .
   docker save <registry>/es-rapp:<tag> | sudo ctr -n k8s.io images import -
   ```

   This command builds the Docker image and loads it directly into Kubernetes without pushing to a registry.

6. Create a namespace for KServe test workloads:
   ```bash
   kubectl create ns kserve-test
   ```
7. Make sure to expose the rappmanager service in the `nonrtric` namespace. This is done by running the following command:
   ```bash
   kubectl expose service rappmanager --type=NodePort --name=rappmanager-exposed -n nonrtric
   ```
   If `rappmanager-exposed` already exists, skip this command.
8. Find the assigned NodePort of the exposed service:
   ```bash
   kubectl get service rappmanager-exposed -n nonrtric
   ```
   Set `RAPPMANAGER_HOST` to a Kubernetes node address and `8080` to the NodePort shown — a Service `ClusterIP` is internal to the cluster and will not be reachable from outside it. Use this pair in all API calls below.

9. You will be using the Postman collection provided in the main directory of this repository to create the rApp.
10. Open Postman and import the `rapp-energy-saving.postman_collection.json` file.
11. It is important to note the collection-level variables in the Postman collection:

    | Variable | Value |
    |---|---|
    | `REMOTE-IP` | rApp Manager node address |
    | `PORT` | rApp Manager NodePort |
    | `rappId` | A unique rApp identifier, such as `energy-saving-1` |
    | `rappInstanceId` | Filled after the create-instance request |
    | `PREIDCT_PORT` | Prediction service port; use `40077` unless your package specifies otherwise (the supplied collection uses this spelling). |

---

### rApp Deployment via Postman
1. In Postman, select the `Onboard ES rApp` request from the collection. Send this request.
2. Then run the `Get Rapps` request to confirm that the rApp has been onboarded successfully.
3. Run the `Prime rApp` request to prime the rApp.
4. Run `Get All Rapp Instances` to confirm that no rApp instance have been created.
5. Run the `Create Rapp Instance ES` request to create an instance of the rApp.
6. Run the `Get Rapp Instance` request to confirm that the rApp instance has been created successfully.
7. Run the `Deploy Rapp Instance` to trigger installation of the rApp instance.
8. The above deployment can take time, so you can run the `Get Rapp Instance` request to check the status of the rApp instance.
9. You can also monitor the Kubernetes pods in the `smo` namespace to see if the rApp instance helm charts are being deployed.

---

### rApp Deployment via curl

> The rApp name `energy-saving-1` (`$RAPP_ID`) is chosen to describe what is being deployed. It must be consistent across all commands for a given lifecycle.
>
> `$CSAR_PATH` must match the location of the file generated in the preparation steps above.

### End-to-End Loop Test with O1 Server and ns-3

Use the following steps to test the end-to-end loop for the Energy Saving rApp with the O1 server and ns-3, so that the energy saving actions are applied during the scenario.

1. Clone the RIC-TaaP repository:
   ```bash
   git clone https://github.com/Orange-OpenSource/ns-O-RAN-flexric.git
   cd ns-O-RAN-flexric
   ```
2. Navigate to the ns-3 workspace:
   ```bash
   cd mmwave-LENA-oran/
   ```
3. The O1 server and the O1-enabled scenarios are not present on the default checkout of this submodule — check out the `O1_test` branch:
   ```bash
   git fetch origin
   git checkout O1_test
   ```
4. Start the O1 server:
   ```bash
   cd O1_sim/config_data
   python3 O1_server.py
   ```
5. In another terminal, start the ns-3 scenario from `mmwave-LENA-oran/`:
   ```bash
   ./ns3 run 'scratch/Energy_saving_with_cell_utilization_scenario_O1.cc'
   ```
   Or, for the 5G-LENA scenario, run:
   ```bash
   ./ns3 run 'scratch/opl_nr_mimo_demo.cc'
   ```
6. After the O1 server and ns-3 scenario are running, deploy the Energy Saving rApp using the deployment steps described above.

Important sequence:
- Start the O1 server first.
- Start the ns-3 scenario next.
- Deploy the rApp only after both the O1 server and ns-3 are running.

#### 1. Onboard the rApp package

```bash
curl -X POST \
  "http://${RAPPMANAGER_HOST}:8080/rapps/${energy-saving-1}" \
  -F "file=@${CSAR_PATH}"
```
where CSAR_PATH=<absolute-path-to-rapp-energy-saving.csar>

#### 2. Prime the rApp

```bash
curl -X PUT \
  "http://${RAPPMANAGER_HOST}:8080/rapps/${energy-saving-1}" \
  -H "Content-Type: application/json" \
  -d '{"primeOrder": "PRIME"}'
```

#### 3. Check the rApp status

```bash
curl -X GET http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1
```

Wait until the status shows `PRIMED` before proceeding.

#### 4. Create an rApp instance

The keys below must match the ACM/SME descriptors shipped in your CSAR; change them only together with the package descriptors.

```bash
curl -X POST http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1/instance \
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

Note the `rappInstanceId` returned in the response (e.g. `6cf1718e-2b7d-42b9-b606-80854adc9e25`). Set it as `INSTANCE_ID` and use it in the commands below.

#### 5. Deploy the rApp instance

```bash
curl -X PUT \
  http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1/instance/${INSTANCE_ID} \
  -H "Content-Type: application/json" \
  -d '{"deployOrder": "DEPLOY"}'
```

#### 6. Check the rApp instance status

```bash
curl -X GET http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1/instance/${INSTANCE_ID}
```

You can also monitor Kubernetes pods in the `smo` namespace:
```bash
kubectl get pods -n smo | grep energy-saving
```

### Confirmation
1. To confirm successful running of the demo energy saving rApp, we can look at the Kubernetes logs of the pod.
   ```bash
   kubectl logs -f -l app.kubernetes.io/name=energy-saving-rapp -n smo
   ```
2. You should see logs indicating that the rApp is running and managing energy consumption in the network.
3. For example, you should see:
    1. Logs of cells being turned off and on based on the energy consumption in the network.
    2. Predictions of energy consumption based on the current network load being returned to make power management decisions.
    3. NETCONF responses from the O1 server confirming cell state changes.
4. Successful validation requires evidence from both sides of the loop: the rApp selecting cell ON/OFF actions from PM-driven predictions, the O1 server reporting the NETCONF request and response, and the RIC-TaaP/ns-3 scenario showing the corresponding cell-state behavior.

## Undeployment

Undeployment is done in reverse order: undeploy the instance, delete the instance, deprime the rApp, then delete the rApp package.

### Undeploy via Postman
1. Run the `Undeploy Rapp Instance` request to undeploy the rApp instance. This takes some time.
2. Run the `Get Rapp Instance` request to confirm that the rApp instance has been undeployed successfully.
3. Run the `Delete Rapp Instance` request to delete the rApp instance.
4. Run the `Get All Rapp Instances` request to confirm that no rApp instances are present.
5. Run the `Deprime rApp` request to deprime the rApp.
6. Run the `Delete ES Rapp` request to delete the rApp.
7. This should conclude the undeployment of the Energy Saving rApp.

### Undeploy via curl

#### 1. Undeploy the rApp instance

```bash
curl -X PUT \
  http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1/instance/${INSTANCE_ID} \
  -H "Content-Type: application/json" \
  -d '{"deployOrder": "UNDEPLOY"}'
```

#### 2. Delete the rApp instance

```bash
curl -X DELETE http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1/instance/${INSTANCE_ID}
```

#### 3. Deprime the rApp

```bash
curl -X PUT \
  http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1 \
  -H "Content-Type: application/json" \
  -d '{"primeOrder": "DEPRIME"}'
```

#### 4. Delete the rApp package

```bash
curl -X DELETE http://${RAPPMANAGER_HOST}:8080/rapps/energy-saving-1
```

---

### rApp Package Structure (`rapp-energy-saving/`)

| Path | Description |
|---|---|
| `Artifacts/Deployment/HELM/energy-saving-chart/` | Helm chart that deploys the rApp container |
| `Definitions/asd.yaml` | ASD (Application Service Descriptor) referencing the Helm chart |
| `Files/Acm/definition/compositions.json` | ACM composition definition (automation policy types) |
| `Files/Acm/instances/es-instance.json` | ACM instance configuration used when creating the rApp instance |
| `Files/Sme/providers/` | SME provider function registrations (KServe predictor, InfluxDB) |
| `Files/Sme/invokers/` | SME invoker registrations (NCMP, InfluxDB, KServe, TEIV) |
| `Files/Sme/serviceapis/` | SME service API definitions |

---

## Troubleshooting
If you encounter any issues during the deployment or undeployment of the rApp, please check the following:
1. Is deployment of the pods stuck in ACM?
    - Check the logs of the ACM pod and the kubernetes participant in the `onap` namespace.
    - Is there any indication that install of the pods failed?
    - Check the deployment status of the pods.
2. Is there an issue with the SME part of the installation?
    - Check the logs of the servicemanager pod in the `nonrtric` namespace.
    - Is there any indication that the SME is not able to communicate with the rApp Manager?
3. Is the rApp starting but no control action occurring?
    - Verify the O1 host/port in the rApp configuration, that the O1 server has started, that PM data is reaching InfluxDB, and that the O1-enabled ns-3 scenario is running.
4. Is the rApp Manager endpoint unreachable?
    - Use a node address plus its NodePort, or run the client inside the cluster. A Service `ClusterIP` is internal to the cluster by default and is not reachable from outside it.

### Clean Up
If there is a case of a failed deployment that cannot be cleaned up via the API, we can use the following steps.
When cleaning up, it is best to carry out both ACM Cleanup and SME Cleanup - detailed below.

#### ACM Cleanup
Consult the postman collection under the "Cleanup" directory for the ACM cleanup steps.
1. Run `Get All Templates ACM-Direct`
2. Run `Get Template ACM-Direct`
3. Run `Get All Instances ACM-Direct`
4. Run `Get Instance ACM-Direct`
5. The above will populate the postman collection variables with the template and instance IDs.
6. Run `Undeploy Instance ACM-Direct` to undeploy the instance.
   Wait for the pods to undeploy (if they are stuck or leftover).
7. Run `Delete Instance ACM-Direct` to delete the instance.
8. Run `Delete Template ACM-Direct` to delete the template.
9. That should conclude the ACM cleanup.

#### SME Cleanup
SME cleanup requires some manual steps.
1. Delete all the kong services and routes. Put this in some "script.sh" file and run it. The place where you run it
   should have access to the cluster.
   ```bash
      SERVICEMANAGER_POD=$(kubectl get pods -o custom-columns=NAME:.metadata.name -l app.kubernetes.io/name=servicemanager --no-headers -n nonrtric)
      if [[ -n $SERVICEMANAGER_POD ]]; then
      kubectl exec $SERVICEMANAGER_POD -n nonrtric -- ./kongclearup
      else
      echo "Error - Servicemanager pod not found, didn't delete Kong routes and services for ServiceManager."
      fi

   ```
2. Once the above has been run, we must restart some pods.
   ```bash
   kubectl delete pod -l app.kubernetes.io/name=servicemanager -n nonrtric
   kubectl delete pod -l app.kubernetes.io/name=rappmanager -n nonrtric
   kubectl delete pod -l app.kubernetes.io/name=capifcore -n nonrtric
   ```
3. Wait for these pods to come up again to a `Running` state.
4. Now we need to add some preloaded SME configurations.
    1. In the `it/dep` repository, navigate to the `nonrtric/servicemanager-preload` directory.
    2. Preload some of the nonrtric services by running the following command:
       ```bash
       ./servicemanager-preload.sh config-nonrtric.yaml
       ```
    3. Preload the SMO services by running the following command:
       ```bash
       ./servicemanager-preload.sh config-smo.yaml
       ```
5. This should conclude the SME cleanup. Then we can attempt to redeploy the rApp again with whatever
   changes we made to fix the issues.

## Acknowledgements
This demo builds on the [OSC Non-RT RIC Energy Saving rAPP](https://github.com/bmw-ece-ntust/nonrtric-rapp-energysaving). Many thanks to the original authors for their work.

### Citation
```text
Lan, Y., Zhang, H., & Bimo, F. A. (2025). nonrtric-rapp-energysaving (Version 1.0.0) [Computer software]. https://github.com/bmw-ece-ntust/nonrtric-rapp-energysaving
```