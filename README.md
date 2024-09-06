# Securing LKE Cluster Part 2: CIS Benchmarks with Kube-Bench

## Introduction
Securing your Kubernetes clusters is crucial, and one of the best ways to ensure your cluster's security is to benchmark it against the CIS Kubernetes Benchmark. In this guide, we'll walk you through using **Kube-Bench** on a Linode Kubernetes Engine (LKE) cluster to perform security checks and identify potential risks. 

By the end of this guide, you'll be able to:
1. Deploy an LKE cluster.
2. Run Kube-Bench to check for CIS benchmark violations.
3. Identify which issues you can fix at the cluster level and where you'll need assistance from Linode.

---

## Step 1: Deploy LKE Cluster and Connect to It

Before you begin with Kube-Bench, you need an active Kubernetes cluster. Follow these steps to set up your cluster and connect to it:

1. **Deploy the LKE cluster:** Use the Linode Cloud Manager or `linode-cli` to deploy your LKE cluster.
2. **Download the kubeconfig file:** After your cluster is up, download the `kubeconfig` file from the LKE dashboard.
3. **Connect to the cluster:**
   ```bash
   export KUBECONFIG=~/path-to-your-kubeconfig-file
   kubectl get nodes
   ```

---

## Step 2: Create the `kubebench-job.yaml` File

To run Kube-Bench in your LKE cluster, you'll need to define a Kubernetes job that will execute the Kube-Bench container.

1. Create a file named `kubebench-job.yaml` and add the following content:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    metadata:
      labels:
        app: kube-bench
    spec:
      containers:
        - command: ["kube-bench"]
          image: docker.io/aquasec/kube-bench:v0.8.0
          name: kube-bench
          volumeMounts:
            - name: var-lib-cni
              mountPath: /var/lib/cni
              readOnly: true
            - mountPath: /var/lib/etcd
              name: var-lib-etcd
              readOnly: true
            - mountPath: /var/lib/kubelet
              name: var-lib-kubelet
              readOnly: true
            - mountPath: /var/lib/kube-scheduler
              name: var-lib-kube-scheduler
              readOnly: true
            - mountPath: /var/lib/kube-controller-manager
              name: var-lib-kube-controller-manager
              readOnly: true
            - mountPath: /etc/systemd
              name: etc-systemd
              readOnly: true
            - mountPath: /lib/systemd/
              name: lib-systemd
              readOnly: true
            - mountPath: /srv/kubernetes/
              name: srv-kubernetes
              readOnly: true
            - mountPath: /etc/kubernetes
              name: etc-kubernetes
              readOnly: true
            - mountPath: /usr/local/mount-from-host/bin
              name: usr-bin
              readOnly: true
            - mountPath: /etc/cni/net.d/
              name: etc-cni-netd
              readOnly: true
            - mountPath: /opt/cni/bin/
              name: opt-cni-bin
              readOnly: true
      hostPID: true
      restartPolicy: Never
      volumes:
        - name: var-lib-cni
          hostPath:
            path: /var/lib/cni
        - hostPath:
            path: /var/lib/etcd
          name: var-lib-etcd
        - hostPath:
            path: /var/lib/kubelet
          name: var-lib-kubelet
        - hostPath:
            path: /var/lib/kube-scheduler
          name: var-lib-kube-scheduler
        - hostPath:
            path: /var/lib/kube-controller-manager
          name: var-lib-kube-controller-manager
        - hostPath:
            path: /etc/systemd
          name: etc-systemd
        - hostPath:
            path: /lib/systemd
          name: lib-systemd
        - hostPath:
            path: /srv/kubernetes
          name: srv-kubernetes
        - hostPath:
            path: /etc/kubernetes
          name: etc-kubernetes
        - hostPath:
            path: /usr/bin
          name: usr-bin
        - hostPath:
            path: /etc/cni/net.d/
          name: etc-cni-netd
        - hostPath:
            path: /opt/cni/bin/
          name: opt-cni-bin
```

---

## Step 3: Apply the Kube-Bench Job to the Cluster

Now that you have the job definition, run the following command to deploy it:

```bash
kubectl apply -f kubebench-job.yaml
```

This will create a Kubernetes job that runs the Kube-Bench container in your cluster.

---

## Step 4: Check Logs for Results

To check the results of Kube-Bench, you need to retrieve the logs from the job’s pod. Use the following command to find the pod name:

```bash
kubectl get pods
```

Then, view the logs:

```bash
kubectl logs <kube-bench-pod-name>
```

Kube-Bench will report whether your cluster passed or failed each CIS benchmark check. You’ll see results for each section (such as API server, scheduler, etc.).

---

## Step 5: Addressing Issues on LKE

On an LKE cluster, you may have limited access to modify certain security configurations on the worker nodes since Linode manages the infrastructure. However, there are several actions you can take at the control plane or namespace level to enhance security. Below is a breakdown of what you can and cannot address:

### Issues You Can Address

1. **RBAC and Security Issues:**
   - **[FAIL] 5.1.1: Ensure the cluster-admin role is only used where required.**
     - **Fix**: Audit role bindings and remove unnecessary cluster-admin role assignments.
     - Example:
     ```bash
     kubectl get clusterrolebindings
     kubectl delete clusterrolebinding <binding_name>
     ```
   - **[FAIL] 5.1.2: Minimize access to secrets.**
     - **Fix**: Reduce access to Secrets in roles and bindings.
     - Example:
     ```bash
     kubectl edit clusterrole <role_name>
     ```
   - **[FAIL] 5.1.3: Minimize wildcard use in Roles and ClusterRoles.**
     - **Fix**: Modify ClusterRoles or Roles to replace wildcards with specific permissions.

2. **Service Accounts:**
   - **[FAIL] 5.1.5: Ensure default service accounts are not actively used.**
     - **Fix**: Disable service account token mounting where not needed.
     ```yaml
     automountServiceAccountToken: false
     ```

3. **Pod Security Policies and Admission Controls:**
   - **[WARN] 5.2.x: Issues like restricting privileged containers and enforcing security contexts.**
     - **Fix**: Use PodSecurityPolicies or alternatives like Open Policy Agent.

4. **Network Policies:**
   - **[WARN] 5.3.x: Ensure network policies are applied.**
     - **Fix**: Define NetworkPolicies for each namespace to control pod communication.
     ```yaml
     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: default-deny
       namespace: <your-namespace>
     spec:
       podSelector: {}
       policyTypes:
         - Ingress
         - Egress
     ```

5. **Secrets Management:**
   - **[WARN] 5.4.x: Prefer using secrets as files over environment variables.**
     - **Fix**: Mount secrets as files instead of passing them as environment variables.

---

## Issues You Likely Can’t Fix

1. **File Permission Issues on Worker Nodes:**
   - Some file permission issues like `/lib/systemd/system/kubelet.service` or `/var/lib/kubelet/config.yaml` are managed by Linode.
   - **Solution**: You cannot fix these directly, but you can raise these issues with Linode support.
---

## Conclusion

While some aspects of your cluster’s security are out of your control on LKE (due to managed worker nodes), you can still implement significant security improvements at the control plane and namespace levels. Focus on addressing RBAC policies, pod security contexts, network policies, and secrets management. For file permission issues, collaborate with Linode’s support team to resolve these.

By following this guide, you'll be able to secure your LKE clusters against common vulnerabilities and ensure compliance with CIS benchmarks.
