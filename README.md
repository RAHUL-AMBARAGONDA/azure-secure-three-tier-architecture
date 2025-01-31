  **Azure Security Lab: Securing a Three-Tier Architecture**

**Lab Overview**

This lab demonstrates how to deploy and secure a three-tier architecture (Web, App, and Database) on Azure. You will use an Azure Kubernetes Service (AKS) cluster for hosting the application and leverage Azure's security features like Network Security Groups (NSGs), Azure Load Balancer, Azure Sentinel, and more to protect the infrastructure. The lab will guide you through setting up 1 user for role-based access and connecting security services to monitor and detect threats.

**Step-by-Step Guide with Explanations**

**1. Setting Up the Azure Environment**

In this step, we prepare the environment for our three-tier architecture by creating resource groups, a virtual network (VNet), and subnets.

Create Resource Groups
A resource group in Azure is a logical container that holds all the resources related to your project. This helps manage, organize, and monitor related resources in one place.

```bash
az group create --name SecurityLab-RG --location eastus
```

**-What happens?** A resource group named `SecurityLab-RG` is created in the `East US` region. All the resources you create will be placed in this resource group for easy management.

Create a Virtual Network (VNet)
The virtual network (VNet) provides network isolation and is the foundation of your secure infrastructure. It allows communication between resources while enabling control over IP address ranges, subnets, routing, and security.

```bash

--name SecurityVNet \
--resource-group SecurityLab-RG \
--address-prefixes 10.0.0.0/16 \
--subnet-name WebSubnet \
--subnet-prefix 10.0.1.0/24
```

-**What happens** ? A VNet named `SecurityVNet` with the IP range `10.0.0.0/16` is created. A subnet named `WebSubnet` is created within this VNet to host the web tier with the address range `10.0.1.0/24`. Similar subnets can be created for the App and DB tiers.

**Create Additional Subnets (App and DB)**

These subnets isolate the application and database tiers. Subnets segment your network to help with security and performance.

```bash
az network vnet subnet create --resource-group SecurityLab-RG \
--vnet-name SecurityVNet --name AppSubnet --address-prefix 10.0.2.0/24
az network vnet subnet create --resource-group SecurityLab-RG \
--vnet-name SecurityVNet --name DBSubnet --address-prefix 10.0.3.0/24
```

-**What happens**? Two more subnets are created: `AppSubnet` for the application tier and `DBSubnet` for the database tier. These tiers are isolated, providing greater control over traffic flows between the tiers.

---

**2. Deploying AKS Cluster for Web and App Tiers**

Azure Kubernetes Service (AKS) is a managed Kubernetes service that simplifies containerized applications' deployment, management, and scaling. In this lab, we’ll host both the web and app tiers on AKS.

**Create AKS Cluster**

This step creates a Kubernetes cluster on Azure that hosts the containers for your web and app tiers. AKS simplifies Kubernetes management, patching, and scaling.

```bash
az aks create \
--resource-group SecurityLab-RG \
--name AKSCluster \
--node-count 3 \
--enable-addons monitoring \
--generate-ssh-keys
```

-**What happens**? A managed Kubernetes cluster (`AKSCluster`) is created with 3 nodes (VMs) that will host your containers. The `--enable-addons monitoring` flag enables Azure Monitor to collect logs and metrics from your AKS cluster. SSH keys are generated to securely connect to the cluster.

**Connect to AKS Cluster Using Kubectl**

`kubectl` is the command-line tool used to manage Kubernetes clusters. Here, we connect to the AKS cluster.

```bash
az aks get-credentials --resource-group SecurityLab-RG --name AKSCluster
```

-**What happens**? This command retrieves the AKS cluster credentials and configures `kubectl` to manage the cluster. After this step, you can deploy and manage containers on AKS.

**Deploy the Voting App on AKS**

We are deploying a sample voting app that represents the web and app tiers of our architecture.

```bash
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
cd azure-voting-app-redis
kubectl apply -f azure-vote.yaml
```

- **What happens**? The voting app, which has a front-end (web tier) and a Redis-based backend (app tier), is deployed on the AKS cluster. Kubernetes manages the pods, networking, and load balancing for the application.

**Network Policies for AKS**

Network Policies in Kubernetes help control traffic between different pods and services. This is important for isolating communication between the app and database tiers.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  policyTypes:
  - Ingress
