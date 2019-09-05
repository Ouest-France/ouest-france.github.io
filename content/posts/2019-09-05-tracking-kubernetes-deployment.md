---
title: "[EN] Tracking Kubernetes deployment"
date: 2019-09-05T21:43:07+02:00
draft: false
emoji: true
---

![Monitoring deployment](/images/posts/2019-09-05-tracking-kubernetes-deployment/intro.png)


Eventually, here at Ouest France, we achieved migrating our delivery pipeline and we are now able to deliver on demand.

Even if it not yet a real continuous delivery workflow (still a few human validation mostly for sanity checks), we drastically increased the number of time we deliver our CMS. We used to deliver once a week, and we are now able to deliver a few time per day! :tada:

# Monitoring is the key of relialability

At Ouest France, we are conviced than the relialability of our website is based on the code quality, infrastructure reslilance and on proactive detection of potential issues.

This is why we always have a screen displaying the status of the main metrics / dependencies health, ... This screen is visible by anyone in the open space. This helps us to detect strange behavior as soon as possible (be sure we also have dozen of automatic alerting in case we were all having a break...).

This leads to many questions by anyone seeing an indicator becoming orange or red :sweat_smile:.

We worked hard to make the delivery of a new version the smoothest possible. But despite all our efforts, when a new pod of our application joins the running pods, the first requests it handles are a bit more slow than a few seconds after (cache loading, JIT, template compilation, ...).

<center>
![TTFB when delivering](/images/posts/2019-09-05-tracking-kubernetes-deployment/on-deliver.png)
</center>

# Give all the keys to understand

We are conviced that a dashboard must display enough information to detect issues, but it **must** also display enough information to be able to explain why an indicator is becoming red.

As we are delivering multiple time per day, those questions about the TTFB degradation that was happening after each delivery, were becoming more and more disturbing for the team because they were "well knwonw behavior" and easy to explain. 

This is why we searched for an indicator to add on the dashboard to display that we were currently running a rolling upgrade and that it could explain the performance degradation.

# In action

When we deploy a new version, we use the `helm upgrade` command that basically request kubernetes to update the deployment descriptor.

Hopefully, we spot those two metrics natively exposed by kubernetes: 

1. `kube_deployment_status_replicas_updated`: the number of pod matching the deployment configuration
1. `kube_deployment_spec_replicas`: the number of pod requested by the replicaset

In other word, we have the total number of pod requested `kube_deployment_spec_replicas` and number of pod that have been updated `kube_deployment_status_replicas_updated`

Then, by simply adding a new single stats in our dashboard, we can display the status of our pods, and display if we are currently updating them, and even the progress of the installation.

The single stat formula is : `kube_deployment_status_replicas_updated{namespace="$namespace"} / kube_deployment_spec_replicas{namespace="$namespace"}`

This is computing the number of updated pods divided by the expected number of pod.

<center>
![Installation progress](/images/posts/2019-09-05-tracking-kubernetes-deployment/progress.png)
</center>

*Note: we are filtering on namespace because we keep one application per namespace, but you might need to filter on the deployment name if you have organized your deployments in an another maner.*

We ended with a complete dashboard that displays our main metrics AND the installation progress. 

<center>
![Installation progress](/images/posts/2019-09-05-tracking-kubernetes-deployment/full-dashboard.png)
</center>

Shared with :heart:, we hope it will save you some time.

### Author

**Mathieu POUSSE**, lead dev at Ouest France
