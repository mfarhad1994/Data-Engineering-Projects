# Setting up a cloud-based development environment on Google Cloud

## Table of Contents
- [Creating a VM Instance](#creating_a_vm_instance)
- [Configuring the Instance](#configuring_the_instance)
- [Connecting VS Code to the Remote Machine](#connecting_vs_code_to_the_remote_machine)
- [Cloning the Repository](#cloning_the_repository)
- [Setting Up Docker](#setting_up_docker)
- [Running PostgreSQL and pgAdmin](#running_postgresql_and_pgadmin)
- [Running Jupyter Notebook](#running_jupyter_notebook)
- [Installing Terraform](#installing_terraform)
- [Managing the VM Lifecycle](#managing_the_vm_lifecycle)


I would like to demonstrate how to set up an environment in Google Cloud. You can simply rent a virtual machine (VM) instance on Google Cloud. 


To begin, I need to navigate to **Compute Engine → VM Instances**. Before creating an instance, we must first generate an SSH key, which will be used to authenticate our connection. Since I use Git Bash, which already includes an SSH client, we will leverage this tool for establishing the connection. Although I am on Windows, I will follow the instructions for Linux, as Git Bash provides a Linux-like environment with SSH support. First, navigate to the `.ssh/` directory. If this directory does not exist, you need to create it manually.

```bash
Farhad_Mustafayev@farhad MINGW64 ~ $ cd .ssh/
```

Next, we will generate a new SSH key. I execute the following command, specifying the file name as `gcp` and the username as `farhad`. Upon being prompted for a passphrase, I leave it empty.

```bash
Farhad_Mustafayev@farhad MINGW64 ~/.ssh $ ssh-keygen -t rsa -f gpc -C farhad
```

This process generates two keys:
1. Private key (gcp) – Must be kept confidential.
2. Public key (gcp.pub) – Can be shared.

Now, we need to upload the **public key** to Google Cloud. To do this, navigate to **Metadata**, locate the **SSH Keys** tab, and paste the contents of the public key into the metadata section. After clicking "Save," Google Cloud will apply this key to all instances within the project.

```bash
Farhad_Mustafayev@farhad MINGW64 ~/.ssh $ cat gpc.pub
```

---
## Creating a VM Instance

Now, let's create a new VM instance. Under **Machine Configuration**, I  name the instance **vminst1-de**. Since Belgium is relatively close to Germany, I select a Belgian region. The platform also provides an estimated cost per month and per hour. For the machine type, I choose E2-standard. 
In the **OS and Storage** section, I change the boot disk to **Ubuntu** and set the storage size to 30 GB.
Once the instance is created, we will require its **external IP address** for SSH access. In the terminal, we will use the private key (not the public key) along with the external IP to establish an SSH connection.

```bash
Farhad_Mustafayev@farhad MINGW64 ~ $ ssh -i ~/.ssh/gpc farhad@34.140.233.83
```

---
### Errors
- if occurs: `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` then:

```bash
ssh-keygen -R 34.140.233.83
```

- if htop is not installed, then:

```bash
farhad@vminst1-de:~$ sudo apt update && sudo apt install -y htop
```
---

After successfully connecting, we can verify the instance's specifications using a system command. This setup provides us with **4 cores** and **16 GB of RAM**.

```bash
farhad@vminst1-de:~$ htop
```

---
## Configuring the Instance

Since Google Cloud SDK is already installed, there is no need for additional setup. Our first step is to download **Anaconda**, ensuring we select the Linux version.

```bash
farhad@vminst1-de:~$ wget https://repo.anaconda.com/archive/Anaconda3-2024.10-1-Linux-x86_64.sh
```

In a separate terminal, I  create an SSH configuration file called config and open it in Visual Studio Code. This file contains settings to streamline SSH access to the server. We need to specify the following details:
- Alias (host name)
- External IP address
- Username used during key generation
- Absolute path to the private key

```bash
Farhad_Mustafayev@farhad MINGW64 ~/.ssh $ touch config
Farhad_Mustafayev@farhad MINGW64 ~/.ssh $ code config
```

```bash
Host vminst1-de
    HostName 34.140.233.83
    User farhad
    IdentityFile c:/Users/gunay/.ssh/gpc
```

Then we initialize anaconda and check if it is in .bashrc:

```bash
Farhad_Mustafayev@farhad MINGW64 ~ $ ssh vminst1-de
farhad1@vminst1-de:~$ ~/anaconda3/bin/conda init
(base) farhad1@vminst1-de:~$ less .bashrc
```

and rehabilitate bash file:
```bash
farhad@vminst1-de:~$ source ~/.bashrc
```

## Connecting VS Code to the Remote Machine

To work with the remote instance using Visual Studio Code, install the **Remote - SSH** extension. Then, using the **Open Remote Window** feature, select **Connect to Host** and choose **vminst1-de** from the list (since it was defined in the SSH config file).

After successfully connecting, running which python confirms that Python is pointing to Anaconda. I can then launch Python and import pandas.

```bash
(base) farhad@vminst1-de:~$ python
>>> import pandas as pd
>>> pd.__version__
'2.2.2'
```

now let's install docker, before we need to do upgrade **update** to fetch the list of packages:

```bash
(base) farhad@vminst1-de:~$ sudo apt-get update
(base) farhad@vminst1-de:~$ sudo apt-get install docker.io
(base) farhad@vminst1-de:~$ docker
```

## Cloning the Repository

To clone the repository, we have two options:
- SSH (requires SSH configuration on the remote machine)
- HTTPS (recommended, as it provides an anonymous way of checking out the repository)
I will proceed with HTTPS to simplify the process.

```bash
(base) farhad@vminst1-de:~$ git clone https://github.com/mfarhad1994/Data-Engineering-Projects.git
```

## Setting Up Docker

To test Docker, I execute `docker run hello-world`. If permission is denied, we need to configure Docker to run without `sudo`. This requires modifying group permissions, logging out, and logging back in.


```bash
(base) farhad@vminst1-de:~$ sudo groupadd docker
(base) farhad@vminst1-de:~$ sudo gpasswd -a $USER docker
(base) farhad@vminst1-de:~$ sudo service docker restart
(base) farhad@vminst1-de:~$ logout
Farhad_Mustafayev@farhad MINGW64 ~/.ssh $ ssh vminst1-de
(base) farhad@vminst1-de:~$ docker run hello-world
```

Next, we install **Docker Compose**.
- Navigate to the Docker Compose GitHub repository
- Download the latest version (or choose a specific release)
- Create a bin directory to store executable files
- Use wget to download the binary and rename it to docker-compose

```bash
(base) farhad@vminst1-de:~$ mkdir bin
(base) farhad@vminst1-de:~$ cd bin
(base) farhad@vminst1-de:~/bin$ wget https://github.com/docker/compose/releases/download/v2.33.1/docker-compose-linux-x86_64 -O docker-compose
```

By default, the system does not recognize this file as executable, so we must update the PATH variable. To do this:
- Open `~/.bashrc` using nano
- Append the `bin` directory to the `PATH` variable
- Execute source `~/.bashrc` to apply the changes

```bash
(base) farhad@vminst1-de:~/bin$ chmod +x docker-compose
(base) farhad@vminst1-de:~/bin$ cd
(base) farhad@vminst1-de:~$ nano .bashrc
export PATH="${HOME}/bin:${PATH}"
(base) farhad@vminst1-de:~$ source .bashrc
```

Now, running `which docker-compose` confirms that it is accessible from any directory.

## Running PostgreSQL and pgAdmin

We proceed with setting up PostgreSQL and pgAdmin via Docker Compose. 

```bash
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=db_data
    volumes:
      - "./data_postgres:/var/lib/postgresql/data:rw"
    ports:
      - "5432:5432"
  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
    ports:
      - "8080:80"
```

```bash
(base) farhad@vminst1-de:~ $ docker-compose up -d
```
Once the required images are downloaded, we can install pgcli for database management. As an alternative, pgcli can also be installed via conda.

```bash
(base) farhad@vminst1-de:~$ pip install pgcli
(base) farhad@vminst1-de:~$ conda install -c conda-forge pgcli
```

While I have tested this setup locally on Ubuntu via VS Code, I have not yet verified it on Google Cloud. 

```bash
(base) farhad@vminst1-de:~$ pgcli -h localhost -U root -d db_data
Farhad_Mustafayev@farhad MINGW64 ~$ pgcli -h localhost -U root -d db_data
root@localhost:db_data> \dt
```

PostgreSQL runs on port **5432**, and to access it locally, we  use port forwarding in Visual Studio Code:

- Open the terminal (Ctrl + ~)
- Locate the **Ports** section
- Click **Forward a Port**
- Specify **5432**

Now, PostgreSQL is accessible from our local machine. Additional ports, such as 8080 for **pgAdmin**, can also be forwarded. By opening localhost:8080 in a browser, we should now have access to pgAdmin.

```bash
Farhad_Mustafayev@farhad MINGW64 ~$ pgcli -h localhost -p 5432 -u root -d db_data
```

## Running Jupyter Notebook

To start Jupyter Notebook, we need to forward another port, 8888. 

```bash
(base) farhad@vminst1-de:~$ jupyter notebook
(base) farhad@vminst1-de:~$ wget https://d37ci6vzurychx.cloudfront.net/trip-data/data_to_postgres_d.parquet
```

Once forwarded, we can use Jupyter Notebook to interact with data stored in PostgreSQL. Running the appropriate commands within Jupyter allows us to insert and query data successfully.

```python
import pandas as pd
import pyarrow.parquet as pq
conda install -c conda-forge psycopg2
from sqlalchemy import create_engine
file = pq.ParquetFile('data_to_postgres_d.parquet')
table = file.read()
df = table.to_pandas()
engine = create_engine('postgresql://root:root@localhost:5432/db_data')
engine.connect()
df.head(n=0).to_sql(name='data_in_postgres_d', con=engine, if_exists='replace')
```

## Installing Terraform

Next, we install Terraform by navigating to the bin directory (where Docker Compose is
stored) and downloading the Terraform binary. Since it is a ZIP archive, we must extract it
before using Terraform commands such as terraform plan and terraform apply.

```bash
(base) farhad@vminst1-de:~$ cd bin
(base) farhad@vminst1-de:~/bin$ wget https://releases.hashicorp.com/terraform/1.11.0/terraform_1.11.0_linux_amd64.zip
(base) farhad@vminst1-de:~/bin$ sudo apt-get install unzip
(base) farhad@vminst1-de:~/bin$ unzip terraform_1.11.0_linux_amd64.zip
(base) farhad@vminst1-de:~/terraform/$ ls
main.tf  variables.tf
```

main.tf:

```bash

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "6.21.0"
    }
  }
}

provider "google" {
  credentials = file(var.credentials)
  project     = var.project
  region      = var.region
}

resource "google_storage_bucket" "demo-bucket" {
  name          = var.gcs_bucket_name
  location      = var.location
  force_destroy = true

  lifecycle_rule {
    condition {
      age = 1
    }
    action {
      type = "AbortIncompleteMultipartUpload"
    }
  }
}

resource "google_bigquery_dataset" "demo_dataset" {
  dataset_id = var.bq_dataset_name
  location = var.location
```

variables.tf:

```bash
  variable "credentials" {
  description = "My Credentials"
  default     = "<Path to your Service Account json file>"
  #ex: if you have a directory where this file is called keys with your service account json file
  #saved there as my-creds.json you could use default = "./keys/my-creds.json"
}


variable "project" {
  description = "Project"
  default     = "<Your Project ID>"
}

variable "region" {
  description = "Region"
  #Update the below to your desired region
  default     = "europe-west3"
}

variable "location" {
  description = "Project Location"
  #Update the below to your desired location
  default     = "EU"
}

variable "bq_dataset_name" {
  description = "My BigQuery Dataset Name"
  #Update the below to what you want your dataset to be called
  default     = "demo_dataset"
}

variable "gcs_bucket_name" {
  description = "My Storage Bucket Name"
  #Update the below to a unique bucket name
  default     = "terraform-demo-terra-bucket"
}

variable "gcs_storage_class" {
  description = "Bucket Storage Class"
  default     = "STANDARD"
}
```

To authenticate **Terraform** with Google Cloud, we require a service account JSON credentials file. The file must be transferred to the server using **SFTP**. After establishing an SFTP connection to vminst1-de, I will create a directory called gc to store this JSON file.

```bash
Farhad_Mustafayev@farhad MINGW64 ~Farhad_Mustafayev/Documents/data-engineering/basics_setup/te $ sftp vminst1-de
sftp> mkdir .gc
sftp> cd .gc
sftp> put my-creds.json
Uploading my-creds.json to /home/farhad/.gc/my-creds.json
(base) farhad@vminst1-de:~/.gc$ ls
my-creds.json
```

Once uploaded, we configure Google Cloud authentication by setting the **GOOGLE_APPLICATION_CREDENTIALS** environment variable. This ensures that Terraform and other cloud-related tools recognize the credentials.

```bash
(base) farhad@vminst1-de:~/terraform$ export GOOGLE_APPLICATION_CREDENTIALS=~/.gc/my-creds.json
(base) farhad@vminst1-de:~/terraform$ gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
#one has to do changes in main.tf and variables.tf accordingly 
(base) farhad@vminst1-de:~/terraform$ terraform init
(base) farhad@vminst1-de:~/terraform$ terraform plan
(base) farhad@vminst1-de:~/terraform$ terraform apply
```

## Managing the VM Lifecycle

After completing the necessary tasks on this VM, you may need to reconnect later. If the external IP changes, update the SSH config file accordingly.

```bash
Farhad_Mustafayev@farhad MINGW64 ~$ nano .ssh/config
```

If you delete the virtual machine, all installed software and data will be lost permanently. However, if you stop the instance instead, you will not incur compute charges, though storage costs will still apply.

