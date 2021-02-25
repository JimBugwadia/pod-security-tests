# Pod Security Tests

This repository contains test pod YAMLs for each policy type defined in the Kubernetes [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

## Usage

**1. Configure a Policy Instantiation**

Configure a [Policy Instantiation](https://kubernetes.io/docs/concepts/security/pod-security-standards/#policy-instantiation) for your cluster, like Kubernetes [Pod Security Policies (PSPs)](https://kubernetes.io/docs/concepts/security/pod-security-standards/#policy-instantiation), or policy engines like [OPA/Gatekeeper](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/pod-security-policy) and [Kyverno](https://kyverno.io/policies/pod-security/).

**2. Apply test YAMLs**

Apply all test pod YAMLs using the command:

```shell
kustomize build https://github.com/JimBugwadia/pod-security-tests/ | kubectl apply -f -
```

Or, apply `Baseline` or `Restricted` YAMLs by appending the appropriate folder name:
  * Baseline: `kustomize build https://github.com/JimBugwadia/pod-security-tests/baseline`
  * Restricted: `kustomize build https://github.com/JimBugwadia/pod-security-tests/restricted`

**3. Check the created pods**

The command below checks pods in the `default` namespace. For this test, 

```sh
λ kubectl get pod --show-labels
```

Assuming all privileged pods were blocked via admission controls, this should show:

```sh
NAME         READY   STATUS    RESTARTS   AGE   LABELS
good-pod     1/1     Running   0          68s   <none>
proc-mount   1/1     Running   0          68s   pod-security-standards/control=Proc_Mount_Type,pod-security-standards/policy=Baseline
```

The full set of YAMLs includes a `good-pod` that is compliant with the `Restricted` policy.  

The `proc-mount` pod will be created if the `ProcMountType` feature flag is not enabled, as otherwise `securityContext.procMount` will be set to `Default` by the kubelet. This can be verified using the command below:

```sh
kubectl get pods proc-mount -o=jsonpath="{.spec.containers[*].securityContext.procMount}"
```

This output of the above command is expected to be `Default`.


## Test Pod Labels

All test pods are have the following labels defined:
* `pod-security-standards/policy` with value of `Baseline` or `Restricted`.
* `pod-security-standards/control` with a value that identifies the security control being tested.

You can view test pods for a policy level using these labels. 

For example, this command shows pods that test the `Restricted` policy controls:

```sh
kubectl get pods -l "pod-security-standards/policy=Restricted"
```

This command shows pods that test both policies:


```sh
kubectl get pods -l "pod-security-standards/policy in (Restricted, Baseline)" --show-labels
```


## Pod Security Controls to Test Mappings

The tables below show each control specified in the [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) and the associated policies and mappings to test YAMLs.

### Baseline/Default

The Baseline/Default policy is aimed at ease of adoption for common containerized workloads while preventing known privilege escalations. This policy is targeted at application operators and developers of non-critical applications. The following listed controls should be enforced/disallowed:

<table>
	<caption style="display:none">Baseline policy specification</caption>
	<tbody>
		<tr>
			<td><strong>Control</strong></td>
			<td><strong>Policy</strong></td>
			<td><strong>Test YAMLs</strong></td>
		</tr>
		<tr>
			<td>Host Namespaces</td>
			<td>
				Sharing the host namespaces must be disallowed.<br>
				<br><b>Restricted Fields:</b><br>
				spec.hostNetwork<br>
				spec.hostPID<br>
				spec.hostIPC<br>
				<br><b>Allowed Values:</b> false<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-host-namespaces.yaml" target="_blank">test-disallow-host-namespaces.yaml</a>
            </td>
		</tr>
		<tr>
			<td>Privileged Containers</td>
			<td>
				Privileged Pods disable most security mechanisms and must be disallowed.<br>
				<br><b>Restricted Fields:</b><br>
				spec.containers[*].securityContext.privileged<br>
				spec.initContainers[*].securityContext.privileged<br>
				<br><b>Allowed Values:</b> false, undefined/nil<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-privileged-containers.yaml" target="_blank">test-disallow-privileged-containers.yaml</a>
            </td>
		</tr>
		<tr>
			<td>Capabilities</td>
			<td>
				Adding additional capabilities beyond the <a href="https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities">default set</a> must be disallowed.<br>
				<br><b>Restricted Fields:</b><br>
				spec.containers[*].securityContext.capabilities.add<br>
				spec.initContainers[*].securityContext.capabilities.add<br>
				<br><b>Allowed Values:</b> empty (or restricted to a known list)<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-adding-capabilities.yaml" target="_blank">test-disallow-adding-capabilities.yaml</a>
            </td>
		</tr>
		<tr>
			<td>HostPath Volumes</td>
			<td>
				HostPath volumes must be forbidden.<br>
				<br><b>Restricted Fields:</b><br>
				spec.volumes[*].hostPath<br>
				<br><b>Allowed Values:</b> undefined/nil<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-host-path.yaml" target="_blank">test-disallow-host-path.yaml</a>
            </td>            
		</tr>
		<tr>
			<td>Host Ports</td>
			<td>
				HostPorts should be disallowed, or at minimum restricted to a known list.<br>
				<br><b>Restricted Fields:</b><br>
				spec.containers[*].ports[*].hostPort<br>
				spec.initContainers[*].ports[*].hostPort<br>
				<br><b>Allowed Values:</b> 0, undefined (or restricted to a known list)<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-host-ports.yaml" target="_blank">test-disallow-host-ports.yaml</a>
            </td>            
		</tr>
		<tr>
			<td>AppArmor <em>(optional)</em></td>
			<td>
				On supported hosts, the 'runtime/default' AppArmor profile is applied by default. The default policy should prevent overriding or disabling the policy, or restrict overrides to an allowed set of profiles.<br>
				<br><b>Restricted Fields:</b><br>
				metadata.annotations['container.apparmor.security.beta.kubernetes.io/*']<br>
				<br><b>Allowed Values:</b> 'runtime/default', undefined<br>
			</td>
            <td>
                <a href="tests/default/test-restrict-apparmor-profiles.yaml" target="_blank">test-restrict-apparmor-profiles.yaml</a>
            </td>            
		</tr>
		<tr>
			<td>SELinux <em>(optional)</em></td>
			<td>
				Setting custom SELinux options should be disallowed.<br>
				<br><b>Restricted Fields:</b><br>
				spec.securityContext.seLinuxOptions<br>
				spec.containers[*].securityContext.seLinuxOptions<br>
				spec.initContainers[*].securityContext.seLinuxOptions<br>
				<br><b>Allowed Values:</b> undefined/nil<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-selinux.yaml" target="_blank">test-disallow-selinux.yaml</a>
            </td>                 
		</tr>
		<tr>
			<td>/proc Mount Type</td>
			<td>
				The default /proc masks are set up to reduce attack surface, and should be required.<br>
				<br><b>Restricted Fields:</b><br>
				spec.containers[*].securityContext.procMount<br>
				spec.initContainers[*].securityContext.procMount<br>
				<br><b>Allowed Values:</b> undefined/nil, 'Default'<br>
			</td>
            <td>
                <a href="tests/default/test-disallow-proc-mount.yaml" target="_blank">test-disallow-proc-mount.yaml</a>
            </td>      
		</tr>
		<tr>
			<td>Sysctls</td>
			<td>
				Sysctls can disable security mechanisms or affect all containers on a host, and should be disallowed except for an allowed "safe" subset.
				A sysctl is considered safe if it is namespaced in the container or the Pod, and it is isolated from other Pods or processes on the same Node.<br>
				<br><b>Restricted Fields:</b><br>
				spec.securityContext.sysctls<br>
				<br><b>Allowed Values:</b><br>
				kernel.shm_rmid_forced<br>
				net.ipv4.ip_local_port_range<br>
				net.ipv4.tcp_syncookies<br>
				net.ipv4.ping_group_range<br>
				undefined/empty<br>
			</td>
            <td>
                <a href="tests/default/test-restrict-sysctls.yaml" target="_blank">test-restrict-sysctls.yaml</a>
            </td>              
		</tr>
	</tbody>
</table>

### Restricted

The Restricted policy is aimed at enforcing current Pod hardening best practices, at the expense of
some compatibility. It is targeted at operators and developers of security-critical applications, as
well as lower-trust users.The following listed controls should be enforced/disallowed:


<table>
	<caption style="display:none">Restricted policy specification</caption>
	<tbody>
		<tr>
			<td><strong>Control</strong></td>
			<td><strong>Policy</strong></td>
			<td><strong>Test YAMLs</strong></td>
		</tr>
		<tr>
			<td colspan="2"><em>Everything from the default profile.</em></td>
		</tr>
		<tr>
			<td>Volume Types</td>
			<td>
				In addition to restricting HostPath volumes, the restricted profile limits usage of non-core volume types to those defined through PersistentVolumes.<br>
				<br><b>Restricted Fields:</b><br>
				spec.volumes[*].hostPath<br>
				spec.volumes[*].gcePersistentDisk<br>
				spec.volumes[*].awsElasticBlockStore<br>
				spec.volumes[*].gitRepo<br>
				spec.volumes[*].nfs<br>
				spec.volumes[*].iscsi<br>
				spec.volumes[*].glusterfs<br>
				spec.volumes[*].rbd<br>
				spec.volumes[*].flexVolume<br>
				spec.volumes[*].cinder<br>
				spec.volumes[*].cephFS<br>
				spec.volumes[*].flocker<br>
				spec.volumes[*].fc<br>
				spec.volumes[*].azureFile<br>
				spec.volumes[*].vsphereVolume<br>
				spec.volumes[*].quobyte<br>
				spec.volumes[*].azureDisk<br>
				spec.volumes[*].portworxVolume<br>
				spec.volumes[*].scaleIO<br>
				spec.volumes[*].storageos<br>
				spec.volumes[*].csi<br>
				<br><b>Allowed Values:</b> undefined/nil<br>
			</td>
            <td>
                <a href="tests/restricted/test-restrict-volume-types.yaml" target="_blank">test-restrict-volume-types.yaml</a>
            </td>             
		</tr>
		<tr>
			<td>Privilege Escalation</td>
			<td>
				Privilege escalation (such as via set-user-ID or set-group-ID file mode) should not be allowed.<br>
				<br><b>Restricted Fields:</b><br>
				spec.containers[*].securityContext.allowPrivilegeEscalation<br>
				spec.initContainers[*].securityContext.allowPrivilegeEscalation<br>
				<br><b>Allowed Values:</b> false<br>
			</td>
            <td>
                <a href="tests/restricted/test-deny-privilege-escalation.yaml" target="_blank">test-deny-privilege-escalation.yaml</a>
            </td>            
		</tr>
		<tr>
			<td>Running as Non-root</td>
			<td>
				Containers must be required to run as non-root users.<br>
				<br><b>Restricted Fields:</b><br>
				spec.securityContext.runAsNonRoot<br>
				spec.containers[*].securityContext.runAsNonRoot<br>
				spec.initContainers[*].securityContext.runAsNonRoot<br>
				<br><b>Allowed Values:</b> true<br>
			</td>
            <td>
                <a href="tests/restricted/test-require-run-as-nonroot.yaml" target="_blank">test-require-run-as-nonroot.yaml</a>
            </td>            
		</tr>
		<tr>
			<td>Non-root groups <em>(optional)</em></td>
			<td>
				Containers should be forbidden from running with a root primary or supplementary GID.<br>
				<br><b>Restricted Fields:</b><br>
				spec.securityContext.runAsGroup<br>
				spec.securityContext.supplementalGroups[*]<br>
				spec.securityContext.fsGroup<br>
				spec.containers[*].securityContext.runAsGroup<br>
				spec.initContainers[*].securityContext.runAsGroup<br>
				<br><b>Allowed Values:</b><br>
				non-zero<br>
				undefined / nil (except for `*.runAsGroup`)<br>
			</td>
            <td>
                <a href="tests/restricted/test-non-root-groups.yaml" target="_blank">test-non-root-groups.yaml</a>
            </td>            
		</tr>
		<tr>
			<td>Seccomp</td>
			<td>
				The RuntimeDefault seccomp profile must be required, or allow specific additional profiles.<br>
				<br><b>Restricted Fields:</b><br>
				spec.securityContext.seccompProfile.type<br>
				spec.containers[*].securityContext.seccompProfile<br>
				spec.initContainers[*].securityContext.seccompProfile<br>
				<br><b>Allowed Values:</b><br>
				'runtime/default'<br>
				undefined / nil<br>
			</td>
            <td>
                <a href="tests/restricted/test-restrict-seccomp.yaml" target="_blank">test-restrict-seccomp.yaml</a>
            </td>            
		</tr>
	</tbody>
</table>



