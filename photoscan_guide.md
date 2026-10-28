# Using Photoscan Pro with UA-HPC resources

**Author:** Guillermo Ponce
geponce at gmail.com  

The aims of this notebook is to document the steps for processing drone imagery using Photoscan Pro and the High Performance Computing services from The University of Arizona (UA). Users of these resources are part of a collaboration project between USDA-ARS-SWRC and UA.

This guide is fully based on the official support Wiki from Cyverse-GIS that can be found here.

The steps described below have been tested using an account in a Unix server environment. Users connect to it by using VNC (Virtual Network Connection) software to access a graphical user interface (GUI).

> **Note:** Non-Server users could follow these steps from any terminal or command-line where Secure Shell (SSH) is installed. The main reasons I decided to run these test from Unix server are:
> 1. Photoscan license available in UNIX-Server.
> 2. X-Window configuration is already set for all users in Server.
> 3. Other software (e.g. SSH tools, Cisco VPN) already available for every user in UnixServer (`ssh`, `ssh-keygen`, `ssh-copy-id`, `sshfs`, etc.).

*Screencasts are included in the original guide to support users that are not familiarized with some of the concepts mentioned.*

---

## 1. Requirements

* UA-NetID (`your_username@email.arizona.edu`)
* Passcodes (Numbers sent by message by the UA for 2-factor-authentication)
* Access to a terminal/command line with Secure Shell software (SSH) installed
* A folder ready with all the drone imagery and photoscan project. These need to be transferred to UA-HPC.

---

## 2. Connecting to UA-VPN from Unix server

In this step you need to open the software called Cisco AnyConnect VPN and connect to `vpn.hpc.arizona.edu`. In the Unix server desktop you will find an icon named “Cisco Anyconnect Secure Mobile”.

Type `vpn.hpc.arizona.edu` in the input text box in case it is not already there. Then, click connect and type username, password, and passcode.

> **Note:** The 2-factor authentication requires the use of passcodes, thus Cisco interface will indicate the initial number of the passcode you need to use. This number will show up in the bottom part (information bar) of the window (i.e. the section where it says `Unable to contact localhost.`). These passcodes are commonly sent by text message.

---

## 3. Opening a Terminal

Once the connection to the UA-VPN is ready it is time to test the connection to the UA-HPC servers. Open the Terminal software or any other software that provides access to the console/command line. Terminal is just a name commonly used in Unix-like systems to identify the command line tool.

---

## 4. Set up UA-credentials

> *This is a one-time-only step, you will NOT do this every time you want to process some imagery.*

### Why do you need this?
To save time and avoid password and passcode requests every time you need to access a UA-HPC server. Instead, you will type your UA-NetID password and passcode once and that is it. For the following ssh connections you should not get password/passcode requests. Your credentials are divided into public and private, the public part is sent to the UA-HPC and the private stays in your user’s folder.

### 4.1 Generating credentials
This is for creating public and private keys. Leave default values and just hit enter at all of the questions. From the user command line type:
```bash
ssh-keygen -t rsa
```

### 4.2 Connect to the HPC server
Next, connect to the HPC server using ssh:
```bash
ssh geponce@hpc.arizona.edu
```
After some messages are displayed you will get the question: `Are you sure you want to continue connecting (yes/no)?`, just type `yes`. Then you get the password request, type your UA-NetID password. Next, if your password was correct, you need to type the passcode.

If everything went smoothly, you should be connected at the gatekeeper server. Just type `exit` to logout from this server and continue with the next step.

### 4.3 Copy public key to the UA-HPC
After you exited from gatekeeper HPC server you will be at your Unix server command line, where you need to type the following:
```bash
ssh-copy-id -i ~/.ssh/id_rsa geponce@hpc.arizona.edu  
```
After this, your public key was copied into your UA-HPC user and now you should be ready to make further ssh connections without having to type your password and passcode.

---

## 5. Transfer data to UA-HPC

The folder with imagery and Photoscan project should be ready to be copied into UA-HPC.

* The location `/xdisk/username` needs to be created in Unix Server just once. Check the availability of the folder by typing `ls /xdisk/username`. If you can’t list it, please request its creation.
* All the projects can live within that folder since this is just a shared file system in the UA-HPC.
* Users should copy the project folder into this path.

Type the following line in your UnixServer terminal to mount the file system:
```bash
sshfs -o sshfs_debug geponce@filexfer.hpc.arizona.edu:/xdisk/geponce /xdisk/geponce
```

Now you can copy your project folder to your `/xdisk/username` and automatically everything that you copy into it will be sent to the UA-HPC:
```bash
cp /home/username/DATA/project1 /xdisk/geponce/
```
Folder `project1` is copied (sent to UA-HPC) to `/xdisk/geponce/` network file system.

This path (`/xdisk/geponce/`) will be used in the Photoscan Preferences in the Network tab. Photoscan will point to this `/xdisk/username` folder and the physical location of that folder will be at the UA-HPC.

> **Note:** Where should I start/store my project at? E.g., If the project is stored in `/username/projects/proj1/`, Photoscan will save that path in the metadata of the project. Thus, if the project is moved into `/xdisk/username` for doing the heavy lifting processing, Photoscan will look for the former path. Therefore, metadata should be checked before starting the HPC processing to make sure that the references to file paths are correct.

---

## 6. Request a node for Photoscan Server (orchestrator)

Next step is to request/initiate the Photoscan Server. This is a node/computer that will work as the orchestrator of the processing, coordinating the access to Photoscan at each CPU/GPU.

First, open a terminal and connect to HPC-Server (gatekeeper):
```bash
ssh -X geponce@hpc.arizona.edu
```
*(If you did the credentials setup steps, you should not get the password/passcode request).*

Once you are connected type this to request access to Ocelote:
```bash
ocelote -X 
```

At the Ocelote command line, type the following line to request the server node:
```bash
qsub -X -I -N photoscan-server-node-1 -m bea -W group_list=tswetnam -q standard -l select=1:ncpus=12:mem=32gb -l cput=72:0:0 -l walltime=72:0:0 -l place=pack:shared
```
You might need to wait a minute or two before getting the resource access granted. Some parameters have been modified to increase resource reservation time (e.g., `walltime`), in case processing goes beyond a day or two.

---

## 7. Loading Photoscan Pro software for the Server

Once you get access to the server node, you will be at the command line of the server you just requested. Here you need to activate/load the Photoscan Pro License:
```bash
module load photoscan-pro/1/1.4.3
```
> **Note:** For some reason, the first time you try to load the license you might get an error message; try again and it should work.

---

## 8. Start Photoscan as a Server (orchestrator)

In this step, you need to identify the name of the server assigned to you (found on the left side of the cursor in the command line).

Run this line to activate the Photoscan Server (orchestrator):
```bash
# Change the name of the server by the one you got
photoscan.sh --server --dispatch i7n17.ocelote.hpc.arizona.edu --control i7n17.ocelote.hpc.arizona.edu --root /xdisk/geponce
```
The server domain will be different depending on what server was assigned to you. Append `ocelote.hpc.arizona.edu` to your assigned node name. The final part of the command is your folder name (`/xdisk/username`) previously configured.

---

## 9. Request GPU node

Now, it’s time to request nodes with GPU capability. Open a **new** terminal (do not close the one from the previous step with the server).

Run this in the new terminal to connect to gatekeeper:
```bash
ssh -X geponce@hpc.arizona.edu
```

Get into Ocelote:
```bash
ocelote -X 
```

Run the following `qsub` command to request a GPU node:
```bash
qsub -I -X -N jobtest -W group_list=tswetnam -q windfall -l select=1:ncpus=28:mem=196gb:ngpus=1 -l cput=1344:0:0 -l walltime=72:0:0 
```
The node reservation time (`cput`) is set for 2 days ($28 \text{ cpus} \times 48 \text{ hrs}$).

---

## 10. Load Photoscan @ Node

At the command line of the assigned node/gpu, load the Photoscan license:
```bash
module load photoscan-pro/1/1.4.3
```
*(Remember to run this line twice if you get an error message the first time).*

---

## 11. Start Photoscan as a Node (GPU)

The final step in setting up the GPU node is to start it as a Photoscan Pro node by running:
```bash
photoscan.sh --node --dispatch i7n17.ocelote.hpc.arizona.edu --capability any --cpu_enable 1 --gpu_mask 1 --root /xdisk/geponce
```
*Do not forget to change the server name to match your assigned node.*

> **Note:** You will need a new request for every GPU node you want. By switching between terminal tabs, you can monitor the Server/Orchestrator and all active GPU nodes. Right now (Jan/2019), the maximum number of GPU nodes you are able to request is 10.

---

## 12. Start Photoscan Network Monitor

At the UnixServer user’s Desktop, find the program `Photoscan_Monitor`. This will start the software allowing you to monitor the activity of the GPU network. Connect to the Photoscan Server (orchestrator) to view active servers and GPU nodes working in progress.

---

## 13. Testing from the Photoscan Pro GUI

Open the Photoscan GUI in Unix Server (using UnixServer Photoscan License), open a project, and run the process **“Build Dense Point Cloud”** over the established network setup.

---

## 14. Comments & Remarks

* Adding preprocessing steps to the “Build Dense Point Cloud” workflow (e.g., alignment with RTK) will further improve this document.
* Documenting additional workflows for generating other end-products beyond point clouds would be useful.
* **File paths investigation:** Investigate how to configure projects with relative file paths instead of absolute paths in metadata. When files are copied to UA-HPC (`/xdisk/username`), Photoscan looks for imagery in the original local path unless adjusted for distributed computing.
* While these steps involve command-line typing, this workflow is currently the optimal choice for processing large volumes of imagery across multiple flights.
* **Performance Benchmark:** This process took ~1 day for building a "Dense Point Cloud" with a project of 2,000 photos at High quality and mild depth filtering, compared to 7 to 10 days using a single computer.
