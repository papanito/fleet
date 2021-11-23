# Troubleshooting

This section contains commands and tips to troubleshoot Fleet.

## Check status of fleet controller

You can check the status of the Fleet controller pods by running the commands below:

```bash
kubectl -n fleet-system logs -l app=fleet-controller
kubectl -n fleet-system get pods -l app=fleet-controller
```

```bash
NAME                                READY   STATUS    RESTARTS   AGE
fleet-controller-64f49d756b-n57wq   1/1     Running   0          3m21s
```

## Fleet fails with bad response code: 403

If your GitJob returns the error below, the problem may be that Fleet cannot access the Helm repo you specified in your [`fleet.yaml`](./gitrepo-structure.md):

```
time="2021-11-04T09:21:24Z" level=fatal msg="bad response code: 403"
```

Perform the following steps to assess:

- Check that your repo is accessible from your dev machine, and that you can download the Helm chart successfully
- Check that your credentials for the git repo are valid

## Helm chart repo: certificate signed by unknown authority

If your GitJob returns the error below, you may have added the wrong certificate chain:

```
time="2021-11-11T05:55:08Z" level=fatal msg="Get \"https://helm.intra/virtual-helm/index.yaml\": x509: certificate signed by unknown authority" 
```

Please verify your certificate with the following command:

```bash
context=playground-local
kubectl get secret -n fleet-default helm-repo -o jsonpath="{['data']['cacerts']}" --context $context | base64 -d | openssl x509 -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7a:1e:df:79:5f:b0:e0:be:49:de:11:5e:d9:9c:a9:71
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C = CH, O = MY COMPANY, CN = NOP Root CA G3
...

```

## Fleet deployment stuck in modified state

When you deploy bundles to Fleet, some of the components are modified, and this causes the "modified" flag in the Fleet environment.

To ignore the modified flag for the differences between the Helm install generated by `fleet.yaml` and the resource in your cluster, add a `diff.comparePatches` to the `fleet.yaml` for your Deployment, as shown in this example:


```yaml
defaultNamespace: <namespace name> 
helm:  
  releaseName: <release name>  
  repo: <repo name> 
  chart: <chart name>
diff:  
  comparePatches:  
  - apiVersion: apps/v1
    kind: Deployment
    operations:
    - {"op":"remove", "path":"/spec/template/spec/hostNetwork"}
    - {"op":"remove", "path":"/spec/template/spec/nodeSelector"}
    jsonPointers: # jsonPointers allows to ignore diffs at certain json path
    - "/spec/template/spec/priorityClassName"
    - "/spec/template/spec/tolerations" 
```

To determine which operations should be removed, observe the logs from `fleet-agent` on the target cluster. You should see entries similar to the following:

```text
level=error msg="bundle monitoring-monitoring: deployment.apps monitoring/monitoring-monitoring-kube-state-metrics modified {\"spec\":{\"template\":{\"spec\":{\"hostNetwork\":false}}}}"
```

Based on the above log, you can add the following entry to remove the operation:

```json
{"op":"remove", "path":"/spec/template/spec/hostNetwork"}
```