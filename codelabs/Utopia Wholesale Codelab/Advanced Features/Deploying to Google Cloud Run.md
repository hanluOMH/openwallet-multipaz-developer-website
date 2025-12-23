# Deploying Multipaz Servers to Google Cloud Run

## **Before You Begin**

This tutorial guides you through deploying the Multipaz OpenID4VCI Server , Records Server and Verifier Server to Google Cloud Run. These servers work together to provide a complete credential issuance system where the OpenID4VCI server issues credentials based on identity data stored in the Records server. The deployment methods for the three servers are similar. Here, I’ll use the OpenID4VCI and Records servers as examples. The verifier server setup is similar to the OpenID4VCI flow.

---

## **Prerequisites**

* A Google Cloud Platform (GCP) account with billing enabled
* Google Cloud SDK (`gcloud`) installed and configured
* Docker installed (for local testing, though Cloud Build handles this)
* Basic understanding of containerization and cloud deployment
* Familiarity with the Multipaz project structure

---

## **What You'll Learn**

* How to configure servers for Cloud Run deployment
* How to build fat JAR files with all dependencies
* How to create Docker images for the servers
* How to deploy servers to Google Cloud Run
* How to connect the OpenID4VCI server to the Records server

---

## **What You'll Need**

* Access to the `identity-credential` repository
* Google Cloud project with Cloud Run API enabled
* Terminal/command line access
* Java 17+ (for building the JAR files)

---

## **Configuration Changes**

