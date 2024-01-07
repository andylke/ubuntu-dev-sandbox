# ubuntu-dev-sandbox


## Configure Network IP Address
```
Subnet: 192.168.100.0/24 
IP: 192.168.100.200 
Gateway: 192.168.100.1 
Name Servers: 8.8.8.8, 1.1.1.1 

hostname: dev-sandbox 
user: dev
password: dev
```

Upgrade Packages

```
sudo apt update
sudo apt upgrade -y
```

Install Nano, Less

```
sudo apt install nano less -y
```

Set FQDN


```
sudo nano /etc/hosts
```

```
127.0.0.1 dev-sandbox.vm.internal dev-sandbox
```

## Install VirtualBox Guest Additions


Installs the required software packages for building kernel modules.
```
sudo apt install build-essential dkms linux-headers-$(uname -r) libxt6 libxmu6 -y
```

First, try using the blkid command to see what device file your CD is using. Usually, this is going to be /dev/sr0, but it’s possible that yours is something different. You’ll know it’s the right one because it should say ISO9660.
```
blkid | grep "iso9660"
```

```
/dev/sr0: BLOCK_SIZE="2048" UUID="2022-10-19-20-19-46-75" LABEL="VBox_GAs_7.0.2" TYPE="iso9660"
```

create directory /media/cdrom and mount the device file to it.
```
sudo mkdir /media/cdrom
```

```
sudo mount /dev/sr0 /media/cdrom
```

```
mount: /media/cdrom: WARNING: source write-protected, mounted read-only.
```

Run the VirtualBox guest additions installer.
```
sudo /media/cdrom/VBoxLinuxAdditions.run
```

Reboot 
```
sudo reboot
```

## Copy CA Certificates

Mount host certs directory to /media/certs directory

Copy certificate key from mounted /media/certs to /etc/ssl/private directory
```
sudo cp /media/certs/Internal_VM_Dev-Sandbox_CA.key /media/certs/Internal_VM_Dev_CA.key /etc/ssl/private/
```

```
sudo ls /etc/ssl/private | grep "Internal_VM"
```

```
Internal_VM_Dev-Sandbox_CA.key 
Internal_VM_Root_CA.key
```

Copy certificate key from mounted /media/certs to /etc/ssl/cert directory
```
sudo cp /media/certs/Internal_VM_Dev-Sandbox_CA.crt /media/certs/Internal_VM_Dev_CA.crt /etc/ssl/certs
```

```
sudo ls /etc/ssl/certs | grep "Internal_VM"
```

```
Internal_VM_Dev-Sandbox_CA.crt 
Internal_VM_Root_CA.crt
```


## Install OpenJDK 17

```
sudo apt install openjdk-17-jdk-headless -y
```

Verify installed version
```
java -version
```

```
openjdk version "17.0.5" 2022-10-18 
OpenJDK Runtime Environment (build 17.0.5+8-Ubuntu-2ubuntu122.04) 
OpenJDK 64-Bit Server VM (build 17.0.5+8-Ubuntu-2ubuntu122.04, mixed mode, sharing)
```


## Install Firewall

```
sudo apt install ufw -y
```

Deny all incoming, allow all outgoing
```
sudo ufw default deny incoming
```

```
sudo ufw default allow outgoing
```

Allow SSH port 22
```
sudo ufw allow ssh
```

