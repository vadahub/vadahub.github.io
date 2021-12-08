
CKAN 관리자
===

# 환경

1. 참조 링크: 
    https://cntechsystems.tistory.com/9#%C2%A0%20%C2%A01-1.%20.%20%2Fusr%2Flib%2Fckan%2Fdefault%2Fbin%2Factivate%20%EB%AA%85%EB%A0%B9%EC%96%B4%EB%A1%9C%20%EA%B0%80%EC%83%81%ED%99%98%EA%B2%BD%EC%97%90%20%EB%93%A4%EC%96%B4%EA%B0%91%EB%8B%88%EB%8B%A4.


# 관리자계정 생성

## 1. 가상환경에서 관리자용 계정 생성 

```
. /usr/lib/ckan/default/bin/activate

cd /usr/lib/ckan/default/src/ckan
paster sysadmin add ckanadmin email=ckanadmin@synctechno.com name=ckanadmin -c /etc/ckan/default/development.ini
```

> 이 명령어는 꼭 ckan폴더에서 실행해야만 함
>
> 관리자 이름 및 email을 각각 ckanadmin, ckanadmin@synctechno.com 으로 가정
>
> 초기 비밀번호: 관12XX 

## 2. 생성한 계정을 관리자로 지정                                       

```
paster sysadmin add ckanadmin -c /etc/ckan/default/development.ini
```

## 3. 테스트 데이터 생성
    paster create-test-data -c /etc/ckan/default/development.ini

## 4. 업로드 기능 설정 

* 데이터를 저장할 디렉토리 생성 

```
sudo mkdir -p /var/lib/ckan/default
```

* CKAN 설정에서 저장경로(storage_path) 설정 
    
```
vi /etc/ckan/default/development.ini
```

```ini
## Storage Settings

ckan.storage_path = /var/lib/ckan/default
ckan.max_resource_size = 50
#ckan.max_image_size = 2
```

* 업로드 파일을 web으로 다운로드 받을 수 있도로 권한 설정 

```
sudo chown -R www-data /var/lib/ckan/default
sudo chmod -R u+rwx /var/lib/ckan/default
```

> 데이터가 들어가면 스토리지 폴더안에 다른 폴더들이 생성이 되는데 이 폴더들의 주인도 www-data가 맞는지 확인 필요
>
> 다른 폴더의 주인이 www-data가 아닐 경우 500 internal server error 발생

> 500 internal server error를 보신다면 항상 이곳부터 의심

* DB 초기화 하고, CKAN 재실행

```
cd /usr/lib/ckan/default/src/ckan
paster db init -c /etc/ckan/default/development.ini
paster serve /etc/ckan/default/development.ini & 
```   


## 5. 업로드 기능 확장 (DataStore)

* 자동 업로드 기능이 있는 DataPusher를 설치

```
sudo apt-get install python-dev python-virtualenv build-essential libxslt1-dev libxml2-dev zlib1g-dev git libffi-dev 
```

* DataPusher 다운로드 및 설치 

```
    cd ~/ckan-src
    git clone https://github.com/ckan/datapusher
    cd datapusher

    pip install -r requirements.txt
    pip install -e .

    python datapusher/main.py deployment/datapusher_settings.py
```

>   datapusher 폴더에서 실행

> (Problem 1) WARNING: This is a development server. Do not use it in a production deployment.
>
> (Solution) export FLASK_ENV=development  

> (Problem 2) RequestsDependencyWarning: urllib3 (1.25.11) or chardet (4.0.0) doesn't match a supported version!
>
> (Solution) pip install --upgrade requests

> (Problem 3) werkzeug.contrib.fixers import ProxyFix 관련 에러
>
> (Solution) pip install --upgrade requests

> (Problem 4) 포트 번호 충돌 
>
> (Solution) ~/ckan-src/datapusher/deployment/datapusher_settings.py의 "PORT" 수정 


* DataStore용 사용자 생성(DB) 

```
sudo -u postgres createuser -S -D -R -P -l datastore_default
```

> 비밀번호: 관12XX

* 데이터베이스 생성 

```
sudo -u postgres createdb -O ckan_default datastore_default -E utf-8
```

* CKAN 플러그인 설정 편집 

```
vi /etc/ckan/default/development.ini
```

```ini
## Plugins Settings
ckan.plugins = datastore datapusher stats text_view image_view recline_view


## Datapusher settings
ckan.datapusher.url = http://datahub.oda-ocean.com:8800/
ckan.datapusher.assume_task_stale_after = 3600

## Database Settings
ckan.datastore.write_url = postgresql://ckan_default:rhksflwk12#$@localhost/datastore_default
ckan.datastore.read_url = postgresql://datastore_default:rhksflwk12#$@localhost/datastore_default
```

* datastore 설정

```
cd /usr/lib/ckan/default/src/ckan
paster db init -c /etc/ckan/default/development.ini

paster --plugin=ckan datastore set-permissions -c /etc/ckan/default/development.ini | sudo -u postgres psql --set ON_ERROR_STOP=1

paster serve /etc/ckan/default/development.ini
```
 

 

# 한글로 변경 #

CKAN 설정파일(/etc/ckan/default/development.ini)에서 해당 문자열을 찾아서 en을 ko_KR로 변경하고 저장합니다

```
## Internationalisation Settings
ckan.locale_default = ko_KR
```
