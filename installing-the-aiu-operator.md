# Installing the AIU Operator

Install the AIU Operator to use IBM AIU hardware accelerators in your OpenShift cluster.
The AIU Operator deploys the AIU device plug-in and handles node labeling so that workloads can target AIU nodes.

* You have installed the OpenShift CLI (`oc`).
* You have logged in as a user with `cluster-admin` privileges.
* You have installed the Node Feature Discovery Operator.
* The cluster hosts have one or more IBM AIU accelerators installed.

1. Create the `Namespace` custom resource (CR) for the AIU Operator. Run the following command:

   ```yaml
   oc apply -f - <<'EOF'
   apiVersion: v1
   kind: Namespace
   metadata:
     name: aiu-operator
   EOF
   ```
2. Create the `OperatorGroup` CR:

   ```yaml
   oc apply -f - <<'EOF'
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: aiu-operator
     namespace: aiu-operator
   spec:
     targetNamespaces:
     - aiu-operator
   EOF
   ```
3. Create the `Subscription` CR:

   ```yaml
   oc apply -f - <<'EOF'
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: aiu-operator
     namespace: aiu-operator
   spec:
     channel: "stable"
     installPlanApproval: Manual
     name: aiu-operator
     source: certified-operators
     sourceNamespace: openshift-marketplace
   EOF
   ```

Verify that the AIU Operator deployment is successful by running the following command:

```terminal
$ oc get pods -n aiu-operator
```

**Example output**

```terminal
NAME                                    READY   STATUS    RESTARTS   AGE
aiu-device-plugin-n9cnf                 1/1     Running   0          53d
aiu-operator-5546c5798d-6cf92           2/2     Running   2          111d
aiu-webhook-validator-b777cfddc-9sjg2   1/1     Running   1          111d
```

**Additional resources**

* [AIU Operator](https://catalog.redhat.com/en/software/container-stacks/detail/66b72ba7d4e6fd9bd48d99ea#overview)