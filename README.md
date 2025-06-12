# Docker Registry Deployment Script for vSphere Supervisor

This script automates the deployment of a private Docker registry on a vSphere Supervisor. It sets up a Virtual Machine, configures it with Docker, and runs the official Docker registry image, making it accessible via a LoadBalancer service.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Script Overview](#script-overview)
- [Usage](#usage)
  - [Input Parameters](#input-parameters)
  - [Execution Steps](#execution-steps)
- [Kubernetes Resources Created](#kubernetes-resources-created)
- [Accessing the Registry](#accessing-the-registry)
  - [Authenticating](#authenticating)
  - [Pushing Images](#pushing-images)
  - [Pulling Images](#pulling-images)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

## Prerequisites

Before running this script, ensure you have the following:

1.  **Command-Line Tools**:
    *   `htpasswd`: For creating the authentication file for the registry.
    *   `openssl`: For generating self-signed certificates (if not providing your own).
    *   `kubectl`: For interacting with your Kubernetes cluster.
    *   `ssh`: For secure shell access to the VM.
    *   `scp`: For secure file copying to the VM.
    *   `base64`: For encoding data.
    *   `tanzu`: (Optional, depending on your vSphere with Tanzu setup for `kubectl` access).
2.  **`registry.tar` File**:
    *   A Docker image save of the official registry. You can create this using:
        ```bash
        docker save registry:2 -o registry.tar
        ```
    *   This file must be present in the same directory where you run the `deploy-docker-registry.sh` script.
3.  **vSphere Supervisor Environment**:
    *   Access to a vSphere Supervisor Cluster and a Namespace where you can deploy resources.
    *   Sufficient permissions to create PVCs, Services, VMs, Secrets, and ConfigMaps.
4.  **Photon 5.0 VMI**:
    *   A valid and **unique** Photon OS 5.0 VirtualMachineImage (VMI) must be available in the target vSphere Namespace. The script will verify this and exit if zero or multiple such VMIs are found, requiring manual intervention to ensure a single, correct VMI is present.
5.  **Storage Class**:
    *   A StorageClass available in your Kubernetes cluster for provisioning the PersistentVolumeClaim.

## Script Overview

The `deploy-docker-registry.sh` script performs the following actions:

1.  **Collects User Input**: Prompts for necessary details, such as namespace, hostname, credentials, etc.
2.  **Sets up Authentication**: Creates an `htpasswd` file for registry authentication.
3.  **Handles TLS Certificates**: Allows you to provide your own TLS certificate and key or generate self-signed ones for the specified hostname and LoadBalancer IP.
4.  **Provisions Kubernetes Resources**:
    *   `PersistentVolumeClaim (PVC)`: For registry data storage.
    *   `VirtualMachineService (LoadBalancer)`: To expose the registry externally.
    *   `Secret (TLS)`: To store the TLS certificate and key.
    *   `Secret (SSH)`: To store SSH keys for VM access.
    *   `ConfigMap (cloud-init)`: To provide startup configuration to the VM.
    *   `VirtualMachine`: The actual VM that will host the Docker registry.
5.  **Configures the VM**: Uses `cloud-init` to:
    *   Install Docker.
    *   Set up user accounts and SSH access.
    *   Prepare directories and decode TLS/auth files.
    *   Load the `registry.tar` image.
    *   Start the Docker registry container.
6.  **Validates Deployment**: Checks if the registry is accessible.

## Usage

### Input Parameters

The script will prompt you for the following information:

1.  **VSPHERE NAMESPACE**: The vSphere Namespace in your vSphere Supervisor environment where the registry and its associated resources will be deployed.
2.  **STORAGECLASS**: The name of the Kubernetes StorageClass to be used for the registry's persistent storage.
3.  **HOSTNAME**: The fully qualified domain name (FQDN) for your registry (e.g., `registry.yourcompany.com`). This will be used for the TLS certificate.
4.  **USERNAME**: The username used for authenticating to the Docker registry (for pushing and pulling images).
5.  **PASSWORD**: The password for the registry user.
6.  **Provide TLS cert and key files? (y/n)**:
    *   `y`: You will be prompted for the paths to your existing TLS certificate (`.crt`) and private key (`.key`) files.
    *   `n`: The script will generate a self-signed TLS certificate and key.

*Note: The script now includes validation for VSPHERE NAMESPACE and STORAGECLASS to ensure they adhere to Kubernetes naming conventions.*

### Execution Steps

1.  **Clone the Repository (if applicable) or Download the Script**:
    ```bash
    # git clone <repository_url>
    # cd <repository_directory>
    ```
2.  **Prepare `registry.tar`**:
    ```bash
    docker save registry:2 -o registry.tar
    ```
    Ensure `registry.tar` is in the current directory.
3.  **Make the Script Executable**:
    ```bash
    chmod +x deploy-docker-registry.sh
    ```
4.  **Run the Script**:
    ```bash
    ./deploy-docker-registry.sh
    ```
5.  **Follow Prompts**: Enter the required information as prompted by the script.
6.  **DNS Configuration**: After the script completes, it will output the `EXTERNAL_IP` of the LoadBalancer. You **must** create a DNS A record pointing your chosen `HOSTNAME` to this `EXTERNAL_IP`.

## Kubernetes Resources Created

The script creates the following Kubernetes resources in the specified namespace:

*   **PersistentVolumeClaim**: `vmware-docker-registry-pvc` (stores registry data)
*   **VirtualMachineService**: `vmware-docker-registry-vmservices` (Type: LoadBalancer, exposes ports 443 and 22)
*   **Secret**: `vmware-docker-registry-tls-secret` (stores TLS certificate and key)
*   **Secret**: `vmware-docker-registry-ssh-secret` (stores SSH keys for the VM)
*   **ConfigMap**: `vmware-docker-registry-configmap` (contains `cloud-init` data for VM configuration)
*   **VirtualMachine**: `vmware-docker-registry-vm` (hosts the Docker registry)

## Accessing the Registry

Once the script completes successfully and DNS is configured:

### Authenticating

Use the username and password you provided during the script execution.

```bash
docker login $HOSTNAME
# Enter Username and Password when prompted
```

### Pushing Images

Tag your local image and then push it:

```bash
docker tag my-image:latest $HOSTNAME/my-image:latest
docker push $HOSTNAME/my-image:latest
```

### Pulling Images

```bash
docker pull $HOSTNAME/my-image:latest
```

## Security Considerations

*   **Passwords**: Choose strong, unique passwords for the registry user.
*   **TLS Certificates**:
    *   For production environments, it is **highly recommended** to use certificates issued by a trusted Certificate Authority (CA) instead of the self-signed option.
    *   If using self-signed certificates, Docker clients must be configured to trust the certificate. This typically involves placing the `domain.crt` (generated or provided) into the Docker daemon's certificate directory (e.g., `/etc/docker/certs.d/$HOSTNAME/ca.crt` on Linux clients) and restarting the Docker daemon.
*   **VM Root Access**: The primary user is `vmware-system-user`. All administrative tasks on the VM should be performed using this user. Direct root login with a password is not configured.
*   **SSH Access**:
    *   The script uses `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null` for `scp` and `ssh` operations during setup. This is for automation convenience but disables protection against man-in-the-middle (MITM) attacks during the script's execution. This risk is confined to the script's runtime.
    *   The generated SSH private key (`id_rsa`) is used to configure the VM and then deleted from the deployment machine by the `cleanup` function. Access to the `vmware-system-user` is via the public key injected into the VM.
*   **`registry.tar` Integrity**: The script does not verify the integrity or source of the `registry.tar` file. Please ensure you are using an official and unmodified Docker registry image.
*   **File Permissions**:
    *   The script now explicitly sets `0600` permissions on the locally generated SSH private key (`id_rsa`) before its use.
    *   Within the VM, the `cloud-init` process now explicitly sets secure permissions after decoding sensitive files:
        *   `0600` for `/opt/registry/auth/htpasswd`
        *   `0600` for `/opt/registry/certs/domain.key`
        *   `0644` for `/opt/registry/certs/domain.crt`
    *   Users providing their own TLS keys (`CERT_CHOICE=y`) should ensure their private key file also has appropriate restrictive permissions.
*   **Secrets in Kubernetes**: SSH keys and TLS certificates/keys are stored as Kubernetes Secrets. Please make sure your Kubernetes RBAC policies restrict access to these secrets appropriately. The script base64-encodes these files before placing them in `cloud-init`, which is then stored in a ConfigMap; this is not encryption.
*   **LoadBalancer Exposure**: The LoadBalancer exposes the registry (port 443) and SSH (port 22) to the network. Ensure your network security rules (firewalls, NSGs) restrict access to these ports as needed.

## Troubleshooting

*   **LoadBalancer IP Not Assigning**:
    *   Check your Kubernetes cluster's LoadBalancer controller logs.
    *   Please ensure your vSphere environment has available IP addresses in the Load Balancer pool.
    *   The script waits for 5 minutes (60 attempts * 5 seconds); this might need adjustment for slower environments.
*   **VM Not Starting or Not Reachable**:
    *   Check `kubectl get vm vmware-docker-registry-vm -n $NAMESPACE -o yaml` for status and events.
    *   Check `kubectl get pods -n $NAMESPACE` to see if the VM operator pods are running correctly.
    *   Verify the VMI is valid and accessible.
    *   Check `cloud-init` logs on the VM if you can access it via console: `/var/log/cloud-init.log` and `/var/log/cloud-init-output.log`.
*   **Registry Not Accessible (HTTP 401 errors after login)**:
    *   Verify the `htpasswd` file was correctly generated and populated on the VM in `/opt/registry/auth/htpasswd`.
    *   Ensure the Docker client is correctly authenticating.
*   **TLS/SSL Errors**:
    *   If using self-signed certificates, ensure the client Docker daemon trusts the CA certificate (`domain.crt`).
    *   Verify the hostname used to access the registry matches the Common Name (CN) or a Subject Alternative Name (SAN) in the certificate.
*   **Permission Denied (Pushing/Pulling)**:
    *   Double-check the username and password used for `docker login`.
*   **`registry.tar` not found**:
    *   Ensure the file is named `registry.tar` and is in the same directory as `deploy-docker-registry.sh`.

## Cleanup

The script includes a `cleanup` function that attempts to remove temporary files created during its execution on the machine running the script:

*   `cloud-init.yaml`
*   `htpasswd`
*   `id_rsa` (private SSH key)
*   `id_rsa.pub` (public SSH key)
*   `domain.crt` and `domain.key` (if generated by the script)

**To remove the deployed resources from your Kubernetes cluster:**

```bash
kubectl delete vm vmware-docker-registry-vm -n $NAMESPACE
kubectl delete service vmware-docker-registry-vmservices -n $NAMESPACE
kubectl delete pvc vmware-docker-registry-pvc -n $NAMESPACE
kubectl delete secret vmware-docker-registry-tls-secret -n $NAMESPACE
kubectl delete secret vmware-docker-registry-ssh-secret -n $NAMESPACE
kubectl delete configmap vmware-docker-registry-configmap -n $NAMESPACE
```
Replace `$NAMESPACE` with the actual namespace used.
**Note**: Deleting the PVC will delete the registry data. Back up data if needed before deleting the PVC.
