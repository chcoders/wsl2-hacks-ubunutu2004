# All changes here are specific to ubuntu 20.04
# wsl2-hacks
Useful snippets / tools for using WSL2 as a development environment

---

**Auto-start/services** (`systemd` and `snap` support)

I've done a few methods that have had various levels of success. My goal was to make it feel seamless for my workflow and have commands work as expected. What's below is the current version of the setup I use. It allows me to use the MS Terminal as well as VSCode's Remote WSL plugin.

With this setup your shells will be able to run `systemctl` commands, have auto-starting services, as well as be able to run [snaps](https://tutorials.ubuntu.com/tutorial/basic-snap-usage).

1. Install deps

    ```shell
    $ sudo apt update
    $ sudo apt install dbus policykit-1 daemonize
    ```

2. Exit out of / close the WSL2 shell

    The next step is to shutdown WSL2 and to change the default user to `root`.

    In a PowerShell terminal run:
    
    ```
    > wsl --shutdown
    > ubuntu2004.exe config --default-user root

3. Create a fake-`bash`

    This fake shell will intercept calls to `wsl.exe bash ...` and forward them to a real bash running in the right environment for `systemd`. If this sounds like a hack-- well, it is. However, I've tested various workflows and use this daily. That being said, your mileage may vary.

    ```
    $ touch /usr/bin/bash-bootstrap-services
    $ chmod +x /usr/bin/bash-bootstrap-services
    $ code /usr/bin/bash-bootstrap-services
    ```
    Code can be replaced with your editor of choice
    
    Add the following, be sure to replace `<YOURUSER>` with your WSL2 Linux username

    ```sh
    #!/bin/bash
    # your WSL2 username
    UNAME="<YOURUSER>"

    UUID=$(id -u "${UNAME}")
    UGID=$(id -g "${UNAME}")
    UHOME=$(getent passwd "${UNAME}" | cut -d: -f6)
    USHELL=$(getent passwd "${UNAME}" | cut -d: -f7)

    if [[ -p /dev/stdin || "${BASH_ARGC}" > 0 && "${BASH_ARGV[1]}" != "-c" ]]; then
        USHELL=/bin/bash
    fi

    if [[ "${PWD}" = "/root" ]]; then
        cd "${UHOME}"
    fi

    # get pid of systemd
    SYSTEMD_PID=$(pgrep -xo systemd)

    # if we're already in the systemd environment
    if [[ "${SYSTEMD_PID}" -eq "1" ]]; then
        exec "${USHELL}" "$@"
    fi

    # start systemd if not started
    /usr/bin/daemonize -l "${HOME}/.systemd.lock" /usr/bin/unshare -fp --mount-proc /lib/systemd/systemd --system-unit=basic.target 2>/dev/null
    # wait for systemd to start
    while [[ "${SYSTEMD_PID}" = "" ]]; do
        sleep 0.05
        SYSTEMD_PID=$(pgrep -xo systemd)
    done

    # enter systemd namespace
    exec /usr/bin/nsenter -t "${SYSTEMD_PID}" -m -p --wd="${PWD}" /sbin/runuser -s "${USHELL}" "${UNAME}" -- "${@}"
    ```

4. Test the Script
    
    In a PowerShell terminal run:
    
    ```
    > wsl --shutdown
    ```

    Fire up WSL via the MS Terminal or just `wsl.exe`.
    Start ubunutu and test the script you created by running 
    ```sh
    $  bash /usr/bin/bash-bootstrap-services
    ```
    
    If You are logged in as your normal user then the script is working good and `systemd` should be running
    
    You can test by running the following in WSL2:
    
    ```sh
    $ systemctl is-active dbus
    active
    ```
    
5. Set the fake-`bash` as our `root` user's shell

    We need `root` level permission to get `systemd` setup and enter the environment. The way I went about solving this is to
    have WSL2 default to the `root` user and when `wsl.exe` is executed the fake-`bash` will do the right thing.
    
    The next step in getting this working is to change the default shell for our `root` user.
    
    Edit the `/etc/passwd` file:
    
    `$ sudo editor /etc/passwd`
    
    Find the line starting with `root:`, it should be the first line.
    Change it to:
    
    `root:x:0:0:root:/root:/usr/bin/bash-bootstrap-services`
    
    *Note the `/usr/bin/bash-bootstrap-services` here, slight difference*
    
    Save and close this file. Close the WSL and start again. Eveything should be working fine.

    ```
    


6. Create `/etc/rc.local` (optional)

    If you want to run certain commands when the WSL2 VM starts up, this is a useful file that's automatically ran by systemd.
    
    ```shell
    $ sudo touch /etc/rc.local
    $ sudo chmod +x /etc/rc.local
    $ sudo editor /etc/rc.local
    ```
    
    Add the following:
    ```sh
    #!/bin/sh -e
    
    # your commands here...
    
    exit 0
    ```

`/etc/rc.local` is only run on "boot", so only when you first access WSL2 (or it's shutdown due to inactivity/no-processes).
To test you can shutdown WSL via PowerShell/CMD `wsl --shutdown` then start it back up with `wsl`.

---


**Increase `max_user_watches`**

If devtools are watching for file changes, the default is too low.

```
# /etc/rc.local runs as root by default
# if you run these yourself add 'sudo' to the beginning of each command

sysctl -w fs.inotify.max_user_watches=524288
```

---

**Open MS Terminal to home directory by default**

Open your MS Terminal configuration <kbd>Ctrl+,</kbd>

Find the `"commandLine":...` config for the WSL profile.

Change to something like:

```json
"commandline": "wsl.exe ~ -d Ubuntu-18.04",
```

**Get current IP of WSL2 into Windows clipboard** :

Create a file called getip with the following command 
    ```sh
    $  sudo touch /usr/local/bin/getip
    $  sudo cjmod +x /usr/local/bin/getip 
    ```
 Put the following code inside the getip file using your editor   
```
hostname -I | awk '{print $1}' | awk '{printf "%s", $0}' | clip.exe
hostname -I | awk '{print $1}' | awk '{printf "%s", $0}' 
echo
```

Now everytime you type getip you will be shown the IP and it will be copied into the clipboard.

**Run sudo code command for files with Previliges**

Currently you cannot run sudo code command. Using defualt user as root is not recommended. So best approach is to use rmate.

1. Install rmate on your WSL VM
sudo wget -O /usr/bin/rmate https://raw.githubusercontent.com/aurora/rmate/master/rmate
sudo chmod a+x /usr/bin/rmate

2. Install the Remote VS Code plugin
make sure the Extension is enabled on WSL: after adding the plugin.
Here is how I configured the remote VS Code plugin
File -> Preferences -> Settings

3. Start the VSCode rmate server
Press F1 and run Search for the Remote: Start Server command.


4. Edit your privileged files
Start your WSL instance and open a terminal. If you've done everything correctly you should be able to now edit your files with sudo priveledges in your editor, even if you are not the root user.

sudo rmate /etc/profile.d/custom-profile.sh

**Make it possible to run sudo code to change root files from wsl2**

Currently sudo code doesnt work from wsl2. To make sudo code run you need to modify to configuration file ```/etc/sudoers```. Please open the file and modify the secure_path to enable run VS code with sudo command as below
``` Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/mnt/c/Program Files/Vscodium/bin"```
Just add the path to the bin folder of the VS code installation. Please note I use VScodium instead of VS Code so slight difference.


