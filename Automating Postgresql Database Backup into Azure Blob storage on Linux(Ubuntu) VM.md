# Automating Operamadic Postgresql Database Backup into Azure Blob storage on Linux(Ubuntu 20.04) VM 

## TASK 1: Configure all preliminary packages
### Step 1: Run the following command to setup necessary packages
	sudo apt update

	wget --content-disposition "https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb"

	#sudo dpkg -i packages-microsoft-prod.deb
	#rm packages-microsoft-prod.deb

	sudo apt install apt-transport-https
	sudo apt update

	#sudo apt install -y dotnet-sdk-6.0
	sudo apt install -y dotnet-sdk-7.0

	sudo apt install -y powershell

	sudo apt install -y ca-certificates curl apt-transport-https lsb-release gnupg

	curl -sL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null

	AZ_REPO=$(lsb_release -cs)

	echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

	sudo apt update

	sudo apt install -y azure-cli


## TASK 2: Install postgresql on the linux machine

### Step 1: Setup necessary environment for the postgresql backup 

1. Add the GPG key for connecting with the official PostgreSQL repository using the below command: 
    
        wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
    
2.	Add the official PostgreSQL repository in your source list and also add it’s certificate using the below command: 

        sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'

3.	Run the below command to install PostgreSQL

        sudo apt update
        sudo apt install postgresql postgresql-contrib

4. Verify the installation of PostgreSQL by connecting to the local instance

        sudo su postgres
    
5. Enter the PostgreSQL shell with the below command
        
        psql
    
## TASK 3: Enable connection to the remote postgresql server

### Step 1: Edit the pg config file to support remote server connection
1. Edit the pg config file to support remote server connection. change "localhost" to "*"

        sudo vi  /etc/postgresql/15/main/postgresql.conf 

   
 
### Step 2: Edit the ph_hba.conf file to support remote server connection
1. Edit the ph_hba.conf file to support remote server connection

        sudo vi  /etc/postgresql/15/main/pg_hba.conf 

2. Add the follwing line accordingly

        host		all	all	0.0.0.0/0

        host 		all	all	:/0

        
 
### Step 3: Creating a .pgpass file for login handling
 
 1. Create a .pgpass file in home directory for the root user
    
        vi .pgpass
    
 2. Edit the file to contain database login credentials. Replace the password with the right information
 
        
		trdopmadtrial-db.postgres.database.azure.com:5432:*:grifadmin:<password_here>
    
 3. Set the permissions on the file to 600 to keep it secure i.e restrict read, write and execute access to only owner
       
        chmod 600 ~/ .pgpass
     
 4. Check the file privileges now
 
        l -l .pgpass
        
     
5. Test connection to the remote server, replacing the placeholder(coiling bracket) with necessary server paramenters
    
        
		psql -h trdopmadtrial-db.postgres.database.azure.com -U grifadmin CONTOSO_TEST_HYDRASTORE 
        
6. Restart the server after making above changes

        sudo systemctl restart postgresql
 
 
 
## TASK 4: mount Azure Blob Storage as a file system with BlobFuse v2


