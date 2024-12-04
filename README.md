# Assignment_3_Part_2

#### ip address for load balancer: 146.190.13.17
#### ip address for web1: 165.232.148.70
#### ip address for web2: 165.232.153.16

### Part 1: Setting Up A New Server

### Step 1: Create a System User

The reason we are creating a system user instaed of a regular user is because a system user has better security due to limited privelages and a system user is better suited for running system service tasks such as running nginx due to its isolation from regular users. 

We will create a new system user called `webgen` using the command below:

```bash
sudo useradd --system -d /var/lib/webgen -s /usr/sbin/nologin webgen
```

`--system`: This option creates a system user (UID below 1000) instead of a regular user

`-d`: This option specifies the home directory of the new user

`-s`: This option specifies the login shell of the new user (in this case we will set nologin because it is a system user)

Now we will create the sub-directories inside of the new users home directory using the commands below:

```bash
sudo mkdir -p /var/lib/webgen/HTML
sudo mkdir -p /var/lib/webgen/bin
```

And finally we will change the ownership of the newly created directory using the command below:

```bash
sudo chown -R webgen:webgen /var/lib/webgen
```

`-R`: This option means the ownership will be applied recursively which means it will change the ownership of the `/var/lib/webgen` directory as well as all its subdirectories (such as bin and HTML). 

#### For steps 2, 3, and 4 I am showing the commands using `cp` but you can also use `mv`

`cp` will copy the sripts to the specified locations but also leave the original file in the original downloaded location

`mv` will move the file to the specified location and will not leave a copy anywhere

### Step 2: Copy the generate_index script

After downloading the generate_index script use the copy command below to copy it to the correct location:

```bash
sudo cp generate_index /var/lib/webgen/bin/
```

And we also need to make sure the script can be executed and we can do this by using the command below to change the permissions:

```bash
sudo chmod +x /var/lib/webgen/bin/generate_index
```

### Step 3: Copy the systemd service file and timer file

After downloading the `generate-index.server` and `generate-index.timer` files we need to copy the files to the correct locations using the command below. Since these are .service and .timer files, they will need to be moved into the `/etc/systemd/system` directory because that is where those types of files belong.

```bash
sudo cp generate-index.service /etc/systemd/system/
sudo cp generate-index.timer /etc/systemd/system/
```

After copying the files to the correct locations we will use the commands below to reload systemctl so it can see the new files and also enable the timer so it is actively running right away.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now generate-index.timer
```

To check if the timer is active we can run the command below:

```bash
sudo systemctl status generate-index.timer
```

To check if the service runs correctly we can run the command below:

```bash
sudo systemctl status generate-index.service
```

### Step 4: Copy the nginx configuration files

After downloading the `nginx.conf` files we will need to move it to the nginx configuration directory which is `/etc/nginx/nginx.conf` using the command below:

```bash
sudo cp nginx.conf /etc/nginx/nginx.conf
```

Next we will create the directories `sites-available` and `sites-enabled` using the command below:

```bash
sudo mkdir -p /etc/nginx/sites-available
sudo mkdir -p /etc/nginx/sites-enabled
```

After downlaodint the `server_block.conf` file we will need to copy it to the correct location as well as creating a symlink using the command below to move it into the enabled sites directory:

```bash
sudo cp server_block.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/server_block.conf /etc/nginx/sites-enabled/
```

For more information as to why we created these directories refer to section 3.2.3.1 on the nginx Arch Wiki page
https://wiki.archlinux.org/title/Nginx

The main reason it is important to use a separate server block file instead of modifying the main `nginx.conf` file directly is because we can enable or disable certain sites using symbolic links. We can include sites-enabled to the end of the http block in the `nginx.conf` file so we can control which sites to be enabled. I have already added this in the `nginx.conf` file. 

Finally to test our `nginx` we can use the command below:

```bash
sudo nginx -t
```

We can also view the status of `nginx` using the command below:

```bash
sudo systemctl status nginx
```

### Step 5: Configure UFW (Uncomplicated Firewall)

First we'll install and enable `UFW` using the commands below:

It's also important to allow an ssh connection before enabling the firewall so we will go in the order below:

```bash
sudo pacman -S ufw
sudo ufw allow ssh
sudo systemctl enable --now ufw.service
```

Now we'll configure `UFW` to:

- allow ssh and http from anywhere (in practice we would probably only allow ssh from our home or work ip)
- enable ssh rate limiting

```bash
sudo ufw allow ssh # already did it above
sudo ufw allow http
sudo ufw limit ssh
```

Finally we can check the status of our firewall using the command below:

```bash
sudo ufw status verbose
```

You should see something like this:

```bash
To                         Action      From
--                         ------      ----
80/tcp                     ALLOW IN    Anywhere
22/tcp                     ALLOW IN    Anywhere
80/tcp (v6)                ALLOW IN    Anywhere (v6)
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

Done! Now your server is up and running!

### Part 2: Setting Up Load Balancer



