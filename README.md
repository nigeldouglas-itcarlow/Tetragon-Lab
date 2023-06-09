# Tetragon-Lab
Lab documentation for testing Tetragon Security Capabilities <br/>
<br/>
### Create a Lab Environment for Tetragon [without Cilium]
Run the below eksctl command to create a minimal one node EKS cluster
```
eksctl create cluster nigel-eks-cluster --node-type t3.xlarge --nodes=1 --nodes-min=0 --nodes-max=3 --max-pods-per-node 58
```

This process can take a good 10 mins to complete:

<img width="1105" alt="Screenshot 2023-06-09 at 20 00 43" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/a62527e7-80dc-4002-8e81-1c31a1f3f172">


### Create a Lab Environment for Tetragon [with Cilium as the CNI]
Run the below eksctl command to create a one-node cluster using the optimized ```Bottlerocket AMI``` that was designed for eBPF.

```
#!/usr/bin/env bash

export CLUSTER_NAME="nigel-eks-cluster"
export AWS_DEFAULT_REGION="eu-west-1"
export KUBECONFIG="/tmp/kubeconfig-${CLUSTER_NAME}.conf"

export TAGS="Owner=C00292053@itcarlow.ie Environment=staging"

set -euxo pipefail

cat > "/tmp/eksctl-${CLUSTER_NAME}.yaml" << EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  tags: &tags
$(echo "${TAGS}" | sed "s/ /\\n    /g; s/^/    /g; s/=/: /g")
iam:
  withOIDC: true
managedNodeGroups:
  - name: managed-ng-1
    amiFamily: AmazonLinux2
    # amiFamily: Bottlerocket
    instanceType: t3a.medium
    desiredCapacity: 1
    privateNetworking: true
    minSize: 0
    maxSize: 3
    volumeSize: 20
    volumeType: gp3
    maxPodsPerNode: 100    
    tags:
      <<: *tags
      compliance:na:defender: eks-node
      # compliance:na:defender: bottlerocket
    volumeEncrypted: false
    disableIMDSv1: true
    taints:
    - key: "node.cilium.io/agent-not-ready"
      value: "true"
      effect: "NoSchedule"
EOF

eksctl create cluster --config-file "/tmp/eksctl-${CLUSTER_NAME}.yaml" --kubeconfig "${KUBECONFIG}"

sleep 30

cilium install --helm-set cluster.name="${CLUSTER_NAME}"

cilium status --wait
# cilium connectivity test

echo -e "*****\n  export KUBECONFIG=${KUBECONFIG} \n*****"
```

## Background Checks

Confirm that ```AWS CLI``` is connected to your ```EKS Cluster``` in order to work from terminal. 
```
aws configure --profile nigel-aws-profile
export AWS_PROFILE=nigel-aws-profile                                            
aws sts get-caller-identity --profile nigel-aws-profile
aws eks update-kubeconfig --region eu-west-1 --name nigel-eks-cluster
```

Remember to scale-down the cluster to ```0 Nodes``` or ```delete the cluster``` when unused.
```
eksctl get cluster
eksctl get nodegroup --cluster nigel-eks-cluster
eksctl scale nodegroup --cluster nigel-eks-cluster --name ng-6194909f --nodes 0
eksctl delete cluster --name nigel-eks-cluster  
```
## Installation Workflow
There is actually an operator that installs the CRD, so you need to update the operator container image that will include the new CRD as well. 
```
helm -n kube-system upgrade \
--install tetragon cilium/tetragon \
--set tetragon.image.override=quay.io/cilium/tetragon-ci:latest \
--set tetragonOperator.image.override=quay.io/cilium/tetragon-operator-ci:latest \
--set imagePullPolicy=Always
```
Need to make sure that the CRDs are updated <br/>
You can delete them and restart the tetragon pod if they are not. <br/>
<br/>
Looks like you need to delete the old CRD created from earlier run of Tetra:
```
kubectl delete crd tracingpoliciesnamespaced.cilium.io
kubectl delete crd tracingpolicies.cilium.io
```
and reinstall Tetra should allow you to create network policy.
