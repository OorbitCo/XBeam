##### Oorbit Inc. ©2024

# XBeam™  


### Quick Setup Guide (v4.24.0.1)


##### Welcome to the XBeam™ access source code and infra Auto-Installer.

With the code in this project, you can build the FrontEnd requirements of XBeam™, for a variety of targeted platforms, including desktop, mobile, embedded links (social media apps) and LG TV platform (TBA). Automatically install and deploy all infrastructure requirements on your AWS clusters without the need to code or DevOps.

Prerequisite:

* AWS account and AWS CLI for windows.
* Latest Go Lang installation.
* Latest Pulumi for windows.
* Git for windows
* RabbitMQ Installation
* SMB enabled network drive 
* Requires a domain with Wildcard SSL certificate
* UE5 Windows or Linux games with ‘PixelStreamingPlugin’ enabled
* Windows PC to run the installation code


##### Follow these steps to setup your own Cloud Streaming service:

1. Create an AWS account (if you don't have one yet)

2. Download and install AWS CLI for windows, follow this link: [Getting Started Install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

3. Install GO Lang: [Go Installer](https://go.dev/dl/go1.22.2.windows-amd64.msi)

4. Install Pulumi: [Pulumi Installer](https://github.com/pulumi/pulumi-winget/releases/download/v3.113.0/pulumi-3.113.0-windows-x64.msi)

* **Note: How to setup AWS credentials to use with Pulumi:** [Installation Configuration](https://www.pulumi.com/registry/packages/aws/installation-configuration/)
 

5. Make sure you have installed git for windows: [Github Desktop Installer](https://central.github.com/deployments/desktop/desktop/latest/win32)

6. Clone Oorbit XBeam repository: 
Run these commands in Windows terminal:
```powershell
git clone https://github.com/OorbitCo/XBeam
git submodule update --init
cd WorkloadIaaC
```

7. Edit [Pulumi.dev.yaml](https://github.com/OorbitCo/WorkloadIaaC/blob/master/Pulumi.dev.yaml) based on your needs. (e.g `eks:adminUsername` , `eks:accountId` , `worker:windowsPassword`, etc…)


8. run this command in windows terminal: 
```powershell
pulumi up --config-file .\Pulumi.dev.yaml
```
* **Note: this process may take up to 60 minutes. (Wait until finished before moving to next step)**

9. Setup and deploy SMB enabled network drive:
The network drive must be set up and present within the same region as this cluster, otherwise will result in severe throughput issues and long spin up times for the games.
Minimum required IOPS: 64000
Minimum required bandwidth: 5 Gbits (depending on required concurrency)
Make sure port 445 is open. (OS firewall and AWS cluster security group)


10. Install and deploy RabbitMQ on AWS (**DO NOT use active MQ**)
(Refer to AWS documentation)

11. Once you have installed step 9 and 10 continue to windows terminal, change directory to `WorkloadInstaller` folder and run the following commands: 
```powershell
go build -o installer.exe .
Installer.exe -k ..\WorkloadIaaC\kubeconfig -s smb://username:password@ip/sharename -c domain-pub.cert -p domain-priv.key -r amqps://username:password@hostname -g region
```
***Usage Notes***
```
-s smb://username:password   #Username and password to access network drive
@ip/sharename     #Ip address for the network drive / shared folder name: e.g mygames
(shared folder name should not contain SPACE,symbols, or unicode characters)
-c domain-pub.cert    #SSL Public cert file local path for your domain
-p domain-priv.key     #SSL Private cert file local path for your domain
-r amqps://username:password@hostname  #RabbitMQ User/Pass and Hostname
-g region                   #Desired AWS region to setup the Kubernetes cluster.
```
Full command example:
```powershell
Installer.exe -k c:\users\oorbit\desktop\XBeam\WorkloadIaaC\kubeconfig -s smb://shareduser:oorbitpassword@217.219.65.84/mygames -c oorbit-pub.cert -p oorbit-priv.key -r amqps://oorbitbroker:oorbitpasword@b-94d8e31a-243c-4c29-a87a-6bb8b7822f3e.mq.us-west-2.amazonaws.com -g us-east-1
```

13. Upload UE5 pixel streaming enabled games to your SMB enabled network drive. 

14. To start a game, publish the Json below on `create.pod` exchange in your RabbitMQ. 
Set your desired AWS region as the `routing-key`.
Use the following example path `Games/Linux/Spaceship/SpaceshipCruiser.sh` that is relative to the shared directory, e.g `mygames`.

```json
{
 "os": "linux",
 "type": "shared",
 "request": {
   "args": "-ForceRes -ResX=1920 -ResY=1080 -RenderOffscreen -Unattended -AudioMixer -PixelStreamingURL=ws://127.0.0.1:8888 -AllowPixelStreamingCommands=false -PixelStreamingH264Profile=BASELINE -PixelStreamingWebRTCMaxBitrate=30000000 -PixelStreamingEncoderMultipass=DISABLED -dx11",
   "path": "/smb/Games/theone/Linux/sd2/SpaceshipCruiser.sh",
   "custom_args": "-x -y",
   "env": [
     {
       "key": "X",
       "value": "Y"
     }
   ]
 },
 "username": "theone",
 "source": "browser",
 "duration": 60,
 "pass_id": "243283fa71ec4c858244b834e9deae4c",
 "app": "Fruit_Zombies",
 "preview": false,
 "streamer": "pre5",
 "resource": {
   "type": "dynamic",
   "cpu_min": 1000,
   "cpu_max": 1600,
   "memory_min": 1000,
   "memory_max": 1000
 }
}
```

***Usage Notes***  
Parameter Details:   
`os` : game-compatible operating system. Valid values are `windows`,`linux`  
`type`: type of the application, Valid value `shared`  
`request`: an object that refers to game details such as launch arguments , game path, custom arguments, custom environment variables  
`username`: `username`   
`source`: tracking accessed platforms, for example, such as Web,Mobile,Ads, etc …   
`duration` : maximum duration of the game , (will be used in future versions)   
`pass_id` : unique id of the session (should only contain a-zA-Z)  
`app` : name of the game  
`preview` : reserved for future version  
`streamer`: Unreal compatible streamer, available values are : (`post5`,`pre5`)  
`resource`: resource limits for the game  
`resource.type` : sets compute availability limits , available values are : `static`,`dynamic`    
`static`: will use values set in `cpu_min` / `cpu_max` and `memory_min` / `memory_max`      
`dynamic`: uses predefined values set by Oorbit infra.  
GPU resources are managed by Oorbit infra.  

15. Establish connection to a running game: 
Listen to `on.pod.created` Queue in RabbitMQ, this will provide the IP and port of the running game.
For a successful stream, the IP and port must be converted to a URL via the domain provided in step 11. 
(you can use any conversion service such as [sslip.io](https://sslip.io/) for easy, on the fly conversion)

16. The game will auto stop on two conditions:
	60s timer if no connection is detected.
60s after the stream has been disconnected.



***This Quick Guide is updated frequently, please check time to time for new features and changes.***



Any questions or further assistance please contact us: support@oorbit.com


	
