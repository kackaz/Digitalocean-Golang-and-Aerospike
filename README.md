# How to setup Go and Aerospike in digitalocean Ubuntu 15.10

## Setup ssh keys
* For UNIX
    1. Check if key already exists.
        * `cat ~/.ssh/id_rsa.pub`
    1. If key does not exist, generate it.
        * `ssh-keygen`
        * The default location is best.
        * A passcode is only needed for high security sites, just don't lose your computer.
        * You can find your public key in the file with a .pub extension.
* For Windows
    1. Run puttyGen and generate a key.
    1. You can get the public key in puttyGen in the upper window.
    1. Save the private key somewhere.
    1. Make sure to point putty and WinSCP to the private key.

## Setup Ubuntu user account
1. Create a server.
    * Make sure to set the server to ubuntu 15.10.
    * 32-bit is recommended unless you have more than 4gb of RAM, except for if you use Aerospike, which requires 64-bit for both main database and API.
    * Make sure you turn on private networking so you can connect to your database without using up bandwidth.
    * Make sure you have put in an ssh key here to avoid unsecure password logins.
1. Connect to server's root account.
    * Use ssh on unix machines.
        * `ssh root@<ip_address>`
    * Make sure you have setup ssh keys with your computer.
    * Windows user should use putty.
1. Create a user with sudo access.
    * `adduser <username>`
    * Make sure to put in a good password!
    * You can leave the other settings blank, just keep pressing enter.
    * Give the user sudo access.
        * `gpasswd -a <username> sudo`
1. Add ssh key access to new user account.
    * Flip your access to the new user.
        * `su <username>`
    * Move to the home directory.
        * `cd`
    * Create folder and restrict access to only yourself.
        * `mkdir .ssh`
        * `chmod 700 .ssh`
    * Create a file and add ssh key to it.
        * `nano .ssh/authorized_keys`
        * Paste key into file and save and exit with Ctrl-X.
            * Ctrl-Shift-V for unix users to paste.
            * Right-click in window to paste for putty.
    * Restrict the permissions of the file.
        * `chmod 600 .ssh/authorized_keys`
    * Return to root.
        * `exit`
    * Test if it worked.
        * Connect to your new account with either putty or ssh
            * `ssh <username>@<ip_address>`
        * If it asks for your password, something went wrong.
1. Restrict ssh access to root and password connections.
    * As root, edit the settings in the ssh config file.
        * `nano /etc/ssh/sshd_config`
        * Set the line `PermitRootLogin` to no to disable root login.
        * Set the line `PasswordAuthentication` to no to disable logging in with a password.
            * Make sure to uncomment the line as well.
        * Restart the ssh service.
          * `service ssh restart`
    * Make sure you test if you can still access it with normal connection before you disconnect the root terminal.
    * And test that the root login really is disabled.

## Setup additional helpful items
1. Setup firewall.
    * Allow ssh through the firewall.
        * `sudo ufw allow ssh`
        * or `sudo ufw allow 22/tcp`
    * Examine the rules.
        * `sudo ufw show added`
    * If everything looks right, enable the firewall.
        * `sudo ufw enable`
    * Make sure everything is running right.
        * `sudo ufw status`
1. Synchronize the system clock.
    * Set timezone.
        * `sudo dpkg-reconfigure tzdata`
        * A graphical menu will allow you to choose a city to sync time with.
    * Install NTP.
        * If you have not used apt-get yet, run `sudo apt-get update`
        * `sudo apt-get install ntp`
        * ntp will automatically place enable run on boot.
1. Create a swapspace.
    * Reserve the space.
        * `sudo fallocate -l <size> /swapfile`
        * `<size>` is something like `1G` or `512M`
    * Restrict access to root only.
        * `sudo chmod 600 /swapfile`
    * Configure into a swapfile.
        * `sudo mkswap /swapfile`
    * Start using the swapfile.
        * `sudo swapon /swapfile`
    * Setup automatically using the swapfile on boot.
        * `sudo sh -c 'echo "/swapfile none swap sw 0 0" >> /etc/fstab'`
1. This is a good point to make a snapshot of your server.
    * Shut the server down.
        * `sudo poweroff`
    * Save a snapshot in the digitalocean console.

## Get a Go server running
1. (optional) install Go.
    * If you have Go 1.5 or newer, you can cross compile most programs and transfer the executable.
    * Some packages still require a native Go install to build though.
    * Download Go.
        * `wget <url>`
        * The url for 1.5.1 is `https://storage.googleapis.com/golang/go1.5.1.linux-amd64.tar.gz`
    * Extract Go from the archive file.
        * `tar -xzf <filename>`
    * Move Go to the default install location.
        * `sudo mv go /usr/local/go`
    * Change owner to root and alter permissions.
        * `sudo chown root:root /usr/local/go`
        * `sudo chmod 755 /usr/local/go`
    * Create workspace folder.
        * `mkdir <workspace_name>{,/bin,/pkg,/src}`
    * Edit environment variables.
        * Add `export PATH=$PATH:/usr/local/go/bin` to `/etc/profile`
        * Add `export GOPATH=$HOME/<workspace_name>` to `~/.profile`
        * Add `export PATH=$HOME/<workspace_name>/bin:$PATH` to `~/.profile`
    * Delete the go archive file.
        * `rm <filename>`
    * Install git.
        * `sudo apt-get install git`
    * Reconnect to the server to allow environment variables to update.
1. Adjust firewall to allow http connections.
    * Allow http through the firewall.
        * `sudo ufw allow http`
        * or `sudo ufw allow 80/tcp`
    * Allow https through the firewall, if needed.
        * `sudo ufw allow https`
        * or `sudo ufw allow 443/tcp`
