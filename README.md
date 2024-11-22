# grafana 대시보드 구축

## 0. 목적
> 📜 **그라파나로 모니터링하는데 파일로그랑 컨테이너로그 둘다 캐치할수있어야한다.!**

## # 1. Grafana로 로그 수집 방법 🙆‍♂️
> 📜 **로그 메트릭 수집 서버랑 모니터링 서버는 다른 서버입니다.**
- **`A` Container(파일로그 수집)**
    - **Promtail 컨테이너로 진행**
- **`B` Container(컨테이너 로그 생성)**
    - **도커 플러그인으로 진행**

## 2. 모니터링 서버 구축 ⚒️
📘 **모니터링 서버 사양**
- Instance : **`t3.small`**
- CPU : **`2vCPU(x86_64)`**
- Memory : **`2GiB`**
- Storage : **`32GiB`**

### 2-0. 도커 설치
```
{
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ubuntu
sudo mkdir /dockerimages
sudo sed -i 's#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --data-root=/dockerimages#' /lib/systemd/system/docker.service
sudo systemctl daemon-reload
sudo service docker restart
}
```

### 2-1. 모니터링 서버 구축
```
.
├── README.md
├── data
│   ├── grafana
│   └── loki
├── docker-compose.yaml
└── infra
    ├── grafana
    │   └── provisioning
    │       ├── dashboards
    │       │   ├── dashboard.yaml
    │       │   └── systemMetric.json
    │       └── datasources
    │           └── datasource.yaml
    └── loki
        └── loki.yaml
```
> docker compose up -d

## 3. 계측 서버 구축 ⚒️
### 3-1. 파일 로그 수집
```
.
├── config
│   └── config.yaml
└── docker-compose.yaml
```
- docker-compose.yaml file
  ```
  services:
    promtail:
      container_name: promtail
      image: grafana/promtail:3.1.0
      restart: always
      command:
        - '-config.file=/etc/promtail/config.yaml'
      volumes:
        - '/var/log/apache2/:/var/log/access'
        - '${PWD}/config/config.yaml:/etc/promtail/config.yaml:ro'
        -  '/var/lib/docker/containers:/var/lib/docker/containers:ro'
        -  /var/run/docker.sock:/var/run/docker.sock 
  ```
- config file
  ```
  server:
    http_listen_port: 9080
    grpc_listen_port: 0
    log_level: "info"

  positions:
    filename: /tmp/positions.yaml

  clients: ## 모니터링 인스턴스 IP 주소:포트 삽입
    - url: http://[모니터링 인스턴스 IP 주소:포트]/loki/api/v1/push

  scrape_configs:
  ########################### 파일 로그 #################################
  - job_name: filelogs
    static_configs:
    - targets:
        - localhost
      labels:
        job: varlogs
        __path__: [컨테이너 내의 파일 로그가 저장되는 위치]
  ```
### 3-2. 컨테이너 로그 수집
**Loki 플러그인 설치**
  ```
  docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
  ```

**Loki 플러그인 삭제**
  ```
  docker plugin disable loki --force
  docker plugin rm loki
  ```

**[수정 및 생성] /etc/docker/daemon.json**
> /etc/docker/daemon.json 파일 내용을 아래와 같이 수정
```
{
  "debug": true,
  "log-driver": "loki",
  "log-opts": {
        "loki-url": "http://[IP주소]:3100/loki/api/v1/push",
        "loki-batch-size": "400"
    }
}
```
- /etc/docker/daemon.json 적용
  ```
  sudo systemctl restart docker
  ```

- 로그 수집 컨테이너에 적용(docker-compose)
  ```
  x-default-logging: &logging
    driver: 'loki'
    options:
      loki-url : "http://[로키IP 주소]:3100/loki/api/v1/push"

  services:
    slackbot:
      image: slackbot:latest
      container_name: slackbot_container
      build:
        context: .
        dockerfile: dockerfile
      ulimits:
        memlock: -1
        stack: 67108864
      tty: true
      restart: always
      logging: *logging ## 이부분
      privileged: true
  ```