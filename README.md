# Tetragon-Lab
Lab documentation for testing Tetragon Security Capabilities <br/>
<br/>
### Create a Lab Environment for Tetragon [without Cilium]
Run the below eksctl command to create a minimal one node EKS cluster
```
eksctl create cluster nigel-eks-cluster --node-type t3.xlarge --nodes=1 --nodes-min=0 --nodes-max=3 --max-pods-per-node 58
```
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

