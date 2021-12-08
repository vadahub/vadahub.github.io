CKAN 설치 (source파일로부터)
===

# 환경

1. Ubuntu 20.04 (Python 3.6)
2. 참조 링크: 
    https://docs.ckan.org/en/2.9/maintaining/installing/install-from-source.html

    https://cntechsystems.tistory.com/25?category=749270

3. 전체 과정: Solr 설치 및 구성 > CKAN 설치 및 구성 > Postgre 설치 및 구성 
3. Python 3.6 이상 필요 


# 1. 필수 package 설치 


```
sudo apt-get update 
sudo apt-get install python3-dev postgresql libpq-dev python3-pip python3-venv git-core solr-jetty openjdk-8-jdk redis-server
```


| Package  | Description  |
|---|---|
|Python|	The Python programming language, v3.6 이상|
|PostgreSQL|	The PostgreSQL database system, v9.5 이상|
|libpq|	The C programmer’s interface to PostgreSQL|
|pip|	A tool for installing and managing Python packages|
|python3-venv|	The Python3 virtual environment builder |
|Git|	A distributed version control system|
|Apache Solr|	A search platform|
|Jetty|	An HTTP server (used for Solr).|
|OpenJDK JDK|	The Java Development Kit (used by Jetty)|
|Redis|	An in-memory data structure store|


# 3. Python virtual environment에 CKAN 설치 

> TIP
> 
> 개발용으로 개발자 계정의 home directory에 설치를 하고자 할 때는 simbolic link를 아래와 같이 설정하는게 좋다.
>

```
mkdir -p ~/ckan/lib
sudo ln -s ~/ckan/lib /usr/lib/ckan
mkdir -p ~/ckan/etc
sudo ln -s ~/ckan/etc /etc/ckan
```

## a. Python 가상 환경(virtual environment) 생성하고 가상환경 활성화

```
sudo mkdir -p /usr/lib/ckan/default
sudo chown `whoami` /usr/lib/ckan/default
python3 -m venv /usr/lib/ckan/default
. /usr/lib/ckan/default/bin/activate
```



<<<<<<  old >>>>>>

# Solr 설치 및 구성

## 1. apt-get update

    sudo apt-get update
 
## 2. 관련 모듈 설치 (CKAN 공식 설칯 가이드에서 solr-jetty 제외)
 
    sudo apt-get install python-dev postgresql libpq-dev python-pip python-virtualenv git-core openjdk-8-jdk redis-server

    sudo apt-get install python3-dev postgresql libpq-dev python3-pip python3-venv git-core solr-jetty openjdk-8-jdk redis-server




## 3. JAVA 설치 확인 

    sudo update-alternatives --set java /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
    
## 4. ckan 디렉토리 생성  

    mkdir ckan-src; cd ckan-src

## 5. source code download (solr 6.6 버전 부터는 ckan과 호환 문제가 있으므로 6.6미만 버전을 설치) 

    wget http://archive.apache.org/dist/lucene/solr/6.5.1/solr-6.5.1.tgz
    
    wget http://archive.apache.org/dist/lucene/solr/8.8.2/solr-8.8.2.tgz
    

## 6. 설치 스크립트 압축 풀기 

    tar xzf solr-6.5.1.tgz solr-6.5.1/bin/install_solr_service.sh --strip-components=2

    tar xzf solr-8.8.2.tgz solr-8.8.2/bin/install_solr_service.sh --strip-components=2

## 7. solr 설치 

    sudo bash ./install_solr_service.sh solr-6.5.1.tgz

    sudo bash ./install_solr_service.sh solr-8.8.2.tgz

## 8. solr 사용자로 접속 (whoami로 확인) 

    sudo -u solr bash

## 9. CKAN 관련 solr 설치 (오류 날 시 처음부터 다시진행)

    cd /opt/solr/bin
    ./solr create -c ckan

## 10. solr 설정 파일 수정

    cd /var/solr/data/ckan/conf
    sed -i '/<config>/a <schemaFactory class="ClassicIndexSchemaFactory"/>' solrconfig.xml
    sed -i '/<initParams path="\/update\/\*\*">/,/<\/initParams>/ s/.*/<!--&-->/' solrconfig.xml
    sed -i '/<processor class="solr.AddSchemaFieldsUpdateProcessorFactory">/,/<\/processor>/ s/.*/<!--&-->/' solrconfig.xml 

## 11. ckan용 스키마 가져오기위해 기존 스키마 삭제 

    rm managed-schema
 
## 12. solr 사용자 exit

    exit



# CKAN 설치 

## 1. CKAN 관련 디렉토리 생성 및 링크 생성

    mkdir -p ~/ckan/lib
    sudo ln -s ~/ckan/lib /usr/lib/ckan
    mkdir -p ~/ckan/etc
    sudo ln -s ~/ckan/etc /etc/ckan

> python --version 으로 python version 확인하여 2.7가 아닌 경우 [python의 기본을 2.7로 설정](##-우분투에서-파이썬-버전-변경하기)


## 2. root 계정으로 가상 환경에 접속 

    sudo su 

    sudo mkdir -p /usr/lib/ckan/default
    sudo chown `whoami` /usr/lib/ckan/default

    virtualenv --no-site-packages /usr/lib/ckan/default
    . /usr/lib/ckan/default/bin/activate

> (default) root@ ~~~ 가 떠야 진행이 가능

> . /usr/lib/ckan/default/bin/activate 명령으로 가상환경 접속 


## 3. setuptools 설치 

    pip install setuptools==36.1

## 4. git으로부터 CKAN (2.8) 설치 

    pip install -e 'git+https://github.com/ckan/ckan.git@ckan-2.8.2#egg=ckan'
 
## 5. CKAN 모듈 설치 

    pip install -r /usr/lib/ckan/default/src/ckan/requirements.txt

## 6. paster 명령 실행을 위해 가상환경 나갔다가 다시 접속 

    deactivate
    . /usr/lib/ckan/default/bin/activate




# PostgreSQL 설치하고 Solr 설정을 마무리

## 1. PostgreSQL 설치 확인 
    
    sudo -u postgres psql -l 

>   ### Exception(1-1) 스키마들이 보이지 않는 경우 
>
>   PostgreSQL 실행 
>
>   ``` sudo service postgresql start ```
>   
>   Cluster 확인 
>
>   ``` pg_lsclusters ``` 
>
>   Online이고 초록색 글씨라면 다시 "1. PostgreSQL 설치 확인" 다시 진행
>
>   Online이 아니고 빨간색 글씨라면 pg_ctlcluster <version> <cluster> start 명령어로 실행
>
>   ``` pg_ctlcluster 10 main start ```
>
>   clusters 다시 확인 
>
>   ``` pg_lsclusters ``` 

## 2. user 생성 

    sudo -u postgres createuser -S -D -R -P ckan_default

> 비밀번호: 관12XX

## 3. DB 생성 

    sudo -u postgres createdb -O ckan_default ckan_default -E utf-8

## 4. /etc/postgresql/10/main 경로의 conf 파일(postgresql.conf 파일과 pg_hba.conf) 수정  

> postgresql 버전에 따라 경로의 숫자가 차이가 있습니다.
>
> 두 설정 다 상용화할 때는 설정을 달리 해야 보안에 안전합니다.
>
> postgresql.conf, pg_hba.conf 파일이 비어 있다면 sudo로 접속해서 편집


* postgresql.conf 의 listen_address 수정 
```
listen_addresses = '*'          # what IP address(es) to listen on;
```

* pg_hba.conf 에 host all all 0.0.0.0/0 md5 추가 
```
host    all             all             0.0.0.0/0            md5
```

## 5. /etc/ckan/default 생성 

```
sudo mkdir -p /etc/ckan/default
sudo chown -R `whoami` /etc/ckan/
sudo chown -R `whoami` ~/ckan/etc
```


# 마무리

## 1. 설정파일 생성 

    paster make-config ckan /etc/ckan/default/development.ini

## 2. 설정파일 수정 (DB 접속 URL)

    vi /etc/ckan/default/development.ini 명령어로 설정파일을 열어서 sqlalchemy.url설정을 합니다.

```
## Database Settings
sqlalchemy.url = postgresql://ckan_default:관12XX@localhost/ckan_default


## Site Settings
ckan.site_url = http://datahub.oda-ocean.com:5000
```

## 3. solr 설정 마무리

* ckan용 solr 스키마 옮기고 solr 서비스 재시작 

```
sudo -u solr bash
cd /var/solr/data/ckan/conf

cp /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml .

exit

sudo service solr restart
```

* ckan 설정파일 수정 (solr url) 

```
sudo sed -i '/#solr_url = /s/.*/solr_url = http:\/\/127.0.0.1:8983\/solr\/ckan\//g' /etc/ckan/default/development.ini
``` 

* 설정파일 복사 

``` 
ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini
```

* ckan 설정파일 적용 

```
cd /usr/lib/ckan/default/src/ckan
paster db init -c /etc/ckan/default/development.ini
```

> SUCCESS가 나와야 함

* 서버 실행 


서버 실행 전 현재 디렉토리와 사용자 권한, virtual machine 환경 확인
```
sudo su
cd /usr/lib/ckan/default/src/ckan
. /usr/lib/ckan/default/bin/activate
```

백그라운드로 서버 실행 
```
paster serve /etc/ckan/default/development.ini &
```

> http://datahub.oda-ocean.com:5000 접속해서 확인



# 참고 

# 우분투에서 파이썬 버전 변경하기

Linux의 alternative 이용하여 python/pip의 기본 버전을 변경하는 방법 

## 참조 링크:

    https://seongkyun.github.io/others/2019/05/09/ubuntu_python/

## 아래와 같은 상황을 가정


```
$ python -V
Python 2.7.14

$ which python
/usr/bin/python

$ ls -al /usr/bin/python
lrwxrwxrwx 1 root root 24  4월 18 19:28 /usr/bin/python -> /usr/bin/python2.7

$ ls /usr/bin/ | grep python
python
python2
python2.7
python3
python3.6
.....
```

## 설정 방법

--config python 옵션은 python의 버전을 변경하는 옵션이다.
$ sudo update-alternatives --config python 를 입력하면 python의 버전을 변경 할 수 있다.

```
$ sudo update-alternatives --config python
update-alternatives: error: no alternatives for python
```

만약 위와같이 error로 python에 대한 alternative가 설정된것이 없다 뜰 경우, --install [symbolic link path] python [real path] number 명령어로 2.7이나 3.6같은 파이썬 버전을 등록해주면 된다.

```
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2.7 1
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.6 2

sudo update-alternatives --install /usr/bin/pip pip /usr/bin/pip2 1
sudo update-alternatives --install /usr/bin/pip pip /usr/bin/pip3 2
```

이 후 다시 sudo update-alternatives --config python를 입력하면 설치되어있는 python 버전 선택 메뉴가 등장한다.

```
$ sudo update-alternatives --config python

There are 2 choices for the alternative python (providing /usr/bin/python).

  Selection    Path                Priority   Status
------------------------------------------------------------
* 0            /usr/bin/python3.6   2         auto mode
  1            /usr/bin/python2.7   1         manual mode
  2            /usr/bin/python3.6   2         manual mode

Press <enter> to keep the current choice[*], or type selection number: 2
```

원하는 python 버전의 번호를 선택한 후 엔터를 치면 해당 버전이 default path로 설정되게 된다.

```
$ python --version
Python 3.6.3
```
