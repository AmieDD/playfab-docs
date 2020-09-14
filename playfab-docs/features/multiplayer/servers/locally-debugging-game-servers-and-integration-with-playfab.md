---
title: Locally debugging game servers and integration with PlayFab
author: joannaleecy
description: Describes how to integrate PlayFab multiplayer game servers with the PlayFab Game Server SDK (GSDK) and verify and debug the integration.  
ms.author: joanlee
ms.date: 01/15/2019
ms.topic: article
ms.prod: playfab
keywords: playfab, multiplayer servers, debugging, integration, thunderhead, playfab game server sdk
ms.localizationpriority: medium
---

# Locally debugging game servers and integration with PlayFab

## Overview

PlayFab multiplayer game servers require integration with [PlayFab Game Server SDK (GSDK)](integrating-game-servers-with-gsdk.md). In addition, game servers are run as containerized applications on the PlayFab Multiplayer platform.  

Running them as containerized applications enables running and debugging the server locally, in an environment that matches that of the PlayFab platform in Azure. This facilitates faster development iterations. This article helps you verify that your PlayFab game server conforms to the platform requirements.  

The PlayFab local debugging toolset includes a mock PlayFab VmAgent that provides mock responses to the GSDK and verifies whether your game server is integrated with the GSDK correctly. With the mock responses, the VmAgent cycles the game server through various states in its lifecycle on the PlayFab Multiplayer platform.

You can configure the agent to run the game server as a containerized application and verify that your game server is packaged with all the required dependencies and will run without issues on the PlayFab Multiplayer platform.

## Basic Setup

- Integrate your game server with the GSDK and build it. For more information, see [Integrating game servers with the PlayFab Game Server SDK (GSDK)](integrating-game-servers-with-gsdk.md).
- Compress your game server and its dependencies to a zip archive. To run properly in container mode, the zip archive must contain any system DLLs that are not included in the container image. For more information, see [determine required system DLLs](determining-required-dlls.md)..

  > [!NOTE]
  > Avoid this common mistake - do not accidentally zip a folder *within* a folder in the zip. After zipping, browse the zip folder and double-check that your compression software did not add an extra layer of file hierarchy.

