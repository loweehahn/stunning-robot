# 04-storage

Hands-on notes on Kubernetes storage, tested on a single-node K3s cluster (default StorageClass `local-path`).

## Files

| File | What it does |
|---|---|
| `emptydir-pod.yaml` | Pod `emptydir-test` (nginx) with an `emptyDir` volume `html` mounted at `/usr/share/nginx/html`. |
| `pvc.yaml` | PVC `html-pvc` (`ReadWriteOnce`, `100Mi`) with no `storageClassName`, so it uses the default class. |
| `pvc-pod.yaml` | Pod `pvc-test` (nginx) whose `html` volume comes from the PVC `html-pvc`. |
| `storageclass-retain.yaml` | StorageClass `local-path-retain`: same provisioner as `local-path`, but `reclaimPolicy: Retain`. |
| `retain-pvc.yaml` | PVC `retain-pvc` (`ReadWriteOnce`, `100Mi`) that uses the `local-path-retain` class. |
| `retain-pod.yaml` | Pod `retain-test` (nginx) whose `html` volume comes from the PVC `retain-pvc`. |
| `retain-pvc-restore.yaml` | PVC `retain-pvc` that binds to an existing PV by name (`volumeName`), used to recover retained data. |

## Findings proved on the cluster

### 1. `emptyDir` survives a container restart, but not a Pod delete

- A file written to the container's own filesystem was lost when the container was replaced or the Pod was deleted.
- A file on the `emptyDir` survived `kill 1` (same Pod UID, new container ID, `RESTARTS` +1).
- The same file was lost after deleting and re-applying the Pod (new Pod UID), because the volume lives under `/var/lib/kubelet/pods/<pod-uid>/` and is removed with the Pod.

### 2. On a `WaitForFirstConsumer` class, a PVC stays `Pending` until a Pod uses it

- The `local-path` class has `volumeBindingMode: WaitForFirstConsumer`, so the PVC showed `Pending` and `kubectl get pv` was empty.
- Once a Pod using the claim was scheduled, a PV was provisioned (`pvc-<PVC UID>`) with node affinity to the node and the PVC became `Bound`.
- Data on the PVC survived a Pod delete and re-apply. The directory on the node is named from the PVC UID, not the Pod UID.
- This comes from the class's `volumeBindingMode`, not from every StorageClass: with `Immediate` (the default when unset) a PVC binds as soon as it is created. A claim that names its PV with `volumeName` (`retain-pvc-restore.yaml`) also reached `Bound` before any Pod existed.

### 3. `Retain` and `Delete` differ in what is left after the PVC is deleted

- With `Delete` (the `local-path` default), deleting the PVC removed the PV and the data directory.
- With `Retain` (`local-path-retain`), the PV became `Released` and the data stayed on disk.
- While a Pod still used the PVC, the `pvc-protection` finalizer kept it in `Terminating`. Delete the Pod first.

### 4. A `Released` PV is not reused by a new claim of the same name

- Re-creating `retain-pvc` gave a brand new, empty PV. The old PV's `claimRef.uid` still held the deleted PVC's UID, so the new claim (same name, different UID) did not match.
- The data was recovered by clearing only `claimRef.uid` on the old PV and binding a claim to it with `volumeName`. The ordered steps are under "Recovering a retained PV" below.
- Deleting a `Retain` PV object does not delete its directory. That has to be removed by hand.

## Recovering a retained PV

Use this when `retain-pvc` was deleted, the PV is `Released` and you want its data back. Run the steps in this order.

1. **Find the PV to recover.** Look for `STATUS` `Released` and `CLAIM` `default/retain-pvc`, and note its name (`pvc-<uid>`).

   ```bash
   kubectl get pv
   ```

   If more than one PV is `Released` with that claim, do not pick one by name. Before deleting anything, list the directories on the node. Each is named `<pv-name>_default_retain-pvc`, and the one holding your files is the PV to recover. A directory with nothing listed under its name is empty, so it is not the one to recover.

   ```bash
   sudo sh -c 'ls -l /var/lib/rancher/k3s/storage/*/'
   ```

2. **Remove any Pod and claim that exist under the same name.** Delete the Pod first, otherwise the PVC stays in `Terminating` because of the `pvc-protection` finalizer. If a new `retain-pvc` was created in the meantime, it will have provisioned a new, empty PV. That PV also becomes `Released` when its claim is deleted (clean it up at step 7).

   ```bash
   kubectl delete pod retain-test --wait=true --ignore-not-found
   kubectl delete pvc retain-pvc --wait=true --ignore-not-found
   ```

3. **Clear only the claim UID on the old PV.** It becomes `Available` and stays reserved for `default/retain-pvc`. Replace `<pv-name>` with the name from step 1, then check the phase.

   ```bash
   kubectl patch pv <pv-name> -p '{"spec":{"claimRef":{"uid":null}}}'
   kubectl get pv <pv-name> -o jsonpath='{.status.phase}{"\n"}'
   ```

   The phase should now be `Available`.

4. **Point the restore PVC at that PV, then apply it straight away.** Edit `volumeName` in `retain-pvc-restore.yaml` to the PV name from step 1 (the file ships with the name from the test cluster), then apply it.

   ```bash
   kubectl apply -f retain-pvc-restore.yaml
   ```

   Straight after `apply` the claim showed `Pending` with `CAPACITY` 0, and about 78 seconds later it was `Bound`, before any Pod existed. No new PV should appear.

5. **Start the Pod.**

   ```bash
   kubectl apply -f retain-pod.yaml
   kubectl wait --for=condition=Ready pod/retain-test --timeout=60s
   ```

6. **Check the data is back.**

   ```bash
   kubectl get pvc,pv
   kubectl exec retain-test -- ls -la /usr/share/nginx/html
   ```

   `retain-pvc` should be `Bound` to the PV from step 1 and your files should be listed.

7. **Clean up the leftovers (only after step 6 works).** If step 2 left an empty `Released` PV, delete it. Deleting a `Retain` PV does not delete its directory, so remove that directory on the node by hand, naming it in full (the directory names differ only in the UID).

   ```bash
   kubectl delete pv <empty-pv-name>
   sudo rmdir /var/lib/rancher/k3s/storage/<empty-pv-name>_default_retain-pvc
   ```

   `rmdir` only removes an empty directory, so it will refuse to touch the one holding your data.

## Notes

- In `emptydir-pod.yaml`, the empty `emptyDir` mounted over `/usr/share/nginx/html` leaves that folder with no `index.html`, so `/` returned 403 and `/index.html` returned 404.
- Apply order for the Retain set: `storageclass-retain.yaml`, `retain-pvc.yaml`, then `retain-pod.yaml`.
- Run the recovery steps from the `04-storage` folder, or give the full path to the YAML files.
- Clearing only `claimRef.uid` (not the whole `claimRef`) keeps the PV reserved for one claim name, so an unrelated claim cannot take it before step 4.
