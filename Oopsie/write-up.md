# 1. Reconnaissance

We are given an IP address, so we start our recon with Nmap

## Nmap

![alt text](images/image.png)

In this scan we can see the SSH service and a website


Landing page:

![alt text](images/image-1.png)

After checking the website we realize that the buttons and different labels have no use and no other directories can be found.

So two ideas came to my mind, to try fuzzing and inspecting source code for some clues.

![alt text](images/image-2.png)


![alt text](images/image-3.png)

An interesting directory that gives us the idea of possible vulnerabilities is the /uploads one, hinting a possible file upload capabilities.

Also the source code exposes a potential log in portal under /cdn-cgi/


We found the login portal

![alt text](images/image-4.png)

We see there is a Login as Guest option

![alt text](images/image-5.png)

# 2. Vulnerability Discovery


Website allows cookie tampering by changing name and value of the user logged in

![alt text](images/image-6.png)

At this step, even if we know we can tamper the cookie, we need extra clues on what to try, we can brute-force and try many options, but another string to pull appeared on the URL

I realize that this Guest page has the id=2, so i tried a basic URL manipulation

![alt text](images/image-7.png)


# 3. Exploitation and Post Exploitation

Changing the id=2 for id=1

![alt text](images/image-8.png)

We found a new page, this time with the admin ID and name, hinting me to try these new values into the cookie.

![alt text](images/image-9.png)


We manage to get access to the admin only upload system

![alt text](images/image-10.png)

![alt text](images/image-11.png)

![alt text](images/image-12.png)

![alt text](images/image-13.png)

Trying the first /uploads directory from the fuzzing, with the name of the file uploaded strongly suggested the possibility of RCE on the target with an adequate .php file.

![alt text](images/image-14.png)

![alt text](images/image-15.png)

Successful RCE


At this point I could use the basic "system(_GET)..." to pass shell commands using the URL, but making it stateless mades the target enumeration way slower and tedious, so I opted for a reverse shell.

Started to listen on 4444

![alt text](images/image-16.png)

PHP code calling for a TCP connection into my IP

![alt text](images/image-17.png)

We upload the file and call it with an HTTP request

![alt text](images/image-18.png)

Success with the www-data user.

![alt text](images/image-19.png)


At this point we have a lower prio access on the target machine, so our focus shifts to privilege escalation

![alt text](images/image-20.png)

Also, by just searching through unprotected files we managed to find a flag for the user robert

![alt text](images/image-21.png)


## Privilege Escalation

While we keep searching we stumble onto another valuable information while listing hidden files

![alt text](images/image-22.png)

We found some credentials for mysql, so we check for credentials reusage.

Using this new creds, we try to switch to a more privileged user "robert"

![alt text](images/image-23.png)

We first have to spawn a proper terminal so the target allows us to switch user calling "su"

And we manage to get inside robert's user

But 'robert' is not the root of the machine, so our work is not finished

It is common for many websites to run common bug trackers like MantisBT, Jira, Bugzilla and they can be source of sensitive information

So in one of those checks using the tool find inside the target machine

![alt text](images/image-24.png)

We found a directory of potential interest

Using a known vulnerability from SUID and bugtracker, we know that bugtracker calls for cat in an insecure manner doing something like:

`system("cat /root/reports/bug_report.txt");`

This means, it relies on the PATH variable to find where cat is

So we can modify and create a fake cat executable that does what we want

First we create the logic of what the new 'cat' will do, spawn root shell.

![alt text](images/image-25.png)

We make it an executable and set it on the PATH variable so it gets found before the actual cat tool.

![alt text](images/image-26.png)

We run the actual bugtracker program that we know is gonna call for 'cat' but in this case is gonna call for our new 'cat' tool

![alt text](images/image-27.png)

We make a random input to trigger the logic and we spawn a root shell

![alt text](images/image-28.png)

Now, all it is left is searching for the root directories to find the flag


# 4. Remediations

- Big Vulnerability the fact that with an URL manipulation we could access an admin information

- Cookie tampering allowed

- File uploading system letting us execute .php code

- Plain text credentials

- Known Bugtracker vulnerability under (CWE-426: Untrusted Search Path or CWE-427: Uncontrolled Search Path Element.)

The application uses partial path for the cat tool, making it exploitable with newly created cat tools.


