## FreeIPA
**NB:** No need to change your computer's hostname

**STEP 1:** Make ipa-data file

```         
sudo mkdir ipa-data/
```

**STEP 2:** Edit `/etc/docker/daemon.json` using “nano” to allow remapping:

```
{ 
      "userns-remap" : "user_name" 
} 
```
Restart docker(`sudo systemctl restart docker`)and then check uid using this command:
```
docker run --rm busybox cat /proc/self/uid_map 
```
You must have this display:
```
0     100000     65536
```

**STEP 3:** Change ownership to correct uid (from user remapping, e.g. `docker run --rm busybox cat /proc/self/uid_map`):

```         
sudo chown -R 100000:100000 ipa-data
```

## Gitlab

**STEP 1:** Data backup from vm

To do this, you need to connect to the vm using ssh. The user must then execute the following command:
```
sudo gitlab-rake gitlab:backup:create
```

This will create a backup file in the `/var/opt/gitlab/backups/` directory of the vm, containing all files, databases and git repositories.

**STEP 2:** Transfer backup to Gitlab docker container host

To do this, use the scp command :
```
scp -r root@VMgitlab: /var/opt/gitlab/backups/file_gitlab_backup.tar /home/user/backup/filename_choose_gitlab_backup.tar
```
 This command transfers a copy of the backup created in step 1 to the backup directory of the gitlab container host.

**STEP 3:** Restore data in the Docker container

Copy the backup file contained in the container host to a docker backup volume:
```
sudo cp /home/user/backup/file_gitlab_backup.tar ./srv/gitlab/data/backups/
```

**STEP 4:** Finalizing dockerization

Once all these prerequisites have been met, the user must log on as root to the gitlab container created:
```
docker exec -it gitlab-ee /bin/bash
```

Next, it will need to start restoring the copied data to the `srv/gitlab/data/backups` volume of the container (without **gitlab_backup.tar** extension):
```
gitlab-backup restore BACKUP= backup_file_name
```

## Hosts

Add on `/etc/hosts` of your computer:

```
192.168.1.11 ipa.mcfaden.local
192.168.1.12 gitlab.mcfaden.local
```

## Nginx

Create `nginx.conf` :

```
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name ipa.mcfaden.local;

        location / {
            proxy_pass http://192.168.1.10:80; # Proxy to app1's IP
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    server {
        listen 80;
        server_name gitlab.mcfaden.local;

        location / {
            proxy_pass http://192.168.1.12:80; # Proxy to app1's IP
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## Docker compose:
**NB:** create your nginx.conf file before launch your docker-compose.yml
```
services:
  freeipa_server:
    image: freeipa/freeipa-server:almalinux-9
    container_name: freeipa-srv
    hostname: ipa.mcfaden.local
    environment:
      PASSWORD: Frodo123
    volumes:
      - ./ipa-data:/data:Z
    command: ipa-server-install -U -r MCFADEN.LOCAL --no-ntp
    stdin_open: true
    tty: true
    expose:
      - "80"
      - "443"
      - "389"
      - "636"
      - "88"
      - "464"
    networks:
      docker_net:
        ipv4_address: 192.168.1.11

  nginx-proxy:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      - "80:80" # Exposing port 80 to the host
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # Mount the nginx config file
    networks:
      docker_net:
        ipv4_address: 192.168.1.10
        
  web:
    image: gitlab/gitlab-ee:latest
    restart: always
    hostname: 'localhost'
    container_name: gitlab-ee
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.mcfaden.local'
    ports:
      - '8080:80'
      - '8443:443'
    volumes:
      - ./srv/gitlab/config:/etc/gitlab
      - ./srv/gitlab/logs:/var/log/gitlab
      - ./srv/gitlab/data:/var/opt/gitlab
      # - ./srv/gitlab/backups:/var/opt/gitlab/backups
    networks:
      docker_net:
          ipv4_address: 192.168.1.12
  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
    container_name: gitlab-runner    
    restart: always
    depends_on:
      - web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - './gitlab-runner:/etc/gitlab-runner'
    networks:
      docker_net:
          ipv4_address: 192.168.1.13


networks:
  docker_net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.1.0/24
```

Then launch it with `docker-compose up -d`. Gitlab will take some time to create all files.

**FreeIPA is available at: ipa.mcfaden.local**
**Gitlab is available at: gitlab.mcfaden.local**

Finally, at least the admin user needs to be added in the docker container:
```
docker exec -it freeipa-srv /bin/bash
```
Then use this command to add an admin:
```
kinit admin
```
Using password `Frodo123`


