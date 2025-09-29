# PermX
You can find the machine [here](https://app.hackthebox.com/machines/PermX).
During all this writeup, I'll be using the variable `$TARGET` as the machine IP address, and `$ATTACKER` as the attacker IP address (us).

## Step 1 - Recognition

First of all, we will do some recognition using nmap with the flag `-p-` to discover all the opened ports on the VM :
![Untitled](assets/screenshot_1.png)

I found an HTTP server which redirect me to `http://permx.htb` :
![Untitled](assets/screenshot_2.png)

I add it to my `/etc/hosts` file and go back to the website :
![Untitled](assets/screenshot_3.png)

I search for subdomains using `ffuf` and found the subdomain `lms` :
![Untitled](assets/screenshot_4.png)

I add it to my `/etc/hosts` and go to the website. I found a login page :
![Untitled](assets/screenshot_5.png)

I found that the website use a CMS called `Chamilo`. I search for exploits and found an exploit that does not require authentication :
![Untitled](assets/screenshot_6.png)

I clone the repository and use the exploit to get a reverse-shell :
![Untitled](assets/screenshot_7.png)
![Untitled](assets/screenshot_8.png)

I run linpeas and find a password in a configuration file of the application :
![Untitled](assets/screenshot_9.png)

I search for a user in the `/home` directory and find the user `mtz`. I try to log in with SSH with the credentials found previously `mtz:03F6lY3uXAP2bkW8` :
![Untitled](assets/screenshot_10.png)

And I can finally get the `user.txt` flag.
#### Flag root.txt

I execute the command `sudo -l` to see what command I can execute without password and find a script :
![Untitled](assets/screenshot_11.png)
![Untitled](assets/screenshot_12.png)

This script allow a user to change its permissions on a file. But the script has some protections, such as that the target file cannot be in another directory than `/home/mtz`, or that the target file cannot be a directory.

To exploit the script, I will use symbolic links *(= shotcuts)* to allow me to edit a file that is not in the `/home/mtz` directory.

Here is the plan :

1. We create a symlink to the `/etc/passwd` file.
2. We give permissions to `mtz` to edit this file.
3. We add a new user equivalent to the `root` user by adding a new line in this file.

![Untitled](assets/screenshot_13.png)

⚠️ Warning : Do not forget to add  `\\` before each `§` when adding the new line in the `passwd` file, as without it, the `$` will be interpreted. ⚠️

And I can finally get the flag root.txt.