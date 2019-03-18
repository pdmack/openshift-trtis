# openshift-trtis
OpenShift deployment for TensorRT Inference Server and client pod

## Requirements

* NVIDIA Tesla GPU (Pascal or Volta)
* OpenShift 3.11 with [a properly configured CUDA and nvidia-device-plugin node installation](https://blog.openshift.com/how-to-use-gpus-with-deviceplugin-in-openshift-3-10/)

## Install

Create the project

`oc new-project trtis`

Set up a local hostPath model storage. This step and the following may be modified to suit available PV or StorageClass options.

`mkdir -p /tmp/trtis-models && chmod 777 /tmp/trtis-models`

With selinux enabled, we need to relabel our hostpath

`chcon -R -u system_u -r object_r -t svirt_sandbox_file_t -l s0 /tmp/trtis-models`

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

`oc rsh trtis-client curl -s http://inference-server:8000/api/status`

## Testing
If everything looks good, use the client to exercise the server.

`oc rsh trtis-client /bin/bash`

Basic HTTP test with C++ client

```
root@trtis-client:/workspace# /opt/tensorrtserver/bin/image_client -m resnet50_netdef -s INCEPTION qa/images/mug.jpg -u http://inference-server:8000
Request 0, batch size 1
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
```

gRPC of the same test

```
root@trtis-client:/workspace# /opt/tensorrtserver/bin/image_client -i grpc -m resnet50_netdef -s INCEPTION qa/images/mug.jpg -u inference-server:8001
Request 0, batch size 1
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
```
Ask for other classifications

```
root@trtis-client:/workspace# /opt/tensorrtserver/bin/image_client -m resnet50_netdef -s INCEPTION -c 3 qa/images/mug.jpg -u inference-server:8001 -i gRPC
Request 0, batch size 1
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115996
```

Batch

```
root@trtis-client:/workspace# /opt/tensorrtserver/bin/image_client -m resnet50_netdef -s INCEPTION -c 3 qa/images/mug.jpg -u inference-server:8001 -i gRPC -b 3
Request 0, batch size 3
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115996
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115996
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115996
```

Multi-image classification by directory scan

```
root@trtis-client:/workspace# /opt/tensorrtserver/bin/image_client -m resnet50_netdef -s INCEPTION -c 3 -b 2 qa/images -u inference-server:8001 -i gRPC
Request 0, batch size 2
Image 'qa/images/car.jpg':
    817 (SPORTS CAR) = 0.836187
    511 (CONVERTIBLE) = 0.0708251
    751 (RACER) = 0.0597549
Image 'qa/images/mug.jpg':
    504 (COFFEE MUG) = 0.723991
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115996
Request 1, batch size 2
Image 'qa/images/vulture.jpeg':
    23 (VULTURE) = 0.992326
    8 (HEN) = 0.00231854
    84 (PEACOCK) = 0.00201471
Image 'qa/images/car.jpg':
    817 (SPORTS CAR) = 0.836187
    511 (CONVERTIBLE) = 0.0708251
    751 (RACER) = 0.0597549
```

Performance benchmark

```
root@trtis-client:/workspace# /opt/tensorrtserver/bin/perf_client -m resnet50_netdef -p3000 -t4 -v -u inference-server:8001 -i gRPC
*** Measurement Settings ***
  Batch size: 1
  Measurement window: 3000 msec

Request concurrency: 4
  Pass [1] throughput: 175 infer/sec. Avg latency: 22809 usec (std 1677 usec)
  Pass [2] throughput: 172 infer/sec. Avg latency: 23180 usec (std 2768 usec)
  Pass [3] throughput: 175 infer/sec. Avg latency: 22732 usec (std 1742 usec)
  Client:
    Request count: 525
    Throughput: 175 infer/sec
    Avg latency: 22732 usec (standard deviation 1742 usec)
    Avg gRPC time: 22723 usec (marshal 92 usec + response wait 22614 usec + unmarshal 17 usec)
  Server:
    Request count: 633
    Avg request latency: 21778 usec (overhead 18446714931933763777 usec + queue 29141775787793 usec + compute 21824 usec)
```

TODO: Python install of TRTIS in the client image appears to be broken
