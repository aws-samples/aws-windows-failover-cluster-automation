# aws-windows-failover-cluster-automation
This solution describes the automated approach for setting up the infrastructure needed to setup SQL Server Failover Cluster Instances using Amazon FSx for windows in Multi Availability Zone(AZ). AWS services required for this setup will be provisioned in automated way using CloudFormation template and SQL Server installation and cluster node creation on Amazon EC2 carried out using PowerShell commands.

It uses a highly available Multi-AZ Amazon FSx file system as the shared witness used to store the SQL Server database files. The Amazon FSx file system and EC2 Windows instances that host SQL Server are joined to the same Active Directory domain.

# Prerequisites
- An Active AWS Account
- AWS User with sufficient permission to provision resources using CloudFormation templates
- Active Directory in AWS
- Credentials in secrets manager to authenticate to AD in Key Value Pair
  ADDomainName: <Domain Name>
  ADDomainJoinUserName: <Domain Username>
  ADDomainJoinPassword:<Domain User Password>
  TargetOU: <Target OU Value>
  Note: Keep same Key Name as it will be used in SSM automation for AD Join Activity.
- SQL Server media files for SQL server installation and windows service/domain accounts created which will be used during cluster creation.
- VPC, 2 Public Subnet (In separate AZ), 2 Private Subnets (In separate AZ), IGW, Nat Gateway, Route tables associations and jump server
- Keep below parameters details handy, It will be required during Infrastructure creations with Cloudformation script.
   | Parameter Name | Description |
   | ---------------| ------------|
   | ActiveDirectoryId | Active Directory ID in AWS |
   | ADDnsIpAddresses1 | Primary DNS IP of AD |
   | ADDnsIpAddresses2 | Secondary DNS IP of AD |
   | FSXSecurityGroupName | Name of FSX SG |
   | FSxWindowsFileSystemName | Name of FSx Drive |
   | ImageID | Base windows 2012 R2 Image/AMI ID from which SQL Instance node is to be created |
   | KeyPairName | Key Pair to attach to EC2 nodes for access |
   | Node1SecurityGroupName | Name of First Node SG |
   | Node2SecurityGroupName | Name of Second Node SG |
   | OUSecretName | Name of Secret containing AD related information |
   | PrivateSubnet IDs | IDs of private subnets |
   | SqlFSxFCIName | FSx FCI Name to be provided |
   | SqlFSxServerNetBIOSName1 | Name of Primary EC2 node (Limit 15 characters) |
   | SqlFSxServerNetBIOSName2 | Name of Secondary EC2 node (Limit 15 characters) |
   | VPC | VPC ID |
   | WorkloadInstanceType | Type of EC2 Instance |

# Product Versions
- Windows Server 2012 R2 and SQL Server 2016

# Target Technology Stack
- AWS EC2 instances
- AWS FSx for windows server
- SSM Automation Runbook (Systems Manager)
- Network Configurations (Security Groups, VPC, Subnets, Internet/NAT Gateways)
- Secret Manager
- Active Directory
- Event Bridge
- IAM

# Automation and Scale
- Use Cloudformation to setup infrastructure for multiple clusters.
- SSM Documentation to join instance with Active Directory in automated way.
- Set up windows sql fci cluster with automation.

# Tools
- AWS CloudFormation – AWS CloudFormation is an infrastructure as code (IaC) service that allows you to easily model, provision, and manage AWS and third-party resources.
- AWS System Manager – AWS Systems Manager is the operations hub for your AWS applications and resources and a secure end-to-end management solution for hybrid cloud environments that enables safe and secure operations at scale.
- AWS Identity and Access Management (IAM) – AWS IAM helps you to manage access to AWS services and resources securely.
- AWS Event Bridge - Amazon EventBridge is a serverless event bus that makes it easier to build event-driven applications at scale using events generated from your applications.
- PowerShell Scripts

# Code
The code for this pattern is available on gitlab, in the windows-sql-fci-cluster repository. The code repository contains the following files and folders:

- infra-cf.yaml file - Contain cloudformation template to create infrastructure required to setup windows sql fsx failover cluster.
- ssm.yaml file - Contain cloudformation template to create SSM automation document which gets triggerd upon EC2 launch with tag ADJoined: FSXADD and add the instance to active directory.

# Setup your environment
To pull down the repo 
```
git clone https://gitlab.aws.dev/gwmanish/windows-sql-fci-cluster.git
```
This creates a folder named `windows-sql-fci-cluster`

## Setup Infrastructure
This describes the steps to deploy infrastructure needed to setup SQL Server FCI Cluster.
- Complete all steps mentioned in Pre-Requisites section.
- Go to AWS Account, login with IAM credentials. Navigate to CloudFormation console and deploy SSM automation document stack. The file name is ssm.yaml. Some parameters need to be provided to run the cf stack for SSM which are listed below.

    StateUnJoinAssociationLoggingBucketName: Will create a s3 bucket for logging purpose so need to provide the name of bucket.

    SSMAssociationADUnjoinName: Provide name to be given for SSM Association

    SSMAutomationDocumentName: Provide name to be given for SSM document

    EventBridgeName: Provide name to be given for event bridge

    Deploy the SSM CF stack. It will create SSM automation document which will get triggered when new EC2 server having tag ADJoined: FSXADD launches and will add the instance to active directory.
