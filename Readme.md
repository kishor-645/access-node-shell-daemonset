To run commands directly on a Kubernetes node when you don't have SSH access, you can use a **privileged DaemonSet**. This method uses `nsenter` to "break out" of the container and execute commands in the host's namespace.

### Step 1: Create the YAML file
Create a file named `node-access.yaml`. This configuration grants the pod root access to the host's process tree and filesystem.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-shell
  namespace: kube-system
  labels:
    app: node-shell
spec:
  selector:
    matchLabels:
      app: node-shell
  template:
    metadata:
      labels:
        app: node-shell
    spec:
      # Access host namespaces
      hostPID: true
      hostNetwork: true
      hostIPC: true
      containers:
      - name: shell
        image: alpine
        securityContext:
          privileged: true
        # This command keeps the pod running so you can exec into it
        command: ["/bin/sh", "-c", "sleep infinity"]
        volumeMounts:
        - name: host-root
          mountPath: /host
      volumes:
      - name: host-root
        hostPath:
          path: /
```

### Step 2: Apply the DaemonSet
Run this command to deploy the pods to every node in your cluster:
```bash
kubectl apply -f node-access.yaml
```

### Step 3: Run a command on a specific node
Once the pods are running, find the pod name for the node you want to target:
```bash
kubectl get pods -n kube-system -l app=node-shell -o wide
```

Now, use `kubectl exec` combined with `nsenter`. The `nsenter` command switches the context to the host's PID 1 (init process), effectively giving you a host shell.

**To get an interactive shell on the node:**
```bash
kubectl exec -it <POD_NAME> -n kube-system -- nsenter -t 1 -m -u -n -i sh
```

**To run a single command (e.g., check `docker ps` or `crictl ps` on the host):**
```bash
kubectl exec -it <POD_NAME> -n kube-system -- nsenter -t 1 -m -u -n -i -- ls /etc/kubernetes
```

### How it works:
*   **`hostPID: true`**: Allows the container to see all processes on the host.
*   **`privileged: true`**: Gives the container the permissions needed to escape its own namespace.
*   **`nsenter -t 1`**: Targets Process ID 1 (the host's init system).
*   **`-m -u -n -i`**: These flags tell `nsenter` to enter the host's Mount, UTS (hostname), Network, and IPC namespaces.

### Cleanup
When you are finished, delete the DaemonSet for security reasons:
```bash
kubectl delete -f node-access.yaml
```

### Important Security Note
This method is extremely powerful and effectively bypasses all container security boundaries. 
1.  **RBAC**: You must have permissions to create DaemonSets and use privileged security contexts.
2.  **Pod Security Admission**: If your cluster has Pod Security Standards enabled (e.g., `restricted` or `baseline`), this pod will be blocked unless the namespace is set to `privileged`.
3.  **Auditing**: Most enterprise security tools will flag this activity immediately as it is a common technique for lateral movement or container escape.