Enable firewall
```
sudo ufw enable
```

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y 
Firewall is active and enabled on system startup
```

Verify status
```
sudo ufw status
```

```
Status: active 
To                         Action      From 
--                         ------      ---- 
22/tcp                     ALLOW       Anywhere 
22/tcp (v6)                ALLOW       Anywhere (v6)
```


## Install Nginx


```
sudo apt install nginx -y
```

Verify installed version
```
nginx -v
```

```
nginx version: nginx/1.18.0 (Ubuntu)
```

Verify service status
```
sudo systemctl status nginx
```

Enable service to start automatically during system startup
```
sudo systemctl enable nginx
```

```
Synchronizing state of nginx.service with SysV service script with /lib/systemd/systemd-sysv-install. 
Executing: /lib/systemd/systemd-sysv-install enable nginx
```

Configure firewall to allow http 80 and https 443
```
sudo ufw allow 'Nginx Full'
```

```
Rule added 
Rule added (v6)
```

Verify status
```
sudo ufw status
```

```
Status: active 
To                         Action      From 
--                         ------      ---- 
22/tcp                     ALLOW       Anywhere 
Nginx Full                 ALLOW       Anywhere 
22/tcp (v6)                ALLOW       Anywhere (v6) 
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

Configure SSL
```
sudo nano /etc/nginx/sites-enabled/default
```
Replace with following config. Use CTRL+SHIFT+K to delete line.
```
server { 
    listen 80; 
    listen [::]:80; 
    server_name dev-sandbox.vm.internal; 
    location / { 
        rewrite ^ https://$host$request_uri? permanent; 
    } 
} 

server { 
    listen 443 ssl; 
    listen [::]:443 ssl; 
    server_name dev-sandbox.vm.internal; 
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    ssl_certificate /etc/ssl/certs/Internal_VM_Dev-Sandbox_CA.crt;
    ssl_certificate_key /etc/ssl/private/Internal_VM_Dev-Sandbox_CA.key; 
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / { 
        try_files $uri $uri/ =404; 
    }

}
```

Verify syntax
```
sudo nginx -t
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok 
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Restart Nginx service
```
sudo systemctl restart nginx
```
Open URL https://dev-sandbox for nginx homepage


## Install Jenkins


Add the repository key for stable release
```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
```
Add stable release package repository address to source.list
```
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```
Update and install
```
sudo apt update
sudo apt install -y jenkins
```

Configure URL prefix
```
sudo systemctl edit jenkins
```
Add  
```
### Anything between here and the comment below will become the new contents of the file
[Service]
Environment="JENKINS_PREFIX=/jenkins"
```

Start and verify service status
```
sudo systemctl daemon-reload
sudo systemctl restart jenkins.service
sudo systemctl status jenkins.service
```

Enable service to start automatically during system startup
```
sudo systemctl enable jenkins.service
```

Configure Nginx reverse proxy
```
sudo nano /etc/nginx/sites-enabled/default
```
Append the following location config in server listening to 443
```
    location /jenkins/ {  
        proxy_pass http://localhost:8080/jenkins/;
        proxy_set_header Host                 $http_host; 
        proxy_set_header X-Real-IP            $remote_addr;  
        proxy_set_header X-Forwarded-For      $proxy_add_x_forwarded_for;  
        proxy_set_header X-Forwarded-Proto    $scheme; 
        proxy_set_header X-Forwarded-Host     $host;
        proxy_set_header X-Nginx-Proxy        true;
        proxy_http_version 1.1;  
        proxy_set_header Upgrade $http_upgrade;  
        proxy_set_header Connection "upgrade";

        proxy_max_temp_file_size 0; 
        client_max_body_size       10m; 
        client_body_buffer_size    128k; 
        proxy_connect_timeout      90; 
        proxy_send_timeout         90; 
        proxy_read_timeout         90; 
        proxy_buffering            off; 
        proxy_request_buffering    off;
    } 

