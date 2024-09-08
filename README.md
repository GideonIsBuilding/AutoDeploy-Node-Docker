### **Complete Guide to Dockerizing and Automating the Deployment of a Node.js Application using GitHub Actions**

---

This guide walks through the complete process of Dockerizing a Node.js application, automating the build, push, and deployment using GitHub Actions, and resolving common issues along the way. The steps include Dockerizing your application, setting up GitHub Actions to automate the process, pushing the image to Docker Hub, and deploying it to a remote server using SSH.

---

## **1. Prerequisites**

### Before starting, ensure you have the following tools and accounts:
- **Node.js and npm**: Installed locally for building the application.
- **Docker**: Installed locally for testing containerization.
- **GitHub Repository**: Where the Node.js project is hosted.
- **Docker Hub Account**: Where the Docker image will be pushed.
- **Remote Server**: Running Linux, where the Docker container will be deployed.
- **SSH Access**: Set up between GitHub and the remote server using SSH keys.

---

## **2. Dockerizing a Node.js Application**

The first step is to create a `Dockerfile` that defines how to package your Node.js application as a container.

### **2.1. Create a `Dockerfile`**
In the root of your Node.js project, create a file called `Dockerfile`. This file will describe how to build a Docker image of your application.

```Dockerfile
# Step 1: Use an official Node.js runtime as a parent image
FROM node:20

# Step 2: Set the working directory inside the container
WORKDIR /usr/src/app

# Step 3: Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Step 4: Install the app dependencies
RUN npm install

# Step 5: Copy the rest of the application files to the working directory
COPY . .

# Step 6: Expose the port the app will run on (this will be mapped in the Docker run command)
EXPOSE 3000

# Step 7: Command to run the app
CMD ["node", "index.js"]
```

### **2.2. Test Docker Image Locally**
To ensure everything works, build and run the Docker image locally.

#### Build the Docker Image:
```bash
docker build -t node-demo .
```

#### Run the Docker Container:
```bash
docker run -p 3000:3000 node-demo
```

Open your browser and visit `http://localhost:3000`. You should see your application running.

---

## **3. Automating the Build, Push, and Deployment with GitHub Actions**

Next, we’ll set up a CI/CD pipeline with GitHub Actions to automate the following:
1. Build the Docker image.
2. Push the image to Docker Hub.
3. SSH into the remote server and update the running container.

### **3.1. Create the GitHub Actions Workflow**

In your GitHub repository, create a new directory called `.github/workflows` and add a YAML file, e.g., `docker-deploy.yml`. This file will contain your GitHub Actions pipeline.

### **3.2. Example Workflow (`docker-deploy.yml`)**

```yaml
name: Docker Build and Deploy

on:
  push:
    branches:
      - main  # Trigger the workflow on pushes to the main branch

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Check out the repository
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKER_USERNAME }}/node-demo:latest

  deploy:
    runs-on: ubuntu-latest
    needs: build

    steps:
    - name: Deploy to remote server
      uses: appleboy/ssh-action@v0.1.9
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SERVER_SSH_KEY }}
        port: ${{ secrets.SERVER_SSH_PORT }}
        script: |
          docker pull ${{ secrets.DOCKER_USERNAME }}/node-demo:latest
          docker stop node-demo || true
          docker rm node-demo || true
          docker run -d --name node-demo -p 80:3000 ${{ secrets.DOCKER_USERNAME }}/node-demo:latest
```

---

## **4. Setting Up GitHub Secrets**

To securely pass sensitive information like Docker Hub credentials and SSH keys, you’ll need to add them to GitHub Secrets.

### **4.1. Adding GitHub Secrets**

1. Navigate to your GitHub repository.
2. Go to **Settings** -> **Secrets and Variables** -> **Actions** -> **New Repository Secret**.
3. Add the following secrets:
   - **DOCKER_USERNAME**: Your Docker Hub username.
   - **DOCKER_PASSWORD**: Your Docker Hub password.
   - **SERVER_HOST**: IP address of your remote server.
   - **SERVER_USER**: The username for SSH (e.g., `ubuntu`).
   - **SERVER_SSH_KEY**: Your private SSH key used for accessing the server.
   - **SERVER_SSH_PORT**: SSH port (default is `22`).

---

## **5. Setting Up Docker and SSH on the Remote Server**

### **5.1. Install Docker on the Remote Server**

If Docker is not installed on your remote server, you can do so using the following commands:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

### **5.2. Allow Non-Root User to Use Docker**

To avoid needing `sudo` for Docker commands, add your SSH user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

### **5.3. Verify Docker Permissions**

Log out and log back in, then verify that Docker commands work without `sudo`:

```bash
docker ps
```

### **5.4. Enable SSH Access for GitHub Actions**

To allow GitHub Actions to SSH into your server, you need to set up SSH keys:

1. On your local machine, generate an SSH key pair:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```

2. Copy the public key to the remote server:
   ```bash
   ssh-copy-id username@server-ip
   ```

3. Store the private key in GitHub Secrets (`SERVER_SSH_KEY`).

---

## **6. Running the GitHub Actions Pipeline**

Once everything is set up, the pipeline will be triggered every time you push code to the `main` branch of your GitHub repository. The process will be as follows:

1. **Check Out Code**: The workflow checks out your repository.
2. **Build Docker Image**: Docker Buildx builds the image based on your `Dockerfile`.
3. **Push to Docker Hub**: The Docker image is pushed to Docker Hub under the specified tag.
4. **Deploy on Remote Server**: The workflow SSHs into the remote server, pulls the latest Docker image, stops any running containers, and runs the new one.

---

## **7. Common Issues and Fixes**

### **7.1. Docker Daemon Permission Denied**

**Error**:
```
permission denied while trying to connect to the Docker daemon socket
```

**Solution**:
- Add your user to the `docker` group:
  ```bash
  sudo usermod -aG docker $USER
  ```

### **7.2. Docker Image Not Tagging Correctly**

**Error**:
```
ERROR: invalid tag "*** /node-demo:latest": invalid reference format
```

**Solution**:
- Ensure the correct formatting of the Docker tag and verify the GitHub secrets are set correctly.

---

## **8. Final Steps and Verification**

After successfully setting up and running the pipeline:
1. **Access the Application**: Visit your public server IP or domain at port 80 (or the port you've mapped) to verify that the application is running in a container.
2. **Monitor the Workflow**: GitHub Actions logs will provide real-time feedback on the progress of each step.

You now have a fully automated CI/CD pipeline that builds, pushes, and deploys your Node.js application as a Docker container using GitHub Actions.

---