- After successful deployment of SSM stack, next step is to create infra component which includes EC2 nodes, Security Groups, FSx and IAM role. Navigate to CloudFormation console and deploy infra stack. The file name is infra-cf.yaml. Some inputs parameters are required to run this stack and those are mentioned in prerequisites section.

    Deploy the infra stack and it will launch all the needed infra components required to setup Windows SQL FCI Failover Cluster.
- Once EC2 nodes are launched , SSM document will get trigger to join these instance to Active Directory. To track the progress go to Systems Manager —> Automation page.
- After the successful completion of SSM document, next step is to setup SQL FailOver cluster.

## Setup FCI (FailOver Cluster instance)
This describes the steps to deploy and configure FCI FailOver Cluster Instance using FSx.
- Log in to Primary Node (EC2 instance or Primary node eg. Node 1). Execute below powershell script. This script will install windows feature (AD and FCI Tools).
```
Install-WindowsFeature -Name RSAT-AD-Powershell,Failover-Clustering -IncludeManagementTools
Install-WindowsFeature -Name RSAT-Clustering,RSAT-ADDS-Tools,RSAT-AD-Powershell,RSAT-DHCP,RSAT-DNS-Server
```
Similarly, log in to Secondary Node (Node 2)  and run same script to enable features on Node 2.
- Prestage Cluster Computer Objects in Active Directory Domain Services (AD DS).

    - Prestage the CNO in AD DS

        - Login to AD Admin user on either of node Node 1 or 2 , go to Server Manager -> Tool -> click on “Active Directory Users and Computers”.

        - Expand the AD and go to required/existing OU where cluster computer objects needs to be created. Right click on OU where CNO needs to be created and point to “New” and click on computer.

        - Enter computer name which we will be using as a cluster.

        - Disable the the newly created computer accounts by right-clicking and “Disable Account”.

        - Grant permissions to CNO to Create Cluster. Here the CNO is AWSSQLDBCL.

        - Right Click on CNO , go to Properties -> Security -> click Add -> “Select Users, Computers, or Groups” dialog box -> search and add AD Service account and grant Full control to them. -> click OK.

        Note:- If in properties security tab is not visible then go to Active Directory Users and Computers window then click on view → tick Advanced Features.

        - Grant permissions to CNO to OU - Right Click on CNO , go to Properties -> Security -> Advanced -> “Advanced security settings” click Add.

        - Click on “Select a principal”. In Object Types, select the Computers check box and then click OK. Enter the CNO name in text box “Enter the object names to select” and click Check Names and then click OK.

        - In response to the warning message that says that you are about to add a disabled object, click OK

        - In the Permission Entry dialog box, make sure that the Type list is set to Allow, and the Applies to list is set to This object and all descendant objects. Under Permissions, select the Create Computer objects  or Full Control check box.

        - Click OK until you return to the Active Directory Users and Computers snap-in.

        This completes the grant permission step. Now, administrator on the failover cluster can create clustered roles with client access points, and bring the resources online.

    - Prestage VCO For a Clustered role

        In Active Directory Users and Computers, on the View menu, make sure that Advanced Features is selected.

        You can follow similar steps as that of Prestage CNO.

        - Login to AD Admin user on either of node Node 1 or 2 , go to Server Manager -> Tool ->  click on “Active Directory Users and Computers”.

        - Expand the AD and go to OU where CNO was created in previous steps. Right click on that OU , point to “New”  and click on computer.

        - Similarly in same OU create VCO object, whose name is Network Name of SQL Server or Clustered Role.

        - Right-click the computer account that you just created, and then click Properties.

        - On the Security tab, click Add -> In the Select User, Computer, Service Account, or Groups dialog box, click Object Types, select the Computers check box, and then click OK.

        - Under Enter the object names to select, enter the name of the CNO, click Check Names, and then click OK. If you receive a warning message that says that you are about to add a disabled object, click OK.

        - Make sure that the CNO is selected, and then next to Full control, select the Allow check box.

- Create WSFC (Windows Server Failover Cluster)

    - Log in to Primary Node (EC2 instance or Primary node eg. Node 1).

    - On Primary Node (Node 1) , execute below poweshell command to create FSx Share and grant full access to listed AD service account.
    This command will also creates Continuously Available (CA) file share, which are optimised for use by Microsoft SQL Server.

    ```
    Invoke-Command -ComputerName "<FSx Windows Remote PowerShell Endpoint>" -ConfigurationName FSxRemoteAdmin -scriptblock {
    New-FSxSmbShare -Name "SQLDB" -Path "D:\share" -Description "SQL Databases Share" -ContinuouslyAvailable $true -FolderEnumerationMode AccessBased -EncryptData $true
    grant-fsxsmbshareaccess -name SQLDB -AccountName "<domain\user>" -accessRight Full
    }
    ```

    - Run below command on Primary Node (Node 1) to create Failover Cluster.
    ```
    New-Cluster -Name <CNO Name> -Node  <Node1 Name>, <Node2 Name> -StaticAddress <Node1 Secondary Private IP>, <Node2 Secondary Private IP>
    ```

    In above command,  details of parameters given below.

      Name - Name of Cluster (CNO)

      Node - Name of primary and secondary node respectively

      StaticAddress - Secondary IP addresses of primary and secondary nodes respectively.

      Important Note - Domain admin/regular user must have administrator permission on both the nodes to create WSFC cluster otherwise above command will fail stating "You do not have administrator privilege on servers". 

    - Once cluster is created, run below command to attach FileShare Witness to it.
    ```
    Set-ClusterQuorum -FileShareWitness \\<FSx Windows Remote PowerShell Endpoint>\share\witness
    ```
