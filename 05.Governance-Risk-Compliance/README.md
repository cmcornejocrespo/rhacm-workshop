# Exercise 5 - Governance Risk and Compliance

In this exercise you will go through the Compliance features that come with Red Hat Advanced Cluster Management for Kubernetes. You will apply a number of policies to the cluster in order to comply with global security and management standards.

**NOTE!** This exercise depends on the ACM application deployed in the previous exercise (NOT the application deployed using ArgoCD). If the application is not available in your environment, run the next command to deploy it -

```sh
<hub> $ oc apply -f https://raw.githubusercontent.com/cmcornejocrespo/rhacm-workshop/master/04.Application-Lifecycle/exercise-application/rhacm-resources/application.yaml
```

**NOTE!** Make sure that the `environment=production` label is associated to any of the managed clusters!

Before you start creating the policies, make sure to create a namespace to populate the CRs that associate with RHACM policies.

```yaml
<hub> $ cat <<EOF | oc apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: rhacm-policies
EOF
```

After the namespace is created, create a PlacementRule resource. We will use the PlacementRule to associate the below policies with all clusters that are associated with the environment=production label.

```yaml
<hub> $ cat <<EOF | oc apply -f -
---
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: prod-policies-clusters
  namespace: rhacm-policies
spec:
  clusterConditions:
    - type: ManagedClusterConditionAvailable
      status: "True"
  clusterSelector:
    matchLabels:
      environment: production
EOF
```

## Policy #1 - Network Security

In this section you will apply a NetworkPolicy object onto the cluster in order to limit access to the application you have created in the previous exercise. You will only allow traffic that comes from OpenShift’s Ingress Controller in port 8080. All other traffic will be dropped.

The policy you’ll create in this section will use the _enforce_ remediation action in order to create the NetworkPolicy objects if they do not exist.

We will configure the policy definition in two stages -

### Stage 1 - Deny all traffic to the application namespace

The policy you will configure in this section is enforcing a _deny all_ NetworkPolicy in the webserver-acm namespace on the managed cluster. A _deny all_ NetworkPolicy object example -

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-by-default
spec:
  podSelector:
  ingress: []
```

In order to create a _deny all_ NetworkPolicy object on the managed cluster using Red Hat Advanced Cluster Management for Kubernetes, apply the next commands to the hub cluster -

```yaml
<hub> $ cat <<EOF | oc apply -f -
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-networkpolicy-webserver
  namespace: rhacm-policies
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: SC System and Communications Protection
    policy.open-cluster-management.io/controls: SC-7 Boundary Protection
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-networkpolicy-denyall-webserver
        spec:
          remediationAction: enforce # the policy-template spec.remediationAction is overridden by the preceding parameter value for spec.remediationAction.
          severity: medium
          namespaceSelector:
            include: ["webserver-acm"]
          object-templates:
            - complianceType: musthave
              objectDefinition:
                kind: NetworkPolicy
                apiVersion: networking.k8s.io/v1
                metadata:
                  name: deny-by-default
                spec:
                  podSelector:
                  ingress: []
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policy-networkpolicy-webserver
  namespace: rhacm-policies
placementRef:
  name: prod-policies-clusters
  kind: PlacementRule
  apiGroup: apps.open-cluster-management.io
subjects:
- name: policy-networkpolicy-webserver
  kind: Policy
  apiGroup: policy.open-cluster-management.io
EOF
```

The above command creates two objects _Policy_ and _PlacementBinding_.

* The _Policy_ objects define the NetworkPolicy that will be deployed on the managed cluster. It associates the NetworkPolicy to the webserver-acm namespace, and enforces it.
* The _PlacementRule_ resource associates the _Policy_ object with the _PlacementRule _resource that was created in the beginning of the exercise. Thereby, allowing the Policy to apply to all clusters with the _environment=production_ label.

After the creation of the objects, navigate to **Governance Risk and Compliance** in the Red Hat Advanced Cluster Management for Kubernetes console. Note that the policy is configured, and the managed cluster is compliant.

Make sure that the policy is effective by trying to navigate to the application once again - **https://&lt;webserver application route>/application.html**. (The application should not be accessible).

### Stage 2 - Allow traffic from the Ingress Controller

In this section, you will modify the policy you have created in the previous section. You will add another ObjectDefinition entry to the policy. The ObjectDefinition will apply a second NetworkPolicy object onto the webserver-acm namespace in the managed cluster. The NetworkPolicy object will allow traffic from the Ingress Controller to reach the webserver application in port 8080. An example definition of the NetworkPolicy object -

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
spec:
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
  podSelector: {}
  policyTypes:
  - Ingress
```

