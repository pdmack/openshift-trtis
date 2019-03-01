# openshift-trtis
OpenShift deployment for TensorRT Inference Server and client pod

Create the project

`oc new-project trtis`

Set up a local hostPath model storage

`mkdir -p /tmp/trtis-models && chmod 777 /tmp/trtis-models`

Download the models and copy them to the local storage

`sh fetch_models.sh && cp -R model_repository/* /tmp/trtis-models`

Create a PV and a PVC for the storage

`oc create -f trtis-volume.json`

`oc create -f trtis-claim.json`

Launch the TensorRT Inference Server

`oc create -f trtis-server.json`

Launch the pre-built client image

`oc create -f trtis-client.json`

Check that the server is healthy and models are ready to be served for inference

`oc exec trtis-client curl http://inference-server:8000/api/status`
