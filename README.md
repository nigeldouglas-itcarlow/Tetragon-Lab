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

## Deploy Tetragon
To install and deploy Tetragon, run the following commands:
```
helm repo add cilium https://helm.cilium.io
helm repo update
helm install tetragon cilium/tetragon -n kube-system
kubectl rollout status -n kube-system ds/tetragon -w
```

<img width="1105" alt="Screenshot 2023-06-09 at 20 10 21" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/1a8cf86e-b826-476b-825a-ce9ace72c609">

## Create a Privileged Pod
```
kubectl apply -f https://raw.githubusercontent.com/nigeldouglas-itcarlow/Tetragon-Lab/main/privileged-pod.yaml
```

<img width="1105" alt="Screenshot 2023-06-09 at 20 13 55" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/07e33971-351c-4b81-84ba-f9b4d1099605">

## Detect Cryptomining using Tetragon
Enable visibility to capability & namespace changes via the ```configmap``` by setting ```enable-process-cred``` and ```enable-process-ns``` to ```true```:
```
kubectl edit cm -n kube-system tetragon-config
```

<img width="1105" alt="Screenshot 2023-06-09 at 20 20 58" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/e1d176ec-7ecd-4e89-a41d-737423fb6e60">


Restart the Tetragon daemonset to enforce those changes:
```
kubectl rollout restart -n kube-system ds/tetragon
```

The Tetragon agent has since restarted and therefore is only 21 seconds old:

<img width="1105" alt="Screenshot 2023-06-09 at 20 22 15" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/a7ced293-c58b-4150-9029-e8e4bf5031ca">


Let's then open a new terminal window to monitor the events from the overly-permissive pod:
```
kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f | tetra getevents -o compact --namespace default --pod test-pod-1
```

We are already alerted on the fact that our pod has elevated admin privileges - ```CAP_SYS_ADMIN```
<img width="1416" alt="Screenshot 2023-06-09 at 20 31 14" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/f31065a1-4081-43ee-9e7f-354e384df9f2">


In the first window, terminal shell into the overly-permissive pod:
```
kubectl exec -it nigel-app -- bash
```
We receive a bunch of process activity after we shell into the pod. <br/>
However, the data is not so usefil in its current state.
<img width="1416" alt="Screenshot 2023-06-09 at 20 33 20" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/d9ab78ab-3dd3-405f-8391-0ced449d4bcc">


Download the ```xmrig``` binary from the official Github repository:
```
curl -OL https://github.com/xmrig/xmrig/releases/download/v6.16.4/xmrig-6.16.4-linux-static-x64.tar.gz
```

We are definitely seeing the activity in realtime. <br/>
However, the only context is that the process started and then there was an exit:

<img width="1416" alt="Screenshot 2023-06-09 at 20 35 04" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/f4c6abc8-f556-4ae4-8afb-a48ce186464c">


Unzip the tarbal package to access the malicious files:
```
tar -xvf xmrig-6.16.4-linux-static-x64.tar.gz
```

