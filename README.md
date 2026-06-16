# VPN Development in Visual Studio Code with Golang

> [!NOTE]  
> This guide is part of the [simplest-vpn](https://github.com/developer3389/simplest-vpn) project — a minimalist VPN proof of concept written in Go in roughly 200 lines of code.

> [!TIP]  
> Once you have everything set up as described in this guide, you will be able to write almost no code by hand.  
> AI agents will have full access to both the client and server parts of the project and can implement features, fix bugs, and refactor code for you — all you need to do is describe what you want.

## Table of Contents
- [Workflow and Architecture](#workflow-and-architecture)
- [1. Install Visual Studio Code and Extensions](#1-install-visual-studio-code-and-extensions)
- [2. Prepare SSH Access to Both Machines](#2-prepare-ssh-access-to-both-machines)
- [3. Add Both Machines to Your SSH Config](#3-add-both-machines-to-your-ssh-config)
- [4. Connect to the Client from VS Code](#4-connect-to-the-client-from-vs-code)
- [5. Open the Project on the Client](#5-open-the-project-on-the-client)
- [6. Open the Second Machine in a Second Window](#6-open-the-second-machine-in-a-second-window)
- [7. Set Up Go Tools in VS Code](#7-set-up-go-tools-in-vs-code)
- [8. Configure Run and Debug](#8-configure-run-and-debug)
- [9. How to Combine Client and Server into One AI Context](#9-how-to-combine-client-and-server-into-one-ai-context)
- [10. Use AI and Experiment](#10-use-ai-and-experiment)

---

## Workflow and Architecture
This guide describes a distributed development workflow where after setup you will have:

- **Window 1**: VS Code connected directly to the Client machine via `Remote-SSH`.

- **Window 2**: VS Code connected directly to the Server machine via `Remote-SSH`.

- The ability to edit, run, and debug code natively on the remote hosts.

- A **unified project context** for your `AI` assistant (via `sshfs`).

### Bare-Metal Deployment
```plain
+-----------------------------------------------------------+
|                  Local Workstation                        |
|  +-----------------------+     +-----------------------+  |
|  |  VS Code Window 1     |     |  VS Code Window 2     |  |
|  |   (VPN Client)        |     |   (VPN Server)        |  |
|  +-----------+-----------+     +-----------+-----------+  |
|              |                             |              |
|        Remote-SSH (22)               Remote-SSH (22)      |
|              |                             |              |
+--------------|-----------------------------|--------------+
               |                             |
      +--------v--------+           +--------v--------+
      |  Remote Linux   |           |  Remote Linux   |
      |     Client      <-- sshfs -->     Server      |
      +-----------------+           +-----------------+
```

### Hypervisor-Based Deployment
```plain
+-------------------------------------------------------------------+
|                         Local Workstation                         |
|                                                                   |
|   +-----------------------+           +-----------------------+   |
|   |   VS Code Window 1    |           |   VS Code Window 2    |   |
|   |     (VPN Client)      |           |     (VPN Server)      |   |
|   +-----------+-----------+           +-----------+-----------+   |
|               |                                   |               |
|         Remote-SSH (22)                     Remote-SSH (22)       |
|               |                                   |               |
|   +-----------v-----------------------------------v-----------+   |
|   |                    HyperV / VirtualBox                    |   |
|   |                                                           |   |
|   |  +-----------------+               +-----------------+    |   |
|   |  |   VM 1 (Linux)  |               |    VM 2 (Linux) |    |   |
|   |  |     Client      <---- sshfs ---->      Server     |    |   |
|   |  +--------+--------+               +--------+--------+    |   |
|   +-----------|-----------------------------|-----------------+   |
|               |                             |                     |
|        Bridged (vNIC)                Bridged (vNIC)               |
+---------------|-----------------------------|---------------------+
                |                             |
          +-----v-----------------------------v-----+
          |              Real Router DHCP           |
          +-----------------------------------------+
```

The examples below use these addresses:

- client: `192.168.0.3`;
- server: `192.168.0.4`.

Replace them with the actual IP addresses of your machines.

## 1. Install Visual Studio Code and Extensions

Download and install Visual Studio Code:

https://code.visualstudio.com/

After installation, open Extensions (`Ctrl+Shift+X`) and install:

- `Remote - SSH`;
- `Go` (`golang.go`).

## 2. Prepare SSH Access to Both Machines

For remote development, VS Code must be able to connect over SSH to both the client and the server.

### Option Used in This Guide

The instructions below use login as `root` because it is simpler for debugging and matches your current setup.

> [!CAUTION]
> Use this only in an isolated test environment. For production servers, it is better to create a separate user and use key-based authentication.

### What to Do on Each Machine

1. Set a password for `root`:

```bash
sudo passwd root
```

2. Open the SSH configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

3. Find the parameter:

```text
PermitRootLogin
```

4. Set it to:

```text
PermitRootLogin yes
```

5. Restart the SSH server:

```bash
sudo systemctl restart ssh
```

## 3. Add Both Machines to Your SSH Config

1. In VS Code, press `F1`.
2. Run **Remote-SSH: Connect to Host...**.
3. Select **Configure SSH Hosts...**.
4. Open the `~/.ssh/config` file.
5. Add entries for the client and server.

> [!TIP]
> It is recommended to use clear aliases instead of raw IP addresses:

```sshconfig
Host vpn-client
  HostName 192.168.0.3
  User root
  Port 22

Host vpn-server
  HostName 192.168.0.4
  User root
  Port 22
```

After that, you will see readable names like `vpn-client` and `vpn-server` in the connection list.

## 4. Connect to the Client from VS Code

1. Press `F1`.
2. Select **Remote-SSH: Connect to Host...**.
3. Choose `vpn-client`.
4. If VS Code asks for the remote OS type, choose `Linux`.
5. On first connection, confirm the remote machine fingerprint.
6. Enter the `root` password.

If the connection succeeds, a new VS Code window will open, and you will see an indicator like this in the lower-left corner:

```text
SSH: vpn-client
```

This means you are working directly with the files on the remote machine.

## 5. Open the Project on the Client

In the remote window, select **File -> Open Folder...** and open the root folder of the project: `/root/vpn/simplest-vpn`.

Since you are using a monorepo structure, opening the root folder provides you with full access to the entire project context at once:

- `/root/vpn/simplest-vpn/client`
- `/root/vpn/simplest-vpn/server`

After that, you will be able to:

- edit files on the remote machine;
- run a terminal on the remote machine;
- debug the application;
- work with Git as if the project were local.

## 6. Open the Second Machine in a Second Window

Once the client window is already open:

1. Select **File -> New Window**.
2. In the new window, press `F1`.
3. Run **Remote-SSH: Connect to Host...**.
4. Choose `vpn-server`.
5. Open the project directory on the server.

As a result, you will have two independent VS Code windows:

```text
Window 1 -> VPN Client
Window 2 -> VPN Server
```

This is convenient because the client and server run independently, while you can change code in both at the same time.

## 7. Set Up Go Tools in VS Code

After [installing Go](https://go.dev/doc/install) on both Linux machines, configure the development tools.

### Install Additional Tools

1. Open any `.go` file.
2. Press `F1`.
3. Run the command `Go: Install/Update Tools`.
4. Select all tools and click `OK`.

Usually it is enough to install:

- `gopls` for autocomplete and navigation;
- `goimports` for auto-formatting and import management;
- `dlv` for debugging;
- `golangci-lint` if your project uses it.

### Quick Check

Create a test file named `main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello World!")
}
```

If autocomplete works, errors are highlighted, and imports are formatted automatically, then the environment is configured correctly.

## 8. Configure Run and Debug

The project requires a `.vscode/launch.json` file to enable `F5` debugging.  
Since you are working on two separate machines, you need to create this file locally on each host.

> [!IMPORTANT]
> Although your project folder structure is identical on both machines, the `.vscode/launch.json` file is unique to each host.  
> This ensures that when you press `F5` in the respective VS Code window, it automatically targets the correct local binary.

### Setup Instructions:

On each machine, create the directory: `/root/vpn/simplest-vpn/.vscode/`

Inside this directory, create the `launch.json` file tailored for that specific host:

On **the Client** machine (`/root/vpn/simplest-vpn/.vscode/launch.json`):

#### Client Configuration

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Client",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/client",
      "asRoot": true,
      "output": "${workspaceFolder}/build/client"
    }
  ]
}
```

On **the Server** machine (`/root/vpn/simplest-vpn/.vscode/launch.json`):
#### Server Configuration

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Server",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/server",
      "asRoot": true,
      "output": "${workspaceFolder}/build/server"
    }
  ]
}
```

> [!NOTE]
> `${workspaceFolder}` is a VS Code variable that resolves to the root folder open in the window (`/root/vpn/simplest-vpn/`).  
> By keeping these files separate on each machine, you avoid configuration conflicts and enable "one-click" debugging on both the client and server.

### Running the Application

- `F5` starts the application with the debugger attached;
- `Ctrl+F5` starts the application without the debugger.

## 9. How to Combine Client and Server into One AI Context

If the client and server code lives on different machines, but you want the AI to see both parts of the project at once, you can use `sshfs`.

### Install `sshfs`

Run this on both machines:

```bash
apt update && apt install -y sshfs
```

### Expected Structure

Assume the shared project path is:

```text
/root/vpn/simplest-vpn
```

Inside it:

- `client` contains the client code;
- `server` contains the server code.

### Mounting on the Client Machine (`192.168.0.3`)

On the client machine, mount the remote `server` directory from the server machine.

```bash
mkdir -p /root/vpn/simplest-vpn/server
sshfs root@192.168.0.4:/root/vpn/simplest-vpn/server /root/vpn/simplest-vpn/server \
  -o reconnect,ServerAliveInterval=5,ServerAliveCountMax=2
```

### Mounting on the Server Machine (`192.168.0.4`)

On the server machine, mount the remote `client` directory from the client machine.

```bash
mkdir -p /root/vpn/simplest-vpn/client
sshfs root@192.168.0.3:/root/vpn/simplest-vpn/client /root/vpn/simplest-vpn/client \
  -o reconnect,ServerAliveInterval=5,ServerAliveCountMax=2
```

> [!IMPORTANT]
> The local mount directory must already exist before running the command.
> In the commands above, the IP address points to the **opposite** machine — the one you are mounting the directory **from**.

> [!WARNING]
> If the local directory already contains files, they will not be deleted, but they will be shadowed by the mount point and will become visible again only after unmounting.

> [!NOTE]
> Open the `/root/vpn/simplest-vpn/` folder in VS Code on both hosts.  
> Now, each instance of the editor will see the "neighbor's" files as if they were stored locally.

### How to Unmount

Use these commands to detach the project directories. If a standard unmount hangs due to an interrupted connection, use the **lazy** (`-uz`) option.

| Target | Standard Unmount | Lazy Unmount |
| :--- | :--- | :--- |
| **Server** | `fusermount -u /root/vpn/simplest-vpn/server` | `fusermount -uz /root/vpn/simplest-vpn/server` |
| **Client** | `fusermount -u /root/vpn/simplest-vpn/client` | `fusermount -uz /root/vpn/simplest-vpn/client` |

## 10. Use AI and Experiment

> [!IMPORTANT]
> **Do not waste time on manual boilerplate work.**
> Use AI during development, let it handle routine tasks, and spend your energy on ideas, experiments, and system design.

### AI Assistants Worth Knowing About

**GitHub Copilot** is the most natural choice when working in VS Code. It is built directly into the editor and supports an agent mode where it can read your project, suggest multi-file changes, run commands, and iterate on the result. Install it from the Extensions panel and sign in with your GitHub account.

Other tools worth knowing about:

- **Cursor** — a VS Code fork with deep AI integration built in from the start, popular for agent-style development;
- **Windsurf** — another AI-first editor based on VS Code, focused on multi-step autonomous tasks;
- **Claude** (Anthropic) — a powerful general-purpose AI that works well for architecture discussions, code review, and writing documentation, available via the web or API;
- **ChatGPT** (OpenAI) — widely used for explaining concepts, debugging ideas, and generating boilerplate, available via the web or API;
- **Gemini** (Google) — available in the web and integrated into some Google developer tools.

Most of these can be used alongside VS Code even if they are not embedded in it directly — paste code into the chat, describe what you need, and bring the result back.

## Happy Coding!