Adding the NetworkPolicy to the existing policy can be done by running the next command -

```yaml
<hub> $ cat <<EOF | oc apply -f -
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-networkpolicy-webserver
  namespace: rhacm-policies
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: SC System and Communications Protection
    policy.open-cluster-management.io/controls: SC-7 Boundary Protection
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-networkpolicy-denyall-webserver
        spec:
          remediationAction: enforce # the policy-template spec.remediationAction is overridden by the preceding parameter value for spec.remediationAction.
          severity: medium
          namespaceSelector:
            include: ["webserver-acm"]
          object-templates:
            - complianceType: musthave
              objectDefinition:
                kind: NetworkPolicy
                apiVersion: networking.k8s.io/v1
                metadata:
                  name: deny-by-default
                spec:
                  podSelector:
                  ingress: []
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-networkpolicy-allow-ingress-webserver
        spec:
          remediationAction: enforce # the policy-template spec.remediationAction is overridden by the preceding parameter value for spec.remediationAction.
          severity: medium
          namespaceSelector:
            include: ["webserver-acm"]
          object-templates:
            - complianceType: musthave
              objectDefinition:
                kind: NetworkPolicy
                apiVersion: networking.k8s.io/v1
                metadata:
                  name: allow-ingress-8080
                spec:
                  ingress:
                  - ports:
                    - protocol: TCP
                      port: 8080
                  - from:
                    - namespaceSelector:
                        matchLabels:
                          network.openshift.io/policy-group: ingress
                  podSelector: {}
                  policyTypes:
                  - Ingress
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policy-networkpolicy-webserver
  namespace: rhacm-policies
placementRef:
  name: prod-policies-clusters
  kind: PlacementRule
  apiGroup: apps.open-cluster-management.io
subjects:
- name: policy-networkpolicy-webserver
  kind: Policy
  apiGroup: policy.open-cluster-management.io
EOF
```

After applying the above policy, the application will be reachable from OpenShift’s ingress controller only. Any other traffic will be dropped.

Make sure that the managed cluster is compliant to the policy by navigating to **Governance Risk and Compliance** in the Red Hat Advanced Cluster Management for Kubernetes console.

![networkpolicy-status](images/networkpolicy-status.png)

Make sure that the application is accessible now at - **https://&lt;webserver application route>/application.html**.

## Policy #2 - Quota Management

In this section you will apply a LimitRange object onto the cluster in order to limit the application’s resource consumption. You will configure a LimitRange object that limits the application’s container memory to 512Mb.

The policy you will create defines the next LimitRange object in the webserver-acm namespace:

```yaml
apiVersion: v1
kind: LimitRange # limit memory usage
metadata:
  name: webserver-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
      defaultRequest:
      memory: 256Mi
      type: Container
```

In order to apply the LimitRange object to the managed cluster using Red Hat Advanced Cluster Management for Kubernetes, run the next commands:

```yaml
<hub> $ cat <<EOF | oc apply -f -
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-limitrange
  namespace: rhacm-policies
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: SC System and Communications Protection
    policy.open-cluster-management.io/controls: SC-6 Resource Availability
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-limitrange-example
        spec:
          remediationAction: enforce # the policy-template spec.remediationAction is overridden by the preceding parameter value for spec.remediationAction.
          severity: medium
          namespaceSelector:
            include: ["webserver-acm"]
          object-templates:
            - complianceType: mustonlyhave
              objectDefinition:
                apiVersion: v1
                kind: LimitRange # limit memory usage
                metadata:
                  name: webserver-limit-range
                spec:
                  limits:
                  - default:
                      memory: 512Mi
                    defaultRequest:
                      memory: 256Mi
                    type: Container
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policy-limitrange
  namespace: rhacm-policies
placementRef:
  name: prod-policies-clusters
  kind: PlacementRule
  apiGroup: apps.open-cluster-management.io
subjects:
- name: policy-limitrange
  kind: Policy
  apiGroup: policy.open-cluster-management.io
EOF
```

