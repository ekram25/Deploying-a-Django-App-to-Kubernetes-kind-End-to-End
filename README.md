# Deploying a Django App to Kubernetes (kind) — End-to-End
This project showcases the end-to-end deployment of a Django application on Kubernetes using kind.
The app is containerized with Docker, pushed to Docker Hub, and deployed using Namespaces, Deployments, and Services.
Local access is enabled via kubectl port-forward, with optional in-cluster MySQL integration for database support.
The project highlights containerization, Kubernetes fundamentals, rolling updates, and troubleshooting best practices.

## 1) Prerequisites & What We’re Building

- **Goal**: Containerize a Django application, publish the image to Docker Hub, and deploy it on a local Kubernetes cluster using kind (Kubernetes-in-Docker).

- **Kubernetes Setup**: Create a dedicated Namespace, a Deployment to manage Django pods, and a ClusterIP Service for internal networking.

- **Local Access**: Use kubectl port-forward to route traffic from the host machine to the in-cluster Service.

- **Learning Focus**: Gain hands-on experience with Kubernetes primitives (Namespaces, Deployments, Services) in a cost-free local setup.

- **Scalability**: The same manifests can be applied to managed clusters (EKS, GKE, AKS) with minimal adjustments.
- 
# 2) Install Docker
Docker is required to build and run container images. Make sure Docker Desktop or Docker Engine is installed and running.
Verify installation with docker version and docker info — both should return successfully.
Enable Linux containers (default on Linux).
