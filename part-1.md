# Part 1 - Docker on Windows

We'll start with the basics and get a feel for running Docker on Windows.

## Goals

* Learn how to run interactive, background and task containers
* Learn how to connect to containers from your Docker host
* Learn how applications run inside Windows containers
* Learn how to share your images by pushing them to Docker Hub

---

## Run a task in a Nano Server container

.exercise[
    - This is the simplest kind of container to start with. In PowerShell run:

    ```
    docker container run microsoft/nanoserver hostname
    ```]

You'll see the output written from the `hostname` command.

---

## Check for running containers

.exercise[
    - List all containers and you'll see your Nano Server container in the `Exited` state:

    ```
    docker container ls --all
    ```]

> Note that the container ID *is* the hostname that the container displayed.

???

Docker keeps a container running as long as the process it started inside the container is still running. 

In this case the `hostname` process completes when the output is written, so the container stops. The Docker platform doesn't delete resources by default, so the container still exists. 

---

## About task containers

.extra-details[
    Containers which do one task and then exit can be very useful. You could build a Docker image which installs the Azure PowerShell module and bundles a set of scripts to create a cloud deployment.]

.extra-details[
    Anyone can execute that task just by running the container - they don't need the scripts or the right version of the Azure module, they just need to pull the Docker image.]

???

I use a task container to create all the VMs for this workshop, using my [azure-vm-provisioner](https://github.com/sixeyed/dockerfiles-windows/tree/master/azure-vm-provisioner) image. That image packages up a set of Terraform scripts in an image with Terraform installed, so anyone can use it to provision VMs, using their own Azure subscription details.

---

## Run an interactive Windows Server Core container

.exercise[
    - Run this to start a Windows Server Core container and connect to it:

    ```
    docker container run --interactive --tty --rm microsoft/windowsservercore powershell
    ```]

When the container starts you'll drop into a PowerShell session with the default prompt `PS C:\>`.

???

Docker has attached to the console in the container, relaying input and output between your PowerShell window and the PowerShell session in the container.

The [microsoft/windowsservercore](https://hub.docker.com/r/microsoft/windowsservercore) image is effectively a full Windows Server 2016 OS, without the UI.

---

## Explore Windows Server Core

.exercise[
    - Run some commands to see how the Windows Server Core image is built:

    ```
    ls C:\
    Get-Process
    Get-WindowsFeature
    ```]

Now run `exit` to leave the PowerShell session, which stops the container process. 

???

Using the `--rm` flag means Docker now removes that container (if you run `docker container ls --all` again, you won't see the Windows Server Core container).

---

## About interactive containers

.extra-details[
    Interactive containers are useful when you are putting together your own image. You can run a container and verify all the steps you need to deploy your app, and capture them in a Dockerfile.]

.extra-details[
    You *can* [commit](https://docs.docker.com/engine/reference/commandline/commit/) a container to make an image from it - but you should avoid that wherever possible. It's much better to use a repeatable [Dockerfile](https://docs.docker.com/engine/reference/builder/) to build your image. You'll see that shortly.]

---

## Run a background SQL Server container

Background containers are how you'll run most applications.

.exercise[
    - Run SQL Server in the background as a detached container:

    ```
    docker container run --detach --name sql `
    --env ACCEPT_EULA=Y `
    --env sa_password=DockerCon!!! `
    microsoft/mssql-server-windows-express:2016-sp1
    ```]

> The workshop VM pre-loads a set of Docker images. If you don't have a local copy of an image, Docker will pull it when you first run a container. 

???

This example uses another image from Microsoft - [microsoft/mssql-server-windows-express](https://hub.docker.com/r/microsoft/mssql-server-windows-express/) which builds on top of the Windows Server Core image and comes with SQL Server Express installed.

---

## Exploring SQL Server

As long as the SQL Server process keeps running, Docker keeps the container running in the background.

.exercise[
    - Check what's happening by viewing the logs from the container, and seeing the process list:

    ```
    docker container logs sql
    docker container top sql
    ```]

---

## Connecting to SQL Server

.extra-details[
    
    The SQL Server instance is isolated in the container, because no ports have been made available to the host. Traffic can't get into Docker containers from the host, unless ports are explicitly published.]
    
.extra-details[
    You can't connect an external client - like SQL Server Management Studio - to this container (we'll see how to do that later on). Other containers in the same Docker network can access the SQL Server container, and you can run commands inside the container through Docker.]

---
## Running SQL commands inside the container

.exercise[
    - Check what the time is inside the database container:

    ```
    docker container exec sql `
    powershell "Invoke-SqlCmd -Query 'SELECT GETDATE()' -Database Master"
    ```]

> You can execute SQL statements using PowerShell cmdlets which have been packaged inside the container image.

---

## Connect to a background container

The SQL Server container is stil running in the background. 

.exercise[
    -  Connect an interactive PowerShell session to the container by running `exec`:

    ```
    docker container exec --interactive --tty sql powershell
    ```]

---

## Explore the filesystem and users in Windows containers

.exercise[
    -  Look at the `Program Files` directory, and then drill down into the SQL Server default file locations:

    ```
    ls 'C:\Program Files'
    cd 'C:\Program Files\Microsoft SQL Server'
    ls .\MSSQL13.SQLEXPRESS\MSSQL\data
    ```]

---

## Data stored in containers

.extra-details[
    
    The `.mdf` and `.ldf` files are stored inside the container. You can run SQL statements to store data, but when you remove the container, the data is lost. ]
    
.extra-details[
    For stateful services like database, you'll want to run them with the data physically stored outside of the container, so you can replace the container but retain the data. I'll cover that later in the workshop.]

---

## Processes in the SQL Server container

.exercise[
    - Check the processes running in the container:

    ```
    Get-Process
    ```]

> One is `sqlservr`, which is the database engine. There are also two `powershell` processes, one is the container startup process and the other is this PowerShell session. 

---

## Windows users in the SQL Server container

.exercise[
    - Compare the user accounts for the PowerShell processes:

    ```
    Get-Process -Name sqlservr -IncludeUserName
    Get-Process -Name powershell -IncludeUserName
    ```]

> The normal Windows user groups are there in the container, along with a special account for container processes.

---

## Accounts and user groups in Windows containers

.extra-details[
    
    The SQL Server process runs under the normal `NT AUTHORITY\SYSTEM` account. All the default user groups and accounts are present in the Windows Server Core Docker image, with all the usual access permissions. ]

.extra-details[
    The PowerShell processes are running as `User Manager\ContainerAdministrator`. That's the default account for processes running in Windows Docker containers, and it has admin privileges.]

---

## Check processes on the Windows host

On Windows Server 2016, those processes are actually running in isolated environments on the host. 

.exercise[
    - Open **another PowerShell terminal** and list all the PowerShell processes running on the server:

    ```
    Get-Process -Name powershell -IncludeUserName
    ```]

> You'll see the PowerShell sessions from the container processes - with the same process IDs but with a blank username. The container user doesn't map to any user on the host.

???
There are two important takeaways from this:

- Windows Server container processes run natively on the host, which is why they are so efficient
- container processes run as an unknown user on the host, so a rogue container process wouldn't be able to access host files or other processes.

---

## Disconnect from the container

.exercise[
    - Close the second PowerShell window, and exit the interactive Docker session in the first PowerShell window:

    ```
    exit
    ```]

The container is still running.

.exercise[
   - Now clean up, by removing all containers:

    ```
    docker container rm --force $(docker container ls --quiet --all)
    ```]
---

## Package and run a custom app using Docker

Next you'll learn how to package your own Windows apps as Docker images, using a [Dockerfile](https://docs.docker.com/engine/reference/builder/). 

The Dockerfile syntax is straightforward. In this task you'll walk through two Dockerfiles which package websites to run in Windows Docker containers. 

The first example is very simple, and the second is more involved. By the end of this task you'll have a good understanding of the main Dockerfile instructions.

---

## ASP.NET apps in Docker

Have a look at the [Dockerfile for this app](part-1/hostname-app/Dockerfile), which builds a simple ASP.NET website running that displays the host name of the server. There are only two instructions:

- [FROM](https://docs.docker.com/engine/reference/builder/#from) specifes the image to use as ther starting point for this image. `microsoft/aspnet` is an image owned by Microsoft, that comes with IIS and ASP.NET installed on top of Windows Server Core
- [COPY](https://docs.docker.com/engine/reference/builder/#copy) copies a file from the host into the image, at a known location.

The Dockerfile copies a simple `.aspx` file into the content directory for the default IIS website. 

---

## Build a simple website image

.exercise[
    - Run `docker image build` to execute the steps in the Dockerfile and package the app:

    ```
    cd "$env:workshop\part-1\hostname-app"
    docker image build --tag "$env:dockerId/hostname-app" .
    ```]

> The output shows Docker executing each instruction in the Dockerfile, and tagging the final image with your Docker ID.

---

## Run the new app

.exercise[
    - Run your website in a detached container, just like you did with SQL Server, but this time publishing the HTTP port so traffic can be passed from the host into the container:

    ```
    docker container run --detach --publish 80:80 `
    --name app "$env:dockerId/hostname-app"
    ```]

> Any external traffic coming into the server on port 80 will now be directed into the container. 

---

## Browse to the app

When you're connected to the host, to browse the website you can use the local (virtual) IP address of the container.

.exercise[
    - Get the container IP address:
    ```
    $ip = docker container inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app
    ```

    - Open the browser at the container's IP address and see the ASP.NET site:

    ```
    firefox "http://$ip"
    ```]

???
You need to use the container IP address locally because Windows doesn't have a full loopback networking stack. You can read more about that [on my blog](https://blog.sixeyed.com/published-ports-on-windows-containers-dont-do-loopback/).

---

## Run multiple instances of the website in containers

Let's see how lightweight the containerized application is. 

.exercise[
    - Run a PowerShell loop which starts five containers from the same website image:

    ```
    for ($i=0; $i -lt 5; $i++) {
        & docker container run --detach --publish-all --name "app-$i" "$Env:dockerId/hostname-app"
    }
    ```]

> The `publish-all` flag publishes all the ports from the container to random ports on the host. 

???
Only one process can listen on a port, and we're already using port 80 in the previous container. Using random host ports means we can start multiple containers, and access them using port 80 on the container.

---

## Check all the containers

.exercise[
    - List running containers:
    
    ```
    docker container ls
    ```]

.exercise[
    - Now this loop will fetch the IP address of each container and browse to it:

    ```
    for ($i=0; $i -lt 5; $i++) {
        $ip = & docker container inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' "app-$i"

        firefox "http://$ip"
    }
    ```]

> When the browsers have finished loading, you'll see that each site displays a different hostname, which is the container ID Docker generates. 

---

## See how much compute resources containers use

.extra-details[
    
    On the host you have six `w3wp` processes running, which are the IIS worker processes for each container.  You can see the memory and CPU usage with `Get-Process`:]

```
Get-Process -Name w3wp | select Id, Name, WorkingSet, Cpu
```

.extra-details[
    On my Azure VM, the worker processes average around 50MB of RAM and 5 seconds of CPU time.]

---

## Some issues to fix...

This is a simple ASP.NET website running in Docker, with just two lines in a Dockerfile. But there are two issues we need to fix:

- It took a few seconds for the site to load on first use
- We're not getting any IIS logs from the container

???
The cold-start issue is because the IIS service doesn't start a worker process until the first HTTP request comes in. The first website user takes the hit of starting the worker process.

---

## Logs inside containers

.extra-details[
    
    Run `docker container logs app-0` and you'll see there are no logs from the container.]
    
.extra-details[
    IIS stores request logs in the container filesystem, but Docker is only listening for logs on the standard output from the startup program.]
    
.extra-details[ 
    There's no automatic relay from the log files to the console output, so there are no HTTP access log entries in the containers.]

---

## Build and run a more complex website image

The next [Dockerfile](part-1/tweet-app/Dockerfile) is a better representation of a real-world script. These are the main features:

- it is based [FROM](https://docs.docker.com/engine/reference/builder/#from) `microsoft/iis:windowsservercore`, a clean Windows Server 2016 image, with IIS already installed
- it uses the [SHELL](https://docs.docker.com/engine/reference/builder/#shell) instruction to switch to PowerShell when building the Dockerfile, so the commands to run are all in PowerShell
- it configures IIS to write all log output to a single file, using the `Set-WebConfigurationProperty` cmdlet
- it copies the [start.ps1](part-1/tweet-app/start.ps1) startup script and [index.html](part-1/tweet-app/index.html) files from the host into the image
- it specifies `start.ps1` as the [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) to run when containers start. The script starts the IIS Windows Service and relays the log file entries to the console
- it adds a [HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#healthcheck) which makes an HTTP GET request to the site and returns whether it got a 200 response code

---

## Build the Tweet app

.exercise[
    - Build an image from the Dockerfile in the `tweet-app` directory:

    ```
    cd "$env:workshop\part-1\tweet-app"

    docker image build --tag "$env:dockerId/tweet-app" .
    ```]

> You'll see output on the screen as Docker runs each instruction in the Dockerfile. Once it's built you'll see a `Successfully built...` message. 

???

If you repeat the `docker image build` command again, it will complete in seconds. That's because Docker caches the [image layers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/) and only runs instructions if the Dockerfile has changed since the cached version.

---

## Browse to the new app

.exercise[
    - When the build completes, run the new app in the same way as the last app:

    ```
    docker container run --detach --publish 8080:80 `
    --name tweet-app "$env:dockerId/tweet-app"
    ```

    - Find the container IP address and browse to it:

    ```
    $ip = docker container inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' tweet-app

    firefox "http://$ip"
    ```]

> Feel free to hit the Tweet button, sign in and share your workshop progress :)

---

## List your images

.exercise[
    - List the images and filter on your Docker ID, you'll see the images you've built today, with the newest at the top:

    ```
    docker image ls -f reference="$env:dockerId/*"
    ```]

> Those images are only stored in your Azure VM, and that VM will be deleted after the workshop. Next we'll push the images to a public repository so you can run them from any Windows machine with Docker.
 
???

My Docker ID is `sixeyed`, so my output looks like this:

```
REPOSITORY                      TAG              IMAGE ID            CREATED              SIZE
sixeyed/tweet-app               latest           64fcfbceea4b        About a minute ago   10.8GB
sixeyed/hostname-app            latest           bf41287f7762        35 minutes ago       13.5GB
sixeyed/docker-workshop-verify  latest           dcf4c3874c4e        41 minutes ago       10.4GB
```

---

## Storing images in Docker registries

.extra-details[
    
    Distribution is built into the Docker platform. You can build images locally and push them to a public or private [registry](https://docs.docker.com/registry/), making them available to other users.]

.extra-details[
    Anyone with access can pull that image and run a container from it. The behavior of the app in the container will be the same for everyone, because the image contains the fully-configured app - the only requirements to run it are Windows and Docker.]

---

## Push images to Docker Hub

[Docker Hub](https://hub.docker.com) is the public registry for Docker images. You've already logged in using `docker login`, so now upload your images to the Hub:

.exercise[
    ```
    docker image push $env:dockerId/hostname-app
    docker image push $env:dockerId/tweet-app
    ```]

> You'll see the upload progress for each layer in the Docker image. 

???
The `hostname-app` image uploads quickly as it only adds one small layer on top of Microsoft's ASP.NET image. The `tweet-app` image takes longer to push - there are more layers, and the configured IIS layer runs to 40MB. 

---

## How big are the Docker images?

.extra-details[
    
    The logical size of those images is over 10GB each, but the bulk of that is in the Windows Server Core base image.]

.extra-details[
     Those layers are already stored in Docker Hub, so they don't get uploaded - only the new parts of the image get pushed. And Docker shares layers between images, so every image that builds on Windows Server Core will share the cached layers for that image.]

.extra-details[
    You can browse to [Docker Hub](https://hub.docker.com), login with your Docker ID and see your newly-pushed Docker images. These are public repositories, so anyone can pull the image - you don't even need a Docker ID to pull public images.]

---

## Next Up

That's it for Part 1. Next in [Part 2](part-2.md) we'll get stuck into modernizing an old ASP.NET app, by bringing it to a modern application platform.

.exercise[
    
    - Before we move on, let's clear up all the running containers:

    ```
    docker container rm --force $(docker container ls --quiet --all)
    ```]
