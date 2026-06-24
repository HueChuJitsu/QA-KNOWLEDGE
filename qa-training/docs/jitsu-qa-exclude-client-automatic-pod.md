# How To: Exclude a Client from Automatic POD Analysis

> **Env:** Staging · **Layer:** Backend worker (`s3-webhook`) · **Config store:** Consul
> Add the test client to this list so the captured POD is **not** sent for automatic analysis.
>
> Related: [Delivery & POD](./jitsu-qa-training-document.md) · driver-app POD capture behaviour in [../../apps/driver-app/functions/pod.md](../../apps/driver-app/functions/pod.md)

---

## 1. What it does

`automatic_pod_excluded_client_ids` is a list of **client IDs that are excluded from automatic POD analysis**.

When a client's ID is in this list, a POD that the driver captures on a stop is **not analyzed** after capture — the automatic POD analysis step is skipped for that client's shipments.

This is used when testing POD: add your test client so automatic POD analysis does not run against test deliveries.

## 2. Where it lives (Consul)

| | |
|---|---|
| **Key** | `automatic_pod_excluded_client_ids` |
| **Worker** | `s3-webhook` |
| **Environment** | Staging |
| **Path** | `staging/apps/workers/s3-webhook/attributes/automatic_pod_excluded_client_ids` |

Edit URL:

```
https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/workers/s3-webhook/attributes/automatic_pod_excluded_client_ids/edit
```

## 3. How to use it

1. Get the **client ID** of the test client whose shipments you will deliver.
2. Open the Consul key at the path above.
3. **Add** the test client ID to the `automatic_pod_excluded_client_ids` list and save.
4. Run the delivery / capture the POD on the stop → confirm the POD is **not analyzed** after capture.
5. **Clean up:** once testing is done, **remove** the test client ID from the list so it does not accumulate stale test data.

## 4. Notes

- This config is read by the `s3-webhook` worker, so it affects the backend POD-analysis step — it is separate from the on-device driver-app POD capture/validation described in [../../apps/driver-app/functions/pod.md](../../apps/driver-app/functions/pod.md).