In [this commit](https://github.com/hanluOMH/identity-credential/commit/bd51ad7dc8fd128491da7dff709e7f36eda30330) from the [`deploy_cloud` branch](https://github.com/hanluOMH/identity-credential/tree/deploy_cloud), several configuration changes were made to prepare the servers for Cloud Run deployment:

### **Changes to `multipaz-openid4vci-server`**

The following changes were made to `multipaz-openid4vci-server/src/main/resources/resources/default_configuration.json`:

1. **Port Configuration**: Changed `server_port` from `8007` to `8080`
   - Cloud Run expects services to listen on port 8080 by default
   - The PORT environment variable can override this if needed

2. **Base URL Configuration**: Added `base_url` pointing to the Cloud Run deployment URL
   ```json
   "base_url": "https://multipaz-openid4vci-server-971523157550.us-central1.run.app"
   ```
   - **Important**: Replace this with your own Cloud Run deployment URL after deploying the server
   - This ensures all redirects and generated URLs use the correct public URL
   - Without this, the server would generate URLs with `localhost:8080`

3. **System of Record URL**: Added `system_of_record_url` pointing to the Records server
   ```json
   "system_of_record_url": "https://multipaz-records-server-971523157550.us-central1.run.app"
   ```
   - **Important**: Replace this with your own Records server Cloud Run URL after deploying it
   - This connects the OpenID4VCI server to the Records server
   - The OpenID4VCI server uses this to retrieve identity data when issuing credentials

### **Changes to `multipaz-records-server`**

The following changes were made to `multipaz-records-server/src/main/resources/resources/default_configuration.json`:

1. **Port Configuration**: Changed `server_port` from `8004` to `8080`
   - Consistent with Cloud Run's expected port

2. **Base URL Configuration**: Added `base_url` pointing to the Cloud Run deployment URL
   ```json
   "base_url": "https://multipaz-records-server-971523157550.us-central1.run.app"
   ```
   - **Important**: Replace this with your own Cloud Run deployment URL after deploying the server
   - Ensures proper redirects when accessing the root URL

3. **Admin Password**: Added fixed `admin_password` set to `"multipaz-records"`
   - Previously, the server generated a random password on each startup
   - This makes the password predictable and manageable for deployment

---

## **Deployment Steps**

### **Step 1: Create Dockerfile**

Create a Dockerfile for the OpenID4VCI server in the `multipaz-openid4vci-server` directory:

```bash
mkdir -p multipaz-openid4vci-server && cat > multipaz-openid4vci-server/Dockerfile <<'EOF'
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY app.jar app.jar
# Cloud Run sets PORT environment variable; pass it to the server using -param format
CMD ["sh","-c","java -jar app.jar -param server_port=${PORT:-8080}"]
EOF
```

**Explanation:**
- `FROM eclipse-temurin:17-jre`: Uses Java 17 JRE base image (matches the server's Java version requirement)
- `WORKDIR /app`: Sets the working directory inside the container
- `COPY app.jar app.jar`: Copies the fat JAR file into the container
- `CMD`: Runs the JAR with the PORT environment variable passed as a parameter
  - Note: Uses `-param server_port=` format (not `--server.port=`) because this is a Ktor server, not Spring Boot
  - `${PORT:-8080}` uses Cloud Run's PORT environment variable, defaulting to 8080 if not set

### **Step 2: Build the Fat JAR**

Build the fat JAR file that includes all dependencies:

```bash
./gradlew :multipaz-openid4vci-server:buildFatJar
```

**What this does:**
- Compiles the Kotlin code
- Packages all dependencies into a single JAR file
- Includes the configuration files (including `default_configuration.json`)
- Outputs the JAR to `multipaz-openid4vci-server/build/libs/app.jar`

**Note:** The Ktor Gradle plugin creates a fat JAR named `app.jar` by default.

### **Step 3: Prepare for Cloud Build**

Copy the JAR file to the deployment directory:

```bash
# Rename and copy the JAR file to the multipaz-openid4vci-server directory
cp multipaz-openid4vci-server/build/libs/app.jar multipaz-openid4vci-server/app.jar
```

**Directory structure should look like:**
```
multipaz-openid4vci-server/
├── Dockerfile
└── app.jar
```

### **Step 4: Build Docker Image**

Build the Docker image using Google Cloud Build:

```bash
gcloud builds submit multipaz-openid4vci-server --tag gcr.io/engaged-list-481123-n3/multipaz-openid4vci-server
```

**Explanation:**
- `gcloud builds submit`: Submits a build to Google Cloud Build
- `multipaz-openid4vci-server`: The directory containing the Dockerfile and JAR file
- `--tag`: Tags the resulting image with the specified name
- `gcr.io/engaged-list-481123-n3/`: Your GCP project's Container Registry path
  - Replace `engaged-list-481123-n3` with your actual GCP project ID

**What happens:**
1. Cloud Build reads the Dockerfile
2. Builds a Docker image containing the JAR file
3. Pushes the image to Google Container Registry
4. The image is ready to be deployed to Cloud Run

### **Step 5: Deploy to Cloud Run**

Deploy the containerized application to Cloud Run:

```bash
gcloud run deploy multipaz-openid4vci-server \
  --image gcr.io/engaged-list-481123-n3/multipaz-openid4vci-server \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

**Parameters explained:**
- `--image`: Specifies the Docker image to deploy (from Step 4)
- `--platform managed`: Uses Google's fully managed Cloud Run platform
- `--region us-central1`: Deploys to the specified region
  - Choose a region close to your users for better performance
- `--allow-unauthenticated`: Makes the service publicly accessible
  - Remove this flag if you want to require authentication

**After deployment:**
- Cloud Run will provide a URL like: `https://multipaz-openid4vci-server-XXXXX.us-central1.run.app`
- Update the `base_url` in your configuration file with this URL
- Rebuild and redeploy if you need to update the base URL

---

## **Deploying the Records Server**

Follow the same steps for `multipaz-records-server`:

1. Create a similar Dockerfile in `multipaz-records-server/`
2. Build the fat JAR: `./gradlew :multipaz-records-server:buildFatJar`
3. Copy the JAR: `cp multipaz-records-server/build/libs/app.jar multipaz-records-server/app.jar`
4. **Create environment variables file**: Create an `env-vars.yaml` file in the `multipaz-records-server/` directory with the `system_of_record_jwk` variable:
   
   ```bash
   cat > multipaz-records-server/env-vars.yaml <<'EOF'
   system_of_record_jwk: ''
   EOF
   ```
   
   **Note:** This environment variable contains the JWK (JSON Web Key) used for authenticating with the Records server. Make sure to use your own JWK if you have one, or use the example provided here.
5. Build the image: `gcloud builds submit multipaz-records-server --tag gcr.io/YOUR-PROJECT-ID/multipaz-records-server`
6. Deploy with environment variables: 
   ```bash
   gcloud run deploy multipaz-records-server \
     --image gcr.io/YOUR-PROJECT-ID/multipaz-records-server \
     --platform managed \
     --region us-central1 \
     --env-vars-file multipaz-records-server/env-vars.yaml \
     --allow-unauthenticated
   ```

**Important:** Deploy the Records server first, then update the `system_of_record_url` in the OpenID4VCI server configuration before deploying it.

---

## **Verifying Deployment**

After deployment, verify both servers are working:

1. **Records Server**: Visit `https://YOUR-RECORDS-SERVER-URL`
   - You should see the State Registry interface
   - Should redirect to `/index.html` without redirecting to localhost
   - Login with the admin password: `multipaz-records`

2. **OpenID4VCI Server**: Visit `https://YOUR-OPENID4VCI-SERVER-URL`
   - Should redirect to `/index.html` without redirecting to localhost

---

## **Troubleshooting**

### **Issue: Server redirects to localhost**

**Solution:** Ensure `base_url` is set correctly in `default_configuration.json` and rebuild the JAR.

### **Issue: Port binding errors**

**Solution:** The Dockerfile correctly passes the PORT environment variable. Ensure Cloud Run is setting the PORT variable (it does by default).

### **Issue: Cannot connect to Records server**

**Solution:** 
- Verify the Records server is deployed and accessible
- Check that `system_of_record_url` in OpenID4VCI server matches the Records server URL
- Ensure both servers are in the same region or have proper network access

### **Issue: Build fails with "app.jar not found"**

**Solution:** Ensure you've copied the JAR file to the deployment directory before running `gcloud builds submit`.

---

## **Next Steps**

* Monitor server logs in Cloud Run console
* Set up custom domains for your deployments
* Configure environment variables for different deployment environments
* Set up CI/CD pipelines for automated deployments
* Configure database connections if using persistent storage

---

## **Summary**

In this tutorial, you learned:

1. **Configuration changes** made in [this commit](https://github.com/hanluOMH/identity-credential/commit/bd51ad7dc8fd128491da7dff709e7f36eda30330) to prepare servers for Cloud Run
2. **How to build fat JARs** that include all dependencies
3. **How to create Dockerfiles** for containerizing the servers
4. **How to build and deploy** to Google Cloud Run
5. **How to connect** the OpenID4VCI server to the Records server

The servers are now deployed and ready to issue verifiable credentials based on identity data stored in the Records server.

