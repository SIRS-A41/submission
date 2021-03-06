# Remote document access with ransomware protection

We present a solution that allows users to create projects, commit versions, revisit older versions, and share them with other users. 

Each project has its own symmetric key which is used to encrypt every file before it is submitted to the server. Project keys are safely stored in the remote server using users' asymmetric key pairs.

The document is divided as follows:

  - [General Information](#general-information)
    - [Built With](#built-with)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
      - [Single machine](#single-machine)
      - [Multiple machines (using VMs)](#multiple-machines-using-vms)
    - [Installing](#installing)
      - [Single machine (easier)](#single-machine-easier)
      - [Multiple machines](#multiple-machines)
        - [Auth Server](#auth-server)
        - [Auth REST API](#auth-rest-api)
        - [Resources Server](#resources-server)
        - [Resources REST API](#resources-rest-api)
        - [Backup Server](#backup-server)
        - [Firewall / Reverse Proxy Server](#firewall--reverse-proxy-server)
        - [User](#user)
    - [Testing](#testing)
  - [Demo](#demo)
    - [Mongo Express](#mongo-express)
      - [User Projects](#user-projects)
      - [Project History](#project-history)
    - [Redis Commander](#redis-commander)
      - [List of Users](#list-of-users)
  - [Additional Information](#additional-information)
    - [Authors](#authors)
    - [License](#license)

## General Information

The ability to send and receive files over the Internet without requiring one of the parties to be connected at the same time greatly simplifies file sharing.

However, this comes with risks. By default, the Internet isn't secure, and without the use of a proper communication channel, this data exchange could be exposed to whoever intercepts the connection. Moreover, if files are stored in a remote server for ease of access at all times, there should be a way to keep their contents secret from unauthorized parties, whether they are users of the system or someone with physical access to the server. This should be done in a way that still allows authorized users to transparently view and edit their files. Even if an attacker can???t read the files, they can launch a ransomware attack and encrypt the files on the server. 

Here we propose a client program, called bag, which behaves similar to *git*. This program handles the interaction with the remote server. This client takes care of criptographic key generation, management, file compression and decompression, encryption and decryption, signing, everything so you only have to interact with it using simple commands.

The server side takes care of handling user authentication, providing an architecture similar to OAuth2.0 with refresh and access tokens. The data kept encrypted on the server, so only the users who should have access to it can read it and modify it. A version history is kept on the remote server, so if the users wants to revisit some files he modified a while ago he can do so.

The remote server also counts with a backup server wich will keep users' data secure in case of ransomware attacks.

### Built With

* [Dart](https://dart.dev/) - Dart is a client-optimized language for fast apps on any platform; used for the REST APIs - Auth API and Resources API
* [Docker](https://www.docker.com/) - Virtualization tool used for deploying the different applications
* [Docker Compose](https://docs.docker.com/compose/) - Docker tool used for defining and running multi-container Docker applications 
* [Java](https://openjdk.java.net/) 11 - Programming language and platform used for the client application, bag
* [JWT](https://jwt.io/) - Open, industry standard RFC 7519 method for representing claims securely between two parties
* [Maven](https://maven.apache.org/) - Build tool and dependency management
* [MongoDB](https://www.mongodb.com/) - source-available cross-platform document-oriented database program, used for storing users' public keys, projects metadata and symmetric keys.
* [Mongo Express](https://github.com/mongo-express/mongo-express) - Web-based MongoDB admin interface written with Node.js, Express and Bootstrap3
* [Nginx](https://www.nginx.com/) - advanced load balancer, web server, and reverse proxy
* [Redis](https://redis.io/) - open source, in-memory data structure store, used as a database for users' credentials and refresh tokens
* [Redis Commander](https://joeferner.github.io/redis-commander/) - node.js web application used to view, edit, and manage a Redis Database

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on a live system.

### Prerequisites

There are two options for implementing the developed solution. Self-host all the application in a single machine or divide the application through different machines.

You can start by installing [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/).

Then, clone the different repositories:

**auth**:
```
git clone https://github.com/SIRS-A41/auth-api.git ~/Downloads/auth-api
```

**backup**:
```
git clone https://github.com/SIRS-A41/backup.git ~/Downloads/backup
```

**bag**:
```
git clone https://github.com/SIRS-A41/bag.git ~/Downloads/bag
```

**certificates**:
```
git clone https://github.com/SIRS-A41/certificates.git ~/Downloads/certificates
```

**resources**:
```
git clone https://github.com/SIRS-A41/resources-api.git ~/Downloads/resources-api
```

#### Single machine

You will require a computer running Linux. Since our solution uses Docker, it might work on other platforms, but it was not tested.


#### Multiple machines (using VMs)

You'll need an architecture as the one depicted in the Figure below:

![machines diagram](https://github.com/SIRS-A41/submission/raw/main/images/SIRS-machines.png)

Since multiple machines are involved, it is required to setup firewalls for each machine.

**firewalls**:
```
git clone https://github.com/SIRS-A41/firewalls.git
```

And also install [nginx](https://www.nginx.com/) on the Firewall machine to work as a reverse proxy:

```
sudo apt install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```


### Installing

#### Single machine (easier)

1. Initialize the Redis and Redis commander services:
   
   ```bash
   cd ~/Downloads/auth-api
   docker-compose up -d
   ```

2. Check everything is working by visiting the Redis Commander instance [localhost:8082](localhost:8082).

3. Initialize the MongoDB and Mongo Express services:
   
   ```bash
   cd ~/Downloads/resources-api
   docker-compose up -d
   ```

4. Check everything is working by visiting the Mongo Express instance [localhost:8081](localhost:8081).

5. Setup SSH on your machine:

    ```bash
    sudo apt install openssh-server
    sudo systemctl enable sshd
    sudo systemctl start sshd
    ```

6. Create a user called `sirs` with password `123123` for storing the project files:

    ```bash
    sudo useradd -m -e `date -d "7 days" +"%Y-%m-%d"` sirs 
    sudo passwd sirs
    ```
    *Note: the created user account will expire in 7 days.*

7. Discover your machine IP. It is required for generating a valid certificate:
   
   ```bash
   IP=$(ip -o route get to 8.8.8.8 | sed -n 's/.*src \([0-9.]\+\).*/\1/p')
   echo $IP
   ```

8. Generate a valid certificate and import it to Java truststore:

    ```bash
    cd ~/Downloads/certificates
    ./generate_certificate.sh $IP
    sudo ./import_certificate.sh
    ```

    *Note: the certificate is valid for 30 days.*

9.  Now you must setup the Auth REST API with the correct IP:

    ```bash
    cd ~/Downloads/auth-api
    sed -i "s/192\.168\.2\.2/$IP/g" .env
    ```

10. Make a copy of the generated certificates for the Auth REST API to use:

    ```bash
    cd ~/Downloads/auth-api
    cp -r ~/Downloads/certificates .
    ```

11. You can now build the docker image and run it in a container:

    ```bash
    cd ~/Downloads/auth-api
    docker build --no-cache -t auth-api .
    docker run -p 8443:8443 -p 8445:8445 auth-api
    ```

12. You need to change your local IP for the Resources API as well:

    ```bash
    cd ~/Downloads/resources-api
    sed -i "s/192\.168\.3\.2/$IP/g" .env
    ```

13. Make a copy of the generated certificates for the Resources REST API to use:

    ```bash
    cd ~/Downloads/resources-api
    cp -r ~/Downloads/certificates .
    ```

14. You can now build the docker image and run it in a container:

    ```bash
    cd ~/Downloads/resources-api
    docker build --no-cache -t resources-api .
    docker run -p 8444:8444 resources-api
    ```

15. Edit the URI in the Java project to use your local machine IP:

    ```bash
    cd ~/Downloads/bag 
    sed -i "s/192\.168\.0\.254/$IP/g" src/main/java/com/sirsa41/AuthRequests.java
    sed -i "s/192\.168\.0\.254:8443/$IP:8444/g" src/main/java/com/sirsa41/ResourcesRequests.java
    ```

16. Package to a JAR file:
    ```bash
    mvn assembly:assembly -DdescriptorId=jar-with-dependencies
    ```

17. Setup an alias to execute the compiled JAR:
    ```bash
    alias bag="java -jar ~/Downloads/bag/target/bag-1.0-jar-with-dependencies.jar"
    ```

18. Run the bag client:
    ```bash
    bag
    ```
    Output:
    ```
    Instructions:
    bag register - Create an account
    bag login - Login to account
    bag user - Get current user
    bag logout - Logout of your account
    bag keys - Generate asymmetric key pair
    bag private - Set private key
    bag create <project-name> - Create a new project named <project-name>
    bag clone <project-name> - Clone project named <project-name>
    bag share <username> - Share project with <username>
    bag history - Show project history
    bag push - Push current project files to remote server
    bag pull - Pull current project files from remote server
    bag checkout <commit-version> - Pull <commit-version> project files from remote server
    bag projects - Show user's projects
    ```

*Note: the backup server is not setup when using a single machine, however, you can still do it following the instructions below.*

You're all done!

#### Multiple machines

Make sure you have 7 machines as depicted [above](####multiple-machines-using-vms).

The [`firewalls`](https://github.com/SIRS-A41/firewalls.git) repo contains a script for setting up the Firewall in each machine, *e.g.*, `auth-api.sh`.

##### Auth Server

1. Place the [`auth-api`](https://github.com/SIRS-A41/auth-api.git) repo at `/root/auth-api`.

2. Initialize the MongoDB and Mongo Express services:
   
   ```bash
   cd /root/auth-api
   docker-compose up -d
   ```

##### Auth REST API

1. Place the [`auth-api`](https://github.com/SIRS-A41/auth-api.git) repo at `/root/auth-api`.

2. Place the [`certificates`](https://github.com/SIRS-A41/certificates.git) repo at `/root/auth-api/certificates`.

3. Build the docker image and run it in a container:

    ```bash
    cd /root/auth-api
    docker build --no-cache -t auth-api .
    docker run -p 8443:8443 -p 8445:8445 auth-api
    ```

##### Resources Server

1. Place the [`resources-api`](https://github.com/SIRS-A41/resources-api.git) repo at `/root/resources-api`.

2. Initialize the MongoDB and Mongo Express services:
   
    ```bash
    cd /root/resources-api
    docker-compose up -d
    ```

3. Setup SSH on your machine:

    ```bash
    sudo apt install openssh-server
    sudo systemctl enable sshd
    sudo systemctl start sshd
    sudo apt install btrfs-progs -y
    ```

4. Create a user called `sirs` with password `123123` for storing the project files:

    ```bash
    sudo useradd -m -e `date -d "7 days" +"%Y-%m-%d"` sirs 
    sudo passwd sirs
    ```
    *Note: the created user account will expire in 7 days.*

5. Create a Btrfs Subvolume Subvolumes:

    ```
    sudo mkdir -v /data
    sudo btrds subvolume create /data/projects
    sudo mkdir -v /data/.snapshots
    ```

6. Create a cronjob for taking snapshots of the project files `sudo crontab -e`. Add the following line:

    ```
    20 */24 * * * /usr/bin/python3 /home/backup/snapshot-script.py
    ```

##### Resources REST API

1. Place the [`resources-api`](https://github.com/SIRS-A41/resources-api.git) repo at `/root/resources-api`.

2. Place the [`certificates`](https://github.com/SIRS-A41/certificates.git) repo at `/root/resources-api/certificates`.

3. Build the docker image and run it in a container:

    ```bash
    cd /root/resources-api
    docker build --no-cache -t resources-api .
    docker run -p 8444:8444 resources-api
    ```

##### Backup Server

1. Create user `back`.

    ```bash
    useradd -m back
    ```
    *Note: the created user account will expire in 7 days.*

2. Place the [`backup`](https://github.com/SIRS-A41/backup.git) repo at `/home/back/backup`.

3. Generate ssh key:

    ```bash
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    ```

4. Copy key to Resources server:

    ```bash
    ssh-copy-id -i ~/.ssh/id_rsa.pub sirs@192.168.4.1
    ```

5. Create a cronjob for backing up the project files on the Resources server `sudo crontab -e`. Add one of the following lines:

    **Simpler version using only scp:**
    ```
    0 * * * * /home/back/backup/backup.sh
    ```

    **Version using btrfs, zip, and scp:**
    ```
    30 */24 * * * /usr/bin/python3 /home/backup/backup-script.py
    ```


##### Firewall / Reverse Proxy Server

1. Place the [`certificates`](https://github.com/SIRS-A41/certificates.git) repo at `/root/certificates`.

2. Setup nginx at `/etc/nginx/sites-enabled/default`:

    ```
    server {
        listen 8443 ssl default_server;
        listen [::]:8443 ssl default_server;

        location /auth {
            proxy_bind 192.168.1.254;
            proxy_pass https://192.168.1.1:8443;
        }

        location /resources {
            proxy_http_version 1.1;
            proxy_bind 192.168.1.254;
            proxy_pass https://192.168.1.2:8444;
        }

        ssl_certificate /root/certificates/cert.pem;
        ssl_certificate_key /root/certificates/key.pem;

        access_log /var/log/nginx/default.access.log;
        error_log /var/log/nginx/default.error.log;
    }
    ```

3. Reload nginx:

    ```bash
    nginx -s reload
    ```

##### User

1. Place the [`certificates`](https://github.com/SIRS-A41/certificates.git) repo at `~/certificates`.

2. Import the certificate to Java truststore:

    ```bash
    cd ~/certificates
    sudo ./import_certificate.sh
    ```

1. Place the [`bag`](https://github.com/SIRS-A41/bag.git) repo at `~/bag`.

3. Setup an alias to execute the compiled JAR:
    ```bash
    alias bag="java -jar ~/bag/target/bag-1.0-jar-with-dependencies.jar"
    ```

3. Run the bag client:
    ```bash
    bag
    ```


### Testing

On your machine or on the `user` machine:

1. Create an account for `alice`:

    ```bash
    bag register
    ```

    Expected output:
    ```
    Enter your username: alice
    Enter your password: 
    Confirm your password: 
    Creating an account for user: alice...
    Account successfuly created
    ```

2. Create another account but for `bob`.

3. Login to `alice`'s account:

    ```bash
    bag login
    ```

    Expected output:
    ```
    Enter your username: alice
    Enter your password: 
    Login successful
    Generating user asymmetric keys...
    Uploading public key to remote server...
    Public key successfuly set
    SAVE your private key:
    MIIEvgIBADANBgkqhkiG9w0BAQEF(...)
    ```

4. Create a sample directory:

    ```bash
    mkdir test
    cd test
    echo "Beatriz, Afonso, Guilherme = bag" > students
    ```

5. Create a project:

    ```bash
    bag create sirsa41
    ```

    Expected output:
    ```
    Creating project: sirsa41
    Generating project symmetric key...
    Encrypting project key...
    Uploading encrypted key to server...
    Project successfuly created!
    You can now push your files to the remote server
    ```

6. Push the local files:

    ```bash
    bag push
    ```

    Expected output:
    ```
    Checking local version against remote server...
    Compressing project files...
    Encrypting project files...
    Signing commit...
    Pushing project files to remote server..
    Successful push
    Commit: 6b52f62dbf584f5b6def5b1d814d6c844cbc3ba160827beaab1b4cc3e510a729
    ```

7. Create a new file and modify the previous one:

    ```bash
    echo $(date) > date
    echo "a41" >> students
    bag push
    ```

8. Check the project versions history:

    ```bash
    bag history
    ```

    Expected output:
    ```
      Date                  User    Commit
    > 2022-01-28 19:23      alice   b5a9f034a80703e66918c38c7eddc77aff3541aef762bc282734ca388a2f515f
      2022-01-28 19:23      alice   6b52f62dbf584f5b6def5b1d814d6c844cbc3ba160827beaab1b4cc3e510a729

    ```

9. Share the project with `bob`:

    ```bash
    bag share bob
    ```

    Expected output:
    ```
    Retrieving bob public key...
    Encrypting project key for bob...
    Uploading new key to remote server...
    Project successfuly shared with bob
    ```

10. Logout of your account:

    ```bash
    bag logout
    ```

    Expected output:
    ```
    Logout successful
    ```

11. Login to `bob`'s account:

12. Check `bob`'s projects:

    ```bash
    bag projects
    ```

    Expected output:
    ```
    Date                    Name            Shared
    2022-01-28 19:26        alice/sirsa41
    ```

13. Clone the project `alice/sirsa41`:

    ```bash
    cd
    bag clone alice/sirsa41
    cd sirsa41
    ```

    Expected output:
    ```
    Cloning project: alice/sirsa41
    Retrieving project information from remote server...
    Checking local version against remote server...
    Pulling latest commit from remote server...
    Verifying file signature...
    Decrypting project files...
    Decompressing project files...
    Successful pull!
    Commit: b5a9f034a80703e66918c38c7eddc77aff3541aef762bc282734ca388a2f515f
    Project alice/sirsa41 successfuly cloned
    ```

14. Add a new file to the project and push it:

    ```bash
    echo "20" > grade
    bag push
    ```

15. Check the project history:

    ```bash
    bag history
    ```

    Expected output:
    ```
      Date                  User    Commit
    > 2022-01-28 19:28      bob     275ba06fd8e3b87f58168c5573a3fb3f3be86af1b9f6d273f2815c40826bd733
      2022-01-28 19:23      alice   b5a9f034a80703e66918c38c7eddc77aff3541aef762bc282734ca388a2f515f
      2022-01-28 19:23      alice   6b52f62dbf584f5b6def5b1d814d6c844cbc3ba160827beaab1b4cc3e510a729
    ```
    
16. Checkout an older commit:

    ```bash
    bag checkout <commit-version>
    ```

    Expected output:
    ```
    Pulling commit 6b52f62dbf584f5b6def5b1d814d6c844cbc3ba160827beaab1b4cc3e510a729 from remote server...
    Verifying file signature...
    Decrypting project files...
    Decompressing project files...
    Successful pull!
    Commit: 6b52f62dbf584f5b6def5b1d814d6c844cbc3ba160827beaab1b4cc3e510a729
    ```

## Demo

The main features of our application are described above.

You can also check the Mongo Express and Redis Commander instance to get a better understanding of how data is organized and stored:

### Mongo Express

Accessible at [localhost:8081](localhost:8081).

#### User Projects

![mongo user projects](https://github.com/SIRS-A41/submission/raw/main/images/mongo-user-projects.png)
#### Project History

![mongo project history](https://github.com/SIRS-A41/submission/raw/main/images/mongo-project-history.png)


### Redis Commander

Accessible at [localhost:8082](localhost:8082).
#### List of Users

![redis users](https://github.com/SIRS-A41/submission/raw/main/images/redis-user.png)

## Additional Information

### Authors

* 83805 - [**Afonso Raposo**](https://afonsoraposo.com)
* 89451 - Guilherme Areias

### License

This project is licensed under the GNU General Public License v3.0.
