---
title: Docker 오프라인 설치하기
categories:
    - docker
tags:
    - docker

---

## Docker 오프라인 설치하기

인터넷이 막혀 curl이나 pip 등의 명령을 사용할 수 없는 환경에서 일하다 보면 직접 설치파일을 받아서 설치하는 법을 찾게 된다.



### 1. Setup Docker Daemon

1. Installation File Download (Local PC) : <https://download.docker.com/linux/ubuntu/dists/xenial/pool/stable/amd64/>
    1. docker-ce-cli
    2. containerd
    3. docker-ce
2. Local PC → Server로 파일 이동 (FTP활용)
3. 해당 서버로 이동하여 파일 설치
    ```bash
    sudo dpkg -i docker-ce-cli_~~~~.deb
    sudo dpkg -i containerd.io_~~~.deb
    sudo dpkg -i docker-ce_~~~.deb
    ```

잘 동작하는지 확인

```
sudo docker -v
```

> Ref. <https://docs.docker.com/engine/install/ubuntu/>

---

### 2. setup docker compose

1. File Download (Local PC) : <https://github.com/docker/compose/releases/download/1.27.4/docker-compose-Linux-x86_64>
2. Local PC → Server로 파일 이동
3. 서버의 다음 위치로 파일 이름 변경 및 이동: `/usr/local/bin/docker-compose`
    ```bash
    mv ./docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
    ```

4. 파일 권한 수정
    ```bash
    sudo chmod +x /usr/local/bin/docker-compose
    ```

5. 잘 동작하는지 확인:
    ```bash
    sudo docker-compose --version
    ```

> Ref. <https://docs.docker.com/compose/install/>

---

### 3. Set up Auto-Completion

1. Auto-Completion 주소 : <https://raw.githubusercontent.com/docker/compose/1.27.4/contrib/completion/bash/docker-compose>
2. 위 파일 내용을 copy하여 Local Folder 내 docker-compose 이름으로 파일 저장(개행문자를 LF로 저장해야 한다)
3. 서버의 다음 위치로 이동: /etc/bash_completion.d/docker-compose

> Ref. <https://docs.docker.com/compose/completion/>

---

### 4. set up local pc docker

서버에 Private Registry를 Setup 하거나, 이후에 서버의 Registry에 이미지를 push 하려는 경우 Local PC(오피스)에 Docker setup이 필요하다.

Local PC에 Docker Desktop 설치 시 Hyper-V가 활성화 되므로 보안 예외 결재가 필요하며, 다른 가상화 기술을 사용할 수 없게 된다(이를 피하려면 Docker Desktop 대신 VirtualBox를 사용하는 Docker Toolbox를 설치해야 한다).

<https://docs.docker.com/docker-for-windows/install/> 에서 Docker Desktop Installer 다운받은 후 설치

---

### 5. Set up Private Registry

1. Pull Registry Image (at Local PC Docker)
    ```bash
    docker pull registry:latest
    ```

2. 해당 Image를 FIle로 저장하여 서버로 이동
    ```bash
    docker save -o ./docker_registry.tar registry:latest

    scp ./docker_registry.tar ubuntu@12.26.62.68:/home/ubuntu/docker_install/docker_registry.tar
    ```

3. 서버로 이동 후 Image Load
    ```bash
    sudo docker load -i ./docker_registry.tar
    ```

4. Deploy Registry Container

    1. docker-compose.yml 파일 작성
    ```yml
    registry:
        restart: always
        image: registry:latest
        container_name: docker-registry
        ports:
        - 5000:5000
    ```

    2. deploy
    ```bash
    docker-compose up -d
    ```


5. 확인
    ```bash
    docker ps -a
    ```

> Ref. <https://docs.docker.com/registry/deploying/#deploy-your-registry-using-a-compose-file>

---

### 6. Registry Proxy 해제


1. /etc/docker/daemon.json 파일 생성 후 아래와 같이 설정

    ```json
    {
        "insecure-registries": ["[ip]:5000"]
    }
    ```