- Download the [local debugging toolset](https://github.com/PlayFab/LocalMultiplayerAgent/releases) and extract it to a folder of your choice (such as *C:\PlayFabVmAgent*).
- Navigate to the location of the extracted folder and open the *MultiplayerSettings.json* file in a text editor (such as [Visual Studio Code](https://code.visualstudio.com/download)). Update the following properties:  
  - `LocalFilePath` - Full Local Path (on your workstation) to the game server asset zip file created earlier, for example: *D:\\\\MyAmazingGame\\\\asset.zip* (note that backslashes need to be escaped for JSON formatting).
  - `StartGameCommand` - The full path to the game server executable within the container. For example, if the name of the executable is *mygame.exe*, a sample path would be *C:\\\\Assets\\\\mygame.exe*. The paths for the StartGameCommand are different for a Process and a Container. The StartGameCommand path for a container is an absolute path to a resource in the container or asset folder. The StartGameCommand path for a process is a relative path where the working directory will be the first asset specified.
  - `PortMappingsList` - These are the ports that are available to your game while running. `NodePort` is the port that is opened on your workstation, `GamePort.Number` is the port that your game server needs to bind to when running in a container. Update the GamePort section to match the protocol and port at which your game server is listening for clients. If your game server needs multiple ports, copy/paste the existing port configuration and increment `NodePort` then update `GamePort.Number` and `GamePort.Name` to the required port. When running as a process, `GamePort.Number` is ignored, your process should bind to NodePort. To handle both cases, do one of the following:
    - Set the ports to the same value
    - Check the GSDK config at runtime for the value with the key `GamePort.Name` which always returns the correct port to bind against.
  
- There are additional fields in the *MultiplayerSettings.json* file you might want to edit:
  - `ResourceLimits` (optional) - If specified, docker limits CPU/memory usage. Warning: If your server goes over the allowed memory, it is killed. ResourceLimits can only be specified in container mode.
  - `SessionCookie` (optional) - Any session cookie that is passed to your game server as part of the [RequestMultiplayerServer API](xref:titleid.playfabapi.com.multiplayer.multiplayerserver.requestmultiplayerserver) call.
  - `OutputFolder` (optional) -  Path to a drive or folder where the outputs and config files are generated. Ensure that there is sufficient space available since the game server will be extracted under this path. If not specified, the agent folder is used.
  - `MountPath` - The path within the container at which to mount the asset. This field does not need to be specified when running in process mode. We recommend using the sample value - *C:\\\\Assets* (note that backslashes need to be escaped for JSON formatting).
  - `AgentListeningPort` - Specifies the port to which the agent binds to communicate with the game server. Any open port will work, if you have another process binding to 56001 you must change this value (or kill the other process).

## Verifying GSDK integration

- In the *MultiplayerSettings.json* file, set `RunContainer` to `false`.
- In a Powershell window (as Administrator):  
  - Change your working directory to the folder where the toolset was extracted.
  - Run *MockVmAgent.exe*. At this point, **MockVmAgent** sets up the http listener, unzips the game asset, and starts the game server in a separate process. **MockVmAgent** then waits for heartbeats from the GSDK integrated with your game server.
- If the GSDK is integrated correctly, **MockVmAgent** prints the following outputs:  
  - `CurrentGameState - Initializing` (this is optional and may not show up if your game server directly calls `GSDK::ReadyForPlayers` and does not call `GSDK::Start`)
  - `CurrentGameState - StandingBy`
  - `CurrentGameState - Active`
  - `CurrentGameState - Terminating`
- If the shutdown callbacks are set up correctly, your game server exits soon after the state is set to terminating. It is important to verify that the game server exits to avoid ungraceful shutdowns on the PlayFab platform.
- The **MockVmAgent** should also terminate along with the game.

### Testing connection to your game

When your game server executable is running and **MockVmAgent** prints `CurrentGameState - Active`, you can connect to your game server using IP address **127.0.0.1** and port `NodePort` on which your game server is listening.

After `NumHeartBeatsForActivateResponse` heartbeats, **MockVmAgent** requests the game server to move from standby to active. Then after `NumHeartBeatsForTerminateResponse` heartbeats **MockVmAgent** requests the game server to move from active to terminated. This behavior can be tuned by updating the values in the *MultiplayerSettings.json* file.

## Verifying containerization

### Prerequisites

- Windows 10 Pro (or above) with April 2018 (1803) update.
- Download [Docker](https://download.docker.com/win/stable/Docker%20for%20Windows%20Installer.exe). Alternately, you can download it from the main page of the [Docker website](https://www.docker.com/products/docker-desktop).

### Setup

- Ensure that Docker is set to [use Windows Containers](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers)
- In a Powershell window (as Administrator):  
  - Navigate to the folder where the toolset was extracted.  
  - Run *Setup.ps1* which sets up the docker networks, add firewall rules to communicate with **MockVmAgent**, and pull down the PlayFab docker image from [Microsoft/PlayFab-Multiplayer](https://hub.docker.com/r/microsoft/playfab-multiplayer/). Note that the first time the script runs, it can take a few minutes to download the container image.
      > [!NOTE]
    > To run this setup successfully, you may have to configure the firewall of any 3rd party antivirus program (such as McAfee, Norton, or Avira) that you have installed.  

### Running the game server within a container

- In the *MultiplayerSettings.json* file, set `RunContainer` to `true`.
- Open a **Powershell** window (as Administrator) in the folder where the toolset was extracted (*C:\PlayFabVmAgent*) and run `MockVmAgent.exe`. This starts the game server within a container. Eventually, you should see game state change output in the Powershell window (just like in the Verifying GSDK integration section above).

### Testing connection to your game server running within a container

When **MockVmAgent** output prints `CurrentGameState - Active`, connect to your game server using IP address **127.0.0.1** and port equal to `NodePort` (by default **56100**) that's specified in the *MultiplayerSettings.json* file.

After `NumHeartBeatsForActivateResponse` heartbeats, **MockVmAgent** requests the game server to move from standby to active. Then after `NumHeartBeatsForTerminateResponse` heartbeats **MockVmAgent** requests the game server to move from active to terminated. This behavior can be tuned by updating the values in the *MultiplayerSettings.json* file.

### Troubleshooting

- In container mode, if your game server exits immediately with an error similar to "Container ... exited with exit code 1", but it works fine in process mode, make sure that you have included all required [system DLLs](determining-required-dlls.md) in your asset package.
- All logs are located under `OutputFolder` that is specified in the *MultiplayerSettings.json* file. **MockVmAgent** creates a new folder each time it is started, with the timestamp as folder name. All game server logs emitted via GSDK are located within the GameLogs folder.  
If the game server is running in a container, there might be an additional level of directory hierarchy to sift through.
- The GSDK writes debug logs to the GameLogs folder. These logs are located within the GameLogs folder along with the logs output by the game server.
- Ensure firewalls (windows and other anti-virus) are configured to allow the traffic over the ports.
- If you get an error similar to: `Docker API responded with status code=InternalServerError, response={"message":"failed to create endpoint <container_name> on network playfab: hnsCall failed in Win32: The specified port already exists". It is likely there is already a container running on the specified port.` This can happen if **MockVmAgent** exits prematurely. Use the command `docker ps` to find the container that is running, and then kill it using `docker kill <container_name>`.
- If you get an error that contains `Failed to find network 'playfab'`. Try rerunning *Setup.ps1*

### Known Limitations

1. Containers might not terminate at the end of the debugging. If this occurs, run the following PowerShell commands as Administrator. These commands stop and remove all containers, including those were not started by **MockVmAgent**.

```powershell
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)  
```
