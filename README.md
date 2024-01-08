# icloud2-lcbo

Downloading iCloud Photo Library to your Synology for Backup
June 21, 2022
Thanks to Tom Insam for the inspiration

What are we attempting?
This takes all the photos out of your iCloud Photo Library and syncs them to your Synology. You end up with exactly the same photos on your synology volume as you do in iCloud. It’s important to note that this is a sync — if you delete a photo from iCloud, it’ll also delete from your Synology. This is done by watching the ‘deleted’ folder in iCloud, so if a photo goes missing without appearing in your deleted folder it will remain.

Updated August 11, 2023: I’ve modified this guide to use a virtual environment instead of installing as root. It should run much more reliably and be less likely to break with DSM updates now. You also no longer need sudo.




Why would I do this?
Having your entire photo library in iCloud is a mild risk. It means your photos aren’t stored in a recognizable, hierarchical folder format and your access to the library depends on Apple continuing to offer a reliable service. Any cloud storage that syncs comes with a risk that all your photos could disappear suddenly — this prevents that from happening. If you’re a digital pack rat with every photo you’ve ever taken, this is for you.




Other notes from me:
For even more absurd redundancy, you can use Synology’s Glacier backup to copy your entire photo library up to AWS Glacier for cheap, offsite replication.

These instructions assume you’re using a Mac, but Windows users should be able to accomplish this with WSL (Windows Systems for Linux) terminal commands.

You probably shouldn’t empty the deleted photos folder more frequently than you run this script or it will leave photos on your Synology that you’ve removed from iCloud




Ingredients
iCloud Photo Library

Synology with plenty of extra storage space

Recommended: Not a slow CPU

DSM 7.0 or later

Your Synology’s local IP or domain (something like <synologyname>.local)

Your Synology username and password

Familiarity with ssh, the terminal in general, and other computer inner workings




Instructions
0) Before you Begin
Allow SSH through your Synology firewall (only if your firewall is on)

Open the Control Panel

Go to Security > Firewall > Edit Rules

Make sure ‘Encrypted Terminal Service’ appears on one of the entries with an action of ‘allow’

Turn on SSH

Open the Control Panel

Go to Terminal & SNMP

Check ‘Enable SSH Service’

Click ‘Apply’

SSH to Synology from your local machine

Open Terminal on your computer

Enter the command `ssh <synologyusername>@<synology domain or IP>’ as in ‘ssh sam@harddrive.local'

Enter your synology password when prompted

You’ll see a slightly different command prompt

1) Install and set up Python
It’s probably already done for you!

Check that it is by typing python -V into your terminal and pressing enter

The result needs to be at least 3.6 (Mine was 3.8.8 which I believe is included with DSM 7.1)

Create a virtual environment in which to do all of the further work. This effectively sections off the python configuration being used for this project from anything else on your Synology, making sure different work you (or DSM) does won’t impact the iCloud Photos work. It’s recommended

Change to your user’s home directory by running cd ~

Create a temp directory for use later by running mkdir $HOME/tmp

Create the virtual environment using python3 -m venv icloudpd

Navigate into that folder with cd icloudpd

Activate the virtual environment by source bin/activate

Note: If you experience any disconnections after this point, re-activate your virtual environment using cd ~/icloudpd;source bin/activate

In order to install more Python packages, we need to install an up-to-date version of pip, the Python package manager

I followed this great guide to enable pip

Run the following commands in your terminal

python -m ensurepip

python3 -m pip install --upgrade pip

The order of this step is very important — we need to install WHEEL first

Run the following commands in your terminal

pip install wheel

pip install --upgrade wheel

2) Install the iCloud Photo Downloader
icloudpd is available on GitHub

Run pip install icloudpd

3) Set up iCloud authentication
Run icloud —username <youremailaddress>

Enter your real iCloud password when prompted — app-specific passwords do NOT work because it is the API

Two factor users will get prompted for your code. For some reason this defaults to the SMS fallback (if you’re still not using MFA on iCloud, you really should be)

4) Get your Synology ready
Return to the DSM web user interface but do not disconnect your terminal yet

Create a dedicated folder for your photos

Copy the path to your folder out to a note somewhere and save it for the next step

Remember that you need to add /volume1 (or a different number if you have more than one volume) to the beginning of your path

5) Run some tests
Return to your terminal. If your computer went to sleep at any point, you may have to ssh to your Synology again. Remember to reactivate your virtualenv

Tell your subsequent work to use that temp directory we created earlier. If you don’t do this, everything will break. export TMPDIR=$HOME/tmp

Run icloudpd -d <path_to_your_folder> -u <iCloud_username> --recent 10 --only-print-filenames --threads-num 2

             If it fails — if you have spaces in your path, you’ll need to enclose in single quotes

See if it succeeds (it should print 10 filenames)

Try downloading a few dozen photos

Run icloudpd -d <path_to_your_folder> -u <iCloud_username> --recent 50 --set-exif-datetime --auto-delete --threads-num 2

Check your folder using the Synology file browser to see if the photos are there and whether they open correctly

6) Set up scheduling
In the Synology web UI, open Synology > Control Panel > Task Scheduler

Create an upgrade script to keep the iCloud Photos Downloader up to date

Click Create > Scheduled Task > User-defined Script

Fill out each tab as follows:

General: Give it a name, run as your user

Schedule: your preference, I chose once a month

Task Settings

Notification: Your email here, only if abnormal termination (if you want)

Command: cd icloudpd; source bin/activate; python -m pip install --upgrade icloudpd

Note: This is actually three commands in one — it changes to the icloudpd folder that we installed the python package in, activates the virtual environment, then does the upgrade

Highlight the resulting entry and click ‘run’

Click ‘Action’ > ‘View Result’ and ensure that the Current Status is Normal (0) or Requirement already satisfied…

Set up the script itself

Click Create > Scheduled Task > User-defined script again

Fill out each tab as follows:

General: Give it a name, run as your user

Schedule: Your preference, I chose daily at 4am

Note that this may occasionally take a long time to run, frequencies <1 day aren’t recommended

Because of the way this script tracks (and removes) deleted photos, it must be run at a frequency less than 30 days — deleted photos are pruned from iCloud after 30 days.

Task Settings

Notification: Same as above

Command: export TMPDIR=$HOME/tmp; cd icloudpd; source bin/activate; icloudpd -d /volume1/'Media Storage'/Pictures/'iCloud Photo Library' -u samgross144@gmail.com --set-exif-datetime --no-progress-bar --auto-delete --threads-num 2 --log-level error --recent 10 --only-print-filenames --dry-run

7) (Optional) Save logs which is helpful
Open Synology > Control Panel > Task Scheduler > Settings

Check ‘save output results’ and give it a folder that you are ok with filling up

Click ok

8) Final Test
Highlight the upgrade task by clicking on it and then click Run

Select Action > View Result to see terminal output

Make sure that the current status says ‘Normal (0)’

Highlight the download task and then click Run

This will download your entire iCloud Photo Library don’t do this when you need your bandwidth. My Synology maxed out at about 10Mbps download, a fraction of my available speed

Select Action > View Result to see terminal output

You should see ‘current status: running’ with no data as the output

9) Clean Up
Quit your terminal app

Turn off SSH on the Synology (remember to hit ‘apply’)

Change log level if desired in the task settings by editing your scheduled task and replacing the last word (info) with error

Limit the comparisons to recent photos only if you’d like by adding —-recent 1000 to the end of your configuration (you can change the number if desired)
