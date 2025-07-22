# Install RHAIIS on OCP

GPU Operators already installed.

```cmd
HF_TOKEN=blah

echo $NAMESPACE
aireilly-rhaiis

oc create secret generic hf-secret --from-literal=HF_TOKEN=$HF_TOKEN -n $NAMESPACE
```

```cmd
oc create secret generic docker-secret --from-file=.dockercfg=$HOME/.docker/config.json --type=kubernetes.io/dockercfg -n aireilly-rhaiis
```

```cmd
oc delete deployment granite -n aireilly-rhaiis

oc apply -f deployment.yaml

oc scale deployment granite -n aireilly-rhaiis --replicas=1

oc get deployment -n aireilly-rhaiis --watch

oc logs granite-ffdf944c7-b5zkv -n aireilly-rhaiis -c fetch-model
```

```cmd
oc get route granite -n aireilly-rhaiis -o jsonpath='{.spec.host}'
oc get svc granite -n aireilly-rhaiis -o jsonpath='{.spec.ports[0].targetPort}'
```

Chat

```cmd
curl -v -k \
  http://granite-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com/v1/chat/completions   -H "Content-Type: application/json"   -d '{
    "model":"granite-3-1-8b-instruct-quantized-w8a8",
    "messages":[{"role":"user","content":"What is AI?"}],
    "temperature":0.1
  }'| jq
```

```cmd
oc port-forward svc/granite 8080:80 -n aireilly-rhaiis

curl POST http://localhost:8080/v1/chat/completions   -H "Content-Type: application/json"   -d '{
    "model":"granite-3-1-8b-instruct-quantized-w8a8",
    "messages":[{"role":"user","content":"What is AI?"}],
    "temperature":0.1
  }'| jq
```

## Woes

Create a temporary pod to inspect the PVC:

```cmd
oc run temp-debug --image=registry.access.redhat.com/ubi8/ubi:latest --rm -it --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"temp-debug","image":"registry.access.redhat.com/ubi8/ubi:latest","command":["/bin/bash"],"stdin":true,"tty":true,"volumeMounts":[{"name":"model-vol","mountPath":"/model"}]}],"volumes":[{"name":"model-vol","persistentVolumeClaim":{"claimName":"model-cache"}}]}}' \
  -n aireilly-rhaiis
```

## Developer mode

Set `VLLM_SERVER_DEV_MODE` to enable additional logging and more verbose server output via `/server_info` endpoint.

```yaml
containers:
  - name: granite
    image: 'registry.redhat.io/rhaiis/vllm-cuda-rhel9@sha256:a6645a8e8d7928dce59542c362caf11eca94bb1b427390e78f0f8a87912041cd'
    imagePullPolicy: IfNotPresent
    env:
      - name: VLLM_SERVER_DEV_MODE
        value: '1'
```

```cmd
$ curl -v -k   http://granite-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com/server_info
* Host granite-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com:80 was resolved.
* IPv6: (none)
* IPv4: 52.116.127.252, 52.117.122.220
*   Trying 52.116.127.252:80...
* Connected to granite-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com (52.116.127.252) port 80
* using HTTP/1.x
> GET /server_info HTTP/1.1
> Host: granite-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com
> User-Agent: curl/8.11.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< date: Wed, 23 Jul 2025 10:59:01 GMT
< server: uvicorn
< content-length: 1787
< content-type: application/json
< set-cookie: 8291511e02d6683f0a06073e1c73d065=1d93ea54cd73cbd90d39546d09b4d0e9; path=/; HttpOnly
< 
{"vllm_config":"model='/model', speculative_config=None, tokenizer='/model', skip_tokenizer_init=False, tokenizer_mode=auto, revision=None, override_neuron_config={}, tokenizer_revision=None, trust_remote_code=False, dtype=torch.bfloat16, max_seq_len=131072, download_dir=None, load_format=LoadFormat.AUTO, tensor_parallel_size=1, pipeline_parallel_size=1, disable_custom_all_reduce=False, quantization=compressed-tensors, enforce_eager=False, kv_cache_dtype=auto,  device_config=cuda, decoding_config=DecodingConfig(backend='auto', disable_fallback=False, disable_any_whitespace=False, disable_additional_properties=False, reasoning_backend=''), observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None), seed=0, served_model_name=granite-3-1-8b-instruct-quantized-w8a8, num_scheduler_steps=1, multi_step_stream_outputs=True, enable_prefix_caching=True, chunked_prefill_enabled=True, use_async_output_proc=True, pooler_config=None, compilation_c* Connection #0 to host granite-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com left intact
onfig={\"level\":3,\"debug_dump_path\":\"\",\"cache_dir\":\"\",\"backend\":\"\",\"custom_ops\":[],\"splitting_ops\":[\"vllm.unified_attention\",\"vllm.unified_attention_with_output\"],\"use_inductor\":true,\"compile_sizes\":[],\"inductor_compile_config\":{\"enable_auto_functionalized_v2\":false},\"inductor_passes\":{},\"use_cudagraph\":true,\"cudagraph_num_of_warmups\":1,\"cudagraph_capture_sizes\":[512,504,496,488,480,472,464,456,448,440,432,424,416,408,400,392,384,376,368,360,352,344,336,328,320,312,304,296,288,280,272,264,256,248,240,232,224,216,208,200,192,184,176,168,160,152,144,136,128,120,112,104,96,88,80,72,64,56,48,40,32,24,16,8,4,2,1],\"cudagraph_copy_inputs\":false,\"full_cuda_graph\":false,\"max_capture_size\":512,\"local_cache_dir\":null}"}
```