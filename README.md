# vitejs--ideas

Using the environment variable `VITE_BASE_API_URL` is utilized correctly, allowing the dynamic API URL replacement via ConfigMap at runtime.

### Step-by-Step Implementation

#### Step 1: Use `.env` Files for Default Configuration

Create environment variable files for different environments.

**.env**
```env
VITE_BASE_API_URL=https://default-url.example.com
```

**.env.production**
```env
VITE_BASE_API_URL=https://production-url.example.com
```

**.env.development**
```env
VITE_BASE_API_URL=https://development-url.example.com
```

#### Step 2: Fetch Configuration in `main.js`

Modify `main.js` to use the environment variable with a fallback to the dynamically loaded configuration.

```javascript
const loadConfig = async () => {
  try {
    const response = await fetch('/config.json');
    const config = await response.json();
    return config.apiUrl || import.meta.env.VITE_BASE_API_URL;
  } catch (error) {
    console.error('Failed to load config:', error);
    return import.meta.env.VITE_BASE_API_URL;
  }
};

const initApp = async () => {
  const apiUrl = await loadConfig();
  console.log('API URL:', apiUrl);

  // Example usage in your API call
  const response = await fetch(`${apiUrl}/computers`);
  const data = await response.json();
  computers.value = convertDateTime(data["results"]);
};

initApp();
```

#### Step 3: Modify `index.html` to Include a Placeholder

Include a placeholder in `index.html` that will be replaced by Nginx.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Vite App</title>
  <script>
    window.config = {
      apiUrl: '__API_URL__'
    };
  </script>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.js"></script>
</body>
</html>
```

#### Step 4: Create a Configuration File

Create a `config.json` file in the `public` directory of your Vite project.

```json
{
  "apiUrl": "https://default-url.example.com"
}
```

#### Step 5: Create a ConfigMap in Kubernetes

Create a ConfigMap that contains your API URL configuration.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-sbox
data:
  apiUrl: "https://sbox-api.example.com"
```

Apply the ConfigMap:

```sh
kubectl apply -f configmap-sbox.yaml
```

#### Step 6: Create an Entrypoint Script

Create an entrypoint script (`start-nginx.sh`) to replace the placeholder in the configuration with the actual API URL from the ConfigMap.

```sh
#!/bin/sh

# Validate the Nginx configuration
echo "Checking Nginx configuration..."
nginx -t

# Check if nginx -t command was successful
if [ $? -eq 0 ]; then
  echo "Nginx configuration is valid."
  
  # Read the API URL from the ConfigMap
  API_URL=$(cat /etc/config/apiUrl)
  
  # Replace the placeholder in index.html
  sed -i "s|__API_URL__|$API_URL|g" /usr/share/nginx/html/index.html
  
  echo "Starting Nginx..."
  exec nginx -g 'daemon off;'
else
  echo "Nginx configuration is invalid."
  exit 1
fi
```

Make sure the script has execute permissions:

```sh
chmod +x start-nginx.sh
```

#### Step 7: Update Dockerfile

Ensure your Dockerfile copies the updated `start-nginx.sh` script and sets it as the entrypoint:

```Dockerfile
# Build stage
FROM node:20 as build-stage
WORKDIR /build
COPY . .
RUN npm --version && \
    node --version && \
    npm config set registry "https://artifactory.demo.build/artifactory/api/npm/npm" && \
    npm ci --force && \
    npm run build && \
    chmod +x ./start-nginx.sh

# Production stage
FROM nginx:1.25 as production-stage

# Copy the built files from the build stage
COPY --from=build-stage /build/dist /usr/share/nginx/html

# Copy the custom Nginx configuration file
COPY --from=build-stage /build/nginx.conf /etc/nginx/nginx.conf

# Copy the entrypoint script to replace the placeholder in index.html
COPY --from=build-stage /build/start-nginx.sh /start-nginx.sh

# Set the entrypoint script as executable
RUN chmod +x /start-nginx.sh

EXPOSE 80

# Set the entrypoint to the start-nginx.sh script
ENTRYPOINT ["/start-nginx.sh"]
```

#### Step 8: Build and Push Docker Image

Build your Docker image and push it to your container registry.

```sh
docker build -t your-docker-repo/vite-app:latest .
docker push your-docker-repo/vite-app:latest
```

#### Step 9: Create Kubernetes Deployment for Each Environment

Here's an example Kubernetes deployment for the `sbox` environment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vite-app-sbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vite-app
  template:
    metadata:
      labels:
        app: vite-app
    spec:
      containers:
        - name: vite-app
          image: your-docker-repo/vite-app:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: app-config-sbox
```

Apply the deployment:

```sh
kubectl apply -f deployment-sbox.yaml
```

### Summary

1. **Use `.env` files** for default configuration.
2. **Modify `main.js`** to use the environment variable with a fallback to the dynamically loaded configuration.
3. **Modify `index.html`** to include a placeholder for the API URL.
4. **Create a `config.json`** file for default configuration.
5. **Create a ConfigMap** in Kubernetes to store the API URL.
6. **Create an entrypoint script** to replace the placeholder with the actual API URL from the ConfigMap.
7. **Update the Dockerfile** to copy the entrypoint script and set it as the entrypoint.
8. **Build and push the Docker image** to your container registry.
9. **Create and apply Kubernetes deployments** for each environment, mounting the appropriate ConfigMap.
10. **Test the deployment** to ensure the API URL is correctly set at runtime for each environment.

This combined approach allows you to use Vite's environment variables while still dynamically overriding the API URL at runtime using a ConfigMap in Kubernetes. This provides flexibility to change the configuration without rebuilding the Docker image and ensures the application can adapt to different environments seamlessly.