- Install  SQL Server Failover Cluster

    - After Cluster is set up, proceed with installation of SQL Cluster on primary node1, use below Powershell commands to install SQL server.

    - Create two folders in T drive in both the nodes tempdb and log, the same need to provide as an argument in below command.

    - Copy the SQL Server media files for SQL server installation on both nodes.

    ```
    D:\setup.exe /Q  `
    /ACTION=InstallFailoverCluster `
    /IACCEPTSQLSERVERLICENSETERMS `
    /FEATURES="SQL,IS,BC,Conn"  `
    /INSTALLSHAREDDIR="C:\Program Files\Microsoft SQL Server”  `
    /INSTALLSHAREDWOWDIR="C:\Program Files (x86)\Microsoft SQL Server"  `
    /RSINSTALLMODE="FilesOnlyMode"  `
    /INSTANCEID="MSSQLSERVER" `
    /INSTANCENAME="MSSQLSERVER"  `
    /FAILOVERCLUSTERGROUP="SQL Server (MSSQLSERVER)"  `
    /FAILOVERCLUSTERIPADDRESSES="IPv4;<2nd Sec Private Ip node1>;Cluster Network 1;<subnet mask>"  `
    /FAILOVERCLUSTERNETWORKNAME="<Fail over cluster Network Name>"  `
    /INSTANCEDIR="C:\Program Files\Microsoft SQL Server"  `
    /ENU="True"  `
    /ERRORREPORTING=0  `
    /SQMREPORTING=0  `
    /SAPWD=“<Domain User password>” `
    /SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"  `
    /SQLSYSADMINACCOUNTS="<domain\username>" `
    /SQLSVCACCOUNT="<domain\username>"  /SQLSVCPASSWORD="<Domain User password>" `
    /AGTSVCACCOUNT="<domain\username>"  /AGTSVCPASSWORD="<Domain User password>" `
    /ISSVCACCOUNT="<domain\username>" /ISSVCPASSWORD="<Domain User password>"  `
    /FTSVCACCOUNT="NT Service\MSSQLFDLauncher"  `
    /INSTALLSQLDATADIR="\\<FSX DNS name>\share\Program Files\Microsoft SQL Server"  `
    /SQLUSERDBDIR="\\<FSX DNS name>\share\data"  `
    /SQLUSERDBLOGDIR="\\<FSX DNS name>\share\log" `
    /SQLTEMPDBDIR="T:\tempdb"  `
    /SQLTEMPDBLOGDIR="T:\log"  `
    /SQLBACKUPDIR="\\<FSX DNS name>\share\SQLBackup" `
    /SkipRules=Cluster_VerifyForErrors `
    /INDICATEPROGRESS
    ```
- Add Secondary Node to Cluster
    - Use below Powershell commands to Add secondary node SQL server. Also, create two folders in T drive in both the nodes tempdb and log.

    ```
    D:\setup.exe /Q  `
    /ACTION=AddNode `
    /IACCEPTSQLSERVERLICENSETERMS `
    /INSTANCENAME="MSSQLSERVER"  `
    /FAILOVERCLUSTERGROUP="SQL Server (MSSQLSERVER)" `
    /FAILOVERCLUSTERIPADDRESSES="IPv4;<2nd Sec Private Ip node2>;Cluster Network 2;<subnet mask>" `
    /FAILOVERCLUSTERNETWORKNAME="<Fail over cluster Network Name>" `
    /CONFIRMIPDEPENDENCYCHANGE=1 `
    /SQLSVCACCOUNT="<domain\username>"  /SQLSVCPASSWORD="<Domain User password>" `
    /AGTSVCACCOUNT="domain\username>"  /AGTSVCPASSWORD="<Domain User password>" `
    /FTSVCACCOUNT="NT Service\MSSQLFDLauncher" `
    /SkipRules=Cluster_VerifyForErrors `
    /INDICATEPROGRESS
    ```
- Test  SQL Server Failover Cluster
    - Launch the Failover Cluster Manager tool, within the Administrative Tools.

    - Check both nodes are up and running.

    - Select Roles, right click on SQL Server (MSSQLSERVER) and select Move and Select Node. Now after the node selection the SQL should be running on other node now and vice versa.

## Contributors
   - Manish Garg
   - Rajneesh Tyagi
   - Nishad Mankar
   - T.V.R.L.Phani Kumar Dadi

## Security
See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License
This library is licensed under the MIT-0 License. See the LICENSE file.