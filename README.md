# claimDotNetApp
This project describes the steps for building and deploying a real world Medical Claims Processing microservice application (Claims API) on Azure Kubernetes Service.

Table of Contents
A. Deploy an Azure SQL Server Database
B. Provision a Linux VM (Bastion Host/Jump Box) on Azure and install pre-requisite software
C. Build and run the Claims API microservice locally on the Bastion Host
D. Deploy an Azure Container Registry (ACR)
F. Define and execute a Build Pipeline in Azure DevOps Services
G. Deploy an Azure Kubernetes Service (AKS) cluster
Invoking the Claims API Microservice REST API

This project provides step by step instructions to use Azure DevOps Services to build the application binaries, package the binaries within a container and deploy the container on Azure Kubernetes Service (AKS). The deployed microservice exposes a Web API (REST interface) and supports all CRUD operations for accessing (retrieving / storing) medical claims records from a relational data store. The microservice persists all claims records in an Azure SQL Server Database.


For easy and quick reference, readers can refer to the following on-line resources as needed.

Azure CLI
ASP.NET Core 3.1 Documentation
Docker Documentation
Kubernetes Documentation
Helm 3.x Documentation
Creating an Azure VM
Azure Kubernetes Service (AKS) Documentation
Azure Container Registry Documentation
Azure DevOps Documentation
Application Insights for ASP.NET Core applications


