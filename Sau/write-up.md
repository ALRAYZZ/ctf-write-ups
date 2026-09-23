# Recon

```bash
nmap -sC -sV -T4 10.129.229.26
```

![image1](./images/image1.png)

![image2](./images/image2.png)

![image3](./images/image3.png)

We find 3 services: SSH, a web app, and a custom Golang web server.

We find that port `80` is dropping the packets, which could mean that there is a service listening only on local connections.

![image4](./images/image4.png)

We find that it's powered by a service called `request-baskets`.

![image5](./images/image5.png)

![image6](./images/image6.png)

One of the first things to look for when working with external and custom services and their versions is any relevant OSINT that might reveal issues specific to that version.

# Initial Access

![image7](./images/image7.png)

SSRF is a security vulnerability where we can trick a server into making a request that we normally would not have access to, potentially bypassing security checks or allowing us to access hidden endpoints.

We find the code repository that exposes the REST API.

![image8](./images/image8.png)

![image9](./images/image9.png)

With the API structure, we can try to craft a POST request that calls the internal service that we cannot access directly, allowing us to recon it.

```bash
curl -X POST http://10.129.229.26:55555/api/baskets/pivot \
  -H "Content-Type: application/json" \
  -d '{"forward_url":"http://127.0.0.1:80","proxy_response":true}'
```

Then we curl it to see if we are successfully reaching port `80`.

```bash
curl -i http://10.129.229.26:55555/pivot/
```

![image10](./images/image10.png)

![image11](./images/image11.png)

We find it's running Maltrail v0.53.

# Chain Vulnerabilities for system foothold

Following the same methodology, the first thing to do when finding a service name and its version is to check for OSINT.

![image12](./images/image12.png)

So, for the exploit to work, we need to change the path that the `forward_url` is pointing to, since the vulnerability is found in the `/login` directory.

We first do a simple test based on the documentation.

```bash
curl -X POST http://10.129.229.26:55555/api/baskets/p2 \
  -H "Content-Type: application/json" \
  -d '{
    "forward_url": "http://127.0.0.1:80/login",
    "proxy_response": true,
    "expand_path": true,
    "capacity": 250
  }'
```

```bash
curl -i -X POST 'http://10.129.229.26:55555/p2' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'username=test&password=test'
```

![image13](./images/image13.png)

# Exploit

So now we have prepared the basket correctly to point directly at `/login`.

To ease our task, we grab a script that has already been published:

`https://github.com/spookier/Maltrail-v0.53-Exploit/blob/main/exploit.py`

![image14](./images/image14.png)

![image15](./images/image15.png)

But we edit the `target_URL` variable since we do not need to append the string `/login`.

We run the modified script.

![image16](./images/image16.png)

We land RCE.

![image17](./images/image17.png)

Finding the user flag under Puma's home directory.

![image18](./images/image18.png)

# Privilege Escalation

One of the first commands when looking for misconfigurations as a low-privileged user is to check `sudo -l`, which lists the sudo privileges available to our user.

![image19](./images/image19.png)

We find that `/usr/bin/systemctl` can be invoked by any user and group without a password through sudo.

Then we need to learn what `systemctl` is. It is the command-line tool for controlling `systemd`, which is the init system and service manager on most modern Linux distributions. Its job is to start, stop, and manage services, in our case, `trail.service`.

Triggering this functionality:

![image20](./images/image20.png)

It displays the service status, but with a pager, which instead of dumping all the output in one go, lets us view the output in chunks.

We see the pager running as PID `1310`, so we can check with what privileges the pager is running, since a pager is an interactive program, not just a passive output tool.

`ps -o pid,ppid,user,uid,group,gid,args -p 1307,1309,1310`

![image21](./images/image21.png)

We see the pager running as root.

Now we check what pager is actually running.

`readlink -f /usr/bin/pager`

![image22](./images/image22.png)

![image23](./images/image23.png)

We know from public information that `less` has a shell escape facility while at the `(END)` screen.

![image24](./images/image24.png)

![image25](./images/image25.png)

![image26](./images/image26.png)

The `less` pager is intentionally an interactive program, and one of its features is the ability to invoke another program. The danger comes from the fact that the pager is running as `root`.

The fact that we found `systemctl` as `NOPASSWD` is not the fundamental problem with this privilege escalation.

The important components are:

1. `puma` has the sudo rule.
2. `systemctl status` launches a pager.
3. The pager is launched with root privileges.
4. The pager is interactive and can invoke external functionality.