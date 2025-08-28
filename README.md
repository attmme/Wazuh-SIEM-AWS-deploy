# Wazuh SIEM deployment with AWS

&nbsp;

## INDEX

- [ℹ️ About this project](#ℹ%EF%B8%8F-about-this-project)

- [🤔💭 What is Wazuh](#-what-is-wazuh)

- [💰 Cost Estimate & Instance Selection ](#-cost-estimate--instance-selection)

- [🧾 Requisites](#-requisites)

- [🛠️ Installation & Setup](#%EF%B8%8F-installation--setup)

    - [📦 Setting up AWS](#-setting-up-aws)

    - [🚀 Setting up Wazuh indexer](#-setting-up-wazuh-indexer)

    - [🚀 Setting up Wazuh server](#-setting-up-wazuh-server)
 
    - [🚀 Setting up Wazuh dashboard](#-setting-up-wazuh-dashboard)

- [👋 Personal notes](#%EF%B8%8F-installation--setup)


---

&nbsp;

---

# ℹ️ About this project
This project documents a complete deployment of **Wazuh** on separated machines for the indexer, server, and dashboard on **AWS**. The goal is to have a reusable setup that can serve as a foundation for future projects related to SOC operations.

**❗ Please note:**
- I am not an expert, so there might be significant mistakes. If you spot any, I would greatly appreciate your feedback.
- I mainly created this repository to use as a **cheatsheet** to speed up deployments when helping clients.
- The setup is designed with a **retention period of ~90 days** and is intended for **low log ingestion**.
- The estimated complete deployment time is **around 4 to 6 hours**, under normal circumstances.
- All steps are based on the official documentation available at: [Wazuh Installation Guide](https://documentation.wazuh.com/current/installation-guide/) (as of 25/08/2025).

&nbsp;

---

&nbsp;

# 🤔💭 What is Wazuh
Wazuh is a **SIEM** (Security Information and Event Management), **event and log correlator**, capable of ingesting a large amount of information from agents and endpoints. It can **ingest, correlate, and categorize** anything sent from **rsyslog**.

&nbsp;

# 💰 Cost Estimate & Instance Selection 

## 1. **Wazuh Indexer (Elasticsearch-based)**
| Level       | Instance type | Specs             | USD/h  | USD/30d (720h) |
| ----------- | ------------- | ----------------- | ------ | -------------- |
| Minimum     | t3.large      | 2 vCPU, 8 GB RAM  | 0.0832 | **60.74**      |
| Recommended | m5.2xlarge    | 8 vCPU, 32 GB RAM | 0.384  | **280.32**     |

## 2. **Wazuh Server (Manager)**
| Level          | Instance type | Specs             | USD/h  | USD/30d (720h) |
| -------------- | ------------- | ----------------- | ------ | -------------- |
| Minimum stable | t3.medium     | 2 vCPU, 4 GB RAM  | 0.0416 | **30.37**      |
| Recommended    | t3.xlarge     | 4 vCPU, 16 GB RAM | 0.1664 | **121.47**     |

## 3. **Wazuh Dashboard**
| Level          | Instance type | Specs            | USD/h  | USD/30d (720h) |
| -------------- | ------------- | ---------------- | ------ | -------------- |
| Minimum stable | t3.medium     | 2 vCPU, 4 GB RAM | 0.0416 | **30.37**      |
| Recommended    | t3.medium     | 2 vCPU, 4 GB RAM | 0.0416 | **30.37**      |

## 4. Storage (EBS volumes)
Estimated **~20 USD/month** across all components (gp3 SSD volumes: 50–200 GB, depending on retention).

## 5. Total monthly cost estimate (30 days, 24/7)
|Scenario|Indexer (USD)|Server (USD)|Dashboard (USD)|Storage (USD)|**Total (USD)**|
|---|---|---|---|---|---|
|Minimum stable setup|60.74|30.37|30.37|20|**141.48**|
|Recommended setup|280.32|121.47|30.37|20|**452.16**|

&nbsp;

# 🧾 Requisites

## Wazuh indexer
|         | Minimum (t3.medium) | Recommended (m5.2xlarge) |
| ------- | ------------------- | ------------------------ |
| OS      | Unbuntu 16.04       | Unbuntu 16.04            |
| RAM     | 4 GB                | 16 GB                    |
| CPU     | 2 cores             | 8 cores                  |
| Storage | 50 GB               | 200 GB                   |

## Wazuh server
|         | Minimum (t3.medium) | Recommended   |
| ------- | ------------------- | ------------- |
| OS      | Unbuntu 16.04       | Unbuntu 16.04 |
| RAM     | 2 GB                | 4 GB          |
| CPU     | 2 cores             | 8 cores       |
| Storage | 50 GB               | 200 GB        |

## Wazuh dashboard
|         | Minimum (t3.medium) | Recommended   |
| ------- | ------------------- | ------------- |
| OS      | Unbuntu 16.04       | Unbuntu 16.04 |
| RAM     | 4 GB                | 8 GB          |
| CPU     | 2 cores             | 4 cores       |
| Storage | 50 GB               | 200 GB        |

&nbsp;

---

&nbsp;

# 🛠️ Installation & Setup

## 📦 Setting up AWS

### 1️⃣ VPC
1. Inside of the AWS - **VPC** page, in the left column, select **Your VPCs** option (inside *Virtual private cloud*).

    ![Step 1](assets/0_setting_up/1_vpc/1.png)

2. Then, **Create VPC**.

    ![Step 2](assets/0_setting_up/1_vpc/2.png)

3. Choose a name (recommended to find it between different VPC) and select an IP range, in this case **10.10.10.0/24**.

    ![Step 3](assets/0_setting_up/1_vpc/3.png)

4. Check both options for the DNS hostname and resolution and **Create VPC**.

    ![Step 4](assets/0_setting_up/1_vpc/4.png)

5. Then, inside of Internet gateways, create a new one.

    ![Step 4](assets/0_setting_up/1_vpc/5.png)
   
6. Inside of the new internet gateway, select the option to **attach to VPC**.

    ![Step 4](assets/0_setting_up/1_vpc/6.png)

7. Then, select the VPC we previously create and attach it.

   ![Step 4](assets/0_setting_up/1_vpc/7.png)

8. Enter again to the VPC, and under the resource map, open the route table.

   ![Step 4](assets/0_setting_up/1_vpc/8.png)

9. Edit the routes and add a new one to use the **wazuh-gateway** and save the changes.

   ![Step 4](assets/0_setting_up/1_vpc/9.png)
   ![Step 4](assets/0_setting_up/1_vpc/10.png)


### 2️⃣ Subnet
1. Inside of the AWS - _VPC_ page, in the left column, select **Subnets** option (inside _Virtual private cloud_).

    ![Step 1](assets/0_setting_up/2_subnet/1.png)

2. Then, **Create subnet**.

    ![Step 2](assets/0_setting_up/2_subnet/2.png)

3. Select the VPC that we created in the previous steps.

    ![Step 3](assets/0_setting_up/2_subnet/3.png)

4. Choose a VPN CIDR block and **Create subnet**.

    ![Step 4](assets/0_setting_up/2_subnet/4.png)


### 3️⃣ Security group

1. Inside of the AWS EC2 - _Instances_ page, in the left column, select **Security Groups** option (inside _Network & Security_).

    ![Step 1](assets/0_setting_up/3_security_group/1.png)

2. Then, **Create security group**.

    ![Step 2](assets/0_setting_up/3_security_group/2.png)

3. Select the _WazuhVPC_, created in the VPC step and **Create security group**. For now, we leave it empty because we need the group ID for the next step.

    ![Step 3_1](assets/0_setting_up/3_security_group/3.png)
    ![Step 3_2](assets/0_setting_up/3_security_group/4.png)

5. Inside the security group, edit the inbound rules and allow the ports shown in the image. Add your public IP to have connection with the instances, and the security group ID so the 3 instances will be able to connect to each other.

    ![Step 4](assets/0_setting_up/3_security_group/5.png)

6. Save the changes and check that everything is correct.

    ![Step 6](assets/0_setting_up/3_security_group/6.png)

&nbsp;

## 🚀 Setting up Wazuh indexer

### 1️⃣ EC2 instance

1. Choose a name for the instance.

    ![Step 1](assets/1_wazuh_indexer/aws/1.png)
   
2. Select the last Ubuntu Server.

   ![Step 2](assets/1_wazuh_indexer/aws/2.png)

3. Change the instance type to _t3.medium_.

    ![Step 3](assets/1_wazuh_indexer/aws/3.png)

4. Create a new key with **RSA** pair type and **PEM** format.

    ![Step 4_1](assets/1_wazuh_indexer/aws/4.png)
    ![Step 4_2](assets/1_wazuh_indexer/aws/5.png)
    ![Step 4_3](assets/1_wazuh_indexer/aws/6.png)
   
5. Edit the network settings and put the configuration we created during the **Setting up AWS**. 

    ![Step 5_1](assets/1_wazuh_indexer/aws/7.png)
    ![Step 5_2](assets/1_wazuh_indexer/aws/8.png)
    ![Step 5_3](assets/1_wazuh_indexer/aws/8_2.png)
   
6. Choose the disk capacity.

     ![Step 6](assets/1_wazuh_indexer/aws/9.png)

7. Launch the instance.
  
    ![Step 7](assets/1_wazuh_indexer/aws/10.png)


### 2️⃣ Connection and basic configuration
1. Give the correct permissions to your pem file:

   `$ chmod 0600 Wazuh_PEM.pem`

2. Connect via SSH to your AWS instance (indexer).

   `$ ssh -i Wazuh_PEM.pem ubuntu@<YOUR_INDEXER_PUBLIC_DNS>`

3. Update and install dependencies.

   `$ sudo apt update && sudo apt upgrade -y`

4. 📣 This is an optional step I will do in all the instances, only to make the steps easier to follow. To apply the changes, you need to exit and ssh again to the machine.

   `$ sudo hostnamectl set-hostname wazuh-indexer`


### 3️⃣ Initial Wazuh configuration

1. Download the Wazuh installation assistant and the configuration file.
You can skip the second curl and copy directly the code from **step 2**.

    ```
    $ curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh
    $ curl -sO https://packages.wazuh.com/4.12/config.yml
    ```
    
    ![Step 1](assets/1_wazuh_indexer/indexer/1.png)


2. Modify the `config.yml` file according to your environment and save (or create a new one if you skipped the downloading).
    ```
    nodes:
      indexer:
        - name: wazuh-indexer-1
          ip: "10.10.10.30"

      server:
        - name: wazuh-server
          ip: "10.10.10.31"

      dashboard:
        - name: wazuh-dashboard
          ip: "10.10.10.32"
    ```
    
    ![Step 2](assets/1_wazuh_indexer/indexer/2.png)


3. Run the `wazuh-install.sh` as admin.

    `$ sudo bash wazuh-install.sh --generate-config-files`
   
    ![Step 3](assets/1_wazuh_indexer/indexer/3.png)


4. Later, we will copy the `wazuh-install-files.tar` to the different instances. If you have multiple indexes, it will also be copied on them.

    `$ sudo chmod 744 wazuh-install-files.tar`


### 4️⃣ Wazuh indexer nodes installation

1. Using the previously downloaded `wazuh-install.sh`, add the name of the indexer given in the `config.yml` file and execute it. You will have to repeat this step for each Wazuh indexer node in your cluster, replacing the `wazuh-indexer-1` parameter for the according one. This can a few minutes.

   `$ sudo bash wazuh-install.sh --wazuh-indexer wazuh-indexer-1`

    ![Step 1](assets/1_wazuh_indexer/indexer/4.png)


### 5️⃣ Cluster initialization
   
1. Run again the installer, but changing the parameter to `--start-cluster`.

   `$ sudo bash wazuh-install.sh --start-cluster`

    ![Step 1](assets/1_wazuh_indexer/indexer/5.png)

2. To check if everything was correctly installed, firstly get the admin password.

    `$ tar -axf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O | grep -P "\'admin\'" -A 1` 

    ![Step 2](assets/1_wazuh_indexer/indexer/6.png)

3. Copy it, and replace the placeholders from the next command with your **admin account password** and the **index private IP**. You should have a similar output as shown in the image.

    `$ curl -k -u admin:<ADMIN_PASSWORD> https://<WAZUH_INDEXER_IP>:9200`
   
    ![Step 3](assets/1_wazuh_indexer/indexer/7.png)

4. Then, check the cluster, replacing the same placeholders as the previous step. You should have a similar output as shown in the image.

    `$ curl -k -u admin:<ADMIN_PASSWORD> https://<WAZUH_INDEXER_IP>:9200/_cat/nodes?v`

    ![Step 4](assets/1_wazuh_indexer/indexer/8.png)

5. Finally, it is recommended to disable Wazuh automatic updates. This way, you will avoid future problems with some update breaking your environment.

    ```
    $ sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
    $ sudo apt update
    ```

&nbsp;

## 🚀 Setting up Wazuh server

### 1️⃣ EC2 instance

1. Choose a name for the instance.

    ![Step 1](assets/2_wazuh_server/aws/1.png)
   
3. Select the last Ubuntu Server.

   ![Step 2](assets/1_wazuh_indexer/aws/2.png)

3. Change the instance type to _t3.medium_.

    ![Step 3](assets/1_wazuh_indexer/aws/3.png)

4. Choose the key.

    ![Step 4](assets/1_wazuh_indexer/aws/6.png)
   
5. Edit the network settings and put the configuration we created during the **Setting up AWS**. 

    ![Step 5_1](assets/1_wazuh_indexer/aws/7.png)
    ![Step 5_2](assets/1_wazuh_indexer/aws/8.png)
    ![Step 5_3](assets/2_wazuh_server/aws/8_2.png)
   
6. Choose the disk capacity.

     ![Step 6](assets/1_wazuh_indexer/aws/9.png)

7. Launch the instance.
  
    ![Step 7](assets/1_wazuh_indexer/aws/10.png)


### 2️⃣ Connection and basic configuration

1. From the **main host** (where the Wazuh_PEM.pem file is saved), we will copy the pem file to the indexer machine.

   `$ scp -i Wazuh_PEM.pem Wazuh_PEM.pem ubuntu@<YOUR_INDEXER_PUBLIC_DNS>:/home/ubuntu/`

2. From the **wazuh-index instance**, we will copy the install file that we generated, to the server machine.

    `$ scp -i Wazuh_PEM.pem wazuh-install-files.tar ubuntu@<YOUR_SERVER_PRIVATE_DNS>:/home/ubuntu/`

    ![Step 1](assets/2_wazuh_server/server/1.png)

3. Connect via SSH to your AWS instance (server).

    `$ ssh -i Wazuh_PEM.pem ubuntu@<YOUR_SERVER_PUBLIC_DNS>`

4. Update and install dependencies.

   `$ sudo apt update && sudo apt upgrade -y`

5. 📣 This is an optional step I will do in all the instances, only to make the steps easier to follow. To apply the changes, you need to exit and ssh again to the machine.

   `$ sudo hostnamectl set-hostname wazuh-server`


### 3️⃣ Wazuh server configuration

1. Download the Wazuh installation assistant file.

    `$ curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh`

2. Run the Wazuh installation assistant with the option `--wazuh-server` and the node name (the same used in the `config.yml`) to install the Wazuh server.

    `$ sudo bash wazuh-install.sh --wazuh-server wazuh-server`

    ![Step 2](assets/2_wazuh_server/server/2.png)

3. Finally, it is recommended to disable Wazuh automatic updates. This way, you will avoid future problems with some update breaking your environment.

    ```
    $ sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
    $ sudo apt update
    ```

&nbsp;

## 🚀 Setting up Wazuh dashboard

### 1️⃣ EC2 instance

1. Choose a name for the instance.

    ![Step 1](assets/3_wazuh_dashboard/aws/1.png)
   
3. Select the last Ubuntu Server.

   ![Step 2](assets/1_wazuh_indexer/aws/2.png)

3. Change the instance type to _t3.medium_.

    ![Step 3](assets/1_wazuh_indexer/aws/3.png)

4. Choose the key.

    ![Step 4](assets/1_wazuh_indexer/aws/6.png)
   
5. Edit the network settings and put the configuration we created during the **Setting up AWS**. 

    ![Step 5_1](assets/1_wazuh_indexer/aws/7.png)
    ![Step 5_2](assets/1_wazuh_indexer/aws/8.png)
    ![Step 5_3](assets/3_wazuh_dashboard/aws/8_2.png)
   
6. Choose the disk capacity.

     ![Step 6](assets/1_wazuh_indexer/aws/9.png)

7. Launch the instance.
  
    ![Step 7](assets/1_wazuh_indexer/aws/10.png)


### 2️⃣ Connection and basic configuration

1. From the **wazuh-index instance**, we will copy the install file that we generated, to the server machine.

    `$ scp -i Wazuh_PEM.pem wazuh-install-files.tar ubuntu@<YOUR_SERVER_PRIVATE_DNS>:/home/ubuntu/`

    ![Step 1](assets/3_wazuh_dashboard/dashboard/1.png)

3. Connect via SSH to your AWS instance (server).

    `$ ssh -i Wazuh_PEM.pem ubuntu@<YOUR_SERVER_PUBLIC_DNS>`

4. Update and install dependencies.

   `$ sudo apt update && sudo apt upgrade -y`

5. 📣 This is an optional step I will do in all the instances, only to make the steps easier to follow. To apply the changes, you need to exit and ssh again to the machine.

   `$ sudo hostnamectl set-hostname wazuh-dashboard`


### 3️⃣ Wazuh dashboard configuration

1. Download the Wazuh installation assistant file.

    `$ curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh`

2. Run the Wazuh installation assistant with the option `--wazuh-dashboard` and the node name (the same used in the `config.yml`) to install the Wazuh server.

    `$ sudo bash wazuh-install.sh --wazuh-dashboard wazuh-dashboard`

    ![Step 2](assets/3_wazuh_dashboard/dashboard/2.png)

3. To make it visible from outside (skip this if you plan to use a VPN, which is a highly recommended option) we will edit the configuration file and change the _server_host_ line for `0.0.0.0`.

    `$ sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml`
   
    ![Step 3](assets/3_wazuh_dashboard/dashboard/3.png)


4. Restart the dashboard and enter to the Wazuh dashboard (`https://<YOUR_DASHBOARD_PUBLIC_DNS>`) using your browser.
    `$ sudo systemctl restart wazuh-dashboard`

    ![Step 4](assets/3_wazuh_dashboard/dashboard/4.png)

   
5. To get the login credentials, we will use the following command.

    `$ tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt`

    ![Step 5](assets/3_wazuh_dashboard/dashboard/5.png)


6. Then, login to Wazuh dashboard.

    ![Step 6](assets/3_wazuh_dashboard/dashboard/6.png)
   

7. Finally, it is recommended to disable Wazuh automatic updates. This way, you will avoid future problems with some update breaking your environment.

    ```
    $ sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
    $ sudo apt update
    ```

&nbsp;

---

&nbsp;

# 👋 Personal notes

And here ends the Wazuh configuration. I ran into many issues along the way, but it turned out to be a great learning experience. This setup is by no means optimized for security — there are still several defaults that should be changed and hardened. Still, it works as a functional starting point to get hands-on with Wazuh and better understand how the components fit together.
