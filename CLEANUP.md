# Cleanup Guide for MinIO on EKS with Argo CD

This document explains how to remove the MinIO resources created by this repository.

For deployment, architecture, validation, and operational details, see [README.md](D:/D/POCS/Gitlab_ArgoCD_EKS_MultiEnv/README.md).

## Important Notes

- The MinIO tenant uses `StorageClass` `minio-gp3-retain`.
- The PV reclaim policy is `Retain`.
- Deleting the tenant or PVCs does not automatically delete the EBS volumes.
- The exposed Services are AWS NLB-backed `LoadBalancer` Services, not ALBs.

Cleanup should be done in this order:

1. Delete test objects and buckets if needed
2. Delete the tenant Argo CD application
3. Delete the operator external/internal LB Service
4. Delete the operator Argo CD application
5. Delete PVCs/PVs if still present
6. Delete retained EBS volumes manually in AWS
7. Delete optional StorageClass manifests if no longer needed

## 1. Optional: Delete Test Buckets and Objects

If you created test buckets such as `minio-test-20260522174148`, remove them before deleting the tenant.

Run an `mc` client pod:

```powershell
kubectl -n minio-prod run mc-clean --rm -it --restart=Never --image=minio/mc -- sh
```

Inside the pod:

```sh
mc alias set local https://minio.minio-prod.svc.cluster.local minioadmin minioadmin123 --insecure
mc rm --recursive --force local/minio-test-20260522174148
mc rb local/minio-test-20260522174148
```

## 2. Delete the Tenant Application

Delete the tenant Argo CD application first:

```powershell
kubectl delete -f minio-tenant-app.yaml
```

Or directly by name:

```powershell
kubectl -n argocd delete application minio-prod-tenant
```

Watch tenant-side resources:

```powershell
kubectl -n minio-prod get tenant,pods,svc,pvc
```

## 3. Delete the Operator Console LoadBalancer Service

This removes the internal NLB created for the operator console:

```powershell
kubectl delete -f minio-operator-console-lb.yaml
```

Or directly:

```powershell
kubectl -n minio-operator delete svc minio-operator-console-lb
```

Check deletion:

```powershell
kubectl -n minio-operator get svc
```

## 4. Delete the Operator Application

After the tenant is removed, delete the operator Argo CD application:

```powershell
kubectl delete -f minio-operator-app.yaml
```

Or directly:

```powershell
kubectl -n argocd delete application minio-operator
```

Watch operator-side resources:

```powershell
kubectl -n minio-operator get all
kubectl get crd | Select-String minio
```

## 5. Delete Namespace Resources if Anything Remains

If any resources remain after Argo CD app deletion:

```powershell
kubectl delete namespace minio-prod
kubectl delete namespace minio-operator
```

Check:

```powershell
kubectl get ns
```

## 6. Delete PVCs and PVs

Because the MinIO PVCs use `Retain`, storage objects may remain after tenant deletion.

List PVCs and PVs:

```powershell
kubectl -n minio-prod get pvc
kubectl get pv | Select-String minio-prod
```

Delete PVCs if still present:

```powershell
kubectl -n minio-prod delete pvc --all
```

Delete MinIO PVs if still present:

```powershell
kubectl delete pv pvc-7c617972-90b6-4d26-b149-78e961314dc0
kubectl delete pv pvc-07a74c28-5127-4059-ad3b-644a2923d5b0
kubectl delete pv pvc-7c30d5cc-e0a3-42bd-95e1-fe253544d2a5
kubectl delete pv pvc-f69dec7d-b2bc-42a9-9fdc-e940c631806e
```

If the PV is stuck with a claimRef, inspect first:

```powershell
kubectl get pv <pv-name> -o yaml
```

## 7. Delete Backing EBS Volumes from AWS

Deleting the PV objects does not remove the EBS volumes when reclaim policy is `Retain`.

The MinIO PVs that were created in this setup were:

- `pvc-7c617972-90b6-4d26-b149-78e961314dc0`
- `pvc-07a74c28-5127-4059-ad3b-644a2923d5b0`
- `pvc-7c30d5cc-e0a3-42bd-95e1-fe253544d2a5`
- `pvc-f69dec7d-b2bc-42a9-9fdc-e940c631806e`

Find the EBS volume IDs:

```powershell
kubectl get pv pvc-7c617972-90b6-4d26-b149-78e961314dc0 -o yaml
kubectl get pv pvc-07a74c28-5127-4059-ad3b-644a2923d5b0 -o yaml
kubectl get pv pvc-7c30d5cc-e0a3-42bd-95e1-fe253544d2a5 -o yaml
kubectl get pv pvc-f69dec7d-b2bc-42a9-9fdc-e940c631806e -o yaml
```

Or list them in AWS:

```powershell
aws ec2 describe-volumes --region us-east-1
```

Delete the retained EBS volumes once you confirm they are no longer needed:

```powershell
aws ec2 delete-volume --volume-id <vol-id-1> --region us-east-1
aws ec2 delete-volume --volume-id <vol-id-2> --region us-east-1
aws ec2 delete-volume --volume-id <vol-id-3> --region us-east-1
aws ec2 delete-volume --volume-id <vol-id-4> --region us-east-1
```

## 8. Delete Load Balancers in AWS if They Remain

Normally, deleting the Kubernetes `LoadBalancer` Services removes the NLBs automatically.

Check Services first:

```powershell
kubectl -n minio-prod get svc
kubectl -n minio-operator get svc
```

If an AWS NLB still remains after the Service is gone, inspect it in AWS and remove it manually.

List load balancers:

```powershell
aws elbv2 describe-load-balancers --region us-east-1
```

Delete manually only if Kubernetes no longer manages them:

```powershell
aws elbv2 delete-load-balancer --load-balancer-arn <nlb-arn> --region us-east-1
```

## 9. Optional: Delete StorageClass Manifests

If the storage classes are no longer needed:

```powershell
kubectl delete -f minio-gp3-retain-sc.yaml
kubectl delete -f minio-storageclass.yaml
```

Or directly:

```powershell
kubectl delete storageclass minio-gp3-retain
kubectl delete storageclass gp3-wait
```

## 10. Final Verification

Confirm cleanup:

```powershell
kubectl get application -n argocd
kubectl get ns
kubectl get pv | Select-String minio
kubectl get svc -A | Select-String minio
aws elbv2 describe-load-balancers --region us-east-1
aws ec2 describe-volumes --region us-east-1
```

## Quick Cleanup Sequence

If you want the shortest sequence:

```powershell
kubectl -n argocd delete application minio-prod-tenant
kubectl -n minio-operator delete svc minio-operator-console-lb
kubectl -n argocd delete application minio-operator
kubectl delete namespace minio-prod
kubectl delete namespace minio-operator
kubectl delete storageclass minio-gp3-retain
kubectl delete storageclass gp3-wait
```

Then manually verify and delete retained PVs and EBS volumes in AWS.
