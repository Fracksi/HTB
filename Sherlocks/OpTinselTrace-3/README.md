## Overview

OpTinselTrace-3 is a medium difficulty Sherlock from HacktheBox provided for the 2023 Christmas Holiday and to challenge users DFIR skills with festive themed challenges. OpTinselTrace-3 is meant to challenge users with their ability to perform memory forensics on a .bin of a memory dump.


## Sherlock Scenario

Oh no! Our IT admin is a bit of a cotton-headed ninny-muggins, ByteSparkle left his VPN configuration file in our fancy private S3 location! The nasty attackers may have gained access to our internal network. We think they compromised one of our TinkerTech workstations. Our security team has managed to grab you a memory dump - please analyse it and answer the questions! Santa is waiting…


## Tools utilized

Volatility3 - The main tool used in the memory analysis.
VBSedit  - An IDE for VBS scripting. (For anyone interested, every debug step or script run requires a minimum 5 second wait, with +1 seconds added on every additional time.)


## Task Start

#### Question 1
`What is the name of the file that is likely copied from the shared folder (including the file extension)?`

As with a start of anything you need a good foundation. Outside of the general OS information windows.pslist.PsList is a good first step to seeing what's present on the system at the time of capture. Utilzing this function and reading through we can see a suspicious running process, present.exe, that has it's own spawned conhost instance.

[Path1.png goes here]

Pulling the commands that were used with windows.cmdline.CmdLine we can find when the executable was run and spawned the future malicious events, essentially confirming suspicions on this process.

[Path3 goes here]

Following through with the windows.filescan.FileScan for the root of our issues we can grep the results for the process name and come across a couple of files, one of which is our answer to Question 1.

[Path4 goes here]


#### Question 2
`What is the file name used to trigger the attack (including the file extension)?`

Having found the intial download in filescan we dump the zip files and unzip the contents for information as well as our answer to question 2.

[Path5 goes here]


#### Question 3
`What is the name of the file executed by click_for_present.lnk (including the file extension)?`

#### Question 4
`What is the name of the program used by the vbs script to execute the next stage?`

While the answer to Question 3 is obvious from the contents we will prove this outcome with the follow through. We can pull the exif information from the initial file with exiftool (which also answers question 4)

[Path7 goes here]

Looking at the Command Line Arguments we can identify base64 encoding. Decoding this confirms our hypothesized answer for question 3.

[Path8 goes here]


#### Question 5
`What is the name of the function used for the powershell script obfuscation?`

From here I transfered the vbs code into a windows box with VBSedit, N++, and VSCode to do some cleanup. (For whatever reason VScode was better as the regex cleanup than N++ was)

Inside the vbs code we see the start of a function and many lines of vbs comments as well as a couple dozen variable settings.

[VBSDirty goes here]
[VBSDirty2 goes here]

Scrolling through on a quick manual search we also see portions of code append together and do some final runs.

[VBSDirty4 goes here]
[VBSDirty3 goes here]

We can use some basic regex to clean this up and get a better look at what it's doing overall.

[VBSClean&Run3 goes here]
[VBSClean&Run goes here]

And then attempt to safely print what the output would be. Below is a snippet of an area partway through the VBS code. Remember to use clean, isolated VMs you have no issues with destroying in the event you accidentally step through the debugger to far, forget to set a stop point, or just want to see what the end result does to your PC like some would do with the Malware Museum.

We can spot the function asked for in Question 5 as well.

[VBSClean&Run2 goes here]


#### Question 6
`What is the URL that the next stage was downloaded from?`

We see it's creating a third level of obfuscation with a neat bytecode powershell script so we take the entire output and sending that into PS ISE for some final edits of where to put the print output in this script. Doing a bit of manual cleanup we can identify the $Malice call as a good point to put a print request.

[PSDirty2 goes here]
[PSCleaning2 Goes here]

The printed output gives us the part of the call where it downloads another file, and answers question 6.


#### Question 7
`What is the IP and port that the executable downloaded the shellcode from (IP:Port)?`

The answer to this question was a bit of a logic guess. Looking at windows.netstat.NetStat output for the memory dump we find a connection to the same IP address as Question6 on a different port. We also see a few other uncertain IP connections on port 445/TCP. As 445/TCP is for SMB, commonly used in payload file transfers, and can make the educated guess the payload was transfered over SMB as well, as the protocol is allowed.


#### Question 8
`What is the process ID of the remote process that the shellcode was injected into?`

Another missed photo for showing the process more clearly here, however we can somewhat see for this Question and Question 7 with a previous photo.

[Path4 goes here]

With the connection to the C&C server from windows.netstat.NetStat we see the connection was initiated with svchost with pid 724, which is also shown as a spawned process at the bottom of the handles output for pid 3248, present.exe, if I recall correctly. This should be fixed and added in the near future.


#### Question 9
`After the attacker established a Command & Control connection, what command did they use to clear all event logs?`

As the earlier commands ended up finishing in powershell we can check the powershell eventlogs for traces of other commmands run.

[path9 goes here]

Parsing through the recent commands we find a short one-liner meant to loop through and clear the event logs.

[Question9Answer goes here]

#### Question 10
`What is the full path of the folder that was excluded from defender?`

Similar to the Powershell commands we can pull the Windows Defender evtx files and parse through those as well, finding the removed filepath.

[Question10Answer goes here]


#### Question 11
`What is the original name of the file that was ingressed to the victim?`

In our earlier searches we kept glimpsing PresentForNaughtyChild.exe in the search results. We might not pay it much mind until the powershell event searches but it would be amiss to not perform our diligence on this file beforehand.

Same as earlier we utilize exiftool to pull the information from the exe.img file we pull from the memorydump.

[Question11Answer goes here]

We can see it is a sysinternals program and it's Internal Name a few lines below.

#### Question 12
`What is the name of the process targeted by procdump.exe?`

Going back to the event search logs we can find the above executable called by powershell to act upon the lsass.exe process and utilize the stolen_gift.dmp file.

[path12 goes here.]
