Proof of Concept
Moving a Website from Windows Server 2016 to Windows Server 2019
Prepared for	ezAtlas Pvt Ltd
Environment	AWS EC2 (Windows Server)
Application type	ASP.NET / IIS website
Date	10 September 2026
1. What This Document Covers
This document shows how a website was moved from an old server (Windows Server 2016) to a new server (Windows Server 2019), without reinstalling the website from scratch. Instead of a full rebuild, the website's files were carried over by moving the server's storage disk from the old server to the new one.
2. How It Was Done, in Short
●	Set up the website on the 2016 server and confirm it works.
●	Turn off the website and shut down the 2016 server.
●	Remove the storage disk from the 2016 server.
●	Attach that same disk to the new 2019 server as an extra disk.
●	Copy the website files from that extra disk onto the 2019 server's main disk.
●	Set up the website software on the 2019 server and confirm it works there too.
●	Remove the old disk once everything is confirmed working.
3. Setting Up the Website on the 2016 Server
Step 1: Start the 2016 server
A Windows Server 2016 machine was started on AWS, and accessed remotely.
Step 2: Install the website software (IIS)
Install-WindowsFeature -Name Web-Server,Web-Asp-Net45,Web-ISAPI-Ext,Web-ISAPI-Filter,Web-Mgmt-Console -IncludeManagementTools
This installed IIS, which is the software Windows uses to run websites. It completed successfully.
Step 3: Check the .NET version
The server already had the correct .NET version (4.8) installed, so nothing extra was needed here.
Step 4: Create a folder for the website
New-Item -Path "C:\inetpub\wwwroot\YourApp" -ItemType Directory -Force
The website files were kept on the C: drive (the main drive) so that the whole drive could later be moved to the new server.
Step 5: Set up the website in IIS
Import-Module WebAdministration
New-WebAppPool -Name "YourAppPool"
Set-ItemProperty IIS:\AppPools\YourAppPool -Name managedRuntimeVersion -Value "v4.0"

icacls "C:\inetpub\wwwroot\YourApp" /grant "IIS AppPool\YourAppPool:(OI)(CI)RX"
icacls "C:\inetpub\wwwroot\YourApp" /grant "IIS AppPool\YourAppPool:(OI)(CI)M"

Stop-Website -Name "Default Web Site"
New-Website -Name "YourSite" -PhysicalPath "C:\inetpub\wwwroot\YourApp" -ApplicationPool "YourAppPool" -Port 80 -Force
Start-Website -Name "YourSite"
Note: the default site that comes with IIS had to be turned off first, because it was already using the same address (port 80) that our new site needed.
Step 6: Allow the website through the firewall
New-NetFirewallRule -DisplayName "Allow HTTP 80" -Direction Inbound -LocalPort 80 -Protocol TCP -Action Allow
The AWS security settings for this server were also updated to allow web traffic in.
[ SCREENSHOT: AWS security settings allowing web traffic on this server ]
Step 7: First attempt showed an error
When the website address was opened in a browser, it showed a "403 Forbidden" error. After checking, the reason was simple: the website folder was empty, since no files had been copied into it yet.
 
The error seen before any website files were added to the folder.
A simple test page was added to the folder to confirm everything else was working correctly:
Set-Content -Path "C:\inetpub\wwwroot\YourApp\index.html" -Value "<html><body><h1>Hello from Windows Server 2016 - IIS is working!</h1></body></html>"
Step 8: Website confirmed working on the 2016 server
After adding the test page, the website loaded correctly in the browser.
[ SCREENSHOT: Browser showing the working website on the 2016 server ]
4. Moving the Disk from the 2016 Server to the 2019 Server
Step 9: Turn off the website and stop the 2016 server
Stop-Website -Name "YourSite"
The 2016 server was then stopped completely from the AWS console. A server's main disk can only be removed while the server is stopped.
 
AWS console showing both the 2016 and 2019 servers, stopped.
Step 10: Move the disk to the 2019 server
The disk was removed from the 2016 server and attached to the new 2019 server as a second, extra disk.
[ SCREENSHOT: AWS console showing the disk being attached to the 2019 server ]
Step 11: Switch the disk on inside Windows
Get-Disk
Set-Disk -Number 1 -IsOffline $false
Get-Partition -DiskNumber 1
The disk showed up as "Disk 1" and was initially switched off. It was switched on, and Windows automatically gave it the drive letter D: (since C: was already taken by the 2019 server's own main disk).
Step 12: Confirm the website files made it across
Get-ChildItem "D:\inetpub\wwwroot\YourApp"
Result: the test page from the 2016 server was visible on the 2019 server, through drive D:. This confirmed the disk move worked.
5. Copying the Website to the 2019 Server's Main Drive
Step 13: Copy the files from D: to C:
New-Item -Path "C:\inetpub\wwwroot\YourApp" -ItemType Directory -Force
Copy-Item -Path "D:\inetpub\wwwroot\YourApp\*" -Destination "C:\inetpub\wwwroot\YourApp" -Recurse
Get-ChildItem "C:\inetpub\wwwroot\YourApp"
Result: the website files were successfully copied onto the 2019 server's own main drive.
Step 14: Set up the website software on the 2019 server
Install-WindowsFeature -Name Web-Server,Web-Asp-Net45,Web-ISAPI-Ext,Web-ISAPI-Filter,Web-Mgmt-Console -IncludeManagementTools
Import-Module WebAdministration
New-WebAppPool -Name "YourAppPool"
Set-ItemProperty IIS:\AppPools\YourAppPool -Name managedRuntimeVersion -Value "v4.0"

icacls "C:\inetpub\wwwroot\YourApp" /grant "IIS AppPool\YourAppPool:(OI)(CI)RX"
icacls "C:\inetpub\wwwroot\YourApp" /grant "IIS AppPool\YourAppPool:(OI)(CI)M"

Stop-Website -Name "Default Web Site"
New-Website -Name "YourSite" -PhysicalPath "C:\inetpub\wwwroot\YourApp" -ApplicationPool "YourAppPool" -Port 80 -Force
Start-Website -Name "YourSite"
This step had to be redone on the 2019 server because the website software (IIS) and its settings are part of the operating system, not the disk. Only the website's own files came across with the disk move.
Step 15: Website confirmed working on the 2019 server
The AWS security settings for the 2019 server were updated to allow web traffic, and the website was tested in a browser.
 
The website loading successfully from the 2019 server.
6. Cleaning Up
Once the website was confirmed working correctly on the 2019 server, the old disk (originally from the 2016 server) was removed and deleted from the 2019 server, since it was no longer needed.
[ SCREENSHOT: AWS console showing the old disk removed ]
7. Result
The website was successfully moved from the 2016 server to the 2019 server without rebuilding it from scratch. The website's files travelled across using the disk move; only the website software itself (IIS) had to be reinstalled on the new server. This shows that moving the disk is a safe and workable way to move a website to a new server.
8. Points to Remember for a Real Migration
●	Take a backup (snapshot) of the disk before removing it, so the old server can be recovered if something goes wrong.
●	If the real website is more complex than this test (for example, if it uses a database), those parts need to be set up and tested separately on the new server too.
●	Double-check file permissions after any move between servers, since they don't always carry over cleanly.
●	Make sure the new server has the correct .NET version installed before going live.
