# "Roll Your Own EDR/XDR" Workshop



This guide will provide you with a step-by-step of all the commands we will use throughout this workshop. Please reference it as we move forward. If you have questions, feel free to ask your group moderator.

## Lab 1: Building the Lab

In this lab we will actually be building the lab environment we will be using Virtual Box and Windows. 

### Tools we will use:

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Microsoft Windows 10 Enterprise Evaluation Copy](https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise) (download 64 bit)
- [LimaCharlie Docs](https://docs.limacharlie.io/docs)

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

1. [Create a free LimaCharlie account](https://free.limacharlie.io) (please select BSides Austin for the first option, feel free to use a burner email).


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

3. Name the rule "First Detection - [YOUR FIRST AND LAST NAME] "

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

Save your new detection make sure you add [YOUR FIRST AND LAST NAME]  in the name. 

4. Go into your Windows Virtual Machine in VirtualBox and disable Windows Defender (search Defender in the search box in the main nav)

## Lab 5: Setting up YARA Scans

We will setup a single YARA rule that will run on the endpoint at regular intervals. 

1. In your LimaCharlie account go to "Automation -> YARA Rules" and click on the "+ Add Yara Rule" button

2. You can write your own YARA rule, or leverage rules built by the community, in this exercise we will use [Florian Roth's God Mode YARA rule](https://github.com/Neo23x0/god-mode-rules/blob/master/godmode.yar).

3. In the name field enter "God Mode" and then copy and paste the YARA rule:

```yara

/*
      _____        __  __  ___        __      
     / ___/__  ___/ / /  |/  /__  ___/ /__    
    / (_ / _ \/ _  / / /|_/ / _ \/ _  / -_)   
    \___/\___/\_,_/_/_/__/_/\___/\_,_/\__/    
     \ \/ / _ | / _ \/ _ |   / _ \__ __/ /__  
      \  / __ |/ , _/ __ |  / , _/ // / / -_) 
      /_/_/ |_/_/|_/_/ |_| /_/|_|\_,_/_/\__/  
   
   Florian Roth - v0.8.1 August 2024 - Merry Christmas!

   The 'God Mode Rule' is a proof-of-concept YARA rule designed to 
   identify a wide range of security threats. It includes detections for 
   Mimikatz usage, Metasploit Meterpreter payloads, PowerShell obfuscation 
   and encoded payloads, various malware indicators, and specific hacking 
   tools. This rule also targets ransomware behaviors, such as 
   shadow copy deletion commands, and patterns indicative of crypto mining. 
   It's further enhanced to detect obfuscation techniques and signs of 
   advanced persistent threats (APTs), including unique strings from 
   well-known hacking tools and frameworks. 
*/

rule IDDQD_God_Mode_Rule {
   meta:
      description = "Detects a wide array of cyber threats, from malware and ransomware to advanced persistent threats (APTs)"
      author = "Florian Roth"
      reference = "Internal Research - get a god mode rule set with THOR by Nextron Systems"
      date = "2019-05-15"
      modified = "2024-01-12"
      score = 60
   strings:
      $ = "sekurlsa::logonpasswords" ascii wide nocase           /* Mimikatz Command */
      $ = "ERROR kuhl" wide xor                                  /* Mimikatz Error */
      $ = " -w hidden " ascii wide nocase                        /* Power Shell Params */
      $ = "Koadic." ascii                                        /* Koadic Framework */
      $ = "ReflectiveLoader" fullword ascii wide xor             /* Generic - Common Export Name */
      $ = "%s as %s\\%s: %d" ascii xor                           /* CobaltStrike indicator */
      $ = "[System.Convert]::FromBase64String(" ascii            /* PowerShell - Base64 encoded payload */
      $ = "/meterpreter/" ascii xor                              /* Metasploit Framework - Meterpreter */
      $ = / -[eE][decoman]{0,41} ['"]?(JAB|SUVYI|aWV4I|SQBFAFgA|aQBlAHgA|cgBlAG)/ ascii wide  /* PowerShell encoded code */
      $ = /  (sEt|SEt|SeT|sET|seT)  / ascii wide                 /* Casing Obfuscation */
      $ = ");iex " nocase ascii wide                             /* PowerShell - compact code */ 
      $ = "Nir Sofer" fullword wide                              /* Hack Tool Producer */
      $ = "impacket." ascii                                      /* Impacket Library */
      $ = /\[[\+\-!E]\] (exploit|target|vulnerab|shell|inject)/ nocase  /* Hack Tool Output Pattern */
      $ = "0000FEEDACDC}" ascii wide                             /* Squiblydoo - Class ID */
      $ = "vssadmin delete shadows" ascii nocase                 /* Shadow Copy Deletion via vssadmin - often used in ransomware */
      $ = ".exe delete shadows" ascii nocase                     /* Shadow Copy Deletion via vssadmin - often used in ransomware */
      $ = " shadowcopy delete" ascii wide nocase                 /* Shadow Copy Deletion via WMIC - often used in ransomware */
      $ = " delete catalog -quiet" ascii wide nocase             /* Shadow Copy Deletion via wbadmin - often used in ransomware */
      $ = "stratum+tcp://" ascii wide                            /* Stratum Address - used in Crypto Miners */
      $ = /\\(Debug|Release)\\(Key[lL]og|[Ii]nject|Steal|By[Pp]ass|Amsi|Dropper|Loader|CVE\-)/  /* Typical PDB strings found in malware or hack tools */
      $ = /(Dropper|Bypass|Injection|Potato)\.pdb/ nocase        /* Typical PDP strings found in hack tools */
      $ = "Mozilla/5.0" xor(0x01-0xff) ascii wide                /* XORed Mozilla user agent - often found in implants */
      $ = "amsi.dllATVSH" ascii xor                              /* Havoc C2 */
      $ = "BeaconJitter" xor                                     /* Sliver */
      $ = "main.Merlin" ascii fullword                           /* Merlin C2 */
      $ = "\x48\x83\xec\x50\x4d\x63\x68\x3c\x48\x89\x4d\x10" xor /* Brute Ratel C4 */
      $ = "}{0}\"-f " ascii wide                                 /* PowerShell obfuscation - format string */
      $ = "HISTORY=/dev/null" ascii                              /* Linux HISTORY tampering - found in many samples */
      $ = " /tmp/x;" ascii                                       /* Often used in malicious linux scripts */
      $ = /comsvcs(\.dll)?[, ]{1,2}(MiniDump|#24)/               /* Process dumping method using comsvcs.dll's MiniDump */
      $ = "AmsiScanBuffer" base64 base64wide                     /* AMSI Bypass */
      $ = "AmsiScanBuffer" xor(0x01-0xff)                        /* AMSI Bypass */
      $ = "%%%%%%%%%%%######%%%#%%####%  &%%**#" ascii wide xor  /* SeatBelt */
   condition:
      1 of them
}

```
4. Next we will add the rule to a scanner. In LimaCharlie navigate to "Automation -> YARA Scanners" and click the "+ Add YARA Scanner" 

5. In the drop down you should see the God Mode YARA rule we just created select it. For platforms select "windows" and "linux" and click "Save".

![YARA SCAN](/img/21_yara_scan.png) 

## Bonus Lab 6: Ransomware Domain Threat Intel Detection

Now let's build on what we have learned in the first lab, writing a detection for a single domain is great to test our initial setup, but not particularly practical. Let's add a threat intelligence feed to our LimaCharlie environment and use it in a detection. 

1. In the top navigation click on "Add-ons" and search for "ransomware" you should see a "ransomware-domains" threat list and subscribe to it for your org
![YARA SCAN](/img/subscribe_ransomware_feed.png)


2. Now go back to our org and navigate back to Automation -> D&R Rules and click "New Rule" in the upper right hand corner add [YOUR FIRST AND LAST NAME] to the title of the rule 
3. For our detection this time we will now do a lookup on the ransomware domain threat feed: 

```yaml
event: DNS_REQUEST
op: lookup
path: event/DOMAIN_NAME
resource: 'lcr://lookup/ransomware-domains'

```
4. Let's assume that we have a high confidence in our threat intel feed, if a device contacts a domain we are going to not only trigger an alert, but also isolate the host, in the response section of the detection add: 

```yaml
- action: report
  name: Ransomware Domain Detection Hit
- action: isolate network

```
Your detection should look like this: 
![Ransomware Detection](/img/ransomware_detection.png) 







