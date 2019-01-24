---
layout: article
title: "Running EXEs from Windows Subsystem for Linux (WSL)"
categories: posts
modified: 2019-01-23T20:00:00-04:00
tags: [windows, wsl, tips]
comments: true
ads: false
---
I've been playing around with Ubuntu 18.04 running under the Windows Subsystem for Linux (WSL) and I have to say it's pretty slick. Previously, for a Bash-shell-on-Windows experience, I was using GIT Bash (which runs in MinGW).

One feature of WSL that you might not know about is that you can actually invoke Windows programs directly from the WSL shell and they'll launch in Windows. Even better, the standard out and standard error of the application will actually appear in the WSL shell window just as you'd expect. The GIT Bash shell was able to do this as well, but it didn't always work seamlessly for some applications like NodeJS, Docker, and Kubernetes which were sensitive to detecting the type of TTY.

Unfortunately, WSL requires you to suffix Windows programs with a `.exe` extension, limiting the portability of scripts. So, for example, if you're trying to run a Bash script written for Linux to invoke something like the `docker` or `kubectl` CLI, you'll get errors like `docker: command not found` (since it needs to invoke `docker.exe`). This limitation of WSL isn't necessarily a bad thing -- it helps avoid conflicts between a version of a utility installed inside the WSL OS (e.g. an Ubuntu package) vs a similarly-named application installed in Windows. Still, it can be useful not to have to install the same tools in both WSL Ubuntu and the host Windows system.

**Here's a power-user tip to make this possible:**
1. Run the following command in the WSL shell:

   ```bash
   mkdir -p ~/bin
   ```
   ```bash
   cat > ~/bin/exe_alias
   ```

2. Now, paste the following content in:

   ```bash
    #!/usr/bin/env bash
   ```
   ```bash
    SCRIPT_NAME=$(basename $0)
   ```
   ```bash  
    "${SCRIPT_NAME}.exe" "$@"
   ```

3. Now, make the script executable with the following command:

   ```bash
    chmod u+x ~/bin/exe_alias
   ```

4. Lastly, create symlink aliases for each command you want to be able to invoke from within WSL without the `.exe` extension. For example, if I wanted to be able to run `kubectl` and `docker` inside WSL using the versions of these tools installed in Windows, I'd execute the following commands:

   ```bash
    ln -snf ~/bin/exe_alias ~/bin/kubectl
   ```
   ```bash
    ln -snf ~/bin/exe_alias ~/bin/docker
   ```

Now, I can just run `docker` in the WSL shell while in any folder and `docker.exe` will be executed from the host machine.

The script basically reflects on the name of whatever command you ran (e.g. `~/bin/kubectl`) and then turns that into a call to an application with the same name followed by `.exe` (e.g. `kubectl.exe`).
