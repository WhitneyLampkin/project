# Kubebuilder CronJob Tutorial Project
My attempt at the Kubebuilder CronJob Tutorial & personal notes.

## Introduction

- Who is this for?
	- Users of Kubernetes - build a deeper understanding
	- Kubernetes API Extension Developers - learn core principles and concepts
- Why Kubernetes APIs?
	- Consistent and well-defined endpoints for objects w/ a consistent and rich structure
	- Objects declared with yaml or json config files
	- Objects are managed with common tools
	- Building services as Kubernetes APIs have more advantages than plain REST alone


## CronJob Tutorial

- CronJob Controller - runs one-off tasks on the K8s cluster at regular intervals
	- Build on the Job Controller- runs one-tasks once


## 1.1 What's in a basic Kubebuilder project?

- Build Infrastructure
	- go.mod - New Go module matching our project, with basic dependencies
	- Makefile - Make targets for building and deploying the controller
	- PROJECT - Kubebuilder metadata for scaffolding new components
- Launch Configuration
	- config/ directory 
		- Launch configurations defined as Kustomize YAML definitions required to launch the controller on a cluster
		- CustomResourceDefinitions (CRDs)
		- RBAC configuration
		- WebhookConfigurations
		- Kustomize base for launching the controller with a standard configuration
	- config/manager
		- Launch controllers as pods in the cluster
	- config/base 
		- Permissions required to run controllers under their own service account
- The Entrypoint
	- main.go file


## 1.2 Main.go

- Basic imports
	- Core controller-runtime library
	- Default controller-runtime logging (Zap)
- Scheme
	- Each controller uses one to map Kinds to their Go types
- The main function
	- Sets up basic flags for metrics
	- Instantiates a manager 
		- Keeps track of running controllers
		- Sets up shared caches and clients to the API server
	- Runs the manager
		- Runs all of the controllers and webhooks
		- Continues running until it receives a graceful shutdown signal
			- Allows for nice behavior with graceful pod terminations
		- Managers can restrict the namespace that all controllers watch for resources by


## 1.3 Groups and Versions and Kinds, oh my!

- Groups and Versions
	- Groups - collection of related functionality
	- Versions - change how the API works over time
- Kinds and Resources
	- Kinds - API types stored in API group-versions
		- Can change form between versions but must hold the same data
		- Older API versions thus don't cause newer data to be lost
	- Resources - A use of a Kind within the API
		- Kinds and resources map 1:1 - Kind:Resource (i.e. PodResource:PodKind)
		- Sometimes a Kind may be returned by many resources
		- CRDs each Kind can only map to one Resource
	- GroupVersionKind (GVK) - a kind that belongs to a particular group-version
		- Each GVK corresponds to a root Go type in a package
	- GroupVersionResource (GVR) - a resource that belongs to a particular group-version
- Creating an API
	- Use the command kubebuilder create api
		- Creates a CustomResource (CR) and CustomResourceDefinition (CRD) for our Kind(s)
- Why create APIs?
	- New APIs tell K8s about our custom objects
	- Go structs generate a CRD with the schema of our data
	- Instances of the custom object is created and managed by the controllers
	- CRDs - Definition of our customized objects
	- CRs - Instances of the customized objects
- Example
	- Kubernetes cluster with:
		- Application CRD
		- Database CRD
	- Doesn't not interfere with 
		- Encapsulation
		- Single Responsibility Principle
		- Cohesion
- The Scheme
	- Keeps track of what Go types represent a GVK


## 1.4 Adding a new API

- Steps
	- kubebuilder create api --group batch --version v1 --kind CronJob adds a new API
	- Press 'y' to create resource and controller when prompted
- Result
	- Creates api/v1 directory (1st time)
		- Added Cronjob Kind file 
			- Imports the meta/v1 API group with metadata common to all Kubernetes Kinds
			- Defines types for __Specs and __Status of the Kind
				- Kubernetes works by comparing desired state (Spec) with the actual cluster and external states and recording what it finds in Status (Most types follow this pattern; not all)
			- Defines the actual Kinds (<KindName> and <KindName>List
				- Never modify directly
				- Modify in Spec or Status
			- Markers +kubebuilder:____ acts as extra metadata and tell controller-tools extra info
			- the init() function adds the Go types to the API group
				- Allows us to add the types in this API group to any scheme
		- (Will add more Kind files each time we call this with a different Kind name)
	- Creates controller directory (1st time)


## 1.5 Designing an API (The __types.go file)

- Use camelCase for serialized fields (JSON struct tags)
- omitempty struct tag - omits a field from serialization when empty
- Fields can use most primitive types except numbers, allowed datatypes for numbers are:
	- Int32
	- int64
	- resource.Quantity for decimals
- Spec - holds desired state
	- Inputs to controllers go here
	- CronJob Spec Needs
		- Default
			- Scheduler - represents the 'cron' in CronJob
			- Template - what the job actually runs (represents the job in CronJob)
		- Others
			- Deadline for starting jobs
			- Rule for if multiple jobs would run at once
			- Way to pause running of a CronJob
			- Limits on old job history
			- An old job will be used to determine when our job last ran since we don't track our own status
- Status - holds observed state (information we want users or other controllers to easily obtain)
	- List of actively running jobs
	- Last successful run time (uses metav1.Time instead of time.Time)
- Other api/v1 files - NEVER EDIT - read more if necessary
	- groupversion_info.go - common metadata about the group-version
	- zz_generated.deepcopy.go  - autogenerated implementation of the runtime.Object interface
		- Makes all of the root types as representing Kinds


## 1.6 What's in a controller? (Controller Files)

- Controllers - core of Kubernetes and any operator
	- Ensures actual state matches the desired state
	- Each controller focuses on the root Kind but may interact with any Kinds
- Reconciling
	- Reconciler - logic that implements the reconciling for a specific Kind
		- Takes name of object and returns whether or not we need to try again
- Controller File
	- Standard imports
		- Core controller-runtime library
		- Client package
		- Package for API types
	- Basic reconciler struct
		- Every reconciler needs logging and to fetch objects so that's what's scaffolded initially
		- Some controllers need to run on the cluster, which requires
			- RBAC permissions (using RBAC markers) - bare minimum permissions to run
	- Reconcile() function
		- Performs the reconciling for a single named object
		- Success reconciling returns an empty results and no error (nil)


## 1.7 Implementing the Controller File

- Basic Logic
	1. Load the named CronJob
	2. List all active jobs and update the status
	3. Clean up old jobs according to the history limits
	4. Check if we're suspended (and don't do anything else if we are)
	5. Get the next scheduled run
	6. Run a new job if it's on schedule, not past the deadline and not blocked by our concurrency policy
	7. Requeue when we either see a running job (done automatically) or it's time for the next scheduled run
- Revising the main.go file
	- Kubebuilder added the new API group's package (batchv1)
	- Kubebuilder also added a block to call the CronJob controller's SetupWithManager() method


## 1.8 Implementing defaulting/validating webhooks

- Admission webhooks for CRDs use Defaulter and/or Validator interface
	- Kubebuilder will
		- Create the webhook server
		- Add server to the manager
		- Create handlers for your webhooks
		- Register each handler with a path in your server
- Admission webhooks - HTTP callbacks that receive admission requests, process them and return admission responses
	- Mutating Admission Webhook - mutates the object during creation or updating
	- Validating Admission Webhook - validates the object during creation or updating
- kubebuilder create webhook --group batch --version v1 --kind CronJob --defaulting --programmatic-validation scaffolds the webhooks for CRD (CronJob)
- Cronjob_webhooks.go file Contents
	- Import statements
	- Logger setup for the webhook
	- Webhook with manager setup
	- Kubebuilder marker used to generate mutating webhook manifest
	- The webhook.Defaulter interface used to set defaults to the CRD
	- Use webhook.Validator interface for more validation upon creation, updates and deletion
		- ValidateCreate function
		- VaidateUpdate function
		- VaidateDelete function
	- Validate the name and spec of the Cronjob
- NOTE
	- // +kubebuilder:validation markers are found in the 'Designing an API' section
	- controller-gen crd -w finds ALL supported markers for declaring validations

## 1.9 Running and Deploying the Controller

- Optional
	- make manifests to make changes to the API definitions
- Run and test the controller locally against the cluster
	- Install CRDs with make install
		- Automatically updates the YAML manifests, if needed
	- Run the controller against our cluster
		- No RBAC necessary at this point
	- Use cat -e -t -v makefile_name to check for tabs in the Makefile
	- Use KIND to create a cluster
		- kind create cluster
		- kind delete cluster
	- Optionally add/change API definitions and generate new manifests for new CRs and CRDs with maek manifests
	- Install CRDs with make install
	- Use new terminal to run the controllers with make run ENABLE_WEBHOOKS=false
		- [RESULT] Controller logs but nothing else
	- Write sample/fake CronJob to test the controller with
		- Create the yaml file
		- Create with kubectl create -f config/samples/batch_v1_cronjob.yaml
		- [RESULT] Terminal should show more activity related to the new Cronjob that's running
		- Watch Cronjob Changes
			- kubectl get cronjob.batch.tutorial.kubebuilder.io -o yaml
			- kubectl get job
		- Run in the cluster
			- Stop the make run with ctrl + c
			- Docker Build
				- make docker-build docker-push IMG=<some-registry>/<project-name>:tag 
			- Docker Deploy
				- make deploy IMG=<some-registry>/<project-name>:tag 
			- [RESULTS] List the job against


## 1.9.1 Deploying the Cert Manager

- Download/install Cert Manager
- Use cert-manager.io/inject-ca-from key in the Mutating | ValidatingWebhookConfiguration objects
	- <certificate-namespace>/<certificate-name>


## 1.9.2 Deploying Admission Webhooks

- Requirements
	- Kind cluster
	- Cert Manager
- Build the image
	- make docker-build docker-push IMG=<some-registry>/<project-name>:tag
		- Couldn't get past this point because of issues with Cert Manager...
		- TODO: Return to this at a later time
	- kind load docker-image <your-image-name>:tag --name <your-kind-cluster-name>
- Deploy Webhooks
	- Update config/default/kustomization.yaml and config/crd/kustomization.yaml 
	- Deploy to the cluster
		- Use docker images to list the images if you don't remember the image name

## Summary of Steps to Run Cronjob Project

- Create Kind cluster
- Make Install
- Make Run w/ ENABLE_WEBHOOKS=false
- Confirm controller set up with
	- kubectl create -f config/samples/batch_v1_cronjob.yaml
	- kubectl get cronjob.batch.tutorial.kubebuilder.io -o yaml
	- kubectl get job
- Run in Kind Cluster
	- make docker-build docker-push IMG=<some-registry>/<project-name>:tag
		- Example
	- make deploy IMG=<some-registry>/<project-name>:tag
		- Example
	- [NOTE] The idea of <some-registry> was throwing me off so I left it off to run locally.

1.10 Writing Tests

- Tests use Ginkgo

  
1.11 Epilogue
-
# [ISSUES/ERRORS] Kubebuilder Tutorial Notes

## Quick Start Tutorial

- Prerequisites 
- Installation
- Create Project
- Create an API
- Test it out
	- [ISSUE] error: unable to recognize "STDIN": no matches for kind "CustomResourceDefinition" in version "apiextensions.k8s.io/v1beta1"
	- [SOLUTION] Use older image when creating the kind cluster
		- kind create cluster --image=kindest/node:v1.21.2
- Install Instances of Customer Resources
- Run It on the Cluster
	- [ISSUE] failed to start the controlplane. retried 5 times: fork/exec /usr/local/kubebuilder/bin/etcd: no such file or directory
	- [CAUSE] kubebuilder assumes the etcd is in a certain folder but it's not a guarantee
		- How do I locate the etcd?


## CronJob

- CronJob - runs one-off tasks on the Kubernetes cluster at regular intervals
	- Builds on top of the job controller (job controllers run the tasks only once)
- Webhook Implementation
	- [ISSUE] 2022/03/14 19:12:58 failed to create webhook: unknown project version 3
	- [CAUSE] PROJECT file's version had "3"
	- [SOLUTION] Change PROJECT file's version to "2"; "1" doesn't support all of the flags needed for the command
- Running the CronJob
	- [ISSUE] Makefile: 46: *** missing separator. Stop.
	- [CAUSE] Possibly spaces being used instead of tabs
	- [SOLUTION] cat -e -t -v makefile_name to find and replace spaces with tabs
		- Copied the Makefile from the GitHub repo
- - Also in Running the CronJob
	- [ISSUE] SideEffects is required for creating v1 {Mutating,Validating}WebhookConfiguration & Error: not all generators ran successfully
	- [CAUSE] Missing 'SideEffects' argument for marker used in webhook configuration in cronjob_webhook.go
	- [SOLUTION] added  sideEffects=None argument to the +kubebuilder markers in cronjob_webhook.go
		- If you use the code in the GitHub repo then it's already there; otherwise, you'd have to follow the error message to figure it out.
	- [ISSUE] Go versions don't match
	- [CAUSE] Installed an older version of Go for the validator project but Kubebuilder requires a newer version 
	- [SOLUTION] Use GVM to install the new version
		- Also deleted the older version from WINDOWS 
		- /mnt/c/ uses newer version to run Kubebuilder
		- Linux uses older version since there is where my aks-validator work is done
		- Finally got the make manifests and make install commands to succeed with these changes:
		- [CRD Creation Screenshot](https://github.com/WhitneyLampkin/project/blob/master/images/crd-creation-terminal-screenshot.png?raw=true)
	- [ISSUE] Test Failures
		- [CronJob Test Failures Screenshot](https://github.com/WhitneyLampkin/project/blob/master/images/failed-cronjob-test-screenshot.png?raw=true)
		- [CAUSE] ? 
	- [SOLUTION] ?
- Running the CronJob (Makefile  vet command issue)
	- [ISSUE] Makefile - vet: go vet./...
		- `tutorial.kubebuilder.io/project/api/v1 tested by
			        tutorial.kubebuilder.io/project/api/v1.test imports
			        sigs.k8s.io/controller-runtime/pkg/envtest imports
			        sigs.k8s.io/controller-runtime/pkg/internal/testing/addr imports
			        io/fs: package io/fs is not in GOROOT (/home/wlampkin/.gvm/gos/go1.14.15/src/io/fs)`
	- [CAUSE] I forgot to use `gvm` to change my go version to the newer version. 
		- This package wasn't available with the older version.
	- [SOLUTION] `gvm use 1.17.8`
- Running the CronJob (Deploying webhooks)
	- [ISSUE] make docker push command fails
		- `make docker-push IMG=kind-kind/project:v1
				docker push kind-kind/project:v1
				The push refers to repository [docker.io/kind-kind/project]
				8efc1b9bea2d: Preparing 
				5b1fa8e3e100: Preparing 
				denied: requested access to the resource is denied
				make: *** [Makefile:67: docker-push] Error 1` 
		- [Docker Push Issues Screenshot](docker-push-issues.png)
	- [CAUSE] 
		- Did I need to run the image?
		- Do I need to tag the docker image?
	- [SOLUTION]
		- Used docker documentation to correct image issues
		- Also switched to Ubuntu image
		- Getting activity with make docker-build docker-push IMG=localhost:45875/kind-kind/project:v1  
		- docker-build works but docker-push still fails
		- TODOs:
			- Look into issues with sigs.k8s.io/controller-runtime/pkg/client/config.GetConfigOrDie
			- Why does controller-runtime have so many issues in the project?
		- Totally Uninstalled ALL Go verisons & GVM
			- Installed GVM
			- Installed Go 1.4 as base
			- Installed Go 1.17.8 to get all of the packages needed for the Cronjob project
		- Cloned the repo to Linux and opened with code . from the Ubuntu terminal
- Running the CronJob (Building controllers failed)
	- [ISSUE] => ERROR [builder 8/9] COPY controllers/ controllers/
			------
			 > [builder 8/9] COPY controllers/ controllers/:
			------
			failed to compute cache key: "/controllers" not found: not found
			make: *** [Makefile:64: docker-build] Error 1
	- [CAUSE] Controller weren't in the project because I was using the wrong Kubebuilder path. I had the starter project instead of the completed version.
	- [SOLUTION] Change directory to : /go/GitHub/github.com/whitneylampkin/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project 
		- SUCCESS!!!
	- Running the Cronjob (Error 403 - User Forbidden on Docker Push)
	- [ISSUE] Error 403 - User Forbidden on Docker Push
	- [CAUSE] Line 25 of Dockerfile 
	- [SOLUTION] Commented out the USER 65532:65532 line
- Running the Cronjob (TCP Connection Refused on Docker Push)
	- [ISSUE] 
		- <TODO Add image>
	- [CAUSE] using localhost or 127.0.0.0 for docker tag
	- [SOLUTION] 
		- Change localhost or 127.0.0.0 in docker tag
		- New Error
			- <TODO Add image>
		- I honestly don't know what's going on at this point but there is a new error...
- Running the CronJob (Docker Push Issue)
	- [ISSUE] 
		- <TODO Add image>
	- [CAUSE] Who knows?
	- [SOLUTION] 
		- Used https://docs.docker.com/registry/deploying/ to create a registry and pull down the golang:1.17 image 
		- Disabled 'buildkit' in docker engine file using docker desktop
		- Progress:
			- <TODO Add image>
			- Commented out lines 22-27 in Dockerfile because /workspace/manager couldn't be found
			- <TODO Add image>
			- Reinstalled cert manager now that I know more about it
				- Remember to install CRDs separately if cert-manager-cainjector... is stuck in CrashLoopBackOff
				- Now I'm running into a cert manager issue that is UGHHHHHHH!!!! So annoying! 

# Dependencies

- Cert Manager
- Docker
- Ginkgo
- Kind
- Kubernetes
- Vim (used in writing tests section)

# Helpful Commands

- Kind Commands

		kind create cluster --name=<CLUSTER NAME> --image=<IMAGE NAME>     
		
		kind delete cluster

- Kubectl Commands

		kubectl cluster-info --context <CLUSTER NAME>
		
		[CERT MANAGER] COMMANDS
		
		kubectl get pods --namespace cert-manager 
		
		kubectl get crd 
		
		kubectl describe crd clusterissuers.cert-manager.io 

- Docker Commands

		docker ps

- Other

		sudo apt-get remove golang-go Removes Go from Ubuntu