```
Verify syntax
```
sudo nginx -t
```
Restart Nginx service
```
sudo systemctl restart nginx
```
Open Jenkins at https://dev-sandbox/jenkins
Display initial password
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Create admin admin/admin@5354 
Install plugins
```
Pipeline
Pipeline: Stage View
Folders
Build Timeout
Credentials Binding
Timestamper
Workspace Cleanup
Git
Matrix Authorization Strategy
Email Extension
Mailer
NodeJS Plugin
Maven Integration
Pipeline Maven Integration
```


## Install Nexus


Install JDK-8
```
sudo apt install openjdk-8-jdk-headless -y
```

Set max open files for both hard and soft limits to 65536
```
sudo nano /etc/security/limits.d/nexus.conf
```
Add the following config to set ulimit for specific user nexus with the value 65536 
```
nexus - nofile 65536
```

Download latest and extract to /opt/nexus and /opt/sonatype-work
```
wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz 
tar -zxvf latest-unix.tar.gz
mv $(ls | grep "nexus-3.*") nexus 
sudo mv nexus /opt
sudo mv sonatype-work /opt/sonatype-work 
rm latest-unix.tar.gz
```
Create new user nexus
```
sudo adduser --home /opt/nexus --shell /bin/bash --disabled-password nexus
```

```
Adding user `nexus' ... 
Adding new group `nexus' (1001) ... 
Adding new user `nexus' (1001) with group `nexus' ... 
Creating home directory `/opt/nexus' ... 
Copying files from `/etc/skel' ... 
Changing the user information for nexus 
Enter the new value, or press ENTER for the default 
        Full Name []: 
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] y
```
Change the ownership of both directories to the user and group 'nexus'
```
sudo chown -R nexus:nexus /opt/nexus /opt/sonatype-work
```

Create link to /var/log/nexus
```
sudo ln -s /opt/sonatype-work/nexus3/log /var/log/nexus
```

Change configuration to run as user nexus 
```
sudo nano /opt/nexus/bin/nexus.rc
```
Uncomment and change the following property
```
run_as_user="nexus"
```

Change max heap size for nexus to 512m
```
sudo nano /opt/nexus/bin/nexus.vmoptions
```

```
-Xms512m 
-Xmx512m 
-XX:MaxDirectMemorySize=512m
```

Change nexus to run with JDK-8
```
sudo nano /opt/nexus/bin/nexus
```
Add 
```
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/lib/jvm/java-8-openjdk-amd64
```

To run Nexus with as service, create nexus.service 
```
sudo nano /etc/systemd/system/nexus.service
```

```
[Unit] 
Description=nexus service 
After=network.target 

[Service] 
Type=forking 
LimitNOFILE=65536 
ExecStart=/opt/nexus/bin/nexus start 
ExecStop=/opt/nexus/bin/nexus stop 
User=nexus 
Restart=on-abort 

[Install] 
WantedBy=multi-user.target
```

Reload system-md manager, start and enable running nexus service at system startup
```
sudo systemctl daemon-reload
sudo systemctl start nexus.service 
sudo systemctl enable nexus.service
sudo systemctl status nexus.service
```

Configure Nginx reverse proxy
```
sudo nano /etc/nginx/sites-enabled/default
```
Append the following location config in server listening to 443
```
    location /nexus/ {  
        proxy_pass http://localhost:8081/;
        proxy_set_header Host                 $http_host; 
        proxy_set_header X-Real-IP            $remote_addr; 
        proxy_set_header X-Forwarded-For      $proxy_add_x_forwarded_for; 
        proxy_set_header X-Forwarded-Proto    $scheme;
        proxy_set_header X-Forwarded-Host     $host;
        proxy_set_header X-Nginx-Proxy        true;
        proxy_http_version 1.1; 
        proxy_set_header Upgrade $http_upgrade; 
        proxy_set_header Connection "upgrade";
    } 

