# istio-bug

# overview

This repo basically replicates an issue recently during upgrade from 1.10.4 to 1.11.2.

It includes a `./run` script that spins up a [kind](https://kind.sigs.k8s.io/) cluster, deploys [istio](https://istio.io/) and then proceeds to deploy the following:

- an [nginx](junk/example/deployment.yaml) workload as our target in `example` namespace.
- two [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) + [virtualservice](https://istio.io/latest/docs/reference/config/networking/virtual-service/) pairs - one in `alpha` namespace, the other in `beta`. These are identical other than where they live.
- a [job](junk/debug/job.yaml) that uses a `curl` image to conduct the test.

# the test

The test is simple:

- `curl` the example workload using its service address.
- `curl` the workload using the virtualservice in `alpha` namespace.
- `curl` the workload using the virtualservice in `beta` namespace.

We expect all of the above to work and return a `200` response.
In addition to this, before we conduct the tests we take a [config dump](https://www.envoyproxy.io/docs/envoy/latest/operations/admin#get--config_dump?resource=) from the pod to see how its routing looks.

# results

This `./run` script allows the test to be run with any version of istio.

Using `1.10.4` control plane:

```
http://example.example:8080=200
http://api.alpha:8080=200
http://api.beta:8080=200
```

Using `1.11.2` control plane:

```
http://example.example:8080=200
http://api.alpha:8080=200
http://api.beta:8080=503
```

# config dump

And here lies the issue, `api.beta` is still pointing to the headless service _not_ the `example` service!


```
{
 "name": "api.alpha.svc.cluster.local:8080",
 "domains": [
  "api.alpha.svc.cluster.local",
 ],
 "routes": [
  {
   "route": {
    "cluster": "outbound|8080||example.example.svc.cluster.local",


{
 "name": "api.beta.svc.cluster.local:8080",
 "domains": [
  "api.beta.svc.cluster.local",
 ],
 "routes": [
  {
   "route": {
    "cluster": "outbound|8080||api.beta.svc.cluster.local",
```