2. docker service 재시작

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart docker.service
    ```

3. 확인 - Local PC에서 Image Push
    ```bash
    docker pull jupyter/tensorflow-notebook:latest
    docker tag jupyter/tensorflow-notebook:latest 12.26.62.68:5000/jupyter:v1
    docker push 12.26.62.68:5000/jupyter:v1
    ```

4. 확인 - Server에서 Image Pull
    ```bash
    docker pull 12.26.62.68:5000/jupyter:v1
    ```

---

### 7. Set up Portainer

1. Portainer-CE Image Pull (at Local PC Docker)
    ```bash
    docker pull portainer/portainer-ce
    ```

2. Portainer-CE 및 Portainer-Agent Image tag & push
    ```bash
    docker tag portainer/portainer-ce [ip]:5000/portainer-ce:2.0.0

    docker push portainer/portainer-ce [ip]:5000/portainer-ce:2.0.0
    ```

3. yaml 파일 작성
    ```yml
    portainer:
    restart: always
    image: [ip]:5000/portainer-ce:2.0.0
    container_name: docker-portainer
    ports:
        - "9000:9000"
    volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - portainer_data:/data
    ```

4. 실행하여 확인한다.
    ```bash
    sudo docker-compose --file ~~.yml up -d
    ```
http://[ip]:9000/
최초 접속 시 관리자 암호 설정하는 화면이 나온다. 이후 Local 선택하면 50 서버의 docker를 관리할 수 있다.



> Ref. <https://www.portainer.io/installation/>


---

### 8. Set up nvidia docker


nvidia docker는 컨테이너 안에서 GPU를 사용할 수 있게 하는 docker이다.


<!-- img -->

보통 위 그림으로 많이 표현을 한다.



nvidia docker를 깔기 위해서는 우선 docker가 깔려있어야 하고, NVIDIA driver가 깔려있어야 한다.

NVIDIA driver 설치 확인은 nvidia-smi 명령으로 한다.


```bash
$ nvidia-smi
Wed Dec  2 10:04:56 2020
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.165.02   Driver Version: 418.165.02   CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+

...
```

보통은 repository를 업데이트 한 뒤 apt-get install nvidia-docker2 명령으로 설치를 한다.

오프라인으로 설치 시 deb파일을 받아서 하는데, 의존성 패키지들을 다 설치를 해줘야 한다.



nvidia-docker2 패키지의 의존성 관계는 다음과 같다.
<!--
nvidia-docker2
nvidia-container-runtime
nvidia-container-toolkit
libnvidia-container-tools
libnvidia-container1
-->



즉, libnvidia-container1 패키지부터 역 순서대로 설치를 해야 한다.



1. 설치 파일을 다운로드 받는다.
    1. nvidia-docker2 : https://nvidia.github.io/nvidia-docker/ubuntu18.04/amd64/nvidia-docker2_2.5.0-1_all.deb
    * 다른 버전 또는 OS의 경우 https://github.com/NVIDIA/nvidia-docker/tree/gh-pages 에서 경로와 파일명을 확인하여 수정하여 받는다.
    2. nvidia-container-runtime : https://nvidia.github.io/nvidia-container-runtime/ubuntu18.04/amd64/nvidia-container-runtime_3.4.0-1_amd64.deb
    * 다른 버전 또는 OS의 경우 https://github.com/NVIDIA/nvidia-container-runtime/tree/gh-pages 에서 경로와 파일명을 확인하여 수정하여 받는다.
    3. nvidia-continer-toolkit : https://nvidia.github.io/nvidia-container-runtime/ubuntu18.04/amd64/nvidia-container-toolkit_1.3.0-1_amd64.deb
    * 다른 버전 또는 OS의 경우 https://github.com/NVIDIA/nvidia-container-runtime/tree/gh-pages 에서 경로와 파일명을 확인하여 수정하여 받는다.
    4. libnvidia-container-tools : https://nvidia.github.io/libnvidia-container/ubuntu18.04/amd64/libnvidia-container-tools_1.3.0-1_amd64.deb
    * 다른 버전 또는 OS의 경우 https://github.com/NVIDIA/libnvidia-container/tree/gh-pages 에서 경로와 파일명을 확인하여 수정하여 받는다.
    5. libnvidia-container1 : https://nvidia.github.io/libnvidia-container/ubuntu18.04/amd64/libnvidia-container1_1.3.0-1_amd64.deb
    * 다른 버전 또는 OS의 경우 https://github.com/NVIDIA/libnvidia-container/tree/gh-pages 에서 경로와 파일명을 확인하여 수정하여 받는다.

2. 설치 파일을 서버로 옮긴다.
3. 파일을 설치한다. 다음의 순서로 해야 의존성 오류가 나지 않는다.

```bash
sudo dpkg -i libnvidia-container1_1.3.0-1_amd64.deb
sudo dpkg -i libnvidia-container-tools_1.3.0-1_amd64.deb
sudo dpkg -i nvidia-container-toolkit_1.3.0-1_amd64.deb
sudo dpkg -i nvidia-container-runtime_3.4.0-1_amd64.deb
sudo dpkg -i nvidia-docker2_2.5.0-1_all.deb
```

4. 설치를 하다 보면 사용자가 변경해 놓은 daemon.json을 설치 패키지의 세팅대로 바꿀지 말지 물어볼 수 있는데, 다음과 같다.
    <!-- img -->

    위에서 Registry Proxy 관련하여 설정하였던 파일인데, 우선 설치 패키지의 세팅 대로 덮어 쓰도록 한 후 nvidia-docker2 까지 설치 완료 후에 insecure-registries 부분을 추가하여 수정해 준다.


5. 잘 설치되었는지 확인
    ```bash
    nvidia-docker version
    ```

> Ref. <https://github.com/NVIDIA/nvidia-docker>
