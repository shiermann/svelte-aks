# How to deploy a Svelte project on AKS with an Ingress NGINX Controller
## Requirements:
1. You successfully installed [Docker Desktop](https://www.docker.com/products/docker-desktop/) on your PC.
2. You enabled [Kubernetes](https://docs.docker.com/desktop/kubernetes/) on Docker Desktop (Settings -> Kubernetes -> Enable Kubernetes).
3. You have an [Azure](https://portal.azure.com/) account.
4. You installed the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) on your device.

## Step 1: Create a Svelte project
Follow the guide on the developer page[^1]:

```
npm create svelte@latest myapp
cd myapp
npm install
npm run dev
```

## Step 2: Change the adapter type
When you first build the Svelte project, it provides you with the `@sveltejs-adapter-auto` in the `svelte.config.js` file. <br>
According to the developers page[^2] it's not useful for Kubernetes configuration (because the right adapter is not listed in the auto-feature).<br>
You have to choose the `@sveltejs-adapter-node` adapter[^3]. 
To do this install the adapter in the command line with the following command:

```
npm i -D @sveltejs/adapter-node
```

To change the `adapter-auto`, go to your `svelte.config.js` file and change the adapter to this:

```js
import adapter from '@sveltejs/adapter-node';

export default {
    kit: {
        adapter: adapter({
            out: 'build'
        })
    }
}
```

The `out: 'build'` is an important (but optional) feature. Without it, no `build` folder would be created.<br>
The last step is to build the project, so a `build`-folder will be created in the root of your project:

```
npm run build
```

## Step 3: Create a Dockerfile
To deploy the project to AKS, you first have to dockerize it. Create a new file in your root folder called `Dockerfile`.

```Dockerfile
# Choose the base image
FROM node:18

# Set the working directory
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the source code excluding node_modules
COPY . ./

# Install the correct esbuild platform (optional, but necessary to avoid errors)
RUN npm uninstall esbuild
RUN npm install esbuild

# Build the Svelte application
RUN npm run build

# Copy the contents of the build directory to the Docker image
COPY ./build ./build

# Expose the port the app will run on
ENV HOST=0.0.0.0
EXPOSE 80 3000

# Start the application
CMD ["node", "build"]
```

Build the image with the following command:
```
docker build -t svelte:v1 .
```
⚠️ CAUTION for Mac Users: ⚠️ <br>
Because we'll run this on Azure, you have to check your configuration on the server side! Usually AKS is using the "Standard_DS2_v2" node size[^6]. This means, that the server is running on x86-based chips! If you build your image on an ARM-based device and push it later to ACR (see Step 7), it won't work! <br>
<br>
Run the image for testing:
```
docker run -p 3000:3000 svelte:v1
```
Open it on [http://localhost:3000/](http://localhost:3000/) to see the Svelte project live.<br>
Now we successfully build an image and run it as an container.

## Step 4: Create an Ingress-NGINX-Controller
Now we get Kubernetes in it. Let's create an Ingress-NGINX-Controller to manage and route traffic for ingress resources, which define how traffic should be forwarded to services running in the cluster. <br><br>
You can install the Ingress-NGINX-Controller via `kubectl`:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```
<br>

Or you can install it via `helm`[^4]: <br>
If you don't have `helm` install it via Chocolatey[^7] on Windows or Homebrew[^8] on Mac. <br>
And then create the Ingress-NGINX-Controller with it:
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

For more infos look up the [Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start).

## Step 5: Create a Deployment and Service
For a working Kubernetes environment you always need two things. The deployment, where you specify what image Kubernetes should use and how many replicas it should make, and the service to specify the port it should use. Create a new file called `myapp.yaml` and copy and paste these lines:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: svelte:v1
        ports:
        - containerPort: 3000

---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: 3000
  type: ClusterIP

```

To add these specifications in your Kubernetes environment, run `kubectl apply` in the terminal:
```
kubectl apply -f myapp.yaml -n ingress-nginx
```
`-f` stands for file (which file should I use?) <br>
`-n` stands for namespace (which namespace should I use?)


## Step 6: Create a custom Ingress Configuration
You've successfully installed the Ingress-NGINX-Controller but it still has no clue where our deployments and services are. Also notice that the Ingress-NGINX-Controller is in a specific namespace called `ingress-nginx`. Our deployments and services are not, so we have to tell him where to find it.
Create a new YAML-File called `ingress.yaml` for your custom ingress configuration:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```
To add this specification in your Kubernetes environment, run `kubectl apply` in the terminal:
```
kubectl apply -f ingress.yaml -n ingress-nginx
```
To check if Step 5 and 6 succeeded, look on the current configuration of your Kubernetes environment:
```
kubectl get all -n ingress-nginx
```

## Step 7: Upload it to ACR (Azure Content Registry)
ACR is basically where Azure stores its images. We will also have access to it from the cluster. <br>
If you haven't created an ACR already, you can do that with[^5]
```
az acr create --resource-group <resource-group-name> --name <registry-name> --sku Basic
```
Remember that registry names are unique and if someone else took that name already in your region, you can't use it a second time.
<br>
To login to that registry use the following command:
```
az acr login -n <registry-name>
```
Before you upload anything to Azure, you have to tag your image accourdingly. Find out what the name of your image is with `docker image ls` and tag it for your ACR:
```
docker tag svelte <registry-name>.azurecr.io/svelte:v1
```
`svelte` is the current image, stored in your Docker Desktop. If you have a tag on it use `svelte:v1` for example. Without it, it uses the `latest` tag.
<br><br> 
After you get the confirmation "Login Succeeded", you can push your current image via
```
docker push <registry-name>.azurecr.io/svelte:v1
```

## Step 8: Change the deployment accourdingly to AKS
As you have seen in the step before, we've named the image differently (from `svelte` to `<registry-name>.azurecr.io/svelte:v1`). <br>
You also have to tell Kubernetes, that the name is different now. Change `myapp.yaml` accourdingly:
```yaml
containers:
    - name: myapp
    image: <registry-name>.azurecr.io/svelte:v1
```

## Step 9: Generate a ACR secret
To prevent an `401 Unauthorized` error, you have to generate an ACR secret. That helps Kubernetes to get authorization to your registry. <br>
To generate a secret, give the following prompt in your terminal[^9]:
```yaml
kubectl create secret docker-registry acr-secret \
    --namespace ingress-nginx \
    --docker-server=<registry-name>.azurecr.io \
    --docker-username=<service-principal-ID> \
    --docker-password=<service-principal-password>
```
Learn how to create a Service Principal ID and password here: [Manually create a service principal](https://learn.microsoft.com/en-us/azure/aks/kubernetes-service-principal?tabs=azure-cli#manually-create-a-service-principal)
<br><br>

After that prompt you should have a file called `acr-secret.yaml` with similar content like this:
```
apiVersion: v1
data:
  .dockerconfigjson: [...]
kind: Secret
metadata:
  creationTimestamp: "2023-03-17T12:31:55Z"
  name: acr-secret
  namespace: ingress-nginx
  resourceVersion: "1234568"
  uid: [...]
type: kubernetes.io/dockerconfigjson
```

Change `myapp.yaml` accourdingly:
```yaml
spec:
  imagePullSecrets:
  - name: acr-secret
```

Now apply your changes of `myapp.yaml` with:
```
kubectl apply -f myapp.yaml -n ingress-nginx
```

## Step 10: It works!
Now everything should run! You can check that with:
```
kubectl get all -n ingress-nginx
```
You should see a pod with the name `myapp` with the state `Running`. Now let's check if it is also running on AKS.
In the section `services` you should see an External-IP in the `ingress-nginx-controller`.
If you call that IP on your browser, you should see a running Svelte website. Great, you're done!


# Sources
[^1]: https://svelte.dev/
[^2]: https://kit.svelte.dev/docs/adapter-auto
[^3]: https://kit.svelte.dev/docs/adapter-node
[^4]: https://kubernetes.github.io/ingress-nginx/deploy/#quick-start
[^5]: https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-create
[^6]: https://learn.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series
[^7]: https://docs.chocolatey.org/en-us/choco/setup#more-install-options
[^8]: https://brew.sh/
[^9]: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auth-kubernetes#create-an-image-pull-secret
