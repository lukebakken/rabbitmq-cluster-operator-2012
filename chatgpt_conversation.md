# Full Conversation Transcript (GitHub-Flavored Markdown)

## User
You are an expert in unix signal handling, kubernetes, and docker containers run in that environment. Describe what happens, in detail, when "kubectl delete pod --grace-period 1" is run on a pod.

## Assistant
When you run:

```bash
kubectl delete pod <name> --grace-period=1
```

there are several layers involved: `kubectl` → apiserver → kube-controller-manager / kubelet → container runtime → the Unix process tree inside the container. I will walk through the sequence end-to-end.

Assume:
- Pod is running on some node.
- It is a normal pod managed by the kubelet (not a static pod, not already terminating).
- The container entrypoint is PID 1 inside its PID namespace.

---

## 1. What `kubectl` actually does

`kubectl` does **not** itself send any Unix signals. It makes an HTTP call to the Kubernetes API server:

- It issues a `DELETE` on the Pod resource:
  - Resource: `DELETE /api/v1/namespaces/<ns>/pods/<name>`
  - Body includes:
    - `gracePeriodSeconds: 1`
    - Optionally `propagationPolicy` / `orphanDependents`, etc., but not relevant to signals.

The API server:

1. Validates that the pod exists and is deletable.
2. Sets the pod’s `deletionTimestamp` and `deletionGracePeriodSeconds` to `1`.
3. Returns success to `kubectl`.
4. Does **not** immediately remove the Pod object; it marks it as `terminating`.

From your point of view in `kubectl get pods`, you will see `STATUS` change to `Terminating`.

---

## 2. Controller and kubelet reactions to a terminating pod

Once the pod is marked for deletion:

### 2.1. Kube-controller-manager

For bare pods (not owned by a ReplicaSet/Deployment):

- It mostly just observes the deletion, possibly handling any finalizers.
- If it is a replicated workload (Deployment, StatefulSet, etc.), the higher-level controller may create a replacement pod once it sees this one is going away, depending on the controller’s semantics.

This level does **not** send signals to containers.

### 2.2. Kubelet on the node

The kubelet on the node where the pod is running:

1. Watches the API server and sees:
   - `deletionTimestamp` is set.
   - `deletionGracePeriodSeconds = 1`.

2. Transitions the pod into “terminating” state locally:
   - Stops accepting new readiness/liveness probes as indicative of “running”.
   - Stops attaching the pod to new Services/Endpoints (via the endpoints controller).
   - It starts the **pod termination sequence**:
     - Run preStop hooks.
     - Send SIGTERM.
     - Wait for grace period.
     - Forcefully kill (SIGKILL) if still alive at the end.

---

## 3. PreStop lifecycle hook (if configured)

If the pod spec defines container `lifecycle.preStop` hooks, these run **before** SIGTERM:

```yaml
containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "do-something"]
```

The kubelet:

1. Executes the hook in the container’s context (via the container runtime).
2. The hook runs as a process in the container’s PID namespace (not necessarily as PID 1).
3. Importantly, the **preStop hook eats into the grace period**. With `--grace-period=1`, the entire pod termination, including preStop, SIGTERM, and waiting for exit, must fit within ~1 second.

If preStop takes longer than the grace period, the kubelet will move on to forcible termination.

---

## 4. SIGTERM to the container process

After (or in parallel with, depending on implementation/version) the preStop hook, kubelet tells the container runtime (containerd / dockershim / CRI-O, etc.) to stop the container.

Conceptually:

- `kubelet` → runtime: “Stop this container with gracePeriodSeconds=1”.
- The runtime sends a **SIGTERM** to the container’s PID 1.
  - In Docker terms: `docker stop --time=1 <container>` would do:
    - `kill -TERM <pid1>`
    - Wait up to 1 second
    - If still running: `kill -KILL <pid1>`

So, inside the container:

- PID 1 (your entrypoint) receives SIGTERM.
- Any signal handlers you have for SIGTERM run.
- If PID 1 terminates, the container is considered stopped.
- If PID 1 stays alive beyond the grace period, the runtime will eventually send SIGKILL.

