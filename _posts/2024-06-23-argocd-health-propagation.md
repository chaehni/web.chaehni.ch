---
title: "Customizing Health Checks in Argo CD"
excerpt: Enabling transitive health checks for Argo CD's app of apps pattern
toc: true
last_modified_at: 2024-06-23
header:
  overlay_image: "/assets/images/argo-health/overlay_image.png"
  overlay_filter: 0.5
  teaser: "/assets/images/argo-health/teaser_image.png"
published: true
before_gallery:
  - url: /assets/images/argo-health/broken_component.png
    image_path: /assets/images/argo-health/broken_component.png
    alt: application component that is not healthy
    title: application component that is not healthy
  - url: /assets/images/argo-health/no_health_propagation.png
    image_path: /assets/images/argo-health/no_health_propagation.png
    alt: One level higher, the broken application is not visible
    title: One level higher, the broken application is not visible. Child applications are not showing any health indication and as a result, the parent application assumes healthiness.
after_gallery:
  - url: /assets/images/argo-health/error_propagation.png
    image_path: /assets/images/argo-health/error_propagation.png
    alt: After custom health checks are activated, the error is reported by the child application
    title: After custom health checks are activated, the error is reported by the child application. Consequently, the parent application aggregates both children's health status and reports as `Degraded` itself.
  - url: /assets/images/argo-health/error_propagation_with_hook.png
    image_path: /assets/images/argo-health/error_propagation_with_hook.png
    alt: Since the parent's sync now takes into account the children's health, we can leverage Argo CD hooks to report success and failures of syncing all child components.
    title: Since the parent's sync now takes into account the children's health, we can leverage Argo CD hooks to report success and failures of syncing all child components. Here a `SyncFail` hook was run as one component failed to sync.
  - url: /assets/images/argo-health/all_components_healthy.png
    image_path: /assets/images/argo-health/all_components_healthy.png
    alt: If all components synced successfully on the other hand, the healthy status is reflected by the child applications and the parent sets its status to healthy accordingly.
    title: If all components synced successfully on the other hand, the healthy status is reflected by the child applications and the parent sets its status to healthy accordingly.
  - url: /assets/images/argo-health/no_error_propagation.png
    image_path: /assets/images/argo-health/no_error_propagation.png
    alt: If the annotation on the application resource is not set, the health status is trivially reported as healthy. A message gives us a hint that health checks are not enabled for this resource.
    title: If the annotation on the application resource is not set, the health status is trivially reported as healthy. A message gives us a hint that health checks are not enabled for this resource.
categories:
  - Deployment
tags:
  - GitOps
  - Arog CD
  - Tekton
  - OpenShift
  - Cloud Native
techs:
  - argocd:
    name: "ArgoCD"
    url: "https://argo-cd.readthedocs.io/en/stable/"
    image: "/assets/images/tech/argocd.png"
  - openshift:
    name: "Redhat OpenShift"
    url: "https://www.redhat.com/de/technologies/cloud-computing/openshift"
    image: "/assets/images/tech/openshift.png"
  - lua:
    name: Lua
    url: http://www.lua.org/
    image: /assets/images/tech/lua.png
---

# Managing Application Dependencies with Argo CD in OpenShift

## Introduction

In the world of modern software development, efficient deployment and continuous integration/delivery (CI/CD) pipelines are crucial. At my workplace, we leverage OpenShift for our Kubernetes environment and use GitOps principles to manage our deployments. Our tool of choice for GitOps is Argo CD, which integrates seamlessly with OpenShift. Recently, I encountered a challenging scenario that I believe many others might face as well: coordinating deployments using the "app of apps" pattern with Argo CD and triggering subsequent pipeline tasks only when all components are healthy. In this post, I’ll share the problem, my approach, and the solution I implemented.

## Understanding the Tech Stack

### OpenShift

OpenShift is Red Hat's Kubernetes distribution that provides a robust platform for container orchestration and application development. It enhances Kubernetes with developer and operational tools, CI/CD pipelines, and a strong security model.

### GitOps

GitOps is a set of practices that uses Git repositories as the source of truth for declarative infrastructure and applications. This approach allows for version-controlled, auditable, and consistent deployments. Argo CD is our GitOps operator of choice, providing continuous delivery capabilities.

### Argo CD

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. It automates the deployment of desired application states in a Kubernetes cluster, defined in Git repositories. Argo CD monitors applications and ensures that the live state matches the desired state.

#### Sync Concepts in Argo CD

Understanding the sync process in Argo CD is crucial for configuring and troubleshooting deployments. There are two main sync indicators in Argo CD:

1. **Sync Status**: This indicates whether Argo CD has successfully pulled Git manifests and applied them to the cluster. It essentially reflects the outcome of an `oc apply` or `kubectl apply` operation. A successful sync status means the manifests were applied, but it does not guarantee that the resources are running correctly.

2. **Last Sync**: This indicator provides information about the last deployment's outcome. Even if the manifests were applied successfully, the application might still be in a bad state (e.g., pods not starting correctly). The last sync indicator helps identify such issues by showing the status of the deployment.
This sync indicator is tightly coupled with the health indicators of the resources. Once everything is fully synced (meaning the last sync was successful), the resources should ideally report as healthy.

### Health Indicators in Argo CD

Argo CD uses health indicators to provide a comprehensive view of the application's state. Health checks are integrated into many resources and are reflected in the application's health status:

- **Healthy**: All resources are running as expected.
- **Progressing**: Resources are still being deployed or updated.
- **Degraded**: Some resources are in a bad state.

The health status shown in the Argo CD UI is an aggregation of all the individual child resource health indicators. If one resource is degraded or still progressing, the application health reflects this status.

### Post-Sync Hooks

It's essential to understand when post-sync hooks are executed in Argo CD:
Post-sync hooks are not triggered based on sync status alone. Sync status merely indicates that manifests were applied to the cluster. Post-sync hooks are executed after the last sync is successful, meaning the manifests were not only applied but the application is up and running and healthy.

## The App of Apps Pattern in Argo CD

The "app of apps" pattern is a powerful way to manage complex deployments in Argo CD. This pattern involves a top-level Argo CD application that references other Argo CD applications. Each of these referenced applications can represent a different component of the overall system.

### Benefits

- **Modularity**: Each component can be developed, tested, and deployed independently.
- **Scalability**: Easily manage large numbers of applications.
- **Reusability**: Components can be reused across different top-level applications.

## The Challenge: Propagating Health Status

Our deployment pipeline is divided into two parts:
1. Triggering Argo CD to deploy the application components.
2. Running a pipeline after the successful deployment of all components.

Our goal was to use post sync hooks in Argo CD to trigger the pipeline once all components of an application were deployed. However, we faced a significant challenge: the health status of individual components did not propagate to the top-level application. Recall, that an application's health is the aggreation of the health statusses of its immedaite children. In an app of apps situation, the intermediate children of the parent app are its children apps. So far so good. The problem is, that these child applications, by default, do not have a health status. Since the health aggregation over child resources only takes resources into account that provide health information, in our app of apps pattern the parent app has no information to work with and trivially sets its own health to "Healthy". This in turn transitions the parent application into "post sync" state. With the application now synced and healthy, our post sync webhook is fired and our subsequent pipeline is triggered, even though we have no indication whether our app components comoponents were deployed successfully.

{% include gallery id="before_gallery" caption="initial state" %}

## Exploring Potential Solutions

### Argo CD Capabilities

We have seen that Argo CD provides detailed health checks for individual applications but does not natively aggregate these health statuses to the top-level application in an "app of apps" setup. This means that while each component can report its health status, there is no out-of-the-box mechanism for the top-level application to reflect the aggregated health of all its child applications.

### Solution: Scriptable Health Checks with Lua

Upon further research, it turns out that Argo CD allows us to create custom healt checks for any resource by adding lua scripts to the Argo CD configuration. 

With this, the solution to our problem becomes quite easy: We need to write a script for Argo CD applications to expose their health info. This way, a parent application can pick up on it and set its own health accordingly. Should the deployment of a child application fail, this will set the parent application's state to failed, too, and no webhook is fired.

Doing some more research, it turns out that health information propagation in an "app of apps" pattern [used to be a default feature in Argo CD up to version 1.8](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app). However, this feature was removed because it proved to be confusing for some users. The corresponding Github discussion can be found [here](https://github.com/argoproj/argo-cd/issues/3781). Fortunately, the scriptable health checks feature with Lua allows us to re-enable this functionality in a way that suits our specific needs.

## Implementing the Solution

### Custom Lua Health Checks

The Argo CD documentation shows an example of how to re-enable health statusses for applications in the [documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app). The issue with this implementation is, that it alters the behavior for all applications in ArgoCD. Ideally, I'd like to be able to enable health status for select applications only.

I therefore implemented a slightly modified version of the script from the docs that additionally checks for the annotation "argocd.argproj.io/enable-health-check" to be present. If it is, the health status of the app reflects the health staus of its children. Unfortunately, there seems to be no way to completely skip the health assessment if the annotation is not present. The way the Lua scripting is done, I'm required to return a health status in any case. The only option that preserves the original behavior is returning "Healthy" as it's the only one that will cause my parent app to be healthy, too. (Like before when we didn't have any health information.) Therefore, in case the annotation is missing, I set the health status to "Healthy" regardless of the status of its children.

Here is the lua script that I used:

{% highlight lua %}
  if obj.metadata.annotations ~= nil then
      if obj.metadata.annotations["argocd.argoproj.io/enable-health-check"] == "true" then
          hs = {}
          hs.status = "Progressing"
          hs.message = ""
          if obj.status ~= nil then
              if obj.status.health ~= nil then
                  hs.status = obj.status.health.status
                  if obj.status.health.message ~= nil then
                      hs.message = obj.status.health.message
                  end
              end
          end
          return hs
      end
  end
  hs = {}
  hs.status = "Healthy"
  hs.message = "health check is not enabled for this application"
  return hs
{% endhighlight %}

And here is how I embedded the script in Argo CD's config via the argocd-cm Configmap.

{% highlight yaml %}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.argoproj.io_Application: |
    if obj.metadata.annotations ~= nil then
        if obj.metadata.annotations["argocd.argoproj.io/enable-health-check"] == "true" then
            hs = {}
            hs.status = "Progressing"
            hs.message = ""
            if obj.status ~= nil then
                if obj.status.health ~= nil then
                    hs.status = obj.status.health.status
                    if obj.status.health.message ~= nil then
                        hs.message = obj.status.health.message
                    end
                end
            end
            return hs
        end
    end
    hs = {}
    hs.status = "Healthy"
    hs.message = "health check is not enabled for this application"
    return hs
{% endhighlight %}

## Result

With this Lua script in place, the top-level application is now automatically aggregating the health statuses of its child applications. This approach allows the top-level application to accurately reflect the overall health of all components.
Our post-sync hook now runs only once all components report that they are healthy and therefore our parent application becomes healthy, too.

{% include gallery id="after_gallery" caption="results after applying the custom health checks" %}

## Lessons Learned

### Key Takeaways

- **Custom Solutions**: Sometimes, custom solutions are necessary to meet specific requirements.
- **Scriptable Health Checks**: Leveraging Lua scripts for health checks provides flexibility and control over application health status.
- **Documentation**: Keeping detailed documentation of the deployment process and custom solutions is essential for maintenance and troubleshooting.

## Conclusion

Managing application dependencies and coordinating deployments in a complex environment can be challenging. By leveraging the "app of apps" pattern, custom Lua health checks, and post sync hooks in Argo CD, we were able to streamline my deployment process and ensure that our pipeline runs smoothly after all components are healthy. I hope my experience and solutions provide valuable insights for others facing similar challenges. Feel free to share your experiences and solutions in the comments below!