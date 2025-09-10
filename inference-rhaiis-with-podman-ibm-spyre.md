# Serving and inferencing with Podman using IBM Spyre AI accelerators

Serve and inference a large language model with Red Hat AI Inference Server running on IBM Spyre AI accelerators.

* You have installed Podman or Docker

* You are logged in as a user with root privileges

* You have access to `registry.redhat.io` and have logged in

* You have a Hugging Face account and have generated a Hugging Face access token

* You have access to a Linux server with data center grade IBM Spyre AI accelerators installed

<!-- Standard RH tech preview statement -->
> [!IMPORTANT]
> Deploying Red Hat AI Inference Server on IBM Spyre AI accelerators is a Technology Preview feature only. Technology Preview features are not supported with Red Hat production service level agreements (SLAs) and might not be functionally complete. Red Hat does not recommend using them in production. These features provide early access to upcoming product features, enabling customers to test functionality and provide feedback during the development process. For more information about the support scope of Red Hat Technology Preview features, see link:https://access.redhat.com/support/offerings/techpreview/[Technology Preview Features Support Scope].

1. Open a terminal on your server host, and log in to `registry.redhat.io`:

   ```bash
   $ podman login registry.redhat.io
   ```
2. Pull the IBM Spyre image by running the following command:

   ```bash
   $ podman pull registry.redhat.io/rhaiis/vllm-spyre-rhel9:3.2.2
   ```
3. Create a volume that you will mount into the container.
Adjust the container permissions so that the container can use it.

   ```bash
   $ mkdir ./.cache/rhaiis
   ```

   ```bash
   $ chmod g+rwX ./.cache/rhaiis
   ```
4. Add the `HF_TOKEN` Hugging Face token to the `private.env` file.

   ```bash
   $ echo "export HF_TOKEN=<huggingface_token>" > private.env
   ```
5. Append the `HF_HOME` variable to the `private.env` file.

   ```bash
   $ echo "export HF_HOME=./.cache/rhaiis" >> private.env
   ```

   Source the `private.env` file.

   ```bash
   $ source private.env
   ```
6. Optional: Verify that the container can access the underlying host Spyre AI accelerator hardware.
Run the following command:

   ```bash
   $ podman run -it --device=/dev/vfio -v ./.cache/rhaiis:/opt/app-root/src/.cache -p 8000:8000 --entrypoint="" registry.redhat.io/rhaiis/vllm-spyre-rhel9:3.2.2 /opt/sentient/bin/aiu-query-devices
   ```

   **Example output**

   ```bash
   Detected 1 AIUs
   AIU   0 at 0000:29:00.0
   Topology File: "/etc/aiu/topo.json"
                     |  0   0000:29:00.0 |
   ------------------+-------------------+-
    0   0000:29:00.0 |        ---        |
   ------------------+-------------------+-
   Legend:
   ---        = Self
   P2P (R)DMA = Connection using the P2P DMA protocol traversing the PCIe network
   Host DMA   = Connection using the Host DMA protocol using host shared memory
   ```
7. If your system has SELinux enabled, configure SELinux to allow device access:

   ```bash
   $ sudo setsebool -P container_use_devices 1
   ```
8. Start the AI Inference Server container image.
   1. Start the container:

      ```bash
      podman run --rm -it \
      --device /dev/kfd --device /dev/dri \
      --security-opt=label=disable \ # (a)
      --group-add keep-groups \
      --shm-size=4GB -p 8000:8000 \ # (b)
      --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
      --env "HF_HUB_OFFLINE=0" \
      --env=VLLM_NO_USAGE_STATS=1 \
      -v ./.cache/rhaiis:/opt/app-root/src/.cache \
      registry.redhat.io/rhaiis/vllm-spyre-rhel9:3.2.2 \
      --model ibm-granite/granite-3.3-8b-instruct \
      --tensor-parallel-size 1 # (c)
      ```
      1. `--security-opt=label=disable` prevents SELinux from relabeling files in the volume mount. If you choose not to use this argument, your container might not successfully run.
      2. If you experience an issue with shared memory, increase `--shm-size` to `8GB`.
      3. Set `--tensor-parallel-size` to match the number of AI accelerators when running the AI Inference Server container on multiple AI accelerators.

**Verification**

In a separate tab in your terminal, make a request to your model with the API.

```bash
curl -X POST -H "Content-Type: application/json" -d '{
    "prompt": "What is the capital of France?",
    "max_tokens": 50
}' http://<your_server_ip>:8000/v1/completions | jq
```

**Example output**

```json
{
    "id": "cmpl-b44aeda1d5a4485c9cb9ed4a13072fca",
    "object": "text_completion",
    "created": 1746555521,
    "model": "ibm-granite/granite-3.3-8b-instruct",
    "choices": [
        {
            "index": 0,
            "text": " Paris.\nThe capital of France is Paris.",
            "logprobs": null,
            "finish_reason": "stop",
            "stop_reason": null,
            "prompt_logprobs": null
        }
    ],
    "usage": {
        "prompt_tokens": 8,
        "total_tokens": 18,
        "completion_tokens": 10,
        "prompt_tokens_details": null
    }
}
```