### Step 1: Install BlobFuse2 on Linux

 1. Check the version of your Linux to confirm it support the package. Follow path for supported distros https://github.com/Azure/azure-storage-fuse/releases

       	cat /etc/*-release
   
 2. Installing the necessary packages if it does not exist
 
        sudo wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb
		sudo dpkg -i packages-microsoft-prod.deb
		sudo apt-get update
		sudo apt-get install libfuse3-dev fuse3
		
3. Install BlobFuse2

        sudo apt-get install blobfuse2
	
		
4. Prepare for mounting: Use an SSD as a temporary path. Make sure user has access to the temporary path

        sudo mkdir {/mnt/resource/blobfusetmp} -p
        sudo chown root /mnt/resource/blobfusetmp
 
5. Create a directory where backups will be taken
				
		mkdir ~/contoso-backups 
		
### Step 2: Authorize access to the storage account. 

1. Create an authorization file to the blob storage(config.yaml) and paste the below text. Note the accountName is the name of the storage account, and not the full URL. 

        vi config.yaml 
	
        logging:
		  type: syslog
		  level: log_debug

		components:
		  - libfuse
		  - file_cache
		  - attr_cache
		  - azstorage

		libfuse:
		  attribute-expiration-sec: 120
		  entry-expiration-sec: 120
		  negative-entry-expiration-sec: 240

		file_cache:
		  path: /home/grifadmin/tempcache
		  timeout-sec: 120
		  max-size-mb: 4096
		  attr_cache:
		  timeout-sec: 7200

		azstorage:
		  type: block
		  account-name: trdopmadtrialstore
		  endpoint: https://trdopmadtrialstore.blob.core.windows.net
		  mode: sas
		  sas: {sas_key}
		  container: contoso-backups
		
2. Restrict access to the file so other users can not read it
		
        chmod 600 /path/to/config.yaml 

		
### Step 3: Mount the blob storage

1. To mount BlobFuse, run the following command with your user. The command mounts the container specified in ./config.yaml onto the location ~/contoso-backups:

       sudo blobfuse2 mount ~/contoso-backups --config-file=./config.yaml
    
## TASK 5: Backing up and restoring the Postgresql database

### Step 1: Create the shell script to backup the database

 1. In the home directory create a postgresql-backup.sh file
 
        vi postgresql-backup.sh
   
 2. The <b> BACKUP_DIR </b> is a reference path to where the backup would be taken
 
       	#!/bin/bash

       		BACKUP_DIR="/root/contoso-backups/"

		DB_NAME="CONTOSO_PX_REFERENCE"

		FORMATTED_DATE=$(date +"-%Y%m%d_%H%M%S")
		FILE_NAME="${BACKUP_DIR}${DB_NAME}${FORMATTED_DATE}.sql"

		DB_NAME2="CONTOSO_TEST_PX_REFERENCE"
		FILE_NAME2="${BACKUP_DIR}${DB_NAME2}${FORMATTED_DATE}.sql"

		echo $FILE_NAME
		pg_dump -h trdopmadtrial-db.postgres.database.azure.com -U grifadmin CONTOSO_PX_REFERENCE > $FILE_NAME
		pg_dump -h trdopmadtrial-db.postgres.database.azure.com -U grifadmin CONTOSO_TEST_PX_REFERENCE > $FILE_NAME2

 3. Make the file executable
  	
	chmod a+x /root/postgresql-backup.sh
 4. To restore the database, create a dbrestore.sh file and paste the below command.
 	
		vi dbrestore.sh
			#!/bin/bash


			psql -h trdopmadtrial-db.postgres.database.azure.com -U grifadmin -d CONTOSO_TEST_PX_REFERENCE < /root/contoso-backups/CONTOSO_TEST_PX_REFERENCE-20230510_164758.sql

 ### Step 2: Automate the database backup by leveraging on cron job in Linux environment
 
 1. on the home directory, type the below command
 
        crontab -e
        
 2. Edit the file to backup the database accordingly. For example the command below take a backup of the database daily by 8am based on the system time
 
        00 08 * * * /root/postgresql-backup.sh 
        
 3. Check the backupdir to confirm successful database backup 
 
        vi /root/contoso-backups
        
        
        
  
        
### References

1. "Install PostgreSQL on Linux" https://www.geeksforgeeks.org/install-postgresql-on-linux/
2. "How to take backup PosgreSQL using pg_dump with crontab in Linux?" https://gist.github.com/linuxkathirvel/90771e9d658195fa59e0f0b921f7e22e
4. "34.16. The Password File" https://www.postgresql.org/docs/current/libpq-pgpass.html
5. "How to mount Azure Blob Storage as a file system with BlobFuse2" https://learn.microsoft.com/en-us/azure/storage/blobs/blobfuse2-how-to-deploy?tabs=Ubuntu
6. "Azure/azure-storage-fuse:A virtual file system adapter for Azure Blob Storage" https://github.com/Azure/azure-storage-fuse/tree/main#environment-variables

     
     
 
     
   
 
 
    
    
