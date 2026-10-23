## Step 1: Rebuild your image with a new tag
From the directory containing the correct Dockerfile, run:
```
docker build --no-cache -t <image-name>:v2 .
```

Make sure . is the directory containing the Dockerfile and the intended source files.

## Step 2: Verify the code inside the image
Before touching Kubernetes, run:
```
docker run --rm --entrypoint cat <image-name>:v2 /app/src/app.js
```

Check that the output contains your new imports and all routes. e.g.

```javascript
import { createPod } from "./kubernetes/pod.js";
import { createService } from "./kubernetes/service.js";
import { v7 as uuid } from "uuid";
```

It should also contain the updated endpoints.

If these imports aren't present, stop here: the issue is in the Docker build or source-file paths, not Kubernetes.

## Step 3: Update your Deployment
eg- Your Deployment currently uses <image-name>:latest. Change it to <image-name>:v2:
```
kubectl set image deployment/<deployment-name> <container-name>=<image-name>:v2
```

Here, <container-name> on the left is your container name, which appears to match your image name. Verify it if the command reports that the container wasn't found.
command:
```
kubectl get deployment <deployment-name> -o=jsonpath="{.spec.template.spec.containers[*].name}"
```

## Step 4: Wait for Kubernetes to update the Pod
```
kubectl rollout status deployment/<deployment-name>
```

Then check the image:
```
kubectl get deployment <deployment-name> -o=jsonpath="{.spec.template.spec.containers[*].image}"
```

Expected output:
<image-name>:v2



## Step 5: Verify the new application code
```
kubectl get pods
```

Then inspect the file in the new Pod:
```
kubectl exec deploy/<deployment-name> -- cat /app/src/app.js
```

You should now see the new imports and all routes.


#### The problem was that Kubernetes was still running the old image, which didn't contain your new code. By rebuilding the image with a new tag and updating the Deployment, you ensured that Kubernetes pulled the new image and ran it in the Pod.