1. Setup haproxy.
    * Install haproxy.
        * `sudo apt-get install haproxy`
    * Configure haproxy.
        * Edit `/etc/haproxy/haproxy.cfg`
        * Add `retries 3` to the default section.
        * Add `option redispatch` to the default section.
        * Add the following block to the end of the file:
        ```
        listen serv 0.0.0.0:80
            mode http
            option http-server-close
            timeout http-keep-alive 3000
            server serv 127.0.0.1:9000 check
        ```

    * More information available [here](https://www.digitalocean.com/community/tutorials/how-to-use-haproxy-to-set-up-http-load-balancing-on-an-ubuntu-vps).
  * Reload haproxy
    * `sudo service haproxy reload`
1. Get your code onto the server.
    * If you are on windows, use WinSCP.
    * If you are on a unix machine, use scp.
        * `scp <source> <destination>`
        * Add -rp if it is a folder you are transfering.
        * `scp -rp <source> <destination>`
        * The format for remote connections is `<username>@<ip_address>:<path>`
        * Example: `scp -rp ~/Desktop/testServer daniel@107.170.246.157:~/testServer`
    * Make sure it is built, whether on your system or on the server directly.
1. Configure systemd.
    * Create configuration file: `sudo nano /etc/systemd/system/<filename>.service`
    * Add the following code to the file:
    ```
    [Unit]
    Description=Go Server

    [Service]
    ExecStart=/home/<username>/<exepath>
    WorkingDirectory=/home/<username>/<exefolderpath>
    User=<username>
    Group=<username>
    Restart=always

    [Install]
    WantedBy=multi-user.target
    ```

    * Add the service to systemd.
        * `sudo systemctl enable <filename>.service`
    * Activate the service.
        * `sudo systemctl start <filename>.service`
    * Check if systemd started it.
        * `sudo systemctl status <filename>.service`
    * More information about systemd commands can be found [here](http://www.linux.com/learn/tutorials/788613-understanding-and-using-systemd/).
    * Check if the server is running with your web-browser, just use the server ip address as the url.

## Setup Aerospike server
1. Download and install Aerospike.
  * Aerospike only works for 64-bit machines unless you build it from source yourself, and recommends at least 2gb of RAM.
  * You can get step-by-step instructions for installation [here](http://www.aerospike.com/get-started/#/linux).
  * Download the archive file.
    * `wget -O aerospike.tgz 'http://aerospike.com/download/server/latest/artifact/ubuntu12'`
  * Extract the archive file.
    * `tar -xvf aerospike.tgz`
  * Go into the directory on run the installer.
    * `cd aerospike-server-community-*-ubuntu12`
    * `sudo ./asinstall`
  * Allow the database port through the firewall.
    * `sudo ufw allow in on eth1 to any port 3000 proto tcp`
  * Start the service.
    * `sudo service aerospike start`
    * Check when it is ready with: `sudo tail -f /var/log/aerospike/aerospike.log | grep cake`
  * Delete the aerospike install files.
    * rm -rf aerospike*
1. Install Aerospike management server (optional).
  * Install python2.x, python development libraries, and gcc
    * `sudo apt-get install python gcc python-dev`
  * Download the package file.
    * `wget -O amc.deb http://www.aerospike.com/download/amc/latest/artifact/ubuntu12`
  * Install the server.
    * `sudo dpkg -i amc.deb`
  * Allow the server port through the firewall.
    * `sudo ufw allow 8081/tcp`
  * Start the server.
    * `sudo /etc/init.d/amc start`
  * Examine the amc in your web-browser, address is: `<server_ip>:8081`
    * When it asks you for the ip of a node, enter the localhost ip: `127.0.0.1`
  * Delete amc install file.
    * `rm amc.deb`
1. Configure Aerospike.
  * Add namespaces as needed.
    * `sudo nano /etc/aerospike/aerospike.conf`
    * At the bottom of the file is the test and bar namespaces, comment them out and use them as examples.
    * This is also the file where you can configure having multiple nodes in a cluster. More information on configuring Aerospike [here](http://www.aerospike.com/docs/operations/configure/network/).
  * Restart Aerospike.
    * `sudo service aerospike restart`

## Get Go Aerospike library and test server
  1. Get the go client library (64-bit only).
    * `go get github.com/aerospike/aerospike-client-go`
  1. Run the benchmark tool, (64-bit only).
    * Change into the client code directory, tools/benchmark
      * `cd $GOPATH/src/github.com/aerospike/aerospike-client-go/tools/benchmark`
    * Run the tool.
      * `go run benchmark.go -h <ip_address>`
      * Note this will only work from a server in digitalocean, since the firewall is configured to only allow connections from eth1, which is the private network.
      * Private ip address can be found with: `ifconfig | grep "inet addr"`, the middle address should be the private one.

## Additional API help
  * The [godoc](https://godoc.org/github.com/aerospike/aerospike-client-go) page is very large, but has everything, including enterprise edition commands.
  * Information on connecting can be found [here](http://www.aerospike.com/docs/client/go/usage/connect/).
  * Information on writing a record, including how to write to a single value in a field and how to set an expiration date for data can be found [here](http://www.aerospike.com/docs/client/go/usage/kvs/write.html).
  * Information on reading a record, including only getting parts of an object, can be found [here](http://www.aerospike.com/docs/client/go/usage/kvs/read.html).
  * Information on queries can be found [here](http://www.aerospike.com/docs/client/go/usage/query/query.html).
  * When you are querying on something, make sure you add a secondary index for that field. You can do that programmatically with Go, or using the Aerospike management server.
