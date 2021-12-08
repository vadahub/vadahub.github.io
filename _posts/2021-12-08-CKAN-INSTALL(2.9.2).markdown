CKAN 2.9.2 설치 (Docker 기반)
===


[주의] Docker기반의 설치는 개발간계나 스테이징 환경에 유용하며, production에서 이 설치과정을 이용하기 위해서는 추가적인 주의가 필요



# 1. Environment

이 설치 가이드는 Ubuntu 16.04 LTS에서 테스트됨. 

## Stoage

클라우드 VM을 사용할 때는 VM storage 보다는 external storage volume을 사용하는것이 가격과 백업 편의성때문에 좋다. 이번 케이스를 위해서 16GB storage VM에 100GB btrfs-formatted external storage volume을 사용. external volume를 /var/lib/docker로 symlink했음

external storage volume가 /data로 mount 했다는 가정하에 
```
sudo mkdir /data/docker-volume
ln -s /data/docker-volume /var/lib/docker
```

이렇게 하면 bulky하고 중요한 데이터들(Docker Images, CKAN Database를 위한 Docker data volumes, filestore, config, ...)



## Docker

Docker 공식 설치 안내 (https://docs.docker.com/engine/install/ubuntu/)에 따라 Docker 설치.


기존 버전의 docker (docker, docker.io, docker-engine)이 설치되어 있는 경우 uninstall 먼저 수행
```
 sudo apt-get remove docker docker-engine docker.io containerd runc
```

여기에서는 repository를 이용한 설치를 참고함 

### repository 설정 

apt 패키지 인덱스 업데이트 및 필요한 패키지 설치 

```
 sudo apt-get update
 sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

Docker의 공식 GPG key 추가:

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

stable repository 설정 

```
 echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Docker Engint 설치 

```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

hello-world 이미지 실행을 통한 설치 검증 
```
sudo docker run hello-world
```

현재 사용자를 docker user로 추가
```
sudo usermod -aG docker $USER
```

## Docker Compose

Docker Compose 공식 설치 안내(https://docs.docker.com/compose/install/)에 따라 Docker Compose 설치.

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose version
```

## CKAN 소스코드 다울로드 

~/c
```
cd ~
git clone https://github.com/ckan/ckan.git
git checkout tags/ckan-2.9.2
```

# 2. Build Docker images

이 단계에서는 Docker image들을 build하고 사용자 설정(database 암호와 같은)을 포함하는 Docker data volume들을 생성한다. 

## 사용자 설정과 환경변수 

환경변수 설정을 위해 템플릿파일을 복사하여 환경변수 파일 생성 
```
cp contrib/docker/.env.template contrib/docker/.env
vi contrib/docker/.env
```

수정상항
```
# Image: ckan
CKAN_SITE_ID=default
CKAN_SITE_URL=https://ec2-13-125-255-146.ap-northeast-2.compute.amazonaws.com
CKAN_PORT=5000
# Email settings
CKAN_SMTP_SERVER=smtp.gmail.com
CKAN_SMTP_STARTTLS=True
CKAN_SMTP_USER=
CKAN_SMTP_PASSWORD=전문가!
CKAN_SMTP_MAIL_FROM=oda-ocean@synctechno.com
# Image: db
POSTGRES_PASSWORD=ckan
POSTGRES_PORT=5432
DATASTORE_READONLY_PASSWORD=datastore
```



## 이미지 build 
CKAN 디렉토리에서 실행
```
cd contrib/docker
docker-compose up -d --build
```

이후 남은 단계들에서의 docker-compose 명령 실행은 docker-compose.yml 파일과 .env 파일이 위치한 docntrib/docker 폴더 안에서 실행하는것을 가정한다. 

처음 실행했을 때, postgres container는 긴 시간의 초기화를 필요로 한다.  그래서 ckan 컨테이너를 다시 시작할 필요가 있다. 

```
docker-compose restart ckan
docker ps | grep ckan
docker-compose logs -f ckan
```

이제 CKAN_SITE_URL로 접속할 수 있다. 

실행 확인하는 방법은 다음과 같다. 

docker ps 명령으로 아래의 다섯개 container가 동작하는지 확인 

```
docker ps
```

- ckan: CKAN with standard extensions
- db: CKAN’s database, later also running CKAN’s datastore database
- redis: A pre-built Redis image.
- solr: A pre-built SolR image set up for CKAN.
- datapusher: A pre-built CKAN Datapusher image.

Docker volume 확인 
```
docker volume ls | grep docker
```

- docker_ckan_config: home of production.ini
- docker_ckan_home: home of ckan venv and source, later also additional CKAN extensions
- docker_ckan_storage: home of CKAN’s filestore (resource files)
- docker_pg_data: home of the database files for CKAN’s default and datastore databases

기본적으로 docker volume의 위치는 /var/lib/docker/volumes/ 이다.


# 3. Datastore and datapusher
The datastore database and user are created when the db container is first started, however we need to do some additional configuration before enabling the datastore and datapusher settings in the production.ini.

Configure datastore database
With running CKAN containers, execute the built-in setup script against the db container:


~~docker exec ckan /usr/local/bin/ckan-paster --plugin=ckan datastore set-permissions -c /etc/ckan/production.ini | docker exec -i db psql -U ckan~~

docker exec ckan /usr/local/bin/ckan -c /etc/ckan/production.ini datastore set-permissions | docker exec -i db psql -U ckan






The script pipes in the output of paster ckan set-permissions - however, as this output can change in future versions of CKAN, we set the permissions directly. The effect of this script is persisted in the named volume docker_pg_data.

Note

We re-use the already privileged default user of the CKAN database as read/write user for the datastore. The database user (ckan) is hard-coded, the password is supplied through the``.env`` variable POSTGRES_PASSWORD. A new user datastore_ro is created (and also hard-coded) as readonly user with password DATASTORE_READONLY_USER. Hard-coding the database table and usernames allows to prepare the set-permissions SQL script, while not exposing sensitive information to the world outside the Docker host environment.

After this step, the datastore database is ready to be enabled in the production.ini.

Enable datastore and datapusher in production.ini
Edit the production.ini (note: requires sudo):

sudo vim $VOL_CKAN_CONFIG/production.ini
Add datastore datapusher to ckan.plugins and enable the datapusher option ckan.datapusher.formats.

The remaining settings required for datastore and datapusher are already taken care of:

ckan.storage_path (/var/lib/ckan) is hard-coded in ckan-entrypoint.sh, docker-compose.yml and CKAN’s Dockerfile. This path is hard-coded as it remains internal to the containers, and changing it would have no effect on the host system.
ckan.datastore.write_url = postgresql://ckan:POSTGRES_PASSWORD@db/datastore and ckan.datastore.read_url = postgresql://datastore:DATASTORE_READONLY_PASSWORD@db/datastore are provided by docker-compose.yml.
Restart the ckan container to apply changes to the production.ini:

docker-compose restart ckan
Now the datastore API should return content when visiting:

CKAN_SITE_URL/api/3/action/datastore_search?resource_id=_table_metadata
4. Create CKAN admin user
With all images up and running, create the CKAN admin user (johndoe in this example):

docker exec -it ckan /usr/local/bin/ckan-paster --plugin=ckan sysadmin -c /etc/ckan/production.ini add johndoe
Now you should be able to login to the new, empty CKAN. The admin user’s API key will be instrumental in tranferring data from other instances.

5. Migrate data
This section illustrates the data migration from an existing CKAN instance SOURCE_CKAN into our new Docker Compose CKAN instance assuming direct (ssh) access to SOURCE_CKAN.

Transfer resource files
Assuming the CKAN storage directory on SOURCE_CKAN is located at /path/to/files (containing resource files and uploaded images in resources and storage), we’ll simply rsync SOURCE_CKAN’s storage directory into the named volume docker_ckan_storage:

sudo rsync -Pavvr USER@SOURCE_CKAN:/path/to/files/ $VOL_CKAN_STORAGE
Transfer users
Users could be exported using the python package ckanapi, but their password hashes will be excluded. To transfer users preserving their passwords, we need to dump and restore the user table.

On source CKAN host with access to source db ckan_default, export the user table:

pg_dump -h CKAN_DBHOST -P CKAN_DBPORT -U CKAN_DBUSER -a -O -t user -f user.sql ckan_default
On the target host, make user.sql accessible to the source CKAN container. Transfer user.sql into the named volume docker_ckan_home and chown it to the docker user:

rsync -Pavvr user@ckan-source-host:/path/to/user.sql $VOL_CKAN_HOME/venv/src

# $VOL_CKAN_HOME is owned by the user "ckan" (UID 900) as created in the CKAN Dockerfile
sudo ls -l $VOL_CKAN_HOME
# drwxr-xr-x 1 900 900 62 Jul 17 16:13 venv

# Chown user.sql to the owner of $CKAN_HOME (ckan, UID 900)
sudo chown 900:900 $VOL_CKAN_HOME/venv/src/user.sql
Now the file user.sql is accessible from within the ckan container:

docker exec -it ckan /bin/bash -c "export TERM=xterm; exec bash"

ckan@eca111c06788:/$ psql -U ckan -h db -f $CKAN_VENV/src/user.sql
Export and upload groups, orgs, datasets
Using the python package ckanapi we will dump orgs, groups and datasets from the source CKAN instance, then use ckanapi to load the exported data into the target instance. The datapusher will automatically ingest CSV resources into the datastore.

Rebuild search index
Trigger a Solr index rebuild:

docker exec -it ckan /usr/local/bin/ckan-paster --plugin=ckan search-index rebuild -c /etc/ckan/production.ini
6. Add extensions
There are two scenarios to add extensions:

Maintainers of production instances need extensions to be part of the ckan image and an easy way to enable them in the production.ini. Automating the installation of existing extensions (without needing to change their source) requires customizing CKAN’s Dockerfile and scripted post-processing of the production.ini.
Developers need to read, modify and use version control on the extensions’ source. This adds additional steps to the maintainers’ workflow.
For maintainers, the process is in summary:

Run a bash shell inside the running ckan container, download and install extension. Alternatively, add a pip install step for the extension into a custom CKAN Dockerfile.
Restart ckan service, read logs.
Download and install extension from inside ckan container into docker_ckan_home volume
The process is very similar to installing extensions in a source install. The only difference is that the installation steps will happen inside the running container, and they will use the virtualenv created inside the ckan image by CKAN’s Dockerfile.

The downloaded and installed files will be persisted in the named volume docker_ckan_home.

In this example we’ll enter the running ckan container to install ckanext-geoview from source, ckanext-showcase from GitHub, and ckanext-envvars from PyPi:

# Enter the running ckan container:
docker exec -it ckan /bin/bash -c "export TERM=xterm; exec bash"

# Inside the running container, activate the virtualenv
source $CKAN_VENV/bin/activate && cd $CKAN_VENV/src/

# Option 1: From source
git clone https://github.com/ckan/ckanext-geoview.git
cd ckanext-geoview
pip install -r pip-requirements.txt
python setup.py install
python setup.py develop
cd ..

# Option 2: Pip install from GitHub
pip install -e "git+https://github.com/ckan/ckanext-showcase.git#egg=ckanext-showcase"

# Option 3: Pip install from PyPi
pip install ckanext-envvars

# exit the ckan container:
exit
Some extensions require database upgrades, often through paster scripts. E.g., ckanext-spatial:

# Enter the running ckan container:
docker exec -it ckan /bin/bash -c "export TERM=xterm; exec bash"

# Inside the running ckan container
source $CKAN_VENV/bin/activate && cd $CKAN_VENV/src/
git clone https://github.com/ckan/ckanext-spatial.git
cd ckanext-spatial
pip install -r pip-requirements.txt
python setup.py install && python setup.py develop
exit

# On the host
docker exec -it db psql -U ckan -f 20_postgis_permissions.sql
docker exec -it ckan /usr/local/bin/ckan-paster --plugin=ckanext-spatial spatial initdb -c /etc/ckan/production.ini

sudo vim $VOL_CKAN_CONFIG/production.ini

# Inside production.ini, add to [plugins]:
spatial_metadata spatial_query

ckanext.spatial.search_backend = solr
Modify CKAN config
Follow the respective extension’s instructions to set CKAN config variables:

sudo vim $VOL_CKAN_CONFIG/production.ini
Todo

Demonstrate how to set production.ini settings from environment variables using ckanext-envvars.

Reload and debug
docker-compose restart ckan
docker-compose logs ckan
Develop extensions: modify source, install, use version control
While maintainers will prefer to use stable versions of existing extensions, developers of extensions will need access to the extensions’ source, and be able to use version control.

The use of Docker and the inherent encapsulation of files and permissions makes the development of extensions harder than a CKAN source install.

Firstly, the absence of private SSH keys inside Docker containers will make interacting with GitHub a lot harder. On the other hand, two-factor authentication on GitHub breaks BasicAuth (HTTPS, username and password) and requires a “personal access token” in place of the password.

To use version control from inside the Docker container:

Clone the HTTPS version of the GitHub repo.
On GitHub, create a personal access token with “full control of private repositories”.
Copy the token code and use as password when running git push.
Secondly, the persisted extension source at VOL_CKAN_HOME is owned by the CKAN container’s docker user (UID 900) and therefore not writeable to the developer’s host user account by default. There are various workarounds. The extension source can be accessed from both outside and inside the container.

Option 1: Accessing the source from inside the container:

docker exec -it ckan /bin/bash -c "export TERM=xterm; exec bash"
source $CKAN_VENV/bin/activate && cd $CKAN_VENV/src/
# ... work on extensions, use version control ...
# in extension folder:
python setup.py install
exit
# ... edit extension settings in production.ini and restart ckan container
sudo vim $VOL_CKAN_CONFIG/production.ini
docker-compose restart ckan
Option 2: Accessing the source from outside the container using sudo:

sudo vim $VOL_CKAN_CONFIG/production.ini
sudo vim $VOL_CKAN_HOME/venv/src/ckanext-datawagovautheme/ckanext/datawagovautheme/templates/package/search.html
Option 3: The Ubuntu package bindfs makes the write-protected volumes accessible to a system user:

sudo apt-get install bindfs
mkdir ~/VOL_CKAN_HOME
sudo chown -R `whoami`:docker $VOL_CKAN_HOME
sudo bindfs --map=900/`whoami` $VOL_CKAN_HOME ~/VOL_CKAN_HOME

cd ~/VOL_CKAN_HOME/venv/src

# Do this with your own extension fork
# Assumption: the host user running git clone (you) has write access to the repository
git clone https://github.com/parksandwildlife/ckanext-datawagovautheme.git

# ... change files, use version control...
Changes in HTML templates and CSS will be visible right away. For changes in code, we’ll need to unmount the directory, change ownership back to the ckan user, and follow the previous steps to python setup.py install and pip install -r requirements.txt from within the running container, modify the production.ini and restart the container:

sudo umount ~/VOL_CKAN_HOME
sudo chown -R 900:900 $VOL_CKAN_HOME
# Follow steps a-c
Note

Mounting host folders as volumes instead of using named volumes may result in a simpler development workflow. However, named volumes are Docker’s canonical way to persist data. The steps shown above are only some of several possible approaches.

7. Environment variables
Sensitive settings can be managed in (at least) two ways, either as environment variables, or as Docker secrets. This section illustrates the use of environment variables provided by the Docker Compose .env file.

This section is targeted at CKAN maintainers seeking a deeper understanding of variables, and at CKAN developers seeking to factor out settings as new .env variables.

Variable substitution propagates as follows:

.env.template holds the defaults and the usage instructions for variables.
The maintainer copies .env from .env.template and modifies it following the instructions.
Docker Compose interpolates variables in docker-compose.yml from .env.
Docker Compose can pass on these variables to the containers as build time variables (when building the images) and / or as run time variables (when running the containers).
ckan-entrypoint.sh has access to all run time variables of the ckan service.
ckan-entrypoint.sh injects environment variables (e.g. CKAN_SQLALCHEMY_URL) into the running ckan container, overriding the CKAN config variables from production.ini.
See Configuration Options for a list of environment variables (e.g. CKAN_SQLALCHEMY_URL) which CKAN will accept to override production.ini.

After adding new or changing existing .env variables, locally built images and volumes may need to be dropped and rebuilt. Otherwise, docker will re-use cached images with old or missing variables:

docker-compose down
docker-compose up -d --build

# if that didn't work, try:
docker rmi $(docker images -q -f dangling=true)
docker-compose up -d --build

# if that didn't work, try:
docker rmi $(docker images -q -f dangling=true)
docker volume prune
docker-compose up -d --build
Warning

Removing named volumes will destroy data. docker volume prune will delete any volumes not attached to a running(!) container. Backup all data before doing this in a production setting.

8. Steps towards production
As mentioned above, some design decisions may not be suitable for a production setup.

A possible path towards a production-ready environment is:

Use the above setup to build docker images.
Add and configure extensions.
Make sure that no sensitive settings are hard-coded inside the images.
Push the images to a docker repository.
Create a separate “production” docker-compose.yml which uses the custom built images.
Run the “production” docker-compose.yml on the production server with appropriate settings.
Transfer production data into the new server as described above using volume orchestration tools or transferring files directly.
Bonus: contribute a write-up of working production setups to the CKAN documentation.