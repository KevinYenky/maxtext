<!--
 Copyright 2023-2026 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

(run-elastic-training)=

# Elastic training

This guide runs a standard multi-host MaxText JobSet with Cluster Toolkit and
recovers a failed job by submitting it again from its latest durable
checkpoint.

Cluster Toolkit does not provide the legacy Pathways in-process elastic retry
behavior. The controller and JobSet can restart, and the training command must
restore from a durable checkpoint. This is a restart-and-resume workflow.

## Prerequisites

Follow [At scale with Cluster Toolkit](run_maxtext_via_cluster_toolkit.md) to
install `gcloud`, `kubectl`, and `gcluster`, and to configure a GKE cluster with
healthy Kueue and JobSet components.

## Configure access and workload values

```bash
export PROJECT_ID=<GCP_PROJECT_ID>
export GKE_CLUSTER=<GKE_CLUSTER_NAME>
export ZONE=<GCP_ZONE>
export RUN_NAME=maxtext-recovery-demo
export BASE_OUTPUT_DIRECTORY=<GCS_BUCKET_PATH>
export COMPUTE_TYPE=<CLUSTER_TOOLKIT_COMPUTE_TYPE>
export TOPOLOGY=<TPU_TOPOLOGY>
export BASE_IMAGE=<ARTIFACT_REGISTRY_BASE_IMAGE>

gcloud config set project ${PROJECT_ID?}
gcloud container clusters get-credentials ${GKE_CLUSTER?} \
  --zone ${ZONE?} \
  --project ${PROJECT_ID?}

gcluster job config set project ${PROJECT_ID?}
gcluster job config set cluster ${GKE_CLUSTER?}
gcluster job config set location ${LOCATION?}
```

## Submit a checkpointed workload

Use a checkpoint interval that is appropriate for the amount of work that can
be repeated after a job restart:

```bash
gcluster job submit \
  --base-image ${BASE_IMAGE?} \
  --build-context . \
  --command "python3 -m maxtext.trainers.pre_train.train \
    base_output_directory=${BASE_OUTPUT_DIRECTORY?} \
    run_name=${RUN_NAME?} \
    model_name=qwen3-0.6b \
    dataset_type=synthetic \
    per_device_batch_size=1 \
    steps=5000 \
    enable_checkpointing=true \
    checkpoint_period=100" \
  --name ${RUN_NAME?} \
  --compute-type ${COMPUTE_TYPE?} \
  --topology ${TOPOLOGY?}
```

Monitor the JobSet and logs:

```bash
gcluster job list
gcluster job logs ${RUN_NAME?}
kubectl get jobset -l gcluster.google.com/workload=${RUN_NAME?}
```

## Recover after a failure

After confirming that the latest checkpoint exists in
`${BASE_OUTPUT_DIRECTORY}/${RUN_NAME?}`, cancel the failed JobSet and submit a
new job with a new name. Set `load_parameters_path` to the latest committed
checkpoint:

```bash
gcluster job cancel ${RUN_NAME?}
export RETRY_RUN_NAME=${RUN_NAME?}-retry1
export CHECKPOINT_PATH=${BASE_OUTPUT_DIRECTORY?}/${RUN_NAME?}/checkpoints/<STEP>/items

gcluster job submit \
  --base-image ${BASE_IMAGE?} \
  --build-context . \
  --command "python3 -m maxtext.trainers.pre_train.train \
    base_output_directory=${BASE_OUTPUT_DIRECTORY?} \
    run_name=${RETRY_RUN_NAME?} \
    model_name=qwen3-0.6b \
    dataset_type=synthetic \
    per_device_batch_size=1 \
    steps=5000 \
    load_parameters_path=${CHECKPOINT_PATH?} \
    enable_checkpointing=true \
    checkpoint_period=100" \
  --name ${RETRY_RUN_NAME?} \
  --compute-type ${COMPUTE_TYPE?} \
  --topology ${TOPOLOGY?}
```

Validate recovery by checking that the new job restores the checkpoint and
continues training from the expected step. This restart-and-resume workflow is
the supported Cluster Toolkit equivalent for fault recovery; it does not keep
the original training process alive during the failure.
