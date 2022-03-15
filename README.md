# Kubebuilder CronJob Tutorial
My attempt at the Kubebuilder CronJob Tutorial

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