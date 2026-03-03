---
title: "NCP에서 OCI 프리티어로 이전하고 수동 배포한 기록"
categories:
  - infra
  - cloud
tags:
  - oci
  - oracle-cloud
  - ncp
  - migration
  - docker
  - docker-compose
  - ocir
  - postgresql
  - object-storage
  - nginx
  - certbot
  - rclone
---

서버 생성할 때 인스턴스의 이름은 서비스 이름과 동일하게 지정했다.

### Advanced options
- Advanced options으로 여러가지 선택지가 있다.
	- `On-demand capacity` : 기본값으로, 공유 물리서버 위 VM을 할당받는 방식이다. 회수 위험이 낮다.
	- `Preeptible capacity` : 남는 자원에 태우는 저가형/임시형으로 클라우드가 필요하면 언제든 회수 가능하다.
	- `Capacity reservation` : 내 VM의 자리를 예약해두는 기능으로 대규모/엔터프라이즈 용량 부족 방지용이다.
	- `Dedicated host` : 물리 호스트를 단독으로 쓰는 방식으로 격리/컴플라이언스 목적이다. 비용이 높은 편이다.
	- `Compute cluster` : HPC(초저지연 RDMA) 같은 특수 워크로드용이다.

	![](/images/oracle-freetier/image-0.png)
- 프리티어 서버를 찾아서 Oracle을 이용하기로 했기 때문에 **On-demand**로 선택했다.

## Image 선택
- 기존의 서버가 ubuntu 기반으로 돌아가고 있고, dockerfile 또한 그에 맞게 사용 중이기 때문에 버전과 이미지 그대로 가져갔다.

![](/images/oracle-freetier/image-1.png)

## Shape 선택
아래의 주소에서 프리티어 지원 현황을 확인할 수 있다.
[https://www.oracle.com/cloud/free/](https://www.oracle.com/cloud/free/)

현재 고려해볼 수 있는 컴퓨터 인스턴스는 두 가지이다.
- `AMD compute Instance`: 기존에 쓰던 x86 아키텍쳐지만 1GB RAM으로 운영된다.
- `Arm Compute Instance`: 한도를 넘기지만 않는다면 6GB RAM을 지원하는 등 성능면에서 AMD 기반보다 좀 더 나은 성능을 보인다.

![](/images/oracle-freetier/image-1-1.png)

현재 우리 프로젝트는 x86 아키텍쳐긴 하지만 이미 NCP에서 옮기는 과정에서 데이터 이관이나 서버 재구동 등 귀찮은 일이 많이 예정되어 있어 하나 정도 더 추가해 아키텍쳐가 변경되어도 괜찮지 않을까하는 마음으로 Arm compute Instance를 고르려고 한다.

프로젝트가 정상 궤도에 오르면 앱 출시도 해보고 사용자도 받아보고 싶지만 현재 서비스가 성능이 좋지 않아 너무 낮은 성능의 인스턴스를 쓰면 아예 작동이 안 될 것 같아 후자를 골라보려 한다. 했다가 안 되면 다시 돌아가는 걸로!

![](/images/oracle-freetier/image-1-9.png)

> Arm을 골랐으나 region에 availability가 부족하다는 오류가 뜨면서 생성이 되지 않았다.
> 이틀 간 시도해보았으나 안 되어서 일단 AMD로 다시 선택했다.

## Networking
> 이전의 설정들은 전부 기본값으로 설정했다.

### VNIC

- 기존에 만들어둔 VCN이 없기 때문에 `Create new virtual cloud network`를 선택한다.
![](/images/oracle-freetier/image-1-3.png)

- subnet 또한 새로 만들고, 기본값으로 진행한다.
![](/images/oracle-freetier/image-1-4.png)

- 특별히 할당 받을 고정 사설 IP가 없기 때문에 자동 할당을 선택한다.
![](/images/oracle-freetier/image-1-5.png)

- 현재 Public subnet이 없기 때문에 public IP 할당은 나중에 한다.
- IPv6은 할당하지 않는다.
![](/images/oracle-freetier/image-1-6.png)

- 기존에 사용하던 SSH Key가 있기 때문에 `Paste public key`를 통해 기존에 사용하던 `.pub` 파일 내용을 붙여넣는다.
- 맥북 기준 `.ssh` 폴더 내에 key가 존재한다.
![](/images/oracle-freetier/image-1-7.png)

## Storage

- 기본으로 유지한다.
- `Specify a custom boot volume size and performance setting` : off
	- 200GB까지 무료이기 때문에 Boot volume을 기본값 50GB로 유지하여 한도를 넘지 않도록 한다.
- `Use in-trasit encryption`: on
	- 인스턴스 <-> 부트/블록 볼륨 사이 전송 데이터 암호화
	- 성능 차이가 크게 없고 보안 이점이 있음
- `Encrypt this volume with a key that you manage`: off
	- Oracle 기본 키 대신 직접 관리하는 KMS(Valut) 키로 암호화하고 싶을 때 사용
	- 키 회전/권한/삭제 정책 직접 관리 가능
	- 운영 복잡도가 올라감
	- [How do I manage my own encryption keys?](https://docs.cloud.oracle.com/iaas/Content/KeyManagement/Concepts/keyoverview.htm)
![](/images/oracle-freetier/image-1-8.png)
- 아래의 `Block volumes` 파트는 추가 디스크를 붙일 때 사용하므로 필요할 때 추가한다.

## Review
- 정보를 잘 입력했는지 확인하고 `Create`를 통해 인스턴스를 생성한다.
## Public IP 할당
- 방금 생성한 Instance 선택
- Networking > Attached VNICs에서 생성시 만들었던 vnic 선택
- IP administration 탭에서 이미 존재하는 Primary private IP를 선택하고 edit을 선택하여 public IP를 할당한다.

- CIDR에는 현재 VNIC가 속한 서브넷을 등록
- prefix부터는 다 생략
![](/images/oracle-freetier/image-1-10.png)

- `Reserved public IP` 선택
- `Create new Reserved IP Address` 선택
- `Public IP Name` 입력
- 나머지 기본값 유지
![](/images/oracle-freetier/image-1-11.png)

- `Update`를 누르면 아이피가 할당된다.
## Public IP로 접속
- ubuntu로 선택했기 때문에 계정 이름을 ubuntu로 사용한다.
- `ssh -i ~/.ssh/{ssh key name} ubuntu@{할당된 public IP}`로 접속하면 접속이 완료된다.
![](/images/oracle-freetier/image-1-12.png)

# 데이터 이관
1. OCI VM/PostgreSQL 설치 및 빈 DB 준비
2. OCI Object Storage 버킷 생성
3. NCP Object Storage -> OCI로 객체 **1차 전체 복사** (시간 오래 걸릴 수 있음)
4. DB 이관 **리허설** (덤프/복원 테스트)
5. 서비스 쓰기 중지(업로드/DB write stop)
6. Object Storage **변경분(delta) 2차 sync**
7. DB **최종 dump/restore**
8. 앱 설정을 OCI(DB + Object Storage)로 변경
9. 검증 후 서비스 재개

> 오브젝트 파일은 크고 오래 걸리고, DB는 계속 바뀌기 때문에 오브젝트 스토리지 먼저 카피하고 DB를 마지막에 옮기는 것이 좋다고 판단했다.

# 오브젝트 스토리지 연결
## 버킷 생성

오브젝트 스토리지는 10GiB까지, Archive 스토리지까지 사용하면 20GiB까지 사용할 수 있다고 안내된다.
> You can use **10 GiB** of Object Storage and **10 GiB** of Archive Storage for free in your home region. You are using approximately **0 bytes** of combined Object Storage and Archive Storage. If you use more than **20 GiB** and have not upgraded when your Free Trial ends, your data is deleted.

[https://cloud.oracle.com/object-storage/buckets](https://cloud.oracle.com/object-storage/buckets)
- 위의 사이트에 들어가 `Create bucket` 버튼을 통해 버킷을 생성한다.
- `Bucket name`: 버킷의 이름을 지정한다.
- `Default storage tier`: `Standard` 선택
- `Enable auto-tiering`: off 선택
	- 표준 버킷에 저장된 파일 중 "잘 안 쓰는 파일"을 더 저렴한 계층으로 자동 이동시키는 기능
	- 초기 동작 파악이 어려워질 가능성이 있고, 오래된 파일이 많이 쌓여 자주 안 읽힐 때 키는게 좋음
- `Enable objects events`: off 선택
	- 파일 덮어쓰기/삭제 시 이전 버전을 남기는 기능
	- 저장 용량이 빨리 늘어나고 운영 정책이 따로 필요하기 때문에 필요할 때 키는게 좋음
- `Uncommitted multipart uploads cleanup`: on 선택
	- 대용량 업로드 중 실패/중단되어 미완료 조각이 남는 걸 자동으로 정리하는 기능
	- 저장공간 낭비를 방지하고 무료 한도를 보호하기에 좋기 때문에 키는게 좋음
- `Encryption`: `Encrypt using Oracle managed keys` 선택
![](/images/oracle-freetier/image-1-14.png)

- Tags: 리소스에 메타데이터 라벨을 붙이는 기능인데 필요성을 느낄 때 사용
- Resource logging: 버킷 관련 로그/추적 정보를 OCI Logging으로 보내는 기능으로 디버깅에 도움이 되지만 현재 사용하기에 관리 복잡도가 늘어나 추후에 고려
![](/images/oracle-freetier/image-1-16.png)


### Access Key, Secret Key 생성

- 프로필 > `User Settings` > `Tokens and keys` > Customer secret keys
- Generate secret key에서 name을 입력한다.
- Secret Key는 다시 볼 수 없으니 이 때 잘 저장해놓는다.

### 오브젝트 스토리지 복사

새로운 오브젝트 스토리지에 대해서 알아야하는 부분
- `namespace`: bucket 상세화면에서 확인할 수 있다.
- `region-code`: region management 창에서 확인할 수 있다.
- S3 호환 endpoint: `https://<namespace>.compat.objectstorage.ap-chuncheon-1.oraclecloud.com`

#### `rclone`을 이용해 복사

`rclone` 설치
```bash
brew install rclone
```

`rclone` 설정
```bash
rclone config
```
- ncp -> OCI object storage 이관 설정
	- `n`
	- `s3`
	- `Other`
	- `1`: 직접 입력
	- ncp access key 입력
	- ncp secret key 입력
	- `kr-standard`
	- `https://kr.object.ncloudstorage.com`
	- `location_constraint`는 그냥 엔터
		- 새로운 버킷 생성 하는 것이 아니기 때문에
	- `acl`도 엔터
		- 접근권한 표시를 결정하는 부분인데, 기본값으로 설정 후 추후 필요시 변경하는 것이 안전함
	- 마지막 Edit advanced config? `n`

```
Options:
- type: s3
- provider: Other
- access_key_id: ncp_iam_********************
- secret_access_key: ncp_iam_********************
- region: kr-standard
- endpoint: https://kr.object.ncloudstorage.com
```

결과 확인 후 `y`: Yes this is OK 선택

`n`을 선택하고 oci-s3도 똑같이 진행
- storage: s3
- provider: Other
- region, location_constraint: ap-chuncheon-1
- 나머지는 아까랑 똑같이 진행하면 됨

```
Current remotes:

Name                 Type
====                 ====
ncp                  s3
oci                  s3

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
```

- `rclone lsd {name}:` 으로 연결 확인
- `rclone lsf {name:bucket}` 으로 구조 확인

```
❯ rclone lsd oci:
          -1 2026-02-24 16:21:36        -1 ittda-object-storage
❯ rclone lsd ncp:
          -1 2026-01-29 15:40:00        -1 ittda-prod-assets
          -1 2025-12-18 19:37:37        -1 web25-bucket
❯ rclone lsf ncp:ittda-prod-assets
media/
```

<br/>

**오브젝트 스토리지 내부의 오브젝트들을 옮길 때 사용하는 명령어**
- `rclone copy ncp:ittda-prod-assets/media oci:ittda-object-storage/media --dry-run -P` : `--dry-run`을 이용해 복사전 확인하기
- `rclone copy ncp:ittda-prod-assets/media oci:ittda-object-storage/media -P` 명령어를 통해 복사

<br/>

# 디비 생성
- 기존의 NCP에서는 서버/디비/오브젝트 스토리지 인스턴스를 따로 구동했고, **서버가 10GB**인 대신, **디비가 가용 10GB**였다.
- 이번에는 **서버가 기본 50GB**를 할당받았고, 오라클에서는 postgreSQL를 프리티어로 지원하지 않아 서버에 직접 설치하려 한다.

1. OCI VM 기본 계정(ubuntu/opc)으로 접속
2. migration용 리눅스 유저 생성 (필요하면 sudo 부여)
3. PostgreSQL 설치
4. PostgreSQL에서 DB migration 유저 + 앱 유저 생성
5. 빈 DB 생성
6. extension 설치
7. 리허설 restore

## 마이그레이션용 유저 생성

- `migration` 유저 생성
	- `--disabled-password`: 로컬 비밀번호 로그인 막기
	- `--gecos ""`: 사용자 정보 입력 질문을 빈값으로 넘기기
	- 비대화형으로 리눅스 계정을 생성하는 명령
	```bash
	sudo adduser --disabled-password --gecos "" migration
	```
- sudo 권한 부여
```bash
sudo usermod -aG sudo migration
```
- `ubuntu` SSH 키를 새 유저로 복사
```bash
sudo mkdir -p /home/migration/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/migration/.ssh/authorized_keys
sudo chown -R migration:migration /home/migration/.ssh
sudo chmod 700 /home/migration/.ssh
sudo chmod 600 /home/migration/.ssh/authorized_keys
```

- migration 유저로 SSH 접속하는 법

```bash
ssh -i ~/.ssh/{ssh key name} migration@{할당된 public IP}

# 또는 ubuntu에서 유저 변경
sudo su - migration
```

- (option) 설치/이관이 끝나고 `sudo` 권한 제거
```bash
sudo deluser migration sudo
```

## postgreSQL 설치 및 설정

> DB 옮기는 과정에서 팀원과의 소통문제로 서버가 전부 삭제되고 내려가 백업은 여기까지 진행하고, 설치/적용만 하게 되었다.

### 서버에 디비 설치

<br/>

```bash
sudo apt update
```

<br/>

- `ca-certificates`: HTTPS 인증서 검증용
	- 없으면 `curl https://...`할 때 인증서 검증에 실패할 수 있다.
- `curl` 웹에서 파일 다운로드
- `gnupg`: 저장소 서명키 처리(GPG)
- `lsb-release`: Ubuntu 코드명 확인 (22.04는 `jammy`)
- `-y` : 중간 확인 없이 자동 실행

```bash
# 기본 도구 설치
sudo apt install -y ca-certificates curl gnupg lsb-release
```

<br/>

- APT 저장소 서명키 파일을 넣어둘 폴더를 미리 생성
- `/etc/apt/keyrings`: 폴더 생성
- `-d` 디렉토리 생성
- `-m 0755` : 권한 설정(`rwxr-xr-x`)

```bash
sudo install -d -m 0755 /etc/apt/keyrings
```

<br/>

- PostgreSQL 공식 저장소 서명키 등록
	- `curl`로 PostgreSQL 공식 GPG 키 다운로드
	- `gpg --dearmor`: ASCII 키를 APT가 쓰는 바이너리 형식으로 변환
	- 결과를 `postgresql.gpg`로 저장
	- `-f`: 실패 시 에러, `-s`: 조용히, `-S`: 실패 시 메세지, `-L`: 리다이렉트 따라감

```bash
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg
```

<br/>

- PostgresSQL 공식 APT 저장소 추가
	- APT 저장소 목록 파일 생성
	- `$(lsb_release -cs)`는 Ubuntu 코드명으로 치환됨 (`jammy`, `noble`)
	- `signed-by=...`로 방금 등록한 키만 이 저장소 검증에 사용
	- `tee`를 쓰는 이유: `sudo` 권한으로 파일 쓰기
	- `>dev/null`: `tee` 출력 화면에 안 보이게 함

```bash
echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list >/dev/null
```

<br/>

- 패키지 목록 갱신
- 새로 추가한 저장소(PGDG repo) 포함해서 설치 가능한 패키지 목록 다시 읽기

```bash
sudo apt update
```

<br/>

- PostgreSQL 설치
	- 일단 기존의 개발/배포 단에서는 15버전을 이용중이었기 때문에 15버전, PostGis는 3버전을 그대로 유지하기로 했다.
- `postgresql-<PG_MAJOR>`
  - 서버 본체
- `postgresql-client-<PG_MAJOR>`
  - `psql`, `pg_dump`, `pg_restore` 등 클라이언트 도구
- `postgresql-contrib-<PG_MAJOR>`
  - 자주 쓰는 확장/추가 유틸
- `postgresql-<PG_MAJOR>-postgis-<POSTGIS_MAJOR>`
  - PostgreSQL 15용 PostGIS 3.x 확장 패키지 설치
  - 설치 후 대상 DB에서 `CREATE EXTENSION postgis;`

```bash
sudo apt install -y postgresql-15 postgresql-client-15 postgresql-contrib-15 postgresql-15-postgis-3
```

<br/>

- 설치하다가 커널 업그레이드가 적용 대기 중이며 6.8.0-1044-oracle 커널이 새롭게 적용되어야 하니 직접 재부팅하라는 문구가 등장함
![](/images/oracle-freetier/image-1-17.png)
- 위의 창에서 OK를 선택하고 이 화면에서도 기본값 그대로 유지하고 Tab한 뒤 OK
- 다른 창에서 `uname -r` 을 하면 지금은 **6.8.0-1041-oracle**임을 확인할 수 있음
![](/images/oracle-freetier/image-1-18.png)

<br/>

- 설치가 끝나면 위의 안내대로 `sudo reboot` 한 번 해주기
- 아래 명령어들을 통해 정상 설치되었는지 확인하기

```bash
sudo reboot
uname -r # 6.8.0-1044-oracle
sudo systemctl status postgresql
pg_lsclusters
```

<br/>

- 설치 확인
	- Ubuntu에서 설치하면 보통 기본 클러스터가 자동 생성된다.
	- `pg_lsclusters`: 확인 명령
- 설치 때 자동으로 만들어진 `postgres` 리눅스 유저로 SQL 명령을 한 줄로 실행하는 모양이다.

```bash
sudo -u postgres psql -c "SELECT version();"
```

<br/>

- 서비스 자동 시작 + 즉시 시작 (위에서 active 상태가 아닐 때)
- `enable`: 부팅 시 자동 시작 등록
- `--now`: 지금 즉시 서비스 시작도 함께 수행

```bash
sudo systemctl enable --now postgresql
```

<br/>

- PostGIS/pgcrypto 확인

```bash
sudo -u postgres psql -c "SELECT name, default_version FROM pg_available_extensions WHERE name IN ('postgis','pgcrypto');"
```

<br/>

### 디비 설정

- `psql` 접속

```
sudo -u postgres psql
# 종료는 \q
```

<br/>

- DB 유저 / 빈 DB 생성
	- `openssl rand -hex 24`로 강력한 비밀번호 생성
	- 비밀번호 관리자나 `.env`에 저장

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD '<STRONG_PASSWORD>';
CREATE ROLE db_migration WITH LOGIN PASSWORD '<STRONG_PASSWORD>';

CREATE DATABASE ittda OWNER db_migration ENCODING 'UTF8' TEMPLATE template0;
```

<br/>

- 필요한 확장 추가

```bash
sudo -u postgres psql -d ittda -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
sudo -u postgres psql -d ittda -c "CREATE EXTENSION IF NOT EXISTS postgis;"
```

<br/>

- 필요하면 권한 추가

```sql
-- 이관 리허설/복원 편의용
ALTER ROLE db_migration WITH SUPERUSER;

-- 회수
ALTER ROLE db_migration WITH NOSUPERUSER;
```

<br/>

- 확인

```sql
\du -- DB 유저 확인
\l ittda -- 데이터베이스 목록 중 ittda 확인
sudo -u postgres psql -d ittda -c "\dx"
sudo -u postgres psql -d ittda -c "\dt"
```

<br/>

# 수동 배포

- 도커 설치
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
```
- 설치 확인
```bash
sudo systemctl status docker
docker --version
docker compose version
```

- 배포를 담당하는 리눅스 유저 생성
```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
sudo mkdir -p /home/deploy/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

<br/>

### OCIR 레포 생성
[https://cloud.oracle.com/compute/registry/containers?region=ap-chuncheon-1](https://cloud.oracle.com/compute/registry/containers?region=ap-chuncheon-1)
- 도커 이미지 저장을 위한 레지스트리 레포를 생성
- 백엔드/프론트엔드 레포 두 개를 만든다.
![](/images/oracle-freetier/image-1-19.png)

- 아래 구조대로 환경변수 작성
```
/home/deploy/ittda/
├── docker-compose.yml
├── .env
└── backend/
    └── .env.production
└── frontend/
    └── .env
```

- `.env` 내부 수정 목록
```
REGISTRY_URL=yny.ocir.io/<repo namespace>
DATABASE_URL="postgresql://app_user:<비밀번호>@host.docker.internal:5432/ittda"
S3_ENDPOINT="<namespace>.compat.objectstorage.ap-chuncheon-1.oraclecloud.com"
S3_REGION="ap-chuncheon-1"
S3_ACCESS_KEY= <accesskey>
S3_SECRET_KEY= <secretkey>
S3_BUCKET= <bucket name>
```

---

<br/>

### nginx
- OCI 인바운드 80, 443 Port도 열기

![](/images/oracle-freetier/image-1-20.png)

- 방화벽 열기 (선택 A)

```bash
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
<br/>


- 커널 방화벽 관리, `iptables` 직접 관리 (**선택 B**)
- `-j`: Jump target. 어떤 동작을 할지 옵션을 정함
- `-I`: INPUT 체인 맨 앞에 삽입, `-A`: 맨 뒤에 추가

```bash
# 임시 추가이므로 재부팅 이후에도 유지하기 위해 설정을 해줘야한다.
sudo iptables -I INPUT 1 -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -I INPUT 1 -p tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT
```

<br/>

- 방화벽 영구 유지

```bash
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

<br/>

- 기존 서버와 동일하게 nginx를 적용한 뒤 수동 배포

```bash
# nginx 설치 및 proxy snippet 배치
sudo apt update && sudo apt install -y nginx
sudo mkdir -p /etc/nginx/snippets /etc/nginx/sites-available /etc/nginx/sites-enabled
sudo cp /home/deploy/ittda/deploy/nginx/ittda-be.proxy.conf /etc/nginx/snippets/ittda-be-proxy.conf

# 임시 HTTP 적용 및 certbot 실행
sudo tee /etc/nginx/sites-available/ittda-be >/dev/null <<'EOF'
server {
    listen 80;
    server_name ittda-be.o-r.kr;
    include /etc/nginx/snippets/ittda-be-proxy.conf;
}
EOF

sudo ln -sf /etc/nginx/sites-available/ittda-be /etc/nginx/sites-enabled/ittda-be
sudo nginx -t && sudo systemctl reload nginx
```

<br/>

- certbot 인증
	- [[certbot 인증 오류]]

```bash
sudo apt install -y certbot python3-certbot-nginx

# sudo certbot certonly --nginx -d ittda-be.o-r.kr --dry-run로 미리 확인
sudo certbot --nginx -d ittda-be.o-r.kr

# 방화벽 이슈가 있어 nginx를 직접 관리하고 certbot이 인증서만 관리하도록 함
# 아래를 선택함
sudo certbot certonly --webroot -w /var/www/certbot -d ittda-be.o-r.kr
```

<br/>

- 인증서/엔진 상태 최종 확인

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo certbot renew --dry-run
```

<br/>

- 백업

```bash
TS=$(date +%F_%H%M%S)
sudo mkdir -p /etc/nginx/backup/$TS
sudo cp -a /etc/nginx/sites-available/ittda-be /etc/nginx/backup/$TS/ 2>/dev/null || true
sudo cp -a /etc/nginx/nginx.conf /etc/nginx/backup/$TS/
sudo ls -la /etc/nginx/backup/$TS

sudo mkdir -p /etc/nginx/snippets
sudo cp /home/deploy/ittda/deploy/nginx/ittda-be.proxy.conf /etc/nginx/snippets/ittda-be-proxy.conf

grep -n "include /etc/nginx/snippets/ittda-be-proxy.conf;" /etc/nginx/sites-available/ittda-be || \
echo "include /etc/nginx/snippets/ittda-be-proxy.conf;" | sudo tee -a /etc/nginx/sites-available/ittda-be

sudo nginx -t && sudo systemctl reload nginx
```
<br/>

- nginx 최종 검증
- 아직 백엔드를 띄우지 않았으니 502가 뜨는 것이 정상이다.

```bash
# 서버 내부
curl -I --resolve ittda-be.o-r.kr:443:127.0.0.1 https://ittda-be.o-r.kr
# 외부
curl -I https://ittda-be.o-r.kr
```

<br/>

### 레지스트리 수동 push&pull
<br/>
[OCI Auth Token 발급 페이지](https://cloud.oracle.com/identity/domains/my-profile/auth-tokens?region=ap-chuncheon-1)

1. `auth token` 발급
   - OCI 비밀번호 대신 도커 로그인에 사용하는 전용 토큰이다.
   - 토큰은 **생성 시 1회만 표시**되므로 바로 안전한 곳에 저장해야 한다.
   - Description은 식별용 메모이므로 자유롭게 입력한다. (예: `ocir-deploy-macbook`)

   ![](/images/oracle-freetier/image-1-21.png)

2. 도커 로그인 정보 확인
   - `user settings`에서 로그인에 필요한 값을 확인한다.
   - `tenancy-namespace`: Object Storage namespace와 동일
   - `oci-username`: OCI 계정명
   - 로그인 Username 형식: `<tenancy-namespace>/<oci-username>`

   ```bash
   docker login yny.ocir.io
   # Username: <tenancy-namespace>/<oci-username>
   # Password: (OCI Auth Token)

   # 비대화형 로그인 방식 (스크립트/CI에서 권장)
   # Login Succeed 되면 통과
   echo '<OCI Auth Token>' | docker login yny.ocir.io \
     -u '<tenancy-namespace>/<oci-username>' \
     --password-stdin
   ```

3. 이미지 빌드 및 Push
   - `TAG`를 Git SHA로 두면, 어떤 소스 버전이 배포됐는지 추적이 쉽다.
   - `--platform linux/amd64`는 서버 아키텍처와 맞추기 위한 옵션이다.
   - (서버가 ARM이면 `linux/arm64`로 변경)

   ```bash
   # 1) 로컬/CI에서 이미지 빌드+푸시 (SHA 태그)
   NS=<tenancy-namespace>
   TAG=$(git rev-parse --short=12 HEAD)
   REGISTRY_URL=yny.ocir.io/<tenancy-namespace>/ittda

   # 서비스별 레포 경로 예: ittda/backend, ittda/frontend
   docker buildx build --platform linux/amd64 -f backend/Dockerfile -t yny.ocir.io/$NS/ittda/backend:$TAG --push .

   # latest도 함께 갱신하려면
   docker buildx build --platform linux/amd64 -f backend/Dockerfile \
     -t yny.ocir.io/$NS/ittda/backend:$TAG \
     -t yny.ocir.io/$NS/ittda/backend:latest \
     --push .
   ```

4. Push 결과 확인 (아키텍처/태그 검증)
   - 실제 레지스트리에 어떤 플랫폼/태그로 올라갔는지 확인한다.

   ```bash
   # linux/amd64 확인
   REG=yny.ocir.io/<tenancy-namespace>/ittda

   docker buildx imagetools inspect yny.ocir.io/<tenancy-namespace>/ittda/backend:$TAG

   docker buildx imagetools inspect $REG/backend:$TAG
   docker buildx imagetools inspect $REG/backend:latest
   ```

5. 서버 반영 + 마이그레이션
   - 서버에서 특정 이미지 태그를 pull한 뒤, DB 마이그레이션을 먼저 실행하고 앱을 재기동한다.
   - 이 순서로 해야 스키마 변경이 필요한 배포에서 실패 확률이 낮다.
   [[DB 마이그레이션 오류 해결]]

   ```bash
   cd /home/deploy/ittda
   export REGISTRY_URL=yny.ocir.io/<tenancy-namespace>/ittda
   export BACKEND_IMAGE_TAG=************   # 방금 올린 SHA

   docker compose pull backend
   docker compose run --rm backend npm run db:mig:run:prod
   docker compose up -d backend
   docker compose ps
   docker compose logs --tail=200 backend
   ```

6. 장애 확인용 추가 로그

   ```bash
   docker compose logs -f backend
   docker exec -it backend ls -lah /app/backend/logs
   sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log
   ```
