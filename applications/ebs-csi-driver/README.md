# ebs-csi-driver

```bash
# Create IRSA for ebs csi driver
eksctl create iamserviceaccount \
  --cluster eksdemo \
  --region us-east-1 \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```