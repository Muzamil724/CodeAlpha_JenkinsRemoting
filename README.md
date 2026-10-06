# CodeAlpha DevOps Internship — Task 2: Jenkins Remoting

## Overview
This project demonstrates **Jenkins Remoting** — connecting a remote Jenkins agent (node) to a Jenkins master/controller and running a build job on that remote node instead of the master itself. This mirrors how real organizations distribute build load across multiple machines and isolate build environments.

## Architecture
- **`jenkins-master`** — the Jenkins controller, running in a Docker container, hosting the web UI and scheduling jobs.
- **`jenkins-agent-1`** — a separate Docker container running as a remote inbound agent, registered to the master as node `remote-agent-1` with label `remote-docker`.
- Both containers communicate over a dedicated Docker bridge network (`jenkins-net`), simulating two physically separate machines talking over a network — the master and agent never share a filesystem or process space.

```
┌──────────────────┐          jenkins-net           ┌──────────────────┐
│  jenkins-master   │ <----- JNLP / WebSocket ----> │ jenkins-agent-1   │
│  (controller)      │                                │ (remote node)     │
│  port 8090 (UI)     │                                │ label: remote-docker│
└──────────────────┘                                └──────────────────┘
```

## Setup Steps

1. **Create a dedicated Docker network**
   ```bash
   docker network create jenkins-net
   ```

2. **Run the Jenkins master**
   ```bash
   docker run -d --name jenkins-master --network jenkins-net \
     -p 8090:8080 -p 50000:50000 jenkins/jenkins:lts
   ```

3. **Retrieve the initial admin password and complete setup**
   ```bash
   docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword
   ```
   Install suggested plugins, create an admin user, confirm the Jenkins URL.

4. **Create a remote node** (Manage Jenkins → Nodes → New Node)
   - Name: `remote-agent-1`
   - Type: Permanent Agent
   - Remote root directory: `/home/jenkins/agent`
   - Labels: `remote-docker`
   - Launch method: "Launch agent by connecting it to the controller"

   Save, then open the node page to get its **secret**.

5. **Run the remote agent container**, connecting it to the master over the Docker network (note the internal container name/port, not the host-mapped ones):
   ```bash
   docker run -d --name jenkins-agent-1 --network jenkins-net \
     -e JENKINS_URL=http://jenkins-master:8080 \
     -e JENKINS_AGENT_NAME=remote-agent-1 \
     -e JENKINS_SECRET=<secret-from-node-page> \
     -e JENKINS_WEB_SOCKET=true \
     jenkins/inbound-agent:latest
   ```

6. **Create a Freestyle job** (`remote-build-test`) restricted to run only on label `remote-docker`, with a shell build step:
   ```bash
   echo "Running on: $(hostname)"
   echo "Agent name: $NODE_NAME"
   echo "This proves the build ran on a remote Jenkins agent, not the master."
   ```

## Proof of Remote Execution

Console output from a successful build:

```
Running as SYSTEM
Building remotely on remote-agent-1 (remote-docker) in workspace /home/jenkins/agent/workspace/remote-build-test
+ hostname
Running on: d5b616270cd4
+ echo Agent name: remote-agent-1
Agent name: remote-agent-1
This proves the build ran on a remote Jenkins agent, not the master.
Finished: SUCCESS
```

The hostname printed (`d5b616270cd4`) is the **agent container's** ID — distinct from the master container — confirming the job executed on the remote node, not the controller.

## Key Concepts Demonstrated
- Jenkins master/controller vs. agent/node architecture
- Secure agent-to-master connection (JNLP/WebSocket, port `50000`)
- Node labeling and label-based job restriction (`Restrict where this project can be run`)
- Docker container networking (container-name DNS resolution vs. host-mapped ports)
- Distributed, isolated build execution — proof via live hostname comparison

## Part of
CodeAlpha DevOps Internship — one of three completed tasks (alongside an Azure CI/CD pipeline and a Dockerized Nginx web server with health monitoring).