```
Verify syntax
```
sudo nginx -t
```
Restart Nginx service
```
sudo systemctl restart nginx
```
Open Jenkins at https://dev-sandbox/nexus
Display initial admin password
```
sudo cat /opt/sonatype-work/nexus3/admin.password
```
Change password to admin@5354  and disable anonymous access


## Install MicroK8s


Install latest stable release
```
sudo snap install microk8s --classic --channel=latest/stable
```
add your current user to the group and gain access to the .kube caching directory
```
sudo usermod -a -G microk8s $USER 
sudo chown -f -R $USER ~/.kube
```
re-enter the session for the group update
```
su - $USER
```
Check status
```
microk8s status --wait-ready
```
add an alias
```
sudo nano ~/.bash_aliases
```
append the following command
```
alias k='microk8s kubectl'
alias h='microk8s helm3'
```
change kubectl default editor to nano
```
sudo nano ~/.bashrc
```
append the following command
```
export EDITOR="/usr/bin/nano"
```
reboot 
```
sudo reboot
```
Allow Kubernetes cni0, cbr0 and cali+
```
?? sudo ufw allow in on cbr0 && sudo ufw allow out on cbr0
?? sudo ufw allow in on cni0 && sudo ufw allow out on cni0
sudo ufw allow in on cali+ && sudo ufw allow out on cali+
```
Check ufw rules
```
sudo ufw status
```

```
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
Anywhere on cali+          ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
Anywhere (v6) on cali+     ALLOW       Anywhere (v6)
Anywhere                   ALLOW OUT   Anywhere on cali+
Anywhere (v6)              ALLOW OUT   Anywhere (v6) on cali+
```

Enable hostpath-storage and cert-manager
```
microk8s enable hostpath-storage cert-manager
```

## Enable ingress
```
microk8s enable ingress
```

Configure ingress default port to 5394  
Edit ingress controller YAML
```
k -n ingress edit daemonset.apps/nginx-ingress-microk8s-controller
```
replaces hostPort for http:
```
        name: nginx-ingress-microk8s
        ports:
        - containerPort: 80
          hostPort: 5354
          name: http
          protocol: TCP
```
remove containerPort: 443 for https
```
        - containerPort: 443
          hostPort: 443 
          name: https
          protocol: TCP
```
Check running ingress pod and daemonset
```
k -n ingress get all
```

```
NAME                                          READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-microk8s-controller-rxrz5   0/1     Running   0          61s
NAME                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/nginx-ingress-microk8s-controller   1         1         0       1            0           <none>
61s
```


## Enable Observability
```
microk8s enable observability
```

Edit Grafana deployment YAML
```
k -n observability edit deployment.apps/kube-prom-stack-grafana
```
Replace any value with http://localhost:3000/xxx 
```
        - name: REQ_URL
          value: http://localhost:3000/api/admin/provisioning/dashboards/reload
```
with http://localhost:3000/grafana/xxx 
```
        - name: REQ_URL
          value: http://localhost:3000/grafana/api/admin/provisioning/dashboards/reload
```
Append the following env for before image: grafana/grafana:x.y.z 
```
        - name: GF_SERVER_ROOT_URL
          value: http://localhost:3000/grafana/
        - name: GF_SERVER_SERVE_FROM_SUB_PATH
          value: 'true'
        image: grafana/grafana:9.3.8
```
Check pod is running
```
k -n observability get all | grep pod/kube-prom-stack-grafana
```

```
pod/kube-prom-stack-grafana-7c576749b9-p9vqz                 3/3     Running   0               64s
```

Create ingress for service/kube-prom-stack-grafana 
```
k create ingress kube-prom-stack-grafana -n observability --class=public --rule="/grafana/*=kube-prom-stack-grafana:80"
```
Check ingress created
```
k -n observability get ingress
```

```
NAME                      CLASS    HOSTS   ADDRESS   PORTS   AGE
kube-prom-stack-grafana   public   *                 80      14s
```
Configure Nginx reverse proxy
```
sudo nano /etc/nginx/sites-enabled/default
```
Append the following location
```
    location /grafana/ {  
        proxy_pass http://localhost:5354/grafana/;
        proxy_set_header Host                 $http_host; 
        proxy_set_header X-Real-IP            $remote_addr; 
        proxy_set_header X-Forwarded-For      $proxy_add_x_forwarded_for; 
        proxy_set_header X-Forwarded-Proto    $scheme;
        proxy_set_header X-Nginx-Proxy        true;
    }
```
Restart Nginx service
```
sudo systemctl restart nginx
```
Access Grafana from https://dev-sandbox/grafana 
Login with admin/prom-operator and change password to admin/admin@5354 
