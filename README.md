# Tetragon-Lab
Lab documentation for testing Tetragon Security Capabilities <br/>
<br/>
### Create a Lab Environment for Tetragon

```
#!/usr/bin/env bash

export CLUSTER_NAME="${USER}-cilium"
export AWS_DEFAULT_REGION="eu-central-1"
export KUBECONFIG="/tmp/kubeconfig-${CLUSTER_NAME}.conf"

export TAGS="Owner=petr.ruzicka@gmail.com Environment=dev"

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
    desiredCapacity: 2
    privateNetworking: true
    minSize: 2
    maxSize: 3
    volumeSize: 20
    volumeType: gp3
    maxPodsPerNode: 100    
    tags:
      <<: *tags
      compliance:na:defender: eks-node
      # compliance:na:defender: bottlerocket
    volumeEncrypted: true
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

