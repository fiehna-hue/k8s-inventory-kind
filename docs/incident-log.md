# Incident log

Problems I caused on purpose to practise diagnosing Kubernetes failures. All dated 2026-10-23.

## 1. Wrong image tag stalled a rollout

- **Symptom:** after changing the image, the new pod sat in `ErrImagePull` and then `ImagePullBackOff`, while the two old pods kept serving.
- **Diagnosis:** `kubectl get pods` showed the state, and `kubectl describe pod` showed `Failed to pull image` in the Events.
- **Cause:** the tag did not exist in the cluster, and kind cannot pull local images from anywhere else.
- **Fix:** `kubectl rollout undo deployment/api`.
- **Lesson:** `maxUnavailable: 0` meant a bad image never reduced capacity. The old pods stayed up until a new one was ready.

## 2. Failing readiness probe kept a pod out of service

- **Symptom:** the new pod was `Running` but stayed `0/1`, and the rollout never finished.
- **Diagnosis:** `kubectl describe pod` showed `Readiness probe failed: HTTP probe failed with statuscode: 404`.
- **Cause:** the probe path was changed to one the app does not serve.
- **Fix:** `kubectl rollout undo deployment/api`.
- **Lesson:** a container that is running is not the same as a container that is ready. Readiness decides whether it receives traffic.

## 3. Wrong database password

- **Symptom:** the new pod stayed `0/1`, looking the same as incident 2.
- **Diagnosis:** the API pod pointed nowhere useful. The Postgres pod's logs showed `FATAL: password authentication failed for user "inventory_user"`.
- **Cause:** the pod was given a different password from the one the database was created with.
- **Fix:** `kubectl rollout undo deployment/api`.
- **Lesson:** the same symptom can have different causes, so go to the component that rejected the request. Secrets are read when a pod starts, so changing one needs a rollout.
