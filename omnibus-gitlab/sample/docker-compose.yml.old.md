FWIW, I've got it working (Linux and MacOSX) with the following docker-compose file and helper script to register the runner.

DISCLAIMER: USE AT YOUR OWN. IT IS MEANT TO BE USED FOR TESTING/DEVELOPMENT ONLY.

    docker-compose.yaml

version: '3.7'

services:
  gitlab-web:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab-web
    hostname: gitlab-web
    volumes:
      - './gitlab-config:/etc/gitlab'
      - './gitlab-logs:/var/log/gitlab'
      - './gitlab-data:/var/opt/gitlab'
    ports:
      - '2222:22'
      - '8080:80'
      - '443:443'
      - '4567:4567'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        registry_external_url 'http://localhost:4567'
        registry['enable'] = true
        unicorn['socket'] = '/opt/gitlab/var/unicorn/gitlab.socket'
    networks:
      - gitlab-network

  gitlab-runner1:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner1
    hostname: gitlab-runner1
    volumes:
      - './gitlab-runner1-config:/etc/gitlab-runner:Z'
      - '/var/run/docker.sock:/var/run/docker.sock'
    networks:
      - gitlab-network

networks:
  gitlab-network:
    name: gitlab-network

    gitlab-runner-register.sh

#!/bin/sh
# Get the registration token from:
# http://localhost:8080/root/${project}/settings/ci_cd

registration_token=XXXXXXXXXXXXXXXXX

docker exec -it gitlab-runner1 \
  gitlab-runner register \
    --non-interactive \
    --registration-token ${registration_token} \
    --locked=false \
    --description docker-stable \
    --url http://gitlab-web \
    --executor docker \
    --docker-image docker:stable \
    --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
    --docker-network-mode gitlab-network

    .gitlab-ci.yml

image: docker:latest

services:
  - docker:dind

before_script:
  - docker info

build:
  stage: build
  script:
    - docker ps -a