Make sure that the managed cluster is compliant to the policy by navigating to **Governance Risk and Compliance** in the Red Hat Advanced Cluster Management for Kubernetes console.

Make sure that the LimitRange object is created in your managed cluster:

* Validate that the LimitRange object is created in the webserver-acm namespace:

```sh
<managed cluster> $ oc get limitrange webserver-limit-range -o yaml -n webserver-acm
```

## Policy #3 - Deleting the Kubeadmin Secret

In this section you will deploy a policy that validates the removal of the kubeadmin secret. After we get notified about the not compliancy you will changen the `remediationAction` to enforce to remove the secret

With the following acceptance criteria:
- The policy applies to the **kube-system** namespace.
- The policy defines the **secret** which is flagged for removal - **kubeadmin**.
- The policy defines a compliance type of **mustnothave**.
- The policy is **inform**. RHACM will not remove the secret if it is present.

```yaml
<hub> $ cat <<EOF | oc apply -f -
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-remove-kubeadmin
  namespace: rhacm-policies
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: AC Access Control
    policy.open-cluster-management.io/controls: AC-2 Account Management
spec:
  remediationAction: inform
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-remove-kubeadmin
        spec:
          remediationAction: inform
          severity: high
          namespaceSelector:
            include:
              - kube-system
          object-templates:
            - complianceType: mustnothave
              objectDefinition:
                kind: Secret
                metadata:
                  name: kubeadmin
                type: Opaque
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: remove-kubeadmin-binding
  namespace: rhacm-policies
placementRef:
  name: prod-policies-clusters
  kind: PlacementRule
  apiGroup: apps.open-cluster-management.io
subjects:
- name: policy-remove-kubeadmin
  kind: Policy
  apiGroup: policy.open-cluster-management.io
EOF
```
![policy-remove-kubeadmin-inform](images/policy-remove-kubeadmin-inform.png)

Now navigate to **Governance Risk and Compliance**, select **policy-remove-kubeadmin** and click on **Enforce**

You should see that your cluster is now `compliant`.

![policy-remove-kubeadmin-enforce](images/policy-remove-kubeadmin-enforce.png)

## Policy #4 - Namespace management

The Policy scans security vulnerabilities in the managed clusters’ images using the **container-security-operator**. The policy configures the **container-security-operator** on the managed cluster by applying a Subscription object onto it.

If the **container-security-operator** finds a vulnerability in one of the images, it creates an ImageManifestVuln object. The policy controller will **inform** the hub if it finds any ImageManifestVuln objects in all namespaces that do not begin with *openshift-**.

```yaml
<hub> $ cat <<EOF | oc apply -f -
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-imagemanifestvuln
  namespace: rhacm-policies
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: SI System and Information Integrity
    policy.open-cluster-management.io/controls: SI-4 Information System Monitoring
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-imagemanifestvuln-example-sub
        spec:
          remediationAction: enforce
          severity: high
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: operators.coreos.com/v1alpha1
                kind: Subscription
                metadata:
                  name: container-security-operator
                  namespace: openshift-operators
                spec:
                  channel: quay-v3.4
                  installPlanApproval: Automatic
                  name: container-security-operator
                  source: redhat-operators
                  sourceNamespace: openshift-marketplace
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-imagemanifestvuln-example-imv
        spec:
          remediationAction: inform
          severity: high
          namespaceSelector:
            exclude: ["openshift-*"]
            include: ["*"]
          object-templates:
            - complianceType: mustnothave # mustnothave any ImageManifestVuln object
              objectDefinition:
                apiVersion: secscan.quay.redhat.com/v1alpha1
                kind: ImageManifestVuln # checking for a kind
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: image-placement-binding
  namespace: rhacm-policies
placementRef:
  name: prod-policies-clusters
  kind: PlacementRule
  apiGroup: apps.open-cluster-management.io
subjects:
- name: policy-imagemanifestvuln
  kind: Policy
  apiGroup: policy.open-cluster-management.io
EOF
```