> Note: In most setups, only PID 1 gets the signal. If PID 1 does not propagate SIGTERM down to child processes, you can end up with orphaned processes until the namespace is torn down.

---

## 5. What the grace period of 1 second actually means

The pod spec’s `.spec.terminationGracePeriodSeconds` defaults to 30. Using `--grace-period=1` on `kubectl delete` temporarily overrides it for this deletion.

The semantics:

- Kubelet now has **approximately 1 second** from when it begins termination to when it must force-kill the pod.
- That 1 second covers:
  - PreStop hook execution time
  - SIGTERM handling and process shutdown time
- After the 1 second is up:
  - Kubelet requests the runtime to forcibly stop the containers.
  - Runtime sends SIGKILL to PID 1.
  - Remaining processes in the namespace are reaped by the kernel as the PID namespace goes away.

In practice, timing is approximate (network latency, scheduling, etc.), but the intent is “very little time to shut down gracefully; basically kill it now with only a token grace period.”

---

## 6. SIGKILL and container teardown

If PID 1 is still running after the grace period:

1. Kubelet calls StopContainer again or in “force” mode.
2. Runtime sends **SIGKILL** to PID 1. SIGKILL:
   - Is not catchable.
   - Terminates process immediately.
3. When PID 1 dies:
   - Any remaining processes in that PID namespace are either:
     - Killed when the namespace is torn down.
     - Orphaned and then reparented to the namespace’s PID 1 or host PID 1 (implementation details depend on how the runtime sets up namespaces).
   - Linux cleans up the container’s PID namespace, mount namespace, etc.
4. Runtime marks container as `exited`.
5. Kubelet updates pod status: containers terminated with reason “Killed” or similar.

---

## 7. Pod object deletion from the API

During termination:

- The Pod object remains in the API with a `deletionTimestamp` and status `Terminating`.
- Once kubelet has successfully stopped all containers and reports that the pod is “no longer running”:
  - The API server completes the deletion.
  - The Pod object disappears from `kubectl get pods` (unless you use `--show-all` with old client behavior / watch events, etc.).

If there are any **finalizers** on the Pod (rare for standard pods), the object will not be fully removed until those finalizers complete. That does not affect the Unix signaling; it only affects how long the Pod object lingers in the API.

---

## 8. Signal handling inside the container

From the perspective of Unix signal handling in the container:

1. Your process is PID 1, so it has special semantics:
   - If you do not explicitly handle SIGTERM, default action is to terminate.
   - If you ignore SIGTERM (e.g., trap it and do nothing), then:
     - The process will not exit until SIGKILL arrives.
     - With a 1-second grace period, you will almost always be killed ungracefully.
2. If you have a proper SIGTERM handler:
   - It should start shutdown immediately.
   - With gracePeriod=1, you need to be extremely fast or you will still be killed by SIGKILL.
3. For multi-process containers:
   - If PID 1 is a small “init” like `tini` or a custom init, it should:
     - Forward SIGTERM to children.
     - Wait for them to exit.
   - Again, with only 1 second, you may not complete all cleanup.

---

## 9. Side effects in Kubernetes ecosystem

A few other important details:

- **Endpoints / Services / Load balancers**
  - When the pod enters `Terminating`, controllers stop listing it as a ready endpoint once the readiness condition is false. Often this happens quickly once termination begins.
  - This minimizes new traffic, but in-flight connections may still exist until the process dies.

- **Probes**
  - Liveness/readiness probes typically stop being executed once the pod is terminating. Those results no longer matter for routing.

- **Logs**
  - Container logs for the terminated pod remain in the node filesystem until log rotation / GC. The Pod being deleted does not immediately delete log files.

- **RestartPolicy**
  - For standard pods with `restartPolicy: Always` (via a ReplicaSet/Deployment), once this pod is terminated:
    - The workload controller sees fewer running replicas.
    - It will schedule a replacement pod (depending on minReadySeconds, etc.).

---

## 10. Summary flow

