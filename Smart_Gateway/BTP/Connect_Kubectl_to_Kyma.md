# How to Connect kubectl to a BTP Kyma Environment Using AWS EC2

## Overview

This guide walks through the steps required to connect **`kubectl`** to a BTP Kyma environment using an **AWS EC2 instance** as the intermediary host.

---

## Prerequisites

- EC2 instance running **Amazon Linux**.
- `.pem` key file for the EC2 instance (e.g., `<KEY_PAIR_NAME>.pem`).
- SAP BTP credentials with access to the Kyma environment.
- **OpenSSH on Windows** (included by default in Windows 10 and 11).
- Access to the `oneconnectkyma.zip` deployment file.

Placeholder values are shown in angle brackets `< >`. Replace them with your actual values before executing any command.

| Placeholder | Description |
|---|---|
| `<KEY_PAIR_NAME>.pem` | Name of the EC2 `.pem` key file. |
| `<EC2_USER>` | SSH user for the EC2 instance (e.g., `ec2-user`, `ubuntu`). |
| `<EC2_PUBLIC_IP>` | Public IP address of the EC2 instance. |

---

## Step 1: Connect to the EC2 via SSH

Open **PowerShell on Windows** and set the correct permissions on the `.pem` file so that SSH accepts it:

~~~powershell
icacls ".\<KEY_PAIR_NAME>.pem" /inheritance:r
icacls ".\<KEY_PAIR_NAME>.pem" /grant:r "$($env:USERNAME):(R)"
~~~

Connect to the EC2 instance, including the port tunnels required for Kyma OIDC authentication:

~~~bash
ssh -i ".\<KEY_PAIR_NAME>.pem" \
  -L 8000:localhost:8000 \
  -L 18000:localhost:18000 \
  <EC2_USER>@<EC2_PUBLIC_IP>
~~~

> ℹ️ **Note:** The `-L 8000:localhost:8000` and `-L 18000:localhost:18000` parameters create an SSH tunnel that allows Kyma's OIDC authentication (which opens a browser on port 8000) to work from the local machine.

---

## Step 2: Install kubectl

Run the following commands **inside the EC2 instance**:

~~~bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
~~~

> ℹ️ **Note:** The last command should display the installed kubectl version.

---

## Step 3: Install Helm v3

Run the following commands **inside the EC2 instance**:

~~~bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
~~~

---

## Step 4: Install kubelogin (OIDC Plugin for kubectl)

Kyma uses **OIDC authentication**, so the `kubelogin` plugin is required for kubectl to authenticate:

~~~bash
curl -LO https://github.com/int128/kubelogin/releases/latest/download/kubelogin_linux_amd64.zip
unzip kubelogin_linux_amd64.zip
sudo mv kubelogin /usr/local/bin/kubectl-oidc_login
kubectl oidc-login --version
~~~

---

## Step 5: Download the Project File

Download the project ZIP file and extract it:

~~~bash
curl -O https://oneconnectdeploymentaks.s3.us-east-2.amazonaws.com/oneconnectkyma/oneconnectkyma.zip
unzip oneconnectkyma.zip
~~~

> ℹ️ **Note:** If `unzip` is not installed, run:
>
> ~~~bash
> sudo yum install unzip -y
> ~~~

---

## Step 6: Download the Kyma Kubeconfig

Download the kubeconfig file from the **Kyma Environment Broker** and set it as an environment variable.

1. Log in to **BTP Cockpit** and navigate to your **Subaccount → Kyma Environment**.
2. Click on **"KubeconfigURL"**.
3. The system will open a new browser tab and prompt you to download the **`kubeconfig.yaml`** file. Wait for the file to download in your local drive.
<img width="1053" height="408" alt="image" src="https://github.com/user-attachments/assets/d2750d19-9dfa-41d7-9fdf-cef389e10200" />

Placeholder values are shown in angle brackets `< >`. Replace them with your actual values before executing any command.

| Placeholder | Description |
|---|---|
| `<ENVIRONMENT_INSTANCE_ID>` | Kyma instance ID (from SAP BTP Cockpit). |

Run the following commands **inside the EC2 instance**:

~~~bash
curl -L "https://kyma-env-broker.cp.kyma.cloud.sap/kubeconfig/<ENVIRONMENT_INSTANCE_ID>" -o kubeconfig.yaml
export KUBECONFIG=~/kubeconfig.yaml
~~~

> ℹ️ **Note:** The `<ENVIRONMENT_INSTANCE_ID>` can be found in the SAP BTP Cockpit under the **Kyma Environment** section of your subaccount.

---

## Step 7: Verify the Connection to the Kyma Cluster

Run the following command to verify the connection:

~~~bash
kubectl get namespaces
~~~

When running this command, a message will appear in the console asking you to open `http://localhost:8000` for authentication.

1. Open that URL in your local machine browser (Windows).
2. Sign in with your **SAP BTP credentials**.
3. Return to the terminal.

The command should list the Kyma cluster namespaces, confirming a successful connection.

---

## Summary

The full workflow is:

1. **Set permissions** on the `.pem` key file (Windows).
2. **SSH into the EC2 instance** with port forwarding for OIDC.
3. **Install `kubectl`** on the EC2 instance.
4. **Install `helm v3`** on the EC2 instance.
5. **Install `kubelogin`** for OIDC authentication.
6. **Download the OneConnect Kyma deployment ZIP** and extract it.
7. **Download and export the Kyma kubeconfig**.
8. **Verify the connection** by running `kubectl get namespaces` and completing OIDC login via the browser tunnel.

Once these steps are complete, you can proceed with the Smart Gateway deployment on Kyma. See the [Smart Gateway BTP Kyma Deployment Guide](./SmartGateway_BTP_Kyma_Deployment.md) for the full deployment workflow.
