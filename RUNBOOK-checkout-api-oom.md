Runbook: checkout-api OOMKilled Incident
Purpose

Use this runbook when checkout-api pods are restarting and Kubernetes reports OOMKilled.

The primary goal is to:

Confirm that memory exhaustion is causing the restarts.

Determine whether the issue is a container memory limit, unusual application usage, or another resource condition.

Restore service safely.

Capture enough evidence for follow-up before changing resource limits.

Prerequisites

Access to the Kubernetes cluster.

kubectl configured for the affected cluster.

Permission to inspect and modify the checkout-api workload.

The namespace containing checkout-api (replace <namespace> below if necessary).

Diagnosis
1. Check pod status and restart counts

Start by identifying affected pods:

kubectl get pods -n <namespace> -l app=checkout-api


Look for:

Increasing RESTARTS counts.

Pods repeatedly transitioning between Running and CrashLoopBackOff.

Pods that are not Ready.

For a continuously updating view:

kubectl get pods -n <namespace> -l app=checkout-api -w

2. Confirm OOMKilled

Inspect an affected pod:

kubectl describe pod <pod-name> -n <namespace>


Under the affected container's Last State, look for:

Reason: OOMKilled


You can also query the termination reason directly:

kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'


If the result is OOMKilled, the container exceeded its Kubernetes memory limit.

3. Check the configured memory limit

Inspect the deployment:

kubectl get deployment checkout-api -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[*].resources}{"\n"}'


For a more readable view:

kubectl describe deployment checkout-api -n <namespace>


Look for the Limits section, particularly:

memory: 128Mi


Compare the configured limit with the value that was recently deployed.

For the incident described in this postmortem, the problematic value was:

memory: 8Mi

4. Check current memory usage

If the Metrics Server is available:

kubectl top pods -n <namespace> -l app=checkout-api


For an individual pod:

kubectl top pod <pod-name> -n <namespace>


Compare observed memory usage with the configured limit.

Important: kubectl top shows current/near-current usage and may not capture the peak usage that triggered an OOM. Do not conclude that a pod is healthy simply because its current usage is below the limit.

5. Check recent container logs

Retrieve logs from the current container:

kubectl logs <pod-name> -n <namespace> -c checkout-api


If the container has restarted, inspect the previous instance:

kubectl logs <pod-name> -n <namespace> -c checkout-api --previous


Look for application-level evidence immediately preceding the restart, such as unusually large requests, cache growth, allocation failures, or other abnormal behavior.

6. Check recent rollout/change history

Inspect the deployment's rollout history:

kubectl rollout history deployment/checkout-api -n <namespace>


Inspect the deployment configuration:

kubectl get deployment checkout-api -n <namespace> -o yaml


Pay particular attention to recent changes to:

resources:
  requests:
    memory: ...
  limits:
    memory: ...


If a resource-limit change occurred immediately before the OOMs, record the old and new values.

7. Determine the likely failure mode

Classify the incident before making a permanent change:

A. Limit is clearly below normal usage

Evidence:

Last State reports OOMKilled.

The memory limit was recently reduced.

Observed usage is close to or above the new limit.

The workload was otherwise operating normally.

This matches the incident described in this postmortem.

B. Usage is unexpectedly high

Evidence:

The limit is normally adequate.

Memory usage has increased substantially compared with normal behavior.

Logs or application metrics indicate an abnormal workload.

Do not simply raise the limit indefinitely. Preserve evidence and investigate the application/workload.

C. OOM is caused by node-level memory pressure

Inspect node conditions:

kubectl describe node <node-name>


Look for:

MemoryPressure


and recent eviction events:

kubectl get events -n <namespace> --sort-by='.lastTimestamp'


Container OOMKilled caused by exceeding its own memory limit is different from broader node memory pressure, so distinguish the two before choosing a permanent fix.

Resolution
1. If a recent limit reduction caused the incident, restore the known-good value

For the incident documented here, the known-good memory limit is 128Mi.

First inspect the current configuration:

kubectl get deployment checkout-api -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="checkout-api")].resources.limits.memory}{"\n"}'


If the limit is incorrectly set to 8Mi, restore it:

kubectl set resources deployment checkout-api \
  -n <namespace> \
  --limits=memory=128Mi


If the deployment also has an explicitly defined memory request, preserve or set the intended request rather than accidentally removing it. For example:

kubectl set resources deployment checkout-api \
  -n <namespace> \
  --requests=memory=<known-request> \
  --limits=memory=128Mi

2. Watch the rollout

After changing the deployment:

kubectl rollout status deployment/checkout-api -n <namespace>


Then monitor the pods:

kubectl get pods -n <namespace> -l app=checkout-api -w


Confirm that:

New pods become Ready.

New OOMKilled terminations stop.

Restart counts stop increasing.

Requests return to normal.

3. Verify the resulting configuration
kubectl describe deployment checkout-api -n <namespace>


Confirm that the intended memory limit is present.

Then check actual usage:

kubectl top pods -n <namespace> -l app=checkout-api


Record the observed memory usage for follow-up capacity planning.

4. If the deployment is managed by GitOps or another deployment system

Do not make the emergency change only in the live cluster if a controller will overwrite it.

After service is restored, update the authoritative configuration to the intended value and allow the normal deployment process to reconcile it.

For example, verify that the live value matches the source-controlled manifest:

kubectl get deployment checkout-api -n <namespace> -o yaml

5. If increasing the limit does not resolve the OOM

Do not repeatedly increase the limit without investigation.

Collect:

kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -c checkout-api --previous
kubectl top pod <pod-name> -n <namespace>
kubectl describe node <node-name>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'


Escalate for application-level memory investigation if usage is unexpectedly high or continues growing.

Verification Checklist

The incident can be considered resolved when all of the following are true:

checkout-api pods are Ready.

Restart counts are no longer increasing.

Last State is no longer showing new OOMKilled terminations.

Application requests are succeeding consistently.

The deployment has the intended memory limit.

Current memory usage has been recorded.

Any emergency live-cluster change has been reflected in the authoritative deployment configuration.

Follow-up

After recovery:

Compare the configured memory limit with normal and peak observed usage.

Establish an appropriate memory request and limit based on real workload data.

Add validation to prevent resource limits that are obviously below observed requirements from being deployed.

Review whether resource changes should require comparison with current usage before deployment.

Preserve the incident timeline, commands, observed values, and final configuration for future capacity planning.

Incident-specific reminder: In the documented incident, reducing checkout-api from 128Mi to 8Mi caused the OOMs. The immediate recovery was to restore 128Mi; the longer-term task is to determine an evidence-based resource configuration rather than repeatedly tuning the limit by guesswork.