Putting this all together in a tight sequence:

1. You run:
   ```bash
   kubectl delete pod <name> --grace-period=1
   ```
2. API server:
   - Marks Pod as terminating with `deletionTimestamp` and `deletionGracePeriodSeconds=1`.
3. Kubelet on the node:
   - Sees the pod is terminating.
   - If preStop hook exists, starts running it inside each container.
   - Sends SIGTERM to PID 1 of each container via the runtime.
   - Waits up to ~1 second for containers to exit.
4. Inside the container:
   - PID 1 receives SIGTERM.
   - If it handles SIGTERM and exits fast, container stops cleanly.
   - If not, it is still running when the grace period expires.
5. When 1 second expires:
   - Kubelet asks runtime to forcibly kill container.
   - Runtime sends SIGKILL to PID 1.
   - Container stops abruptly; remaining processes are cleaned up by the kernel.
6. Kubelet updates pod status to “terminated” and informs the API server.
7. API server:
   - Once there are no running containers and no finalizers, deletes the Pod object.
8. From the user’s view:
   - Pod goes from `Running` → `Terminating` → disappears.
   - The actual processes were given at most ~1 second between SIGTERM and SIGKILL.

---

## User
OK, great, this is very informative. A RabbitMQ user is running a cluster in k8s and started this discussion:

----

During an incident where we wanted to force a restart of a RabbitMQ cluster managed by the cluster-operator, we had a hard time forcing a restart of the pods. SIGKILL was getting ignored, and it turns out that's because rabbitmq-server is started as PID 1 in the container, which is allowed to ignore SIGKILL. We resorted to killing the subprocesses of the pod, which pushed rabbitmq-server to crash.

Is there a delibarate rationale to having rabbitmq-server as PID 1, and be able to ignore being killed ?

----

Now, do not assume that his conclusion is correct, only that his description of that specific issue is accurate, and he had a scenario where RabbitMQ pods were not stopping quickly. He also tried "kubectl delete pod --grace-period 1" and the pod remained stuck.

Can you provide an explanation for this behavior?

## Assistant
Here is the most accurate and technically grounded explanation for the behavior the user observed, **without presuming that RabbitMQ or its entrypoint actually ignores SIGKILL** (it cannot), and without assuming user error. What *can* happen in Kubernetes can *look* exactly like “PID 1 ignored SIGKILL,” even though that is impossible at the kernel level.

Below is the real set of mechanisms that can produce this situation.

---

# Why a RabbitMQ pod can appear to “ignore SIGKILL” and fail to terminate even with `--grace-period=1`

You are dealing with a combination of:

1. Kubernetes pod termination edge cases  
2. Container runtime kill-semantics  
3. Filesystem/volume shutdown delays  
4. Erlang/beam.smp supervision tree behavior  
5. `rabbitmq-server` being the PID 1 process *(but PID 1 still cannot ignore SIGKILL)*  

The **key point**:  
**The pod can remain in “Terminating” even if the process was killed**, and this can be misinterpreted as “RabbitMQ ignored SIGKILL,” when the *node*, *runtime*, or *Kubernetes teardown sequence* was what actually stalled.

I will explain the concrete causes that produce this exact symptom in RabbitMQ clusters.

---

# 1. PID 1 cannot ignore SIGKILL — but Kubernetes semantics can make it *look* like it did

In Linux:

- A process can catch or ignore SIGTERM.  
- A process **cannot catch, block, or ignore SIGKILL**.  
- PID 1 has special behavior for some signals, but **SIGKILL is never ignorable**.

Therefore, if a RabbitMQ container is receiving SIGKILL from the runtime, its Erlang VM **is dead immediately**.

But Kubernetes deletion can hang **after** the process is gone.

This is the root of the misunderstanding.

---

# 2. What actually happens: kubelet cannot remove the container or pod sandbox

The common “pod will not die even after SIGKILL” failure mode is:

- The runtime killed the process correctly  
- But kubelet is stuck waiting for the container runtime to report the container as “stopped”  
- Or kubelet is stuck tearing down the **pod sandbox**, **network namespace**, or **mounted volumes**