```

- **What happens**? This policy allows traffic to flow only from the `frontend` pods to the `backend` pods, restricting all other traffic. This ensures that only the web tier can communicate with the app tier.

**Adding Users to AKS Using Azure AD and RBAC**

In this step, we integrate Azure Active Directory (AD) with AKS for authentication and assign role-based access control (RBAC) to 10 users.

**1. Create Users in Azure AD:**
   ```bash
   az ad user create --display-name User1 --user-principal-name user1@contoso.com --password P@ssw0rd123
   ```

   - What happens? A new user (`User1`) is created in Azure AD. This user will have access to the AKS cluster based on assigned roles.
   
**2. Assign RBAC Roles:**
   After adding all users, assign RBAC roles:
   ```bash
   az aks rolebinding create --name reader-role-binding --namespace default --clusterrole reader --user user1@contoso.com
   ```

   - What happens? The users are granted specific roles within the AKS cluster, giving them appropriate permissions based on their role. For example, a `Reader` role will only have view permissions, whereas `Admin` can deploy and manage resources.

**3. Securing with Azure Load Balancer and Network Security Groups (NSGs)**

Create Azure Load Balancer

Azure Load Balancer distributes incoming traffic to your AKS nodes, providing redundancy and high availability.

```bash
az network lb create \
--resource-group SecurityLab-RG \
--name AKSLoadBalancer \
--frontend-ip-name FrontEnd \
--backend-pool-name BackEndPool
```

-What happens? An Azure Load Balancer is created with a frontend IP (`FrontEnd`) that will handle incoming traffic and distribute it to the backend pool (`BackEndPool`), which contains the AKS nodes. This ensures your web and app tiers are highly available and scalable.

Create Network Security Groups (NSGs)

NSGs allow you to filter traffic between the subnets and resources. For instance, allowing only web traffic to the `WebSubnet` and limiting access between the tiers.

```bash
az network nsg create \
--resource-group SecurityLab-RG \
--name WebTier-NSG
```

- **What happens?** A Network Security Group (`NSG`) is created for the web tier. This allows control over traffic entering and exiting the web subnet.

**Define NSG Rules**

Rules within the NSG allow or deny specific traffic. For example, allowing HTTP traffic on port 80.

```bash
az network nsg rule create --resource-group SecurityLab-RG --nsg-name WebTier-NSG --name AllowHTTP \
--priority 100 --source-address-prefixes '*' --source-port-ranges '*' \
--destination-address-prefixes '*' --destination-port-ranges 80 --access Allow --protocol Tcp --direction Inbound
```

- What happens? This rule allows inbound HTTP traffic (port 80) to the web tier. You can define similar rules for HTTPS, database ports, etc.

**4. Protecting the Database**

Securing the database tier is crucial for protecting sensitive data. We’ll use Azure SQL Database for the backend, applying firewall rules and encryption.

Deploy Azure SQL Database

Azure SQL Database is a managed relational database service that provides high availability and security.

```bash
az sql server create \
--name sqlserver-lab \
--resource-group SecurityLab-RG \
--admin-user sqladmin \
--admin-password <YourPassword>
```

**- What happens?** An Azure SQL Server is created with admin credentials. This server will host the SQL database for the application.

**Enable Firewall Rules**

By default, Azure SQL Database is not accessible from outside the Azure environment. Firewall rules

 are needed to restrict or allow specific IPs.

```bash
az sql server firewall-rule create \
--resource-group SecurityLab-RG \
--server sqlserver-lab \
--name AllowYourIP \
--start-ip-address <YourIP> \
--end-ip-address <YourIP>
```

-** What happens?** A firewall rule is created that only allows traffic from your IP address (`<YourIP>`) to the database. This helps prevent unauthorized access.

**Enable Transparent Data Encryption (TDE)**

Transparent Data Encryption encrypts the database at rest, providing protection against offline attacks.

```bash
az sql db tde set --resource-group SecurityLab-RG --server sqlserver-lab --database <DBName> --status Enabled
```

- What happens? TDE is enabled for the database, ensuring all data stored in the database is encrypted, protecting against data breaches or physical theft of storage media.

---

**5. Enabling Azure Sentinel for Threat Detection**

Azure Sentinel is a cloud-native security information and event management (SIEM) solution. It collects, detects, and responds to threats in real time.

Set Up Azure Sentinel

Sentinel requires a Log Analytics workspace to collect logs and perform threat analysis.

```bash
az monitor log-analytics workspace create --resource-group SecurityLab-RG --workspace-name SentinelWorkspace
```

- What happens? A Log Analytics workspace (`SentinelWorkspace`) is created where logs and telemetry from various Azure services (e.g., AKS, SQL, NSGs) will be stored for analysis.

Enable Azure Sentinel

```bash
az sentinel create --resource-group SecurityLab-RG --workspace-name SentinelWorkspace --name SentinelInstance
```

-What happens? Azure Sentinel is enabled on the Log Analytics workspace. It begins analyzing logs and looking for potential security threats.

Connect Data Sources

We connect various Azure services (AKS, SQL, NSGs) to Sentinel to provide real-time monitoring and alerting on potential threats.

```bash
az sentinel data-source connect --workspace-name SentinelWorkspace --resource-group SecurityLab-RG --source-type AKS --source-id <AKSResourceID>
```

- What happens? This command connects AKS to Sentinel, allowing Sentinel to monitor the AKS cluster for security incidents such as anomalous network traffic or suspicious user behavior.

Create Alerts in Sentinel

You can create custom alert rules in Sentinel to detect specific types of threats. For example, an alert can trigger if there’s a brute force login attempt.

```bash
az sentinel alert-rule create --resource-group SecurityLab-RG --workspace-name SentinelWorkspace --name SuspiciousActivityAlert --query <Query for detection>
```

- What happens? An alert is created that triggers when certain criteria (e.g., a failed login attempt) are met. Sentinel will automatically notify the security team of any suspicious activity detected.

**Conclusion**

This lab demonstrates how to build and secure a three-tier architecture on Azure using best practices and a range of Azure security services. By following these steps, you can gain hands-on experience with AKS, NSGs, Load Balancer, Azure SQL Database, and Sentinel for threat monitoring and detection. The security measures taken in this lab can help you safeguard your infrastructure from modern threats.
