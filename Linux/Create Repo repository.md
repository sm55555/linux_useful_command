#### 1. Repo 서버에 웹서버 nginx 를 설치

nginx 설정

```
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /data;
        index repomd.xml;
        
        location / {
                 
          }
        
       }
```

#### 2. 하위폴더를 만들고 아래 명령어로 받아준다. 여기선 /data

```
rsync -avrt --progress rsync://mirror.centos.org/ /data/rocky/
```

#### 3. client 세팅

hosts 파일 Repo 서버 주소 ex) mirro.centos.org

vi /etc/hosts
```
111.222.333.444 mirror.centos.org
111.222.333.444 dl.rockylinux.org
```

/etc/yum.repos.d에서 url 변경

```
CentOS
[base]
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
released updates 


Rocky
[baseos]
name=Rocky-$releasever - Base
mirrorlist=http://mirror.rockylinux/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=http://dl.rockylinux.org/centos/$releasever/os/$basearch/
gpgcheck=1
enable=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial
```

client 서버에서 테스트를 진행해야한다. (특히 centos, rocky) 아래 명령어 결과 후 repomd 관련 내용 출력

```
curl -v http://mirror.centos.org/centos/baseos/repodata/repomd.xml
curl -v http://dl.rockylinux.org/rocky/baseos/repodata/repomd.xml
```


### 도커 올리고하는방법

baseos, appstream, extras, epel

### Rocky linux &&& CentOS 7.9 다운 후 도커 마운트

사전에 필요한 /data 하위에 rocky 등 centos 폴더를 만들어야 한다.

```
docker run -v /data/rocky:/mnt --name rockylinux -d rockylinux/rockylinux sleep infinity
docker run -v /data/centos:/mnt --name cenots -d cetnos:7.9.2009 sleep infinity
```

#### 만약 Host 서버가 proxy 서버(Squid를 통해서) 나갈 경우 doceker를 올릴때 추가 설정이 필요하다.

```
docker run -v /data/rocky:/mnt -e HTTPS_PROXY=http:58.123.456.15:3128 --name rockylinux -d rockylinux/rockylinux sleep infinity
```

기본 rockylinux는 reposync, create repo 가 없기에 아래에 명령이 필요하다.

```
yum update
yum install yum-utils
yum install createrepo_c
```

##### Rockylinux repo 설정 (!! g 옵션 반드시 빼야함)

```
reposync -m --repoid=appstream --download-metadata --newest-only -p=/mnt/
```

Rockylinux는 크게 appstream, baseos, epel, extras 총 4개의 Repository가 있는데

epel은 Rockylinux에서 공식 지원하진 않지만, CentOS 8 과는 호환이 되어서 CentOS 8의 epel repo를 활용한다.

```
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

reposync -m --repoid=epel --download-metadata --newest-only -p=/mnt/
```



### CentOS repo 설정 (!! g 옵션 반드시 빼야함)

```
reposync -m --repoid=appstream --download-metadata --newest-only --download_path=/mnt
```



### createrepo

createrepo는 Yum이든, dnf이든 서버의 Repository Manager들의 설치, 삭제, 관리등을 reposync 후에 Metadata를 통해서 관리하는데, 이 Metadata들을  createrepo로 갱신한다. 그래서 reposync 후에 createrepo를 수행해야 한다.

```
reposync -m --repoid=appstream --download-metadata --newest-only -p=/mnt/
createrepo /mnt/appstream
```

### Error: GPG check FAILED

-g, --gpgcheck
              Remove packages that fail GPG signature checking after
              downloading.  exit status is '1' if at least one package
              was removed.

```
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial 

or

g 옵션을 제외 g를 넣으면 시간을 너무 많이 써야한다.
reposync -m --repoid=appstream --download-metadata --newest-only -p=/mnt/
```

### doceker save (docker image -> tar)
Before do it you must pull image what you want to save image

```
docker save -o nginx.tar nginx.latest
```

### docker load (tar -> docker image)

```
docker load -i nginx.tar
```

## Ubuntu repo 설정

nginx가 있다는 가정하에 현재는 조회하면 /data 하위에 바라보게 되어 있다. centos, rocky 나누어져 있기 떄문

```
sudo cp /usr/bin/apt-mirror /usr/bin/apt-mirror.original
sudo chown root:root /usr/bin/apt-mirror && sudo chmod 755 /usr/bin/apt-mirror
```

/etc/apt/mirror.list

```
############# config ##################
#
# set base_path    /var/spool/apt-mirror
#
# set mirror_path  $base_path/mirror
# set skel_path    $base_path/skel
# set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
# set defaultarch  <running host architecture>
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     20
set _tilde 0
#
############# end config ##############
 
deb http://kr.archive.ubuntu.com/ubuntu focal main restricted
deb http://kr.archive.ubuntu.com/ubuntu focal-updates main restricted
deb http://kr.archive.ubuntu.com/ubuntu focal universe
deb http://kr.archive.ubuntu.com/ubuntu focal-updates universe
deb http://kr.archive.ubuntu.com/ubuntu focal multiverse
deb http://kr.archive.ubuntu.com/ubuntu focal-updates multiverse
deb http://kr.archive.ubuntu.com/ubuntu focal-backports main restricted universe multiverse
deb http://kr.archive.ubuntu.com/ubuntu focal-security main restricted
deb http://kr.archive.ubuntu.com/ubuntu focal-security universe
deb http://kr.archive.ubuntu.com/ubuntu focal-security multiverse
```

대략 250GB 정도된다.

```
apt-mirror

심볼릭 링크 변경

ln -s /var/spool/apt-mirror/mirror/kr.archive.ubuntu.com/ubuntu/ /data
```

or sync 명령어

```
debmirror --verbose --method=http --host=kr.archive.ubuntu.com --root=ubuntu \
--dist=jammy,jammy-updates,jammy-backports,jammy-security,focal,focal-updates,focal-backports,focal-security \
--section=main,restricted,universe,multiverse \
--arch=amd64 --nosource --ignore-release-gpg /data/ubuntu
```

client 설정 /etc/hosts
```
111.222.333.444 archive.ubuntu.com

apt-get install chrony

```


