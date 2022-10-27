# Installing Argo Rollouts

Argo Rollouts is a controller that runs in its own namespace.  
Per default this is argo-rollouts.
The installation through kubectl would be:

     kubectl create namespace argo-rollouts  
     kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

Using a *kustomize.yaml* file following example can be used:

	apiVersion: kustomize.config.k8s.io/v1beta1
	kind: Kustomization
	
	namespace: argo-rollouts
	resources:
	  - https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

If there is a need to start the controller in a different namespace the *install.yaml* also needs to be adapted.
See [Installation](https://argoproj.github.io/argo-rollouts/installation/).

The account that applies the kustomize resource also needs the ability to create new cluster roles.


## Sample Application

The presented sample application can be seen in this [repository](https://github.com/martin-weber-1997/rollouts-demo/tree/master/examples/istio-subset).  
For this sample the load distribution between the stable and the canary set will be handeled by Istio.
For Rollouts following things need to be installed:

- ArgoCD
- Istio
- Argo Rollouts

Following resources are needed:

- [Rollout](#rollout)
- [Service](#service)
- [VirtualService](#virtual-service)
- [DestinationRule](#destination-rule)

### Rollout
Important settings that differ from Deployments are `kind: Rollout` which needs on `apiVersion: argoproj.io/v1alpha1`.
The rollout then needs a `strategy`, in our case that strategy is `canary`. Since `trafficRouting` is handled by `istio` this also needs to be defined.
The `virtualService ` and `destinationRule ` will be explained later on.


		apiVersion: argoproj.io/v1alpha1
		kind: Rollout
		metadata:
		  name: istio-rollout
		spec:
		  revisionHistoryLimit: 2
		  selector:
		    matchLabels:
		      app: istio-rollout
		  template:
		    metadata:
		      labels:
		        app: istio-rollout
		    spec:
		      containers:
		      - name: istio-rollout
		        image: argoproj/rollouts-demo:blue
		        ports:
		        - name: http
		          containerPort: 8080
		          protocol: TCP
		        resources:
		          requests:
		            memory: 32Mi
		            cpu: 5m
		  strategy:
		    canary:
		      trafficRouting:
		        istio:
		          virtualService:
		            name: istio-rollout-vsvc
		            routes:
		            - primary
		          destinationRule:
		            name: rollout-destrule    # required
		            canarySubsetName: canary  # required
		            stableSubsetName: stable  # required
		      steps:
		      - setWeight: 10
		      - pause: {}         # pause indefinitely
		      - setWeight: 20
		      - pause: {duration: 20s}
		      # more steps can be added but are not relevant for the example
		      - setWeight: 90
		      - pause: {duration: 20s}


### Service
The Rollout needs a service that targets the rollout pods.

	apiVersion: v1
	kind: Service
	metadata:
	  name: istio-rollout
	spec:
	  ports:
	  - port: 80
	    targetPort: http
	    protocol: TCP
	    name: http
	  selector:
	    app: istio-rollout

### Virtual service
The VirtualService now defines the route that has destinations. The canary and the stable subsets.
Those subsets are defined in the destination rules.
The VirtualService also contains the weights which will get modified by the rollout controller in the case of a new canary release.
Those weights need to add up to 100 so the base case is that the stable subset recieves all traffic.

	apiVersion: networking.istio.io/v1alpha3
	kind: VirtualService
	metadata:
	  name: istio-rollout-vsvc
	spec:
	  gateways:
	  - istio-rollout-gateway
	  hosts:
	  - '*'
	  http:
	  - name: primary
	    route:
	    - destination:
	        host: istio-rollout
	        subset: stable  # referenced in canary.trafficRouting.istio.destinationRule.stableSubsetName
	      weight: 100
	    - destination:
	        host: istio-rollout
	        subset: canary  # referenced in canary.trafficRouting.istio.destinationRule.canarySubsetName
	      weight: 0
### Destination rule
Finally the DestinationRule contains the subsets that are referenced in the Virtual service and the Rollout itself.

	apiVersion: networking.istio.io/v1alpha3
	kind: DestinationRule
	metadata:
	  name: rollout-destrule
	spec:
	  host: istio-rollout
	  subsets:
	  - name: canary   # referenced in canary.trafficRouting.istio.destinationRule.canarySubsetName
	    labels:        # labels will be injected with canary rollouts-pod-template-hash value
	      app: istio-rollout
	  - name: stable   # referenced in canary.trafficRouting.istio.destinationRule.stableSubsetName
	    labels:        # labels will be injected with canary rollouts-pod-template-hash value
	      app: istio-rollout


While a rollout is in progress the argo rollout controller adapts the weights in the virtual service as well as the destination rules labels to contain the correct hashes.


For further information and details check out the argo rollouts documentations:

[Canary Deployment](https://argoproj.github.io/argo-rollouts/features/canary/)  
[Istio Traffic Splitting](https://argoproj.github.io/argo-rollouts/features/traffic-management/istio/#subset-level-traffic-splitting)


## Migration from Deployments


## Things to consider


## Further concepts