Move to the directory holding the miner:
```
cd xmrig-6.16.4
```
For the purpose of testing, run ```chmod``` to trigger the SetGid Bit detection:
```
chmod u+s xmrig
```
Should trigger the detection, but there's likely no actual change here:
```
find / -perm /6000 -type f
```
Run the cryptominer in background mode (this won't show anything in your shell)
```
./xmrig --donate-level 8 -o xmr-us-east1.nanopool.org:14433 -u 422skia35WvF9mVq9Z9oCMRtoEunYQ5kHPvRqpH1rGCv1BzD5dUY4cD8wiCMp4KQEYLAN1BuawbUEJE99SNrTv9N9gf2TWC --tls --coin monero
```

After performing each of the tasks, we realize that further testing is required:

<img width="1416" alt="Screenshot 2023-06-09 at 20 38 57" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/69b50f50-773f-4a9e-875f-482dfedad62d">

## Use TracingPolicy

```TracingPolicy``` is a user-configurable Kubernetes custom resource that allows users to trace arbitrary events in the kernel and optionally define actions to take on a match. We can enable it by running the below command:
```
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/tracingpolicy/sys_write_follow_fd_prefix.yaml
```

After enabling TracingPolicy, I have far more insight into when files are opened/closed - which is a good start:

<img width="1416" alt="Screenshot 2023-06-09 at 20 43 27" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/40e48e36-ec51-4a87-84f5-07d12d7287ec">

When my shell session did crash, probably due to the sleep configuration, it sent the ```SIGKILL``` action:

<img width="1416" alt="Screenshot 2023-06-09 at 20 47 14" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/9d27c956-6515-43b0-b6ed-59257b07eabf">

To disable the TracingPolicy run:
```
kubectl delete -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/tracingpolicy/sys_write_follow_fd_prefix.yaml
```


## Monitoring Network Activity

To view TCP connect events, apply the example TCP connect TracingPolicy:
```
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/tracingpolicy/tcp-connect.yaml
```

The TracingPolicy "connect" is invalid: <br/>
* spec.kprobes[0].selectors[0].matchArgs[0].operator: Unsupported value: "DAddr": supported values: "Equal", "NotEqual", "Prefix", "Postfix" <br/>
<br/>

Apparently there's an operator that installs the CRD, so you need to update the operator container image that will include the new CRD as well. 
```
helm -n kube-system upgrade \
--install tetragon cilium/tetragon \
--set tetragon.image.override=quay.io/cilium/tetragon-ci:latest \
--set tetragonOperator.image.override=quay.io/cilium/tetragon-operator-ci:latest \
--set imagePullPolicy=Always
```
I did this to no change:

<img width="1416" alt="Screenshot 2023-06-09 at 20 53 25" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/85d76a31-f780-45c2-ac45-95587e6f144d">


Need to make sure that the CRDs are updated <br/>
You can delete them and restart the tetragon pod if they are not. <br/>
<br/>
Looks like you need to delete the old CRD created from earlier run of Tetra:
```
kubectl delete crd tracingpoliciesnamespaced.cilium.io
kubectl delete crd tracingpolicies.cilium.io
```
and reinstalling Tetra should allow you to create network policy. <br/>
Again, this did not work. I turns out I need those Cilium CRD's


<img width="1416" alt="Screenshot 2023-06-09 at 20 55 35" src="https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/ea2e5f2e-25a4-4262-ae71-ba665d2f0a49">

However, I cannot delete the resource associated with ```tracingpolicies.cilium.io``` <br/>
```connect``` was not found in the first place:

```
kubectl delete -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/tracingpolicy/tcp-connect.yaml
```

<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

## The plan was to sigkill processes that use specific protocols - like Stratum:

Use the Stratum protocol
```
./xmrig -o stratum+tcp://xmr.pool.minergate.com:45700 -u lies@lies.lies -p x -t 2
```

I started by killing a process when the user attempts to open any file in the ```/tmp``` directory:
```
kubectl apply -f https://raw.githubusercontent.com/nigeldouglas-itcarlow/Tetragon-Lab/main/sigkill-example.yaml
```

This totally worked!!

![Screenshot 2023-06-30 at 14 56 59](https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/fb10a01e-afa6-4a0b-96f6-de1027c3cd2c)

I used this TracingProfile as the foundation for my Tetragon SigKill rule:
```
https://gist.github.com/henrysachs/1975a8fe862216b4301698c8c3135e85
```

Naturally, I don't need this ```TracingProfile``` in the real world. So I deleted it.
```
kubectl delete -f https://raw.githubusercontent.com/nigeldouglas-itcarlow/Tetragon-Lab/main/sigkill-example.yaml
```

I also wanted to grep for only cases where the SigKill was successful. The rest is just noise in testing:
```
kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f | tetra getevents -o compact --namespace default --pod test-pod-1 | grep exit
```

![Screenshot 2023-06-30 at 15 06 14](https://github.com/nigeldouglas-itcarlow/Tetragon-Lab/assets/126002808/299add2d-b4e7-4989-85e2-2a7e12d8c192)

## Testing DNS Policy

Installing a suspicious networking tool like telnet
```
yum install telnet telnet-server -y
```
If this fails, just apply a few modifications to the registry management:
```
cd /etc/yum.repos.d/
```
```
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```
Update the yum registry manager:
```
yum update -y
```
Now, try to install telnet and telnet server from the registry manager:
```
yum install telnet telnet-server -y
```
```
yum install bind-utils
```
In general, you can search for the ```nslookup``` package provides a command using the yum provides command:
```
yum provides '*bin/nslookup'
```
Just to generate the detection, run nslookup or ```telnet```:
```
nslookup ebpf.io
```

```
telnet
```
Let's also test tcpdump to prove the macro is working:
```
yum install tcpdump -y
tcpdump -D
tcpdump --version
tcpdump -nnSX port 443
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
