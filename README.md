![Twitter header - 1](https://user-images.githubusercontent.com/32911377/152557504-e61a83c6-15e4-404e-816c-e0c7de598e31.png)

This guide is intended to be a one-stop shop for anyone who wants to integrate PlayFab into their Unity Multiplayer game. I’m going to be using and referencing Mirror, the open-source networking library in a lot of these examples, but all of this probably applies to whatever networking library you’re using (such as Unity’s official solution: Netcode for game objects, or Darkrift etc)

Also note that even though this is for Unity, the same principles can be applied while building for Unreal or even your custom game engine.

### A quick word

Hey folks, I’m Rohan Mayya. I have experience with Web, App and Unity development and am fond of Open Source. I spent a good 4 months wrangling around with PlayFab’s Multiplayer Servers trying to get it to work with the multiplayer game I was building (Using Unity + Mirror).
This guide will always remain free, so I would really appreciate it if you reached out on Twitter with feedback on how this guide can be improved, or if you can contribute yourself! 

## Table of Contents

- [Introduction](#introduction)
- [Getting Started](#getting-started)
- Locally Debugging your Game Server Build
- Deploying your Container
- Hosting your Game Server
- Standard Hosting
- Hosting for WebGL Games

# Introduction

Hosting in web development has never been easier. Usually, things happen in one click, or push to your GitHub repository. Nearly everything is automated, and it is dead easy to get started quickly. Tools like Vercel, Heroku, Netlify make doing this a breeze.

However, I’ve seen people in the games industry continuously reinvent the wheel. Between using services like Vultr, AWS’s VPS etc, everyone’s rolling their own logic + scripts for creating and shutting down servers. Everyone’s rolling out their own system for reasons I do not fully understand.

PlayFab abstracts away a lot of this and can easily save you 3+ months of development time (especially if you’re new to all of this), so follow along and we’ll get you up and running in a few hours.

Also, PlayFab gives you multiple options to do certain things (such as choosing between Windows and Linux containers). My guide is, however, very opinionated. I will choose the quickest and easiest path that works for me. Convention over configuration, to get up and running fast.

## Getting Started

Before we begin, if you’re a visual learner, you might prefer watching my YouTube videos instead.

* Part 1 - [https://www.youtube.com/watch?v=YTNdsvTBEcA](https://www.youtube.com/watch?v=YTNdsvTBEcA)
* Part 2 - [https://www.youtube.com/watch?v=9O_tN7pM9v0](https://www.youtube.com/watch?v=9O_tN7pM9v0)

Here are the initial steps:

- Grab the PlayFab Editor Extension for Unity here,
- Follow the steps to add the Editor Extension as a package in your Unity project,
- Create an account on Playfab.com,
- Login to the extension with your PlayFab account,
- Choose the title you created in the PlayFab Editor Extension, after creating a title in PlayFab’s dashboard
- Add `ENABLE_PLAYFABSERVER_API` to your Scripting Define symbols in your Unity’s player settings

PlayFab allows you to create different types of servers: Either a Windows or Linux based server.

With Windows, you can run your server as either a process or a container. With Linux, you can only run it as a linux container.

I discourage using Windows (either process or containers) for starting out, unless you’re really familiar with it. Anything Unix based is easier to use and cheaper (especially by way of servers).

In this guide, I’ll be sticking to Linux containers.

If you’re not familiar with what Docker is, here’s a [guide](https://docs.docker.com/get-started/overview/). In essence, it’s a way to containerize your code (Read: packing all of your application’s code + its dependencies in 1 bucket) so it can be used in any environment easily.

## Locally Debugging your Game Server Build

[Link to Microsoft’s Documentation]

First, grab the PlayFab SDK.

Then, grab the PlayFab GSDK. Place the MultiplayerAgent in your PlayFabSDK folder. 

Make sure you have Mirror’s NetworkManager (or the equivalent if you’re not using Mirror) attached to a GameObject.

 

Create a new GameObject, call it `AgentListener` (or whatever you like). Add a script named `AgentListener.cs` on it as well.

```csharp
public class AgentListener : MonoBehaviour {
	private void Start ()
    {
#if UNITY_SERVER
        PlayFabMultiplayerAgentAPI.Start();

        StartCoroutine(ReadyForPlayers());
#endif
    }

}
```

You can also add these callbacks if you want. I personally had something like this (Set this up in `Start()` again):

```csharp
        PlayFabMultiplayerAgentAPI.IsDebugging = Debugging;
        PlayFabMultiplayerAgentAPI.OnMaintenanceCallback += OnMaintenance;
        PlayFabMultiplayerAgentAPI.OnShutDownCallback += OnShutdown;
        PlayFabMultiplayerAgentAPI.OnServerActiveCallback += OnServerActive;
        PlayFabMultiplayerAgentAPI.OnAgentErrorCallback += OnAgentError;

        UnityNetworkServer.Instance.OnPlayerAdded.AddListener(OnPlayerAdded);
        UnityNetworkServer.Instance.OnPlayerRemoved.AddListener(OnPlayerRemoved);
```

Setup the transport of your choice (for example in Mirror, a popular one is KCP transport). Set 7777 in the `Port` field of the transport you're using.

Build your server. (Make sure it’s Linux x86_64).

Go to the folder where your game server is present.

Add a Dockerfile like so:
`touch Dockerfile`

Your folder should now look like this:
![image](https://user-images.githubusercontent.com/32911377/152561441-eaa977ed-1248-4ec0-bebc-c3421672f657.png)

Add the following to your Dockerfile:

```
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y ca-certificates

ADD . .

CMD ["/game/UnityServer.x86_64", "-nographics", "-batchmode"]
```

You need the second line to install certificates, in case you want to be able to talk to SDKs that talk over HTTPS, such as AWS (I had to do this in my case).

You’re all setup now. If you build this folder as a docker image and run it as a container, you’re free to connect as a client (either from your Unity Editor, or a standalone build on your machine) on IP `127.0.0.1` and port `56100`.

Remember, when you build your server, port must be the server port you want to expose (in our case, 7777). When you connect as a client, use the Connecting Port (in our case, 56100).

To test, you want to grab the PlayFabVMAgent from here. This is used to test running your container locally. If your client (let’s assume your Unity Editor) is able to connect to this, then that confirms your integration is working. 

After extracting the VM agent, `cd` into the folder and open the `Multiplayersettings.json` file. Make sure it looks like this:

[File screenshot]

Then build your docker image, after `cd`ing into your game server folder.

`docker build -t myregistry.io/linuxgameserver:v1 .`

`cd` back into your PlayFabVM agent folder in PowerShell, and run `SetupLinuxContainersOnWindows.ps1`. This sets up the docker network you need for PlayFab to run. 

You can verify this worked by typing `docker network ls`. You will see `playfab` as a network that was just created.

Finally, you can run `LocalMultiplayerAgent.exe -lcow`. This starts your image as a container, which is responsible for starting your game server locally.

You can now connect to it from your Unity Editor. Use IP address `127.0.0.1` and port `56100`

in your hostname field and transport’s port field respectively of your Network Manager.

You should see your PowerShell transitioning between the following states:

- Initialized,
- StandingBy
- Active
- Terminated

When your container is in the `Active` state, that’s when you can connect as a client.

## Deploying your Container

[Link to Microsoft’s documentation]

You can get your customer id, username and password under `Server Details` when creating a new build. Search for “Upload to container registry”.

Assuming you’ve setup WSL2 at this point, navigate to where you have your game server’s image in an Ubuntu (or other Linux-distro based) terminal, and run the following:

**Commands to build and host your new server:**

Login to your Azure Container Registry:
`docker login yourcustomerid.azurecr.io`

Build your local container:
`docker build -t yourcustomerid.azurecr.io/containername:v1 .`

Push your container into their container registry:
`docker push yourcustomerid.azurecr.io/containername:v1`

## Hosting your Game Server

To enable the Multiplayer Servers feature, you need a credit card first. Once you put in your details, you will be allowed to create new server builds.

Next, we shall create a build.
![image](https://user-images.githubusercontent.com/32911377/152562735-f8a5649c-b456-4345-8ad1-8d23c77e48cd.png)


In here, do the following steps:

* Give your build a suitable name,
* Pick Linux Containers,
* Pick the virtual machine type (Dasv4 — 2 cores is fine for now) and pick 1 server per machine (You can vary these after you do significant load testing.),
* You can set Standby and Max servers to 1 for now, for testing purposes,
* Expose ports 7777 on UDP, (this will vary depending on what you chose the port to be when creating your server build),
* Add a region. North Europe/America is fine since you get 750 free hours for testing purposes,
* Hit “Add Build”!

### Testing your Connection

Once your build is created (and assuming you have at least 1 server available), head over to the `Servers` tab in your build, and hit “Request Server” on the top right corner. This will immediately spin up a server for you in one of your existing available VMs.

![image](https://user-images.githubusercontent.com/32911377/152562581-63f8ae02-c819-4183-9e44-333d7168d7d0.png)

Grab the IP and Port that it gives you, start your game client in your Editor or wherever else, use the IP to populate the host name, and port for the port field, (Assuming you’re using KCP/Telepathy/SimpleWebTransport here), and you should be able to connect!

## Standard Hosting

Standard hosting requires that you connect to an IP (or Fully Qualified Domain Name [FQDN]) and Port given to you by PlayFab’s newly spun up server as a client. This is achievable on all types of devices, including but not limited to phone, consoles, PC etc.

## Web Hosting (For WebGL Games)

When you create a Unity WebGL client however (to put your game on the web), you’ll have an issue because a client cannot connect directly to an IP and Port **on an https domain**. (You can connect if your website is not Secure, however). 

You also cannot use the FQDN here because **PlayFab’s Fully Qualified Domain Names are not SSL secured by default.**

This poses a problem, because most legitimate websites will want to have SSL/TLS security, so you cannot compromise by connecting to a WebSocket server that is not secure.

Your end goal is to connect to a url like this: `wss://mydomain.xyz/{data}` where `data` decides which server is picked to connect to.

First off, In Mirror, you want to use Simple Web Transport for your client. In case you are supporting multiple platforms (WebGL, phone etc) you want to use a `Multiplexer` in Mirror, and use that as the Transport in your `NetworkManager`. You can follow similar steps for whatever other networking library you are using, but the point is that your server executable can serve a WebSocket connection over a domain if configured correctly.

This problem can be solved by using a **Reverse Proxy**. 

This [link](https://community.playfab.com/questions/59748/multiplayer-servers-for-webgl-client.html) substantiates the problem and my conversation with Austin ended up yielding fruitful results. Setting up a reverse proxy is out of the scope of this guide, but I will leave you with a few useful links:
[https://www.nginx.com/blog/websocket-nginx/](https://www.nginx.com/blog/websocket-nginx/) (I used NGINX in my working example)

[https://github.com/PlayFab/MultiplayerServerSecureWebsocket](https://github.com/PlayFab/MultiplayerServerSecureWebsocket)

As discussed in the PlayFab Community post above, scale is going to be a problem, because all your clients are going to talk to your reverse proxy. Luckily, nginx and YARP both are built for scale, so it’s up to you, the reader, to figure it out. (If you hit big scale, you’re lucky enough to have that problem in the first place, and you’re bound to figure out a better solution for it than I am.) 

Also, don’t be afraid of using a Reverse Proxy here. One of the core features of Nginx and other Reverse Proxies is SSL/TLS Termination, which means that they are meant to talk to the internal servers (in this case, PlayFab’s servers) without the overhead of SSL for performance purposes. Your connection is still secure because from the client’s perspective, you’re only talking to the secured reverse proxy.