A. Deploy an Azure SQL Server and Database
Approx. time to complete this section: 20 minutes
In this section, we will create an Azure SQL Server instance and create a database (ClaimsDB). This database will be used by the Claims API microservice to persist Medical Claims records.
1. Login to the Azure Cloud Shell. Login to the Azure Portal using your credentials and use a Azure Cloud Shell session to perform the next steps. Azure Cloud Shell is an interactive, browser-accessible shell for managing Azure resources. The first time you access the Cloud Shell, you will be prompted to create a resource group, storage account and file share. You can use the defaults or click on Advanced Settings to customize the defaults. Accessing the Cloud Shell is described in Overview of Azure Cloud Shell. 
2. Create a Resource Group. An Azure Resource Group is a logical container into which Azure resources are deployed and managed. From the Cloud Shell, use Azure CLI to create a Resource Group. Azure CLI is already pre-installed and configured to use your Azure account (subscription) in the Cloud Shell. Alternatively, you can also use Azure Portal to create this resource group. $ az group create --name myResourceGroup --location westus2  NOTE: Keep in mind, if you specify a different name for the resource group (other than myResourceGroup), you will need to substitute the same value in multiple CLI commands in the remainder of this project! If you are new to Azure Cloud, it's best to use the suggested name.  
3. Create an Azure SQL Server (managed instance) and database. In the Azure Portal, click on + Create a resource, Databases and then click on SQL Database as shown in the screenshot below. 
￼
4.  In the Create SQL Database window Basics tab, select the resource group which you created in the previous step and provide a name for the database (ClaimsDB). Then click on Create new besides field Server. 
￼
5.  Fill in the details in the New Server web page. The Server name value should be unique as the SQL database server FQDN will be constructed with this name eg., <SQL_server_name>.database.windows.net. Use a pattern such as <Your_Initial>sqldb for the server name (Replace Your_Initial with your initials). Specify a Server admin login name. Specify a simple Password containing numbers, lower and uppercase letters. Avoid using special characters (eg., * ! # $ ...) in the password! For the Location field, use the same location which you specified for the resource group. See screenshot below. 
￼
6.  Click on OK. In the Basics tab, click on Configure database and select the Basic SKU. See screenshots below. 
￼
7.  
￼
8.  
￼
9.  Click Apply. Click on Next : Networking > at the bottom of the web page. In the Networking tab, select Public endpoint for Connectivity method. Also, enable the button besides Allow Azure services and resources to access this server. Next, click on Review + create. See screenshot below. 
￼
10.  Review and make sure all options you have selected are correct. 
￼
11.  Click Create. It will take approx. 10 minutes for the SQL Server instance and database to get created. 
12. Configure a firewall for Azure SQL Server. Once the SQL Server provisioning process is complete, click on the database ClaimsDB. In the Overview tab, click on Set server firewall as shown in the screenshot below. 
￼
13.  In the Firewall settings tab, configure a rule to allow inbound connections from all public IP addresses as shown in the screenshots below. Alternatively, if you know the Public IP address of your workstation/pc, you can create a rule to only allow inbound connections from your local pc. Leave the setting for Allow access to Azure services ON. See screenshot below. 
￼
14.  Click on Save. NOTE: Remember to delete the firewall rule setting once you have finished working on all labs in this project.  
15. Copy the Azure SQL Server database (ClaimsDB) connection string. In the ClaimsDB tab, click on Connection strings in the left navigational panel (blade). See screenshot below. 
￼
16.  Copy the SQL Server database connection string under the ADO.NET tab. In the connection string, remove the listener port address including the 'comma' (,1433) before saving this string in a file. If the comma and the port number are present in the connection string then application deployments will fail. We will need the SQL Server db connection string in the next sections to configure the SQL Server database for the Claims API microservice. 

B. Provision a Linux CentOS VM on Azure
Approx. time to complete this section: 45 Minutes
The following tools (binaries) will be installed on the Linux VM (~ Bastion Host).
* Azure DevOps Pipelines Agent (docker container). The pipeline container will be used for running application and container builds.
* Azure CLI client. Azure CLI will be used to administer and manage all Azure resources including the AKS cluster resources.
* Git client. The Git client will be used to clone this GitHub repository and then push source code changes to the forked repository.
* .NET Core SDK. This SDK will be used to build and test the microservice application locally.
* Kubernetes CLI (kubectl). This CLI will be used for managing and introspecting the current state of resources deployed on the Kubernetes (AKS) cluster.
* Docker engine and client. 
Follow the steps below to create the Bastion host (Linux VM) and install pre-requisite software on this VM.
1. Fork this GitHub repository to your GitHub account. 
2. Create a Linux CentOS VM (Bastion Host). Open the Azure Cloud Shell in a separate browser tab and use the command below to create a CentOS 7.5 VM on Azure. Make sure you specify the correct resource group name and provide a value for the password. Once the command completes, it will print the VM connection info. in the JSON message (response). Save the Public IP address, Login name and Password info. in a file. Alternatively, if you prefer you can use SSH based authentication to connect to the Linux VM. The steps for creating and using an SSH key pair for Linux VMs in Azure is described here. You can then specify the location of the public key with the --ssh-key-path option to the az vm create ... command. $ az vm create --resource-group myResourceGroup --name k8s-lab --image OpenLogic:CentOS:7.5:latest --size Standard_B2s --data-disk-sizes-gb 128 --generate-ssh-keys --admin-username labuser --admin-password <password> --authentication-type password  
3. Login into the Linux VM via SSH. On a Windows PC, you can use a SSH client such as Putty, Git Bash or the Windows Sub-System for Linux (Windows 10) to login into the VM. NOTE: Use of Cloud Shell to SSH into the VM is NOT recommended.  # SSH into the VM.  Substitute the public IP address for the Linux VM in the command below.
4. $ ssh labuser@x.x.x.x
5. #  
Install Git client and clone this repository. When cloning the repository, make sure to use your Account ID in the GitHub URL. # Switch to home directory
$ cd
#
# Install Git client
$ sudo yum install -y git
#
# Check Git version number
$ git --version
#
# Create a new directory for GitHub repositories.
$ mkdir git-repos
#
# Change the working directory to 'git-repos'
$ cd git-repos
#
# Clone your GitHub repository into directory 'git-repos'.  Cloning this repo. will allow you to make changes to the application artifacts in the forked GitHub project.
# Substitute your GitHub Account ID in the URL.
$ git clone https://github.com/<YOUR-GITHUB-ACCOUNT>/aks-aspnet-sqldb-rest.git
#
# Switch to home directory
$ cd
#
Install a command-line JSON processor. Download jq command line processor and install it on the VM. # Make sure you are in the home directory
$ cd
#
# Create a directory called 'jq'
$ mkdir jq
#
# Switch to the 'jq' directory
$ cd jq
#
# Download the 'jq' binary and save it in this directory
$ wget https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
#
# Grant execute permission for jq
$ chmod 700 ./jq-linux64
#
# Switch back to the home directory
$ cd
#  
Install Azure CLI and login into your Azure account. # Install Azure CLI on this VM.
#
# Import the Microsoft repository key.
$ sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
#
# Create the local azure-cli repository information.
$ sudo sh -c 'echo -e "[azure-cli]\nname=Azure CLI\nbaseurl=https://packages.microsoft.com/yumrepos/azure-cli\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
#
# Install with the yum install command.
$ sudo yum install -y azure-cli
#
# Check the Azure CLI version (At the time of the last update to this document, 2.18.0 was the latest release)
$ az -v
#
# Login to your Azure account.  Use your Azure login ID and password to login.
$ az login -u <user name> -p <password>
# 
Install Kubernetes CLI and .NET Core SDK on this VM. # Switch back to home directory
$ cd
#
# Install Kubernetes CLI
# Create a new directory 'aztools' under home directory to store the kubectl binary
$ mkdir aztools
#
# Install Kubernetes CLI (kubectl + kubelogin) v1.20.x in the 'aztools' directory
$ sudo az aks install-cli --install-location=./aztools/kubectl
# Register the Microsoft key, product repository and required dependencies.
# sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm - Don't run this command !!
#
$ curl https://packages.microsoft.com/config/rhel/7/prod.repo > ./microsoft-prod.repo
$ sudo cp ./microsoft-prod.repo /etc/yum.repos.d/
#
# Update the system libraries.  This command will take a few minutes (~10 mins) to complete.  Be patient!
$ sudo yum update -y
#
# Install .NET Core 3.1 (latest) binaries
$ sudo yum install -y dotnet-sdk-3.1
#
# Check .NET Core version (Should print 3.1.xxx)
$ dotnet --version
#
# Install .NET EF Core 5.0.x
# This command installs dotnet-ef in ~/.dotnet/tools directory. so this directory has to be in
# the Path !!
$ dotnet tool install --global dotnet-ef
#
# Check .NET Core EF version (should print 5.0.2)
# You may need to logout of the terminal session and log back in to view theoutput
$ dotnet-ef --version
#
# Finally, update '.bashrc' file and set the path to jq, Helm and Kubectl binaries
# NOTE: Substitute 'labuser' with your Linux VM login name.
$ JQ=/home/labuser/jq
$ KUBECLI=/home/labuser/aztools

Install docker-ce container runtime.
Refer to the commands below. You can also refer to the Docker CE install docs for CentOS.
#
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum install -y docker-ce docker-ce-cli containerd.io
$ sudo systemctl start docker
$ sudo systemctl enable docker
$ sudo groupadd docker
$ sudo usermod -aG docker labuser


Build and run the Claims API microservice locally on the Linux VM
Approx. time to complete this section: 1 hour
In this section, we will work on the following tasks
* Configure the Azure SQL Server connection string in the Claims API microservice (source code)
* Build the Claims API microservice using the .NET Core 3.1 SDK
* Run the .NET Core Entity Framework migrations to create the relational database tables in Azure SQL Server provisioned in Section A. These tables will be used to persist Claims records.
* Run the Claims API microservice locally using the .NET Core 3.1 CLI
* Build the microservice Docker container and run the container
Before proceeding, login into the Linux VM using SSH.
1. Update the Azure SQL Server database connection string value in the appsettings.json file. The attribute SqlServerDb holds the database connection string and should point to the Azure SQL Server database instance which we provisioned in Section A. You should have saved the SQL Server connection string value in a file. Refer to the comands below to edit the SQL Server database Connection string... # Switch to the source code directory.  This is the directory where you cloned this GitHub repository.
2. $ cd git-repos/aks-aspnet-sqldb-rest
3. #  Edit the appsettings.json file using vi or nano editor and configure the SQL Server connection string value. Replace the variable tokens and specify correct values for SQL_SRV_PREFIX, SQL_USER_ID and SQL_USER_PWD in the connection string.
4.	Variable Token	5.	Description
6.	SQL_SRV_PREFIX	7.	Name of the Azure SQL Server instance. Eg., <Your_Initial>sqldb
8.	SQL_USER_ID	9.	User ID for the SQL Server instance
10.	SQL_USER_PWD	11.	User password for the SQL Server instance
	12.	Do not include the curly braces and the hash symbols (#{ xxx }#) when specifying the values. See the screenshot below. 

1. Build the Claims API microservice using the .NET Core SDK. #
2. # Build the Claims API microservice
3. $ dotnet build
4. #
5. Create and run the database migration scripts. This step will create the database tables for persisting Claims records in the Azure SQL Server database. Refer to the command snippet below. # Run the .NET Core CLI command to create the database migration scripts
6. $ dotnet-ef migrations add InitialCreate
7. #
8. # Run the ef (Entity Framework) migrations
9. $ dotnet-ef database update
10. #  
11. Run the Claims API locally using the .NET Core SDK. Run the Claims API microservice in a Linux terminal window. Refer to the commands below. # Make sure you are in the Claims API source code directory
12. $ pwd
13. /home/labuser/git-repos/aks-aspnet-sqldb-rest
14. #
15. # Run the microservice
16. $ dotnet run
17. #
18. # When you are done testing:
19. #   Press 'Control + C' to exit the program and return to the terminal prompt ($)
20. #  Login to the Linux VM using another SSH terminal session. Use the Curl command to invoke the Claims API end-point. Refer to the command snippet below. # Use curl command to hit the claims api end-point.  
21. $ curl -i http://localhost:5000/api/v1/claims
22. #  The API end-point should return a 200 OK HTTP status code and also return one claim record in the HTTP response body. See screenshot below. 
￼
23.  
24. Build and run the Claims API with Docker for Linux containers. In the SSH terminal window where you started the application (dotnet run), press Control-C to exit the program and return to the terminal prompt ($). Then execute the instructions (see below) in this terminal window. # Make sure you are in the Claims API source code directory.  If not switch ($ cd ...).
25. $ pwd
26. /home/labuser/git-repos/aks-aspnet-sqldb-rest
27. #
28. # Run the docker build.
29. # The build will take a few minutes to download both the .NET core build and run-time 
30. # containers!
31. # NOTE:
32. # The 'docker build' command does the following (Review the 'dockerfile'):
33. # 1. Build the dotnet application
34. # 2. Layer the application binaries on top of a base container image
35. # 3. Create a new application container image
36. # 4. Save the built container image on the host (local machine)
37. #
38. # DO NOT forget the dot '.' at the end of the 'docker build' command !!!!
39. # The '.' is used to set the context directory (path to the dockerfile) for the docker build.
40. #
41. $ docker build -t claims-api .
42. #
43. # List the docker images on this VM.  You should see two container images -
44. # - mcr.microsoft.com/dotnet/core/sdk
45. # - mcr.microsoft.com/dotnet/core/aspnet
46. # Compare the sizes of the two dotnet container images and you will notice the size of the runtime image is pretty small ~ 207MB when compared to the 'build' container image ~ 689MB.
47. $ docker images
48. #
49. # (Optional : Best Practice) Delete the intermediate .NET Core build container as it will consume unnecessary 
50. # space on your build system (VM).
51. $ docker rmi $(docker images -f dangling=true -q)
52. #
53. # Run the application container
54. $ docker run -it --rm -p 5000:80 --name test-claims-api claims-api
55. #
56. # When you are done testing:
57. #   Press 'Control + C' to exit the program and return to the terminal prompt ($)
58. #  
59. Invoke the Claims API HTTP end-point. Switch to the other SSH terminal window and this time invoke the Claims API HTTP end-points using the provided functional test shell script ./shell-scripts/start-load.sh # The shell script './shell-scripts/start-load.sh' calls the Claims API end-points and retrieves,
60. # inserts, updates and deletes (CRUD) both institutional and professional claims records from/to
61. # the Azure SQL Server database.
62. #
63. # Make sure you are in the project root directory (check => $ pwd)
64. #
65. $ chmod 700 ./shell-scripts/start-load.sh
66. #
67. # Substitute values for -
68. #  - No. of runs : eg., 1, 2, 3, 4 ....
69. #  - hostname and port : eg., localhost:5000
70. $ ./shell-scripts/start-load.sh <No. of runs> <hostname and port> ./test-data
71. #In the SSH terminal window where you started the application container (docker run), press Control-C to exit out of the program and return back to the terminal prompt.
You have now successfully tested the Claims API microservice locally on this VM.

D. Deploy Azure Container Registry
Approx. time to complete this section: 10 minutes
In this step, we will deploy an instance of Azure Container Registry (ACR) to store container images which we will build in later steps. A container registry such as ACR allows us to store multiple versions of application container images in one centralized repository and consume them from multiple nodes (VMs/Servers) where our applications are deployed.
1. Login to your Azure portal account. Then click on Container registries in the navigational panel on the left. If you don't see this option in the nav. panel then click on All services, scroll down to the COMPUTE section and click on the star beside Container registries. This will add the Container registries option to the service list in the navigational panel. Now click on the Container registries option. You will see a page as displayed below. 

2. Click on Add to create a new ACR instance. Give a meaningful name to your registry and make a note of it. Select an Azure Subscription, select the Resource group which you created in Section A and leave the Location field as-is. The location should default to the location assigned to the resource group. Select the Basic pricing tier (SKU). Click Create when you are done. IMPORTANT NOTES:
    * Keep in mind, you will need an Premium SKU ACR instance in order to work on Exercise 4. Hence select the Premium SKU if you intend to work on this challenge later.



Create an Azure Kubernetes Service cluster and deploy Claims API microservice
Approx. time to complete this section: 1 Hour

Follow the steps below to provision the AKS cluster and deploy the Claims API microservice.
Ensure the Resource provider for AKS service is enabled (registered) for your subscription. A quick and easy way to verify this is, use the Azure portal and go to ->Azure Portal->Subscriptions->Your Subscription->Resource providers->Microsoft.ContainerService->(Ensure registered). Alternatively, you can use Azure CLI to register all required service providers. See below. $ az provider register -n Microsoft.Network
$ az provider register -n Microsoft.Storage
$ az provider register -n Microsoft.Compute
$ az provider register -n Microsoft.ContainerService  
Check Kubernetes CLI version and available AKS versions. (If you haven't already) Open a SSH terminal window and login to the Linux VM (Bastion host). Refer to the command snippet below. # Check if kubectl is installed OK
$ kubectl version -o yaml
#
# List the AKS versions in a specific (US West 2) region
$ az aks get-versions --location westus2 -o table   
1. Provision an AKS cluster. NOTE: Follow the steps in option A or B below for deploying the AKS cluster. If you would like to explore deploying containers on Virtual Nodes in the extensions projects, follow the steps in option B below. Otherwise, follow the steps in option A.  A. Use the latest supported Kubernetes version to deploy the AKS cluster. At the time this project was last updated, version 1.19.6 was the latest stable AKS version. Refer to the commands below to create the AKS cluster. It will take a few minutes (< 10 mins) for the AKS cluster to get provisioned. # Create a 2 Node AKS cluster v1.19.6.  This is not the latest patch release.
# We will upgrade to the latest patch release in a subsequent lab/Section. 
#
# IMPORTANT: Remember to substitute correct values for 'Resource Group' & 'ACR Name' (Exclude the brackets '<' and '>').
#
# The 'az aks' command below will provision an AKS cluster with the following settings -
# - Kubernetes version ~ 1.19.6
# - No. of application/worker nodes ~ 2
# - Node pool - A single 'system' node pool (AKS supports up to 10 node pools)
# - RBAC ~ Disabled
# - CNI Networking Plugin - Kubenet
# - Azure Load Balancer - Standard SKU
# - Location ~ US West 2
# - DNS Name prefix for API Server ~ akslab
# - The cluster's service principal will have 'pull' permissions to pull container images from the ACR instance
#
$ az aks create --resource-group <Resource Group> --name akscluster --location westus2 --node-count 2 --dns-name-prefix akslab --generate-ssh-keys --disable-rbac --kubernetes-version "1.19.6" --attach-acr <ACR Name>
#
# Verify status of AKS cluster
$ az aks show -g myResourceGroup -n akscluster --output table 

# Verify status of AKS cluster
$ az aks show -g myResourceGroup -n akscluster --output table  
Connect to the AKS cluster. # Configure kubectl to connect to the AKS cluster
# Remember to substitute the correct value for 'Resource Group'
$ az aks get-credentials --resource-group myResourceGroup --name akscluster
#
# Check cluster nodes
$ kubectl get nodes -o wide
#
# Check default namespaces in the cluster
$ kubectl get namespaces  

