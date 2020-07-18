---
title: "Replacing Docker Compose with Ansible"
layout: post
excerpt: " "
tags:
  - docker
  - ansible
---

Source Code: <https://github.com/amard33p/ansible-docker>

We have a simple docker compose file which defines 2 components - ui and api.

```yaml
# docker-compose.yml
version: "3.5"

services:
  ui:
    build: ./ui
    ports:
      - "8080:8000"
    depends_on:
      - api

  api:
    build: ./api
    ports:
      - "8081:5000"
```

A common need during development is to mount the source code inside containers for faster iterations.  
To solve this we generally create a compose override file.

```yaml
# docker-compose.override.yml
version: "3.5"

services:
  ui:
    volumes:
      - ./ui/index.html:/app/index.html
```

The override file should be present in Dev environment but must be absent in Production as we would already have a tested pre-built image by then.

There are pre-processors like [jsonnet](https://jsonnet.org/) that allows the ability to add "logic" to dumb YAML files. However, in my opinion, a simpler solution can be achieved using Ansible Playbooks.

Using the Ansible modules [docker_network](https://docs.ansible.com/ansible/latest/modules/docker_network_module.html), [docker_image](https://docs.ansible.com/ansible/latest/modules/docker_image_module.html) and [docker_container](https://docs.ansible.com/ansible/latest/modules/docker_container_module.html), the equivalent standalone ansible playbook for the above compose files would be:

{% raw %}
```yaml
---
- hosts: localhost
  gather_facts: no
  vars:
    # Compose creates entities based on the directory name
    curdir: "{{ playbook_dir.split('/')[-1] }}"

  tasks:
  # Get value of devenv from extra args
  - set_fact:
      devenv: "{{ devenv | default(false) }}"
  - debug: 
      msg: Dev Environment = {{ devenv }}

  - name: Creating bridge network {{ curdir }}_default
    docker_network:
      name: "{{ curdir }}_default"

  - name: Building image {{ curdir }}_api
    docker_image:
      build:
        path: ./api
      source: build
      name: "{{ curdir }}_api"
    when: devenv|bool

  - name: Starting container {{ curdir }}_api_1
    docker_container:
      name: "{{ curdir }}_api_1"
      image:  "{{ curdir }}_api"
      networks_cli_compatible: yes
      network_mode: "{{ curdir }}_default"
      networks:
      - name: "{{ curdir }}_default"
      ports: 
        - '8081:5000'

  - name: Building image {{ curdir }}_ui
    docker_image:
      build:
        path: ./ui
      source: build
      name: "{{ curdir }}_ui"
    when: devenv|bool

  - name: Starting container {{ curdir }}_ui_1
    docker_container:
      name: "{{ curdir }}_ui_1"
      image:  "{{ curdir }}_ui"
      networks_cli_compatible: yes
      network_mode: "{{ curdir }}_default"
      networks:
      - name: "{{ curdir }}_default"
      ports: 
        - '8080:8000'
      volumes:
        - "{{ './ui/index.html:/app/index.html' if devenv else omit }}"
```
{% endraw %}

We pass in a flag through extra-args to specify the environment and use the volume mount based on that:  
{% raw %}
```
volumes:
  - "{{ './ui/index.html:/app/index.html' if devenv else omit }}"
{% endraw %}```


Run the playbook as follows:
- In Dev: `ansible-playbook -e "devenv=true" compose_playbook.yml`
- In Test/Prod: `ansible-playbook compose_playbook.yml`
