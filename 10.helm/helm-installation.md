# Helm Installation  
Source: https://helm.sh/docs/intro/install/

To install Helm, follow these steps based on your operating system. I'll cover the installation process for Linux, macOS, and Windows.

### 1. **Linux (Ubuntu or other distributions)**

#### Install via Snap:
1. Open a terminal and run:
   ```bash
   sudo snap install helm --classic
   ```

#### Install via Script:   
Source: https://helm.sh/docs/intro/install/

1. Download the latest Helm release:
   ```bash
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

#### Install via Package Manager (APT):
1. Add the Helm repository:
   ```bash
   curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
   sudo apt-get install apt-transport-https --yes
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
   sudo apt-get update
   sudo apt-get install helm
   ```

### 2. **macOS**

#### Install via Homebrew:
1. Open a terminal and run:
   ```bash
   brew install helm
   ```

### 3. **Windows**

#### Install via Chocolatey:
1. Open a PowerShell terminal as Administrator and run:
   ```bash
   choco install kubernetes-helm
   ```

#### Install via Scoop:
1. Open a PowerShell terminal and run:
   ```bash
   scoop install helm
   ```

### 4. **Verify Installation**
To check if Helm is installed correctly, run:
```bash
helm version
```
This will display the version of Helm installed and confirm a successful installation.
