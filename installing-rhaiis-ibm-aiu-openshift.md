# Deploying Red Hat AI Inference Server and inference serving using IBM AIU AI accelerators

Deploy a language model with OpenShift Container Platform and AIU AI accelerators by configuring secrets, persistent storage, and a deployment custom resource (CR) that pulls the model from Hugging Face and uses Red Hat AI Inference Server to inference serve the model.

* You have installed the OpenShift CLI (`oc`).
* You have logged in as a user with `cluster-admin` privileges.
* You have installed NFD and the AIU Operator.

1. Create the `Secret` custom resource (CR) for the Hugging Face token.
The cluster uses the `Secret` CR to pull models from Hugging Face.
   1. Set the `HF_TOKEN` variable using the token you set in [Hugging Face](https://huggingface.co/settings/tokens).

      ```terminal
      $ HF_TOKEN=<your_huggingface_token>
      ```
   2. Set the cluster namespace to match where you deployed the Red Hat AI Inference Server image, for example:

      ```terminal
      $ NAMESPACE=rhaiis-aiu
      ```
   3. Create the `Secret` CR in the cluster:

      ```terminal
      $ oc create secret generic hf-secret --from-literal=HF_TOKEN=$HF_TOKEN -n $NAMESPACE
      ```
2. Create the Docker secret so that the cluster can download the Red Hat AI Inference Server image from the container registry. For example, to create a `Secret` CR that contains the contents of your local `~/.docker/config.json` file, run the following command:

   ```terminal
   oc create secret generic docker-secret --from-file=.dockercfg=$HOME/.docker/config.json --type=kubernetes.io/dockercfg -n rhaiis-aiu
   ```
3. Create a `PersistentVolumeClaim` (`PVC`) custom resource (CR) and apply it in the cluster.
The following example `PVC` CR uses a default IBM VPC Block persistence volume.
You use the `PVC` as the location where you store the models that you download.

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: models
     namespace: rhaiis-aiu
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 20Gi
     storageClassName: ibmc-vpc-block-10iops-tier
   ```

> [!NOTE]
> Configuring cluster storage to meet your requirements is outside the scope of this procedure.
> For more detailed information, see [Configuring persistent storage](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/storage/configuring-persistent-storage).

4. Create a `Deployment` custom resource (CR) that pulls the model from Hugging Face and deploys the Red Hat AI Inference Server container.
Reference the following example `Deployment` CR, which uses AI Inference Server to serve a Granite model on a CUDA accelerator.

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     annotations: {}
     name: vllm-granite-8b-aiu
     # The value that you set for `metadata.namespace` must match the namespace where you configure the Hugging Face `Secret` CR.
     namespace: rhaiis-aiu
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: vllm-granite-8b-aiu
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: vllm-granite-8b-aiu
       spec:
         containers:
           - args:
               - ibm-granite/granite-3.3-8b-instruct
               - --tokenizer
               - ibm-granite/granite-3.3-8b-instruct
               - --port
               - "8001"
               - --host
               - 0.0.0.0
               - --download-dir
               - /models
               - --max-model-len
               - "4096"
               - --max-num-seqs
               - "4"
               - --tensor-parallel-size
               - "1"
             env:
               - name: HUGGING_FACE_HUB_TOKEN
                 valueFrom:
                   secretKeyRef:
                     key: hf_token
                     name: huggingface-secret
               - name: HF_HOME
                 value: /models
               # Setting `VLLM_SPYRE_USE_CB` enables continuous batching for inference serving.
               - name: VLLM_SPYRE_USE_CB 
                 value: "1"
             image: registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.1
             imagePullPolicy: Always
             name: vllm-runner
             ports:
               - containerPort: 8001
                 protocol: TCP
             resources:
               limits:
                 ibm.com/aiu_pf: "1"
               requests:
                 ibm.com/aiu_pf: "1"
             securityContext:
               capabilities:
                 add:
                   - SYS_ADMIN
               privileged: true
             terminationMessagePath: /dev/termination-log
             terminationMessagePolicy: File
             volumeMounts:
               - mountPath: /dev/shm
                 name: dshm
               - mountPath: /models
                 name: model-storage
         dnsPolicy: ClusterFirst
         nodeSelector:
           kubernetes.io/hostname: spyre.example.com
         restartPolicy: Always
         schedulerName: aiu-scheduler
         securityContext:
           fsGroup: 2000
           runAsUser: 2000
         serviceAccount: vllm-sa
         serviceAccountName: vllm-sa
         terminationGracePeriodSeconds: 30
         volumes:
           - emptyDir:
               medium: Memory
               sizeLimit: 32Gi
             name: dshm
           - name: model-storage
             persistentVolumeClaim:
               claimName: vllm-models-pvc
   ```

5. Optional: Watch the deployment and ensure that it succeeds:

   ```terminal
   $ oc get deployment -n rhaiis-aiu --watch
   ```

   **Example output**

   ```terminal
   NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
   vllm-granite-8b-aiu   0/1     1            0           2s
   vllm-granite-8b-aiu   1/1     1            1           14s
   ```

6. Create a `Service` CR for the model inference. For example:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: vllm-granite-8b-aiu
     namespace: rhaiis-aiu
   spec:
     selector:
       app: vllm-granite-8b-aiu
     ports:
       - protocol: TCP
         port: 80
         targetPort: 8001
   ```
7. Optional: Create a `Route` CR to enable public access to the model. For example:

   ```yaml
   apiVersion: route.openshift.io/v1
   kind: Route
   metadata:
     name: vllm-granite-8b-aiu
     namespace: rhaiis-aiu
   spec:
     to:
       kind: Service
       name: vllm-granite-8b-aiu
     port:
       targetPort: 80
   ```
8. Get the URL for the exposed route. Run the following command:

   ```terminal
   $ oc get route vllm-granite-8b-aiu -n rhaiis-aiu -o jsonpath='{.spec.host}'
   ```

   **Example output**

   ```terminal
   vllm-granite-8b-aiu-rhaiis-aiu.apps.example.com
   ```

**Verification**

Ensure that the deployment is successful by querying the model.
Run the following command:

```terminal
curl -X POST http://vllm-granite-8b-aiu-rhaiis-aiu.apps.example.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ibm-granite/granite-3.3-8b-instruct",
    "messages": [{"role": "user", "content": "What is AI?"}],
    "temperature": 0.1
  }'
```