When that happens:

- The container looks like it is still running  
- `kubectl delete pod --grace-period=1` does nothing  
- Users conclude “the process survived SIGKILL,” but the process is actually already dead  
- The pod remains in `Terminating` forever

This is extremely common on StatefulSets using:

- persistent volume mounts (RabbitMQ always does)
- network plugins that leak CNI namespaces
- volume drivers (EBS, Ceph, NFS) that block unmount  
- Sidecar containers that hung  
- Cgroup or overlayfs slowness under heavy IO load

---

# 3. RabbitMQ in particular triggers two conditions that make kubelet teardown fragile

## A. Persistent volumes actively in use  
RabbitMQ does heavy disk IO:

- Mnesia database  
- Message store writes  
- Quorum queue Raft logs  
- File descriptors held open for WAL segments

When the container dies abruptly:

- Kubelet must unmount the PV  
- If the node’s storage driver refuses unmount (because the FDs are not released fast enough, or kernel buffers are dirty), kubelet keeps retrying  
- Pod cannot be fully removed → pod stays Terminating

This is *especially common* if:

- The node is under IO pressure  
- The PV driver is remote (EBS, NFS, Ceph)  
- There was an incident (their report says exactly that)

---

## B. RabbitMQ leaves multiple processes behind BEAM's PID 1

RabbitMQ (via `rabbitmq-server`) ultimately launches:

- `beam.smp`  
- Distribution port listener (`epmd` unless disabled)  
- OS file handle helpers (`beam.*`)  
- Plugins may spawn epmd-like helpers  
- Quorum queues each spawn Raft worker processes inside BEAM  
- Prometheus exporters or TLS renegotiation threads may outlive main BEAM shutdown

If PID 1 dies via SIGKILL:

- Child processes should die with the PID namespace teardown  
- **But if the runtime never tears down the namespace**, orphan processes appear alive from the outside  
- kubelet sees non-empty process tree → still considers container alive

This leads to the exact symptom:

**Pod cannot disappear because container namespace teardown is incomplete.**

---

# 4. Why it happened specifically during an *incident*

During cluster incidents, RabbitMQ nodes often:

- Have large Raft logs to replay  
- Have large, dirty IO buffers  
- Are running recovery code paths  
- Have heavy CPU and disk pressure  

This causes:

- **Slow process exit**  
- **Slow OS release of filesystems**  
- **Slow or failing volume unmounts**  
- **Runtime failure to stop container cleanly**

Under those conditions, kubelet may not deliver SIGKILL until:

- CNI cleanup completes  
- Volume unmount completes  
- Sandbox teardown completes

If any of these stall, you get the “hung terminating RabbitMQ pod” syndrome.

This is widely known in the Kubernetes world:  
**StatefulSet pods with PVs can become unkillable under heavy IO.**

---

# 5. Why it *looks like* RabbitMQ is ignoring SIGKILL

To the user:

- Pod is still Running or Terminating  
- `containerd` or `docker` still reports container PID alive  
- `kubectl delete --force --grace-period=1` hangs  
- They kill sub-processes manually → pod finally dies

Thus users conclude:  
“PID 1 ignored SIGKILL.”

But what actually happened is usually:

1. Process already died  
2. Namespace teardown was blocked  
3. Runtime did not notify kubelet  
4. Pod object was never removed  
5. Kubelet kept retrying unmounts or CNI cleanup  
6. Manual killing of child processes freed resources → cleanup succeeded → container considered dead

RabbitMQ was not ignoring SIGKILL.  
The **node** was unable to tear down the container sandbox.

---

# 6. Is there a rationale for RabbitMQ being PID 1?

Yes, and none of it relates to signal behavior:

## A. RabbitMQ upstream ships `rabbitmq-server` as entrypoint
All official RabbitMQ container images use:

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["rabbitmq-server"]
```

`rabbitmq-server` ends up as PID 1.

This is standard for applications that should receive SIGTERM directly.

## B. The Kubernetes cluster-operator does not wrap it in a “tini” init
Most application containers run PID 1 directly.  
Only special setups use `tini` or dumb-init to absorb zombie processes.

RabbitMQ images intentionally do not add an init because:

- Erlang VM already reaps children  
- RabbitMQ wants to directly receive SIGTERM for graceful shutdown  
- Adding `tini` increases complexity and does not solve this scenario

## C. RabbitMQ does not depend on PID-1 semantics
It is not “designed to ignore kill.”  
The design is simply “PID 1 should be RabbitMQ so SIGTERM leads to graceful cluster leave.”

This has the side effect that the BEAM VM must shut down completely before the namespace can go away, which under heavy IO can take longer than Kubernetes expects.

---

# 7. Final summary

The user’s observation is real, but the conclusion (“RabbitMQ ignored SIGKILL”) is not.

**The actual explanation is:**

1. RabbitMQ was running under heavy IO or recovery load.  
2. Kubelet tried to terminate the pod.  
3. SIGTERM → slow or stuck exit.  
4. SIGKILL was sent — process either died or runtime could not deliver the kill due to node pressure.  
5. Containerd/docker failed to cleanly tear down the container:
   - PV unmount blocked  
   - CNI namespace cleanup blocked  
   - Overlayfs or cgroups stuck  
6. Because teardown didn’t complete, kubelet never removed the pod.  
7. Pod looked alive → user thought SIGKILL was ignored.  
8. Killing remaining processes freed resources and allowed teardown.

RabbitMQ was not resisting SIGKILL; the infrastructure around it was stuck.

---

## User
OK, here is one caveat about how RabbitMQ is run inside of a container. The entrypoint is a shell script that starts the Erlang runtime. The shell script continues running as PID 1. It *DOES* trap signals and forward them to the Erlang's beam.smp process. Does this change anything in this scenario?

## Assistant
Short answer: **Yes, this changes the *surface appearance* of what’s happening, but it does *not* change the fundamental explanation**—because the shell script (PID 1) still *cannot* ignore SIGKILL and cannot prevent the container from being torn down. What it *does* change is how SIGTERM is handled and what the kubelet sees as “still running,” which directly contributes to the very symptoms the user described.

Below is the precise impact.

---

# 1. RabbitMQ’s PID 1 is **not** `beam.smp` — it is a shell wrapper

The RabbitMQ container’s `docker-entrypoint.sh` performs:

- environment setup  
- plugin enablement  
- config file generation  
- and then:

```bash
exec rabbitmq-server
```

OR (for older images):

```bash
rabbitmq-server &
child=$!
trap 'kill ...' TERM
wait "$child"
```

Some older or customized scripts **do not exec**, meaning:

- PID 1 = shell  
- `beam.smp` is a child  
- PID 1 has a trap for SIGTERM  
- PID 1 forwards SIGTERM to the Erlang VM

This is important.

---

# 2. What traps “change” vs. what they *cannot* change

They **change**:

- How fast SIGTERM is forwarded  
- Whether PID 1 exits when `beam.smp` exits  
- Whether PID 1 can remain alive longer than the Erlang VM  
- Whether child processes remain reparented to PID 1  

They **cannot** change:

- PID 1 cannot ignore SIGKILL  
- The shell cannot protect the Erlang VM from SIGKILL if the runtime delivers it  
- The container runtime will still kill PID 1 first  
- The Linux kernel, not the shell, determines when namespaces are torn down  

Thus: the presence of a trap only affects the **SIGTERM phase**, not the **SIGKILL phase**.

---

# 3. Why this *increases* the likelihood of the “hung terminating pod” symptom

Here is the crucial point:

## When PID 1 is a shell script, Kubernetes considers the container “running” **as long as the shell is alive**, even if `beam.smp` has already died.

This **directly increases the odds** of the exact scenario the user encountered.

Let’s walk the sequence.

---

# 4. Termination sequence with a shell wrapper PID 1

## Step A — kubelet sends SIGTERM  
Kubelet → runtime → PID 1 (shell)

- Shell traps SIGTERM  
- Shell forwards SIGTERM to beam.smp  
- Erlang shutdown begins (can be lengthy under load)  

**At this stage, everything is working as expected.**

## Step B — grace period (e.g. 1 second) expires  
Kubelet → runtime → SIGKILL to PID 1

Two subtleties arise:

### 1. The shell script can outlive the Erlang VM

If `beam.smp` dies **first**, but the shell wrapper never exits (because it’s stuck in `wait`, or because the trap handler has not finished), the container appears still alive:

- PID 1 = shell is still running  
- Kubelet still sees “container running”  
- PV unmount is delayed  
- Sandbox teardown does not happen  

This alone creates the illusion that SIGKILL is being ignored.

### 2. If runtime cannot kill PID 1 *immediately*, the kill is deferred

Under IO pressure or CNI teardown slowness, the runtime may fail to immediately deliver the kill signal:

- containerd waits for CRI-level stop  
- network teardown waits  
- filesystem teardown blocks  
- PID namespace teardown is paused  

This yields a **period where the container is “Terminating” and PID 1 still shows up in process listings**, even though an eventual SIGKILL is queued.

To a user:  
“RabbitMQ ignored SIGKILL.”

To Kubernetes:  
“Runtime is stuck trying to terminate container.”  
(not RabbitMQ’s fault)

---

# 5. The important RabbitMQ-specific factor: BEAM may exit while PID 1 does NOT

If the script is of this form (older pattern):

```bash
rabbitmq-server start &
child=$!

trap "kill -TERM $child" TERM
wait "$child"
```

Then possible outcomes during heavy load:

- `beam.smp` receives SIGTERM → begins shutdown  
- `beam.smp` blocks on file sync, mnesia close, queue persistence, etc.  
- After grace period, PID 1 gets SIGKILL → **but if runtime is stuck, kill is delayed**  
- Meanwhile, `beam.smp` finally exits  
- PID 1’s `wait` returns  
- Trap logic may run  
- PID 1 may hang (waiting on zombie reaping, or stuck in trap)  

Now you have:

- RabbitMQ actually dead  
- PID 1 still alive  
- Container cannot be torn down  
- Pod remains Terminating  

This is a **perfect match** for the user symptoms.

---

# 6. What does PID 1 trapping SIGTERM *actually* cause?

## A. Increases likelihood PID 1 continues to run after its child exits  

Trap code might run *after* the child is gone, keeping PID 1 alive.

## B. Makes container state mismatch BEAM state  

Kubelet/containerd consider the container alive as long as PID 1 exists.

## C. Exacerbates teardown delays  

If PID 1 holds any file descriptors or signal handlers active, the namespace tear-down is blocked longer.

## D. User misdiagnosis  

“RabbitMQ ignored SIGKILL.”

When the real cause is:

- The **shell PID 1** did not exit promptly  
- The **container runtime** did not tear down the sandbox  
- Or **volume/CNI teardown** stalled  

---

# 7. Final answer: **Yes, the shell-PID1 design contributes significantly to the observed symptoms**, even though RabbitMQ still cannot ignore SIGKILL.

Here is the precise, correct causal chain:

1. PID 1 in RabbitMQ containers is a shell script.  
2. Shell traps SIGTERM and forwards it to BEAM.  
3. Under heavy RabbitMQ load, BEAM may take time to exit.  
4. Kubelet expires the grace period and requests SIGKILL.  
5. SIGKILL is for PID 1 (the shell), not `beam.smp`.  
6. If the container runtime or node is under stress, PID 1 may not be killed immediately.  
7. Meanwhile, `beam.smp` may already be dead while PID 1 is still stuck in `wait` or in trap logic.  
8. Container runtime still sees PID 1 alive → container is “running”.  
9. Kubelet cannot remove sandbox, PV mounts, or CNI namespaces.  
10. The pod appears unkillable and stuck Terminating.  
11. User concludes: “RabbitMQ (PID 1) ignored SIGKILL.”  
    In reality: container runtime never completed teardown.

This **exactly** matches the user’s observed behavior.
