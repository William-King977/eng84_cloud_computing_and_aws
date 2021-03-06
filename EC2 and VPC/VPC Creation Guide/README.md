# AWS VPC configuration with Subnets
### Step 1: Create the VPC
1. Click **Your VPCs** > **Create VPC**
2. Change the VPC nametag to `eng84_william_vpc`
3. Change IPv4 CIDR block to `0.0.0.0/16` where the first 2 numbers are modified e.g. `59.59.0.0/16`
4. Click **Create VPC**

### Step 2: Create the Internet Gateway and Attach to VPC
This needs to be done, so that your VPC can connect to the internet.
1. Click **Internet Gateways** > **Create internet gateway**
2. Change the nametag to `eng84_william_ig`
3. Click **Create Internet Gatway**
4. Click **Actions** > **Attach to VPC**
5. Select the VPC you created, then click **Attach internet gateway**

### Step 3: Create the Subnets
1. Click **Subnets** > **Create subnet**
2. Select your VPC
3. Add the Subnet name as `eng84_william_public_subnet`
4. Availability zone to `1c`, but `No preference` is fine
5. IPv4 CIDR block to `59.59.1.0/24` as per the VPC IP
6. Click **Create Subnet**
7. Repeat the above steps for the Private Subnet, but with the applicable name and the third number of the IPv4 CIDR block must be unique.

### Step 4: Setting up Routing Tables
First, we'll find our route table and link the public subnet into it.
1. Go on **Routing Tables**
2. Click on the unnamed route tables until you find the one with your VPC
3. Rename it to `eng84_william_public_rt`
4. With the route table selected, select the **Routes** tab
5. Click **Edit routes** and do the following:
   * Set the destination to `0.0.0.0/0`
   * Set the target to `Internet Gateway`, then select your internet gateway
   * Save the configurations
6. Select the **Subnet Associations** tab to link the Public Subnet
7. Click **Edit subnet associations** and do the following:
   * Select the public subnet you have created
   * Click **Save** 

Next, we'll create a separate route table for the private subnet.
1. Click **Create route table**, then do the following:
   * Set the Name tag to `eng84_william_private_rt`
   * Select your VPC
   * Click **Create**
   * NOTE: This route table will not be connected to the internet.
2. With the new route table selected, select the **Subnet Associations** tab
3. Click **Edit subnet associations** and do the following:
   * Select the PRIVATE subnet you have created
   * Click **Save**

Both route tables are now set up!

### Step 5: Creating the EC2 instances
#### NOTE: Skip this step if you have created AMIs already.
First, we'll create the instance for the app.
1. Click **Launch Instance**
2. Choose `Ubuntu Server 16.04 LTS (HVM), SSD Volume Type` as the Amazon Machine Image (AMI)
3. Choose `t2.micro` as the instance type (the default)
4. On Configure Instance Details:
   * Change the VPC to your VPC
   * Change subnet to your public subnet
   * Enable `Auto-assign Public IP`
5. Skip `Add Storage` (keep defaults)
6. Add a tag with the `Key` as `Name` and the value as shown below:
   * `eng84_name_appType`
   * `eng84_william_db`, `eng84_william_app`, etc.
7. Security group name should be `eng84_william_app_sg` for the app and have the following rules:
   * SSH (22) with source `My IP` - allows you to SSH
   * HTTP (80) with source `Anywhere` - allow access to the app (on the browser)
8. Review and Launch
9. Select the existing DevOpsStudent key:pair option for SSH

Now, we'll create an instance for the database.
1. Repeat steps 1 to 3 from the above
2. Repeat step 4, but change the subnet to your private one instead.
3. Repeat steps 5 and 6. Adjust the name applicable for step 6.
4. Repeat step 7, but replace the HTTP rule for All traffic:
   * Custom Source: `the app's security group`
   * This only allows the app to access the database
5. Repeat steps 8 and 9 as normal

### Step 6: Connecting the EC2 instances
Let's start with the app and import the files.
1. Go to the directory before the app folder.
2. Execute this command: `scp -i ~/.ssh/DevOpsStudent.pem -r app/ ubuntu@app_ec2_public_ip:~/app/`
3. For the provisions file, ensure it is in the `UNIX` format. There are a few options:
   * **Sublime Text**: click **View** > **Line Endings** > **Unix**.
   * **Notepad++**: on the bottom-right corner, right-click to select the `Unix (LF)` option.
   * **VS Code**, **Atom**: on the bottom-right corner, ensure `LF` is selected instead of `CTLF`.
   * Ensure you save the file after the change.
4. Go to the directory where the provision file is and execute: `scp -i ~/.ssh/DevOpsStudent.pem -r provision_name.sh ubuntu@app_public_ip:~/`. Now, it's time to SSH into the app.
5. Change to the `~/.ssh` directory
6. On the AWS instance page, click **Connect**, then run the SSH command given from the **SSH client** instructions 
7. If you are asked for a finger print, type yes.
8. Inside the instance, adjust the directories in the provision file as needed (hint: use `pwd`).
9. Run the provision file using `./provision_name.sh`. Change permissions with `chmod` if needed.
   * If you cannot execute it (no such file/directory), enter `sed -i -e 's/\r$//' provision_name.sh`, then run the above again.
   * If all else fails, create the provisions file and copy, paste the contents.
10. Run the environment variable command, so the app can connect to the database: `echo "export DB_HOST=mongodb://db_private_ip:27017/posts" >> ~/.bashrc`
11. Do the following if you want to apply the reverse proxy manually:
    * Execute: `sudo nano /etc/nginx/sites-available/default`
    * Replace it all with the code below:
      ```
      server {
          listen 80;

          server_name _;

          location / {
              proxy_pass http://localhost:3000;
              proxy_http_version 1.1;
              proxy_set_header Upgrade $http_upgrade;
              proxy_set_header Connection 'upgrade';
              proxy_set_header Host $host;
              proxy_cache_bypass $http_upgrade;
          }
      }
      ```

Next, we'll connect to the database instance.
1. The database is not connected to the internet, so a proxy SSH is required.
2. First, we need the Public IPv4 address of our app.
3. Next, we need the Private IPv4 address of our database.
4. Execute this command to SSH into the database: `ssh -i ~/.ssh/DevOpsStudent.pem -o ProxyCommand="ssh -i ~/.ssh/DevOpsStudent.pem -W %h:%p ubuntu@app_public_ip" ubuntu@db_private_ip`

### Step 7: Updating the database instance
First, modify the security group.
1. Go on **Security**, then click the security group
2. Click **Edit inbound rules**
3. Add a HTTP rule and set the Source to `Anywhere`

Next, go to the VPC to modify the route table subnet associations.
1. Click **Route Tables**
2. Select your public route table
3. With your public route table selected, click the **Subnet Associations** tab
4. Edit the subnet associations, then select the private subnet.

The database instance is now accessible to the internet.
1. SSH into the database instance just like your app instance.
2. Import the provisions file just like the app (no need to modify the contents) 
3. Run the provisions file 
4. With the operations complete, remove the private subnet from the public route table.
5. Remove the HTTP rule from the security group.
6. Reconnect (SSH) into both instances.
7. NOTE: for the app, one will need to run `seed.js`

### Step 8: Creating a Public NACL for the VPC
Ensure that you are in the VPC section (not EC2).
1. Go to the **Network ACLs** section under **Security**
2. Click **Create network ACL**
3. Add the appropriate name and allocate the VPC, then create it

Next, lets set the inbound rules for the NACL.
1. With the NACL selected, click on the **Inbound rules** tab
2. Click **Edit inbound rules**
3. Remove the default rule and add the following rules:
   * 100: HTTP (80) with source `0.0.0.0/0` - this allows external HTTP traffic to enter the network
   * 110: SSH (22) with source your_IP_address/32 - allows SSH connections to the VPC
   * 120: Custom TCP with Port range `1024-65535` and source `0.0.0.0/0` - allows inbound return traffic from hosts on the internet that are responding to requests originating in the subnet
   * NOTE: All rules should be **Allow** rules

Now, lets set the outbound rules.
1. Select the **Outbound rules** tab
2. Click **Edit outbound rules**
3. There should be the following rules:
   * 100: All traffic with source `0.0.0.0/0` - allow all the traffic out
   * NOTE: All rules should be **Allow** rules

### Step 9: Creating a Private NACL for the VPC
Create the private NACL and lets set the inbound rules for the NACL.
1. With the NACL selected, click on the **Inbound rules** tab
2. Click **Edit inbound rules**
3. Remove the default rule and add the following rules:
   * 100: Custom TCP with source public subnet CIDR block (`59.59.1.0/32` in this case) - this allows the app subnet to access the database subnet
   * 110: SSH (22) with source your_IP_address/32 - allows SSH connections to the VPC
   * NOTE: All rules should be **Allow** rules

Now, lets set the outbound rules.
1. Select the **Outbound rules** tab
2. Click **Edit outbound rules**
3. There should be the following rules:
   * 100: All traffic with source `0.0.0.0/0` - allow all the traffic out
   * NOTE: All rules should be `Allow` rules

### Step 10: Assigning Subnets to NACLs
Finally, lets assign the subnets to the NACLs.
1. Select the **Subnet associations** tab
2. Select the **Edit subnet associations** tab
3. Select the public/private subnet, depending on the NACL

# Creating Instance AMIs and New Instances (from those AMIs)
### Step 1: Create AMIs of the Instances
For each instance, do the following:
1. Select the instance
2. Select **Actions** > **Image and templates** > **Create image**
3. Enter name as instance with `_ami`
4. Click **Create image**

### Step 2: Create New Instances of the AMIs
For each instance, follow the steps from **Step 5: Creating the EC2 instances** in the **AWS VPC configuration with Subnets** instructions, but change the following:
1. When selecting your AMI, click on **My AMIs** on the side tab
2. Search for your AMI and select it
3. In the Security Group step, select the ones specific to your VPC if you created them separately. Otherwise, create new ones.
4. NOTE: creating instances with these AMIs will have everything installed (and running, like MongoDB)

### Step 3: Connecting Everything
There are a few things to change in the APP to get these AMI instances to work:
1. Change the private database IP in the environment variable (in `~/.bashrc`) to the private database of the AMI.
2. Exit and re-SSH into the app instance to 'save' the changes
3. NOT SURE, but you may have to re-run `seed.js`
4. Now, everything should work when running `node app.js`!

### TOP TIP(s): 
* **First iteration:** when you spin up the AMIs, do it in the default VPC and subnets first.
* **Second iteration:** spin up the AMIs in your own VPC and subnet

# Creating a Bastion Server
Even though we got *everything* working, we cannot SSH into our database instance because its in a private subnet. To solve this, we need to create a bastion server, also known as a jump box, so that we can log in to the bastion and then from there, access our database instance to perform updates.

### Step 1: Creating the Bastion Instance
1. Create a public subnet for the bastion server
2. Create a Linux instance that acts as the bastion server
   * Select your VPC and set the bastion subnet
   * Set the Security Group rule as SSH with source `My IP`
3. Change the following for the app and database instances:
   * In Security Group inbound rules, add SSH with source bastion security group (ID)
   * Add bastion subnet into the public route table (from the Subnet page)

### Step 2: Pre-SSH Configurations
Before the bastion can SSH into our app and database, do the following on your host machine:
1. On the `~/.ssh` directory, execute `ssh-agent bash`
2. Execute `ssh-add pem_file`
3. Now, execute `ssh -A ubuntu@bastion_public_ip` to SSH into the bastion server (instance). NOTE: using this method of SSH will allow you to SSH into the app and database. The *regular* SSH method won't allow this.
4. NOTE: the above needs to be done each time you open a new GitBash/Terminal window.

### Step 3: Bastion Config File
Inside the bastion server, we need to add a `config` file in the `~/.ssh` directory:
1. Execute `cd ~/.ssh`
2. Execute `nano config`
3. Copy and paste the following contents and adjust where applicable:
   ```
   Host db
     Hostname db_private_ip
     User ubuntu
     Port 22
   Host app
     Hostname app_private_ip
     User ubuntu
     Port 22
   ```
4. Now, you can SSH into the app and database with `ssh app` and `ssh db` respectively.
5. NOTE: you *may* have to `exit` on the host machine to stop some additional processes.

# AWS VPC Diagram
Below is a diagram of configuring a two tier architecture in a AWS VPC:

![image](https://user-images.githubusercontent.com/44005332/116392059-e2ef9b80-a817-11eb-8b14-c21d7aeddd8d.png)

## Security Group Rules (Public)
**Inbound rules:**
|Type  |Protocol  |Port Range  |Source      |Description
|:-    |:-        |:-          |:-          |:-
|HTTP  |TCP       |80          |0.0.0.0/0   |HTTP access from the browser
|HTTP  |TCP       |80          |::/0        |HTTP access from the browser
|SSH   |TCP       |22          |My Ip       |SSH access from my machine
|SSH   |TCP       |22          |Bastion SG  |SSH access from the bastion server

**Outbound rules:**
|Type         |Protocol  |Port Range  |Source     |Description
|:-           |:-        |:-          |:-         |:-
|All traffic  |All       |All         |0.0.0.0/0  |Allow all traffic out

## Security Group Rules (Private)
**Inbound rules:**
|Type         |Protocol  |Port Range  |Source      |Description
|:-           |:-        |:-          |:-          |:-
|All traffic  |All       |All         |App SG      |Allow all traffic from the app
|SSH          |TCP       |22          |Bastion SG  |SSH access from the bastion server

**Outbound rules:**
|Type         |Protocol  |Port Range  |Source     |Description
|:-           |:-        |:-          |:-         |:-
|All traffic  |All       |All         |0.0.0.0/0  |Allow all traffic out

## NACL Rules (Public)
**Inbound rules:**
|Type         |Protocol  |Port Range  |Source                 |Allow/Deny  |Why rule is needed
|:-           |:-        |:-          |:-                     |:-          |:-
|HTTP         |TCP (6)   |80          |0.0.0.0/0              |Allow       |Allow access from the app
|SSH          |TCP (6)   |22          |your_ip/32             |Allow       |Allows you SSH access
|Custom TCP   |TCP (6)   |1024-65535  |your_ip/32             |Allow       |Allows inbound returning traffic
|SSH          |TCP (6)   |22          |bastion_private_ip/32  |Allow       |Allows SSH access from bastion

**Outbound rules:**
|Type         |Protocol  |Port Range  |Source     |Allow/Deny  |Why rule is needed
|:-           |:-        |:-          |:-         |:-          |:-
|All traffic  |All       |All         |0.0.0.0/0  |Allow       |Allow all traffic out

## NACL Rules (Private)
**Inbound rules:**
|Type         |Protocol  |Port Range  |Source                      |Allow/Deny  |Why rule is needed
|:-           |:-        |:-          |:-                          |:-          |:-
|Custom TCP   |TCP (6)   |27017       |public_subnet_ipv4_CIDR/24  |Allow       |Allow access from the app
|SSH          |TCP (6)   |22          |your_ip/32                  |Allow       |Allows you SSH access
|SSH          |TCP (6)   |22          |bastion_private_ip/32       |Allow       |Allows SSH access from bastion

**Outbound rules:**
|Type         |Protocol  |Port Range  |Source     |Allow/Deny  |Why rule is needed
|:-           |:-        |:-          |:-         |:-          |:-
|All traffic  |All       |All         |0.0.0.0/0  |Allow       |Allow all traffic out

## Route Table Rules (Public)
|Destination  |Target    |Subnets
|:-           |:-        |:-      
|VPC_IP/16    |Local     |Public
|0.0.0.0/0    |Internet  |Public 

## Route Table Rules (Private)
|Destination  |Target    |Subnets
|:-           |:-        |:-      
|VPC_IP/16    |Local     |Private
