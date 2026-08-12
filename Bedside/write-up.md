# Bedside - Machine write-up from season 11 HTB


## Nmap

Standard Nmap over the target

![alt text](images/image.png)

We find a SSH port, a web app and the domain: bedside.htb

This puts the focus of our enumeration over the web app.

## Web app

![alt text](images/image2.png)

We land on a page that talks about AI, being that a newly attack surface on many web apps since they are starting to use a lot of AI tooling.

### Vhost Fuzz

![alt text](images/image3.png)

We do some Vhost enumeration to find hidden ones and we hit one.

![alt text](images/image4.png)

We find here a potential vector through file uploading that will require more investigation.

Website is telling us how not only is a file upload but some backend is gonna process the file.

![alt text](images/image5.png)

Investigating the found vhost, we see that the headers give us info about pdfminer.six

![alt text](images/image6.png)

We test the functionality with a test picture.

![alt text](images/image7.png)

And we test with wrong format

![alt text](images/image8.png)

We see how the app is leaking its server-side path

![alt text](images/image9.png)

Then we land with out uploaded picture on the directory /uploads/

## Found CVE

After some OSINT investigation around the pdfminer we find a published CVE

![alt text](images/image10.png)

Following instructions we need 2 files

A python file using pickle to desterilize python object, when the pickle module tries to reconstruct a class it looks for the reduce method, this method is HOW to reconstruct it, but we can make that the method literally runs shell commands 

![alt text](images/image11.png)
![alt text](images/image12.png)

Then we need a PDF file, according to documentation, to trigger this pickle parsing. So pdfminer reads the PDF we craft following some encoding rules to trick pdfminer. Since pdfminer uses pickle we can use it to load our RCE payload.

It is important to know the path where the zip file is stored so we can used the pdf file to target the location and trigger the execution.

## Usage of tool

Making the pickle with the reverse shell
![alt text](images/image13.png)

Pdf that triggers the execution of the pickle
![alt text](images/image14.png)

Land shell as datawrangler user
![alt text](images/image15.png)

# Enumerating System as DataWrangler

During system enumeration and the mounts we see that we are inside a docker container.

![alt text](images/image16.png)

![alt text](images/image17.png)

So the focus changes into finding a way to escape the docker container

## Inside Docker

Curious .sh script on tmp directory

![alt text](images/image18.png)

After running it, we see a port 3000 only listening locally

We use chisel to forward the connection

![alt text](images/image19.png)

We found an empty web with HMR websocket protocols

Eventually we try path traversal in this newly found internal port

![alt text](images/image20.png)

![alt text](images/image21.png)

We find the user that potentially has the user flag

Trying path traversal to grab flag over the found user

![alt text](images/image22.png)

Found USER flag

## Escalation of privileges

After grabbing the user flag, next objective is becoming root. So we keep exploring the system with the path traversal vulnerability we found

![alt text](images/image23.png)

We found the keys to SSH into the target machine as developer, which will help us in our system enumeration

![alt text](images/image24.png)

We made the SSH key usable and we SSH into the machine

# Getting Root

Found a script that runs as root without password

Problem is that the script is reading files from a folder that as developer user we have no write perms. We remember that when on datawrangler we could write there.

![alt text](images/image25.png)

![alt text](images/image26.png)

![alt text](images/image27.png)

After testing payloads, we find that the best way is to add our developer ssh key and add it for authorized keys for root

![alt text](images/image28.png)

![alt text](images/image29.png)

Root and flag.