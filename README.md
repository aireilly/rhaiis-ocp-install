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
oc get svc granite -n aireilly-rhaiis -o jsonpath='{.spec.ports[*].port}{" -> "}{.spec.ports[*].targetPort}{"\n"}'
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


# OCI deployment

```cmd
oc apply -f service-oci.yaml
oc apply -f route-oci.yaml
oc logs rhaiis-oci-deploy-6794cfdc6d-4wtht -n aireilly-rhaiis -c fetch-model
oc get route rhaiis-oci-deploy -n aireilly-rhaiis -o jsonpath='{.spec.host}'
oc get svc rhaiis-oci-deploy -n aireilly-rhaiis -o jsonpath='{.spec.ports[*].port}{" -> "}{.spec.ports[*].targetPort}{"\n"}'
oc port-forward svc/rhaiis-oci-deploy 8080:80 -n aireilly-rhaiis
```

Chat

```cmd
curl -v -k \
  http://rhaiis-oci-deploy-aireilly-rhaiis.apps.modelsibm.ibmmodel.rh-ods.com/v1/chat/completions   -H "Content-Type: application/json"   -d '{
    "model":"ibm-granite/granite-3.1-2b-instruct",
    "messages":[{"role":"user","content":"Hello?"}],
    "temperature":0.1
  }'| jq

oc port-forward svc/rhaiis-oci-deploy 8080:80 -n aireilly-rhaiis

curl POST http://localhost:8080/v1/chat/completions   -H "Content-Type: application/json"   -d '{
    "model":"ibm-granite/granite-3.1-2b-instruct",
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

# Spyre

Serve modelcar
https://docs.redhat.com/en/documentation/red_hat_ai_inference_server/3.2/html/validated_models/red_hat_ai_validated_models#redhat-ai-validated-modelcar-images

All three settings must align:

```cmd
  - SPYRE_DEVICES: '0,1,2,3'
  - ibm.com/spyre_pf: '4'     # resources request/limit
  - --tensor-parallel-size=4  # vLLM arg
```


```cmd
$ oc get nodes -o json | jq -r '.items[] | select(.status.capacity["ibm.com/spyre_pf"]) | {name: 
  .metadata.name, total: .status.capacity["ibm.com/spyre_pf"], allocatable: 
  .status.allocatable["ibm.com/spyre_pf"]}'
{
  "name": "spyre-001.nvidia.eng.rdu2.dc.redhat.com",
  "total": "12",
  "allocatable": "12"
}
```


```cmd
oc apply -f spyre_x86/pvc-cleanup-pod.yaml

oc exec -it pvc-cleanup -n vllm-inference -- /bin/sh
```

```cmd
# recreating the PVC, first release the PV
oc patch pv vllm-models-pv-spyre-005 -p '{"spec":{"claimRef": null}}'
```

Clean up locks in /mnt:

```cmd
# List all files in the hub directory (where HuggingFace stores models and locks)
ls -la /mnt/models/hub/

# Navigate and look for lock files
cd /mnt/models/hub
ls -la

# Look for .locks directory
ls -la | grep lock

# If you see a .locks directory, remove it
rm -rf .locks

# Look for any .lock files
ls -la *.lock 2>/dev/null
rm -f *.lock 2>/dev/null

# Also check the ibm-granite directory
cd /mnt/models/ibm-granite
ls -la
rm -rf .locks 2>/dev/null
rm -f *.lock 2>/dev/null

ls -la /mnt/models/
exit
```

```cmd
oc delete pod pvc-cleanup -n vllm-inference
```

Thar she blows:

```cmd
[1;36m(APIServer pid=1)[0;0m INFO 10-23 16:18:20 [api_server.py:1805] vLLM API server version 0.10.1.1
[1;36m(APIServer pid=1)[0;0m INFO 10-23 16:18:20 [utils.py:326] non-default args: {'host': '0.0.0.0', 'port': 3000, 'model': '/mnt/models', 'max_model_len': 4096, 'tensor_parallel_size': 4, 'max_num_seqs': 8}
```

```cmd
$ curl -X POST http://rhaiis-granite-3-1-8b-instruct-vllm-inference.apps.spyre-001.nvidia.eng.rdu2.dc.redhat.com/v1/chat/completions     -H "Content-Type: application/json"     -d '{
      "model": "/mnt/models",
      "messages": [{"role": "user", "content": "What does regret mean?"}],
      "max_tokens": 512,
      "temperature": 0.1
    }' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1251  100  1089  100   162     29      4  0:00:40  0:00:36  0:00:04   262
{
  "id": "chatcmpl-9edc59e5db754bd08edcd6325814bd8a",
  "object": "chat.completion",
  "created": 1761238106,
  "model": "/mnt/models",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Regret is a feeling of sadness, disappointment, or remorse over something that has happened or been done, especially a loss or missed opportunity. It's a common human emotion that often arises when we reflect on past actions or decisions and wish we could change them. Regret can be a powerful motivator for personal growth and change, as it encourages us to learn from our mistakes and make better choices in the future. However, excessive or unproductive regret can also lead to negative psychological states like depression or anxiety.",
        "refusal": null,
        "annotations": null,
        "audio": null,
        "function_call": null,
        "tool_calls": [],
        "reasoning_content": null
      },
      "logprobs": null,
      "finish_reason": "stop",
      "stop_reason": null
    }
  ],
  "service_tier": null,
  "system_fingerprint": null,
  "usage": {
    "prompt_tokens": 65,
    "total_tokens": 183,
    "completion_tokens": 118,
    "prompt_tokens_details": null
  },
  "prompt_logprobs": null,
  "kv_transfer_params": null
}
```