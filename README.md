# "Roll Your Own EDR/XDR" Workshop

This guide will provide you with a step-by-step of all the commands we will use throughout this workshop. Please reference it as we move forward. If you have questions, feel free to ask your group moderator.

## Lab 1: Building the Lab

In this lab we will actually be building the lab environment we will be using Virtual Box and Windows. 

### Tools we will use:

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Microsoft Windows 10 Enterprise Evaluation Copy](https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise) (download 64 bit)

1. Download and install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) on you laptop if you do not already have it, install the appropriate version for your operating system.
 ![Download VirtualBox](/img/1_virtual_box.png)

2. Download the [Microsoft Windows 10 Enterprise Evaluation Copy ISO](https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise) (64 bit) and save it in a folder on your system called "EDR_Workshop"
 ![Download Windows](/img/2_windows_download.png)

3. Open VirtualBox and click on "New" to create a new VM

 ![Open VirtualBox](/img/3_add_vm.png)

4. Configure your new VM.
   - For "Folder" navigate to the "EDR_Workshop" folder you created (where the Windows ISO was saved).
   - For "ISO" navigate to the actual Windows ISO you downloaded.
   - The "Edition" should automatically populate.
   - Select "Skip Unattended Installation"
   - Click on "Next"
  
  ![VM Setup](/img/4_vm_setup.png)

5. On the "Hardware" screen bump the memory up to around 8GB if you system has the RAM available (2GB will still work...things will just be slow)

![VM Setup](/img/5_vm_hardware_setup.png) 

6. Leave the "Virtual Hard disk" screen with the defaults.

  ![VM Setup](/img/6_vm_storage_setup.png)

7. On the next screen review and click "Finish" and then click "Start"

8. Go through the install process for Windows

9. When asked to "Sign in with Microsoft" click on "Domain join intead" in the lower left hand corner and create a local username and password.

![VM Setup](/img/7_windows_setup1.png)

10. In VirtualBox go to the top menu and go to "Devices -> Insert Guest Additions CD image"
![VM Setup](/img/7-2_windowsvmsetup.png)

11. Navigate via explorer to the now mounted CD drive and access the "VirtualBox Guest Additions" and double click on "VBoxWindowsAdditions-amd64" and follow the wizard to install and let the Windows system reboot. 

![VM Setup](/img/7-3_windowsvmsetup.png) 

12. Go to the top menu in VirtualBox and go to "Machine -> Take Snapshot" and name it "Base Install"

## Lab 2: Configuring EDR Telemetry in LimaCharlie

We will use the free tier of LimaCharlie for our lab, this will allow us to easily deploy an agent to start gathering telemetry from our Windows system, and then start writing and deploying detection and response rules. 

1. [Create a free LimaCharlie account](https://free.limacharlie.io) (please select BSidesPDX for the first option, feel free to use a burner email).


![VM Setup](/img/8_lc_setup1.png)

![VM Setup](/img/9_lc_setup2.png)

2. Once you have an organization setup in LimaCharlie, log into your LimaCharlie account ***on your Windows Virtual Machine on VirtualBox*** . Then got to your organization and then to "Sensors" and click on "+ Add Sensor" 

![VM Setup](/img/10_sensor_setup1.png) 

3. Next go to "Endpoint" and click on the Windows option. 

![VM Setup](/img/11_sensor_setup2.png) 

4. You will be asked to select an installation key, as this is our first sensor we will need to create one, so click on "Create New", in the "Description" enter "Windows1" then click "Create"

![VM Setup](/img/12_sensor_setup3.png) 

![VM Setup](/img/13_sensor_setup4.png) 

5. Next we will need to select the specific agent for our operating system, in this case we will select the "x86-64(.exe)" option, next we click on the "Download the selected installer" link to start the download to your Windows system. Find the download in your downloads folder and move it to your desktop.

![VM Setup](/img/14_sensor_setup5.png) 

7. Go to your Windows VM and open the Command Prompt and "Run as administrator" navigate to your desktop and run the installer executable from the Command Prompt with the command line argument with the installer key to install the agent.

![VM Setup](/img/16_sensor_setup7.png) 

![VM Setup](/img/17_sensor_setup8.png) 

## Lab 3: Writing Detection & Response Rules from Scratch

1. Log into your LimaCharlie account and go to the menu on the left and navigate to "Automation -> D&R Rules"

2. Click the "+ New Rule" button to create a new detection rule

3. Name the rule "First Detection - Test"

4. In the "Detect" window copy and past this YAML detection code:

```yaml
event: DNS_REQUEST
op: is
path: event/DOMAIN_NAME
value: superevildomain.com
```
In the "Response" window enter this YAML response code:

```yaml
- action: report
  name: DNS Hit superevildomain.com
```

Save the detection and ensure it is enabled. 

5. Go to your Windows Virtual Machine in VirtualBox and open a browser and visit the superevildomain.com 

6. After a minute go to your LimaCharlie account and navigate to "Detections" on the left hand navigation. You should see an alert for your new detection rule.

![VM Setup](/img/18_first_detection_alert.png) 

## Lab 4: Adding Windows Event Logs to Our Telemetry

So now we will create a more useful rule for Windows, we will use our EDR agent to detect if Windows Defender is disabled on a system. However, we will first need to add Windows Event Logs (WEL) to the telemetry we collect from the system. 

1. In LimaCharlie on the left side navigation menu go to "Sensors -> Artifact Collection" and click on the "+ Add Artifact Collection Rule" button and give it the name of "windows-logs"

![VM Setup](/img/19_artifact_collection_rule.png) 

2. In the patterns field we will add several Windows Event Logs that our EDR sensor will send to LimaCharlie which we can then write detections for. Enter each of these in the pattern field:

```yaml
wel://application:*
```

```yaml
wel://security:*
```

```yaml
wel://system:*
```

```yaml
wel://Microsoft-windows-Windows Defender/Operational:*
```

Set the "Platform(s)" field to "windows"  and save your Artifact Collection Rule

![VM Setup](/img/20_artifact_rules.png) 

3. Now we will write a new rule that will detect when Windows Defender is disabled, go back to your D&R Rules (Automation -> D&R Rules) and click "+ New Rule" and enter this YAML for the detection logic: 

```yaml
event: WEL
op: is
path: event/EVENT/System/EventID
value: '5001'
```

Then enter this YAML for a basic reporting response: 

```yaml
- action: report
  name: Windows Defender Malware Disabled

```

Save your new detection. 

4. Go into your Windows Virtual Machine in VirtualBox and disable Windows Defender (search Defender in the search box in the main nav)

## Lab 5: Setting up YARA Scans

We will setup a single YARA rule that will run on the endpoint at regular intervals. 

1. In your LimaCharlie account go to "Automation -> YARA Rules" and click on the "+ Add Yara Rule" button

2. You can write your own YARA rule, or leverage rules built by the community, in this exercise we will use [Florian Roth's "God Mode" rule]([https://www.virtualbox.org/wiki/Downloads](https://github.com/Neo23x0/god-mode-rules/blob/master/godmode.yar)).

 
   
