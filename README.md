# Assignment_3_Part_2

#### ip address for load balancer: 146.190.13.17
#### ip address for web1: 165.232.148.70
#### ip address for web2: 165.232.153.16

## Information

The purpose of this project is to...

## Part 1: Setting Up Load Balancer

For part 1 of this project we will create 2 droplets on digital ocean that are connected to a load balancer. The first step is to create 2 new droplets running Arch Linux with a tag "web". Then we will create a load balancer using these settings
- Regional, SFO3, same as your servers
- Default VPC, same as your servers
- External(public)
- Use the "web" tag to load balance all servers with a web tag in the SF03 region.

Once created you get an offline error on digital ocean for both droplets which is fine because we haven't logged onto the droplets and configured them with nginx yet.

We'll first add them to the `.ssh/config` file so that we can `ssh <droplet-name>` to log onto them. In my case I created 2 droplets called web1 and web2 and added them as such.

```
Host web1
  HostName 165.232.148.70 # web1's IP
  User arch
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/do-key
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null

Host web2
  HostName 165.232.153.16 # web2's IP
  User arch
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/do-key
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

Now we can log on to both droplets using `ssh web1` and `ssh web2`. Before we set up new users and download the files, we'll log onto both droplets in the terminal and add some packages we may need. Those packages are neovim, nginx, git, and ufw.

First we'll sync, refresh and update all packages.

```bash
sudo pacman -Syu
```

Then we can install the required packages.

```bash
sudo pacman -S neovim nginx git ufw
```

Now that we have the packages installed we can `cd` in both droplets which will take us to our home directory where we will begin part 2.

## Part 2: Setting Up A New Server

### Step 1: Create a System User

>[!note] The steps in part 2 will need to be performed on both droplets, you can do all the steps on one droplet and do it again on the second droplet or you can have both droplets open and do them both side by side.


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

In your droplets home directory perform the command below to download the necessary files to compete steps 2, 3 and 4. 

```bash
git clone https://github.com/KevinDusange/Assignment_3_Part_2.git
```

After downloading the scripts, cd into the git file and use the copy command below.

```bash
sudo cp generate_index /var/lib/webgen/bin/
```

And we also need to make sure the script can be executed and we can do this by using the command below to change the permissions:

```bash
sudo chmod +x /var/lib/webgen/bin/generate_index
```

### Step 3: Copy the systemd service file and timer file

Now we need to copy the `.service` and `.timer` files to the correct locations using the command below. Since these are `.service` and `.timer` files, they will need to be moved into the `/etc/systemd/system` directory.

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

Now we will need to move the `nginx.conf` file to the nginx configuration directory which is `/etc/nginx/nginx.conf` using the command below:

```bash
sudo cp nginx.conf /etc/nginx/nginx.conf
```

Next we will create the directories `sites-available` and `sites-enabled` using the command below:

```bash
sudo mkdir -p /etc/nginx/sites-available
sudo mkdir -p /etc/nginx/sites-enabled
```

Now we will copy the `server_block.conf` file to the correct location as well as creating a symlink using the command below to move it into the `sited-enabled` directory:

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

Done! Now your servers are up and running and using the load balaner IP we can see that it is balaning the load between both servers!