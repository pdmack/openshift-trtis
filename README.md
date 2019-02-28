# openshift-trtis
OpenShift deployment for TensorRT Inference Server and client pod

`oc new-project trtis`

`mkdir -p /tmp/trtis-models && chmod 777 /tmp/trtis-models`

`sh fetch_models.sh && cp -R model_repository/* /tmp/trtis-models`

`oc create -f trtis-volume.json`

`oc create -f trtis-claim.json`

`oc create -f trtis-server.json`
