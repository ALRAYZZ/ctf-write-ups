# NMAP

![image1](./images/image1.png)

Port `6274`.

We found an MCPJam service running a vulnerable version.

![image2](./images/image2.png)

![image3](./images/image3.png)

On port `80`:

![image4](./images/image4.png)

We found a web application with access to internal tooling.

# Running Exploit for CVE-2026-23744

https://github.com/ALRAYZZ/CVE-Exploits/tree/main/exploits/CVE-2026-23744

![image5](./images/image5.png)

![image6](./images/image6.png)

# SSH Persistence Post Exploitation

Before doing any enumeration, we establish stable SSH access by injecting our public key into `mcp-dev`'s `authorized_keys`.

On the attacking machine:

![image7](./images/image7.png)

On the reverse shell:

![image8](./images/image8.png)

We are creating a new SSH key pair and adding the public key to the target machine, allowing us to SSH back into it as the exploited user, `mcp-dev` in this case.

![image9](./images/image9.png)

![image10](./images/image10.png)

We now have a stable shell.

# Lateral Movement

## SSH Port Forwarding

We establish a **Secure SSH Local Port Forwarding Tunnel** to bypass network restrictions and access hidden internal services.

Jupyter on `localhost:8888` and OPSMCP on `localhost:5000` are not reachable directly from outside the machine. We use SSH local port forwarding to tunnel both services to our attacking machine.

![image11](./images/image11.png)

So basically, we make our requests pass through the SSH connection and get forwarded to the target machine, allowing us to access services that are only bound to the target's localhost or otherwise blocked by network restrictions.

# Jupyter

Finding Jupyter on port `8888`:

![image12](./images/image12.png)

![image13](./images/image13.png)

Token:

`a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7`

![image14](./images/image14.png)

![image15](./images/image15.png)

## Executing Code Using the Jupyter WebSocket Protocol

We found online that Jupyter does not execute code through the REST API. The REST API is primarily used for lifecycle and management operations, such as creating and deleting kernels or listing notebooks. Code execution is handled through the **Jupyter Messaging Protocol** over WebSockets, specifically through the `/api/kernels/{kernel_id}/channels` endpoint.

The protocol requires sending a structured JSON message with `msg_type: "execute_request"` on the shell channel. Results are then streamed back as `stream` messages through the IOPub channel.

The key to avoiding a race condition is to associate all message handlers with `parent_header.msg_id`, matching responses specifically to our request. We also trigger completion on `execute_reply`, which arrives after the execution request has been processed, rather than relying on the `idle` status message, which can arrive prematurely.

We used our own tool based on documentation found online:

https://github.com/ALRAYZZ/CVE-Exploits/tree/main/tools/jupyter-remote-exec

This tool allows us to gather the token and kernel ID and pass code for remote execution.

![image16](./images/image16.png)

A small example:

![image17](./images/image17.png)

It also allows us to send a script file as the entire payload

# Accessing server.py from OPSMCP

![image18](./images/image18.png)

API_KEY = `opsmcp_secret_key_4f5a6b7c8d9e0f1a`

![image19](./images/image19.png)

![image20](./images/image20.png)

The `ops._admin_dump` tool can read the `id_rsa` SSH private key.

![image21](./images/image21.png)

We use this key in our `curl` requests:

![image22](./images/image22.png)

![image23](./images/image23.png)

We then change the permissions on the key with `chmod` and use it to log in:

![image24](./images/image24.png)

