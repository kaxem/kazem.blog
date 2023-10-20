---
title: ArgoCD, why not?
date: 2023-10-17
tags: GitOps, K8s, DevOps
---
## Whats GitOps?
GitOps principles uses git repositories as single source of truth to deliver infrastructure as code.
## GitOps workflow?
GitOps uses Git as the version control system for infrastructure configurations.
CI-CD pipelines usually triggered by external event like code being pushed to a repository.
**In GitOps workflow, changes are made using pull requests which modify state in the Git repository.**

## Why we need GitOps?
in Basic CI/CD every one has access to kubernetes cluster and can change it by pushing code into git repository. So it make it hard to trace who executed what in kubernetes.
But in GitOps methode an agent is installed on K8s Cluster, it tracks changes between git state and kubernetes state then if there was an change **Pulls** it from Git repository.

![pull based](/images/ArgoCD/2.webp)


## Pros of GitOps
- Rolling back in GitOps methode is easy
- If a commit breaks k8s we can use `git revert` into last commit
- **Increases Security**: you dont have to give every team member to execute changes. Only CD pipeline needs access.
	- With traditional CI/CD you need to install kubectl or helm in order of applying changes into k8s.
	- you need to config access to K8s in your runner. 
------------------------
# Argo CD
Expect of external access to k8s we have ArgoCD as a part of k8s cluster<br/>
![Argo](/images/ArgoCD/argo.png)
- Argo CD helps us to seperate CI from CD: in other words CI will change bu developers and DevOps engineers but CD is automatically done by Argo. 
## ArgoCD advantages
- Git as source of truth: If someone apply a change in k8s cluster manually, ArgoCD checks diffrents between actual state(kubernetes state) and Desired state(last modified state in git repository), then it change git's state automatically. This helps to k8s manifests in git remain source of truth.
- Easy rollback: expect of using `kubectl delete` or such as things we can easily roll back to last state with argo cd.
- Cluster disaster recovery: if a k8s cluster get completely crashed and not working. we can make a new cluster and install argo agent on it then point argo to git repository, so it will deploy all resources again.
- **K8s Access Control**: as security best practice, you dont give everyone access to k8s cluster. u just give them access to git repositories.



- Working with multiple clusters:<br/>
assume that we have different enviroments like developement and staging and production and we don't want to one config for each of them. thers's two way to handle this<br/> 	1. Use multiple braches and diffrent configs (it's not best practice)<br/> 	2. Use overlays in Kustomize
with this tool we can make base files then overlay configs for each enviroment.

## Argo CD configuration

let's take a look at minimal argo config:
	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	name: guestbook
	namespace: argocd
	spec:
	project: default
	source:
		repoURL: https://github.com/argoproj/argocd-example-apps.git
		targetRevision: HEAD
		path: guestbook
	destination:
		server: https://kubernetes.default.svc
		namespace: guestbook
* apiVersion: latest argo version
* source: source of git repository
* destination: with k8s service to deploy on

## ArgoCD best practices
1. Use separate Git reposiroty for kubernetes manifests to keep Kubernetes configs separate from source code is suggested for cleaner audit log and separation access.
2. Use Git-SHA tags or SemVer tag to immute manifests.


<br/>
<br/>
<br/>







Resources: 

- [Declarative GitOps CD for Kubernetes](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [What is GitOps by RedHat](https://www.redhat.com/en/topics/devops/what-is-gitops)
- [ ArgoCD Tutorial for Beginners by Tech with Nana](https://www.youtube.com/watch?v=MeU5_k9ssrs)