## Using GitOps

In this section you will use RHACM’s built-in GitOps mechanism to manage your policies. You will deploy the above policies, and manage them in a GitOps friendly way.

Before you start this section of the exercise, make sure you delete the namespace containing the policies you used in the previous section.

```sh
<hub> $ oc delete project rhacm-policies
```

1. For this exercise, create a fork of the next GitHub repository - [https://github.com/cmcornejocrespo/rhacm-workshop](https://github.com/cmcornejocrespo/rhacm-workshop)

    As a result, you will have your own version of the repository - [https://github.com/&lt;your-username>/rhacm-workshop](https://github.com/cmcornejocrespo/rhacm-workshop)

2. Afterwards, create a namespace on which you will deploy the RHACM resources (Use the namespace.yaml file in the forked repository) -

```sh
<hub> $ oc apply -f https://raw.githubusercontent.com/<your-github-username>/rhacm-workshop/master/05.Governance-Risk-Compliance/exercise/namespace.yaml
```

3. Now, clone the official policy-collection GitHub repository to your machine. The repository contains a binary named **deploy.sh**. The binary is used to associate policies in a GitHub repository to a running Red Hat Advanced Cluster Management for Kubernetes cluster.

```sh
<hub> $ git clone https://github.com/open-cluster-management/policy-collection.git

<hub> $ cd policy-collection/deploy/
```

4.a. If you are using the kubeadmin user, create an identity provider by running the next commands (It is not possible to create policies via GitOps using the kubeadmin user). The identity provider will create the `workshop-admin` user -

```
<hub> $ htpasswd -c -B -b htpasswd workshop-admin redhat

<hub> $ oc create secret generic localusers --from-file htpasswd=htpasswd -n openshift-config

<hub> $ oc adm policy add-cluster-role-to-user cluster-admin workshop-admin

<hub> $ oc get -o yaml oauth cluster > oauth.yaml
```

4.b. Edit the `oauth.yaml` file. The result should look like -

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
...output omitted...
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: localusers
    mappingMethod: claim
    name: local-users
    type: HTPasswd
```

4.c. Replace the cluster's identity provider by running the next command -

```
<hub> $ oc replace -f oauth.yaml
```

4.d. Login with the created user -

```
<hub> $ oc login -u workshop-admin -p redhat
```

5. Run the next command to allow your username deploy policies via Git (If you're not using the `workshop-admin` user to run the command, make sure to edit the command in order to associate your user with the `subscription-admin` ClusterRole. Make sure to run the command even if you are using an administrative user!) -

```
<hub> $ oc patch clusterrolebinding.rbac open-cluster-management:subscription-admin -p '{"subjects": [{"apiGroup":"rbac.authorization.k8s.io", "kind":"User", "name":"workshop-admin"}]}'
```

6. You can now deploy the policies from your forked repository to Advanced Cluster Management.

```
<hub> $ ./deploy.sh --url https://github.com/<your-github-username>/rhacm-workshop.git --branch master --path 05.Governance-Risk-Compliance/exercise/exercise-policies --namespace rhacm-policies
```

7. Make sure that the policies are deployed in the **Governance Risk and Compliance** tab in the Advanced Cluster Management for Kubernetes console.

![policies-overview](images/policies-overview.png)

8. Edit the LimitRange policy in [https://github.com/&lt;your-username>/rhacm-workshop/blob/master/05.Governance-Risk-Compliance/exercise/exercise-policies/limitrange-policy.yaml](https://github.com/cmcornejocrespo/rhacm-workshop/blob/master/05.Governance-Risk-Compliance/exercise/exercise-policies/limitrange-policy.yaml). Change the default container limit from 512Mi to 1024Mi.

9. Make sure that you commit, and push the change to your fork.

10. Log into managed cluster. Make sure that the change in GitHub was applied to the LimitRange resource.

```yaml
<managed cluster> $ oc get limitrange webserver-limit-range -o yaml -n webserver-acm
...
  limits:
  - default:
    memory: 1Gi
...
```