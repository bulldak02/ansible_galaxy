# /galaxy - WordPress + MySQL + HAProxy + DNS (ansible-galaxy roles)

`ansible-galaxy role init` 으로 생성한 역할 기반 배포 구성입니다.

## 구성

| 그룹 | 호스트 | 역할 | 내용 |
|------|--------|------|------|
| db   | 10.0.0.24 | mysql | mysql-server 설치, `word` DB / `yhlee` 계정 생성, 3306 개방 |
| web  | 10.0.0.22 | httpd + wordpress | Rocky httpd + PHP, WordPress 전개 |
| web1 | 10.0.0.25 | apache + wordpress | Ubuntu apache2 + PHP, WordPress 전개 |
| was  | 10.0.0.23 | haproxy + dns | 80 포트로 web/web1 분산, `yhlee.local` 존 서비스 |

## 역할

- `roles/mysql`      : MySQL 설치/기동, DB·계정 생성, 방화벽
- `roles/wordpress`  : WordPress 압축 해제, wp-config.php DB 설정 (공통)
- `roles/httpd`      : Rocky httpd + PHP 패키지, DirectoryIndex, 방화벽
- `roles/apache`     : Ubuntu apache2 + PHP 패키지, DirectoryIndex
- `roles/haproxy`    : HAProxy 설치/설정, backend 는 inventory `http` 그룹
- `roles/dns`        : BIND 설치, `yhlee.local` 정방향 존, forwarders, 방화벽 53
- `roles/dnsclient`  : Ubuntu netplan DNS 서버 지정 (web1 → 10.0.0.23)

## DNS 레코드 (`yhlee.local`)

| 이름 | 값 | 용도 |
|------|----|------|
| `yhlee.local` | 10.0.0.23 | 대표(HAProxy) |
| `www.yhlee.local` | 10.0.0.23 | WordPress 서비스 |
| `ns.yhlee.local` | 10.0.0.23 | 네임서버 |
| `was.yhlee.local` | 10.0.0.23 | HAProxy/DNS |
| `web.yhlee.local` | 10.0.0.22 | Rocky httpd |
| `web1.yhlee.local` | 10.0.0.25 | Ubuntu apache2 |
| `db.yhlee.local` | 10.0.0.24 | MySQL |

레코드는 `roles/dns/defaults/main.yml` 에서 inventory 그룹 기준으로 생성됩니다.

## 사용법

```bash
cd /galaxy
ansible-playbook apa.yml     # 배포
ansible-playbook rm.yml      # 제거
```

## 변수

`group_vars/all.yml`

```yaml
wp_db_name: word
wp_db_user: yhlee
wp_db_password: 'It12345!'
wp_db_host: 10.0.0.24

dns_zone: yhlee.local
dns_forwarders:
  - 168.126.63.1
  - 8.8.8.8
```

## 접속

- http://www.yhlee.local/  (HAProxy → 22/25 라운드로빈)
- http://10.0.0.22/  (Rocky httpd)
- http://10.0.0.25/  (Ubuntu apache2)
- http://10.0.0.23/  (HAProxy)

## DNS 사용 시 주의

- 클라이언트 DNS 를 `10.0.0.23` 으로 지정하면 `yhlee.local` 도메인을 사용할 수 있습니다.
- `.local` 은 mDNS 예약 도메인이므로 Windows/Ubuntu 에서 mDNS(Bonjour/avahi)와 충돌할 수 있습니다. 문제가 있으면 `dig @10.0.0.23 www.yhlee.local` 로 서버를 직접 검증하세요.
- WordPress 설치를 `http://www.yhlee.local/` 로 진행하면 siteurl 이 도메인으로 고정됩니다. IP 로 설치한 뒤 도메인을 붙이려면 `wp_options` 의 `siteurl`/`home` 을 변경해야 합니다.
- 존 레코드 변경 시 `roles/dns/defaults/main.yml` 의 `dns_serial` 을 올려 주세요.
