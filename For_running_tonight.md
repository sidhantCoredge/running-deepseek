1) GPU free check
kubectl --kubeconfig ~/.kube/saif-config get nodes -o json | python3 -c "
import sys,json
for n in json.load(sys.stdin)['items']:
    if 'gpu-compute' in n['metadata']['name']:
        print(n['metadata']['name'],'free_gpu=',n['status']['allocatable'].get('nvidia.com/gpu','0'))"
2) V4 down (free karna ho to)
kubectl --kubeconfig ~/.kube/saif-config -n vllm scale dgd deepseek-v4-flash-agg-h200 --replicas=0
kubectl --kubeconfig ~/.kube/saif-config -n vllm get pods -w
3) Deploy V4.1 DGD
kubectl --kubeconfig ~/.kube/saif-config -n vllm apply -f <PATH>/deepseek-v4-1-flash_dgd.yaml
kubectl --kubeconfig ~/.kube/saif-config -n vllm get dgd deepseek-v4-1-flash-agg-h200 -w
4) Wait worker → 1/1 Running (load 552B, lamba ~15-40 min)
kubectl --kubeconfig ~/.kube/saif-config -n vllm get pods -l app.kubernetes.io/name=deepseek-v4-1-flash-agg-h200 -w
5) Load logs (progress)
kubectl --kubeconfig ~/.kube/saif-config -n vllm logs -l app.kubernetes.io/name=deepseek-v4-1-flash-agg-h200 --tail=200 -f \
  | grep -iE 'ready to serve|engine.*ready|kv cache|loaded|error|Traceback|OOM'
6) SERVE VERIFY (proof)
# served model list
kubectl --kubeconfig ~/.kube/saif-config -n vllm exec -it <WORKER-POD> -- curl -s localhost:8000/v1/models

# engine liveness
kubectl --kubeconfig ~/.kube/saif-config -n vllm exec -it <WORKER-POD> -- curl -s localhost:9090/live

# completion — golden answer 323
kubectl --kubeconfig ~/.kube/saif-config -n vllm exec -it <WORKER-POD> -- \
  curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"deepseek-ai/DeepSeek-V4.1-Flash","messages":[{"role":"user","content":"What is 17*19? Return only the integer."}],"max_tokens":64}'
# → "323" = 100% serve
7) Agar error aaye (args adjust)
# CUDA-graph/OOM: args mein add
#   --max-model-len 262144  --max-num-seqs 128  --gpu-memory-utilization 0.9
# NCCL shm issue: sharedMemory 16Gi -> 20Gi
# unrecognized flag: logs grep check, wahi flag hatao dobara apply
8) Rollback (zarur/ⁿ ho to)
kubectl --kubeconfig ~/.kube/saif-config -n vllm delete dgd deepseek-v4-1-flash-agg-h200
kubectl --kubeconfig ~/.kube/saif-config -n vllm scale dgd deepseek-v4-flash-agg-h200 --replicas=1
