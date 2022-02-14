#devops-netology
### 08-ansible-05-testing
# Домашнее задание к занятию "08.05 Тестирование Roles"

## Подготовка к выполнению
1. Установите molecule: `pip3 install "molecule==3.4.0"`
2. Соберите локальный образ на основе [Dockerfile](./Dockerfile)

**Подготовка:**
Регистрируем аккаунт для того, чтобы получить доступ к репозиториям, из которого берется образ, на основе которого билдится докер-контейнер.  
**Выполнение:**  
Правим requirements.yml, скорректировав версию на 2.0.3, качаем роли в рабочую папку:  
```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-5$ ansible-galaxy install -r requirements.yml -p roles
- extracting elastic to /home/dusk/devops-netology-ansible-5/roles/elastic
- elastic (2.0.3) was installed successfully
- kibana (1.0.3) is already installed, skipping.
- filebeat (1.0.4) is already installed, skipping.
```
Поднимаем в докере подготовленный ранее контейнер с RedHat, заходим:  
```
docker run --name testrun-8-5-01 --privileged=True -v $(pwd):/opt/elastic -w /opt/elastic -it duskdemon/duskdemon:homework-8-5 /bin/bash
[root@7801051f9a5f elastic]#
```
Ставим молекулу:  
```
[root@7801051f9a5f elastic]# pip install -r test-requirements.txt --force
Collecting molecule==3.4.0
  Downloading molecule-3.4.0-py3-none-any.whl (239 kB)
     |████████████████████████████████| 239 kB 1.4 MB/s            
Collecting molecule_podman==0.3.0
  Downloading molecule_podman-0.3.0-py3-none-any.whl (13 kB)
Collecting ansible<3
...
```
Проверяем, что она работает:  
```
[root@7801051f9a5f elastic]# molecule --version
molecule 3.4.0 using python 3.6 
    ansible:2.10.17
    delegated:3.4.0 from molecule
    podman:0.3.0 from molecule_podman
[root@7801051f9a5f elastic]#
```

## Основная часть

Наша основная цель - настроить тестирование наших ролей. Задача: сделать сценарии тестирования для kibana, logstash. Ожидаемый результат: все сценарии успешно проходят тестирование ролей.

### Molecule

1. Запустите  `molecule test` внутри корневой директории elasticsearch-role, посмотрите на вывод команды.
**Выполнение:**
Запускаем тест, и он фейлится. Похоже, проблемы совместимости. В лекции использовался podman 2.2.1, а в актуальном образе из репозиториев Редхата идет версия 3.4.0. Погуглил, нашлось решение, нужно поправить в файле /etc/containers/containers.conf:  
```
utsns="host" -> utsns="private"
``` 
В ходе прогона теста также наткнулся на опечатку в файле molecule.yml:  
image: docker.io/pycontribs/ubunut:latest  
исправлено: ubuntu  
Результаты теста:  
```
...
PLAY RECAP *********************************************************************
centos7                    : ok=7    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
ubuntu                     : ok=7    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

INFO     Idempotence completed successfully.
INFO     Running default > side_effect
WARNING  Skipping, side effect playbook not configured.
INFO     Running default > verify
INFO     Running Ansible Verifier

PLAY [Verify] ******************************************************************

TASK [Example assertion] *******************************************************
ok: [centos7] => {
    "changed": false,
    "msg": "All assertions passed"
}
ok: [ubuntu] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP *********************************************************************
centos7                    : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
ubuntu                     : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

INFO     Verifier completed successfully.
INFO     Running default > cleanup
WARNING  Skipping, cleanup playbook not configured.
INFO     Running default > destroy

PLAY [Destroy] *****************************************************************

TASK [Destroy molecule instance(s)] ********************************************
changed: [localhost] => (item={'image': 'docker.io/pycontribs/centos:7', 'name': 'centos7', 'pre_build_image': True})
changed: [localhost] => (item={'image': 'docker.io/pycontribs/ubuntu:latest', 'name': 'ubuntu', 'pre_build_image': True})

TASK [Wait for instance(s) deletion to complete] *******************************
FAILED - RETRYING: Wait for instance(s) deletion to complete (300 retries left).
FAILED - RETRYING: Wait for instance(s) deletion to complete (299 retries left).
FAILED - RETRYING: Wait for instance(s) deletion to complete (298 retries left).
changed: [localhost] => (item={'started': 1, 'finished': 0, 'ansible_job_id': '513662985133.14632', 'results_file': '/root/.ansible_async/513662985133.14632', 'changed': True, 'failed': False, 'item': {'image': 'docker.io/pycontribs/centos:7', 'name': 'centos7', 'pre_build_image': True}, 'ansible_loop_var': 'item'})
changed: [localhost] => (item={'started': 1, 'finished': 0, 'ansible_job_id': '191336812751.14652', 'results_file': '/root/.ansible_async/191336812751.14652', 'changed': True, 'failed': False, 'item': {'image': 'docker.io/pycontribs/ubuntu:latest', 'name': 'ubuntu', 'pre_build_image': True}, 'ansible_loop_var': 'item'})

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

INFO     Pruning extra files from scenario ephemeral directory
[root@7801051f9a5f elastic]#
``` 
2. Перейдите в каталог с ролью kibana-role и создайте сценарий тестирования по умолчаню при помощи `molecule init scenario --driver-name docker`.
**Выполнение:**
заходим во вновь созданный контейнер по аналогии с предыдущим, ставим молекулу(для этого предварительно копируем файл requirements.txt). Но с указанием на драйвер докера получаем ошибку
```
[root@2b578202c18f kibana]# molecule init scenario --driver-name docker
Usage: molecule init scenario [OPTIONS] [SCENARIO_NAME]
Try 'molecule init scenario --help' for help.

Error: Invalid value for '--driver-name' / '-d': 'docker' is not one of 'delegated', 'podman'.
[root@2b578202c18f kibana]#
```
Делаем по-дроугому:
```
[root@2b578202c18f kibana]# molecule init scenario scen01 --driver-name podman
INFO     Initializing new scenario scen01...
INFO     Initialized scenario in /opt/kibana/molecule/scen01 successfully.
```
3. Добавьте несколько разных дистрибутивов (centos:8, ubuntu:latest) для инстансов и протестируйте роль, исправьте найденные ошибки, если они есть.
**Выполнение:**
Правим файл molecule.yml, чтобы он выглядел так:  
```yaml
---
dependency:
  name: galaxy
driver:
  name: podman
platforms:
  - name: centos8
    image: docker.io/pycontribs/centos:8
    pre_build_image: true
  - name: ubuntu
    image: docker.io/pycontribs/ubuntu:latest
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```
Получаем ошибку, т.к. в тасках не учтено, что в новой версии centos менеджер пакетов DNF:
```
TASK [kibana : include_tasks] **************************************************
fatal: [centos8]: FAILED! => {"reason": "Could not find or access '/opt/kibana/molecule/scen01/download_dnf.yml' on the Ansible Controller."}
```
копируем файл download_yum.yml и называем download_dnf.yml, добавляем аналогичный install_dnf.yml, но теперь повторный запуск дает следующую ошибку:  
```
TASK [kibana : Install kibana] *************************************************
fatal: [centos8]: FAILED! => {"changed": false, "msg": "Failed to download metadata for repo 'appstream': Cannot prepare internal mirrorlist: No URLs in mirrorlist", "rc": 1, "results": []}
```
Исправить ее я смог, поправив файл install_dnf.yml так:
```yaml
---
- name: Install kibana
  become: true
  dnf:
    name: "/tmp/kibana-{{ kibana_version }}-x86_64.rpm"
    state: present
    disablerepo: "appstream,baseos,epel,epel-modular,extras"
    disable_gpg_check: yes
  notify: restart kibana
```
---

4. Добавьте несколько assert'ов в verify.yml файл, для  проверки работоспособности kibana-role (проверка, что web отвечает, проверка логов, etc). Запустите тестирование роли повторно и проверьте, что оно прошло успешно.  
**Выполнение:**
Добавил проврку наличия файла в каталоге /etc для кибаны/файлбит:  
```
---
# This is an example playbook to execute Ansible tests.

- name: Verify
  hosts: all
  gather_facts: false
  tasks:
  - name: check if file exists
    stat:
      path: /etc/filebeat/filebeat.yml
    register: stat_result
```
5. Повторите шаги 2-4 для filebeat-role.
**Выполнение:**
Делаем аналогичные шаги для файлбит, учитывая ошибки выше:  
```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-5/roles/filebeat$ docker run --name testrun-8-5-07 --privileged=True -v $(pwd):/opt/filebeat -w /opt/filebeat -it duskdemon/duskdemon:homework-8-5 /bin/bash
[root@d646ed4473ac filebeat]# vi /etc/containers/containers.conf 
[root@d646ed4473ac filebeat]# pip install pip install -r test-requirements.txt --force
```
При отладке тестов роли filebeat через molecule не проходит установка на ubuntu:
fatal: [ubuntu]: FAILED! => {“changed”: false, “msg”: “AnsibleUndefinedVariable: ‘dict object’ has no attribute ‘default_ipv4’”}
Пытался перед установкой filebeat доп. таской поставить на ubuntu iproute2 - не помогло. Подскажите, куда копать?

---  

6. Добавьте новый тег на коммит с рабочим сценарием в соответствии с семантическим версионированием.

### Tox

1. Запустите `docker run --privileged=True -v <path_to_repo>:/opt/elasticsearch-role -w /opt/elasticsearch-role -it <image_name> /bin/bash`, где path_to_repo - путь до корня репозитория с elasticsearch-role на вашей файловой системе.
2. Внутри контейнера выполните команду `tox`, посмотрите на вывод.
3. Добавьте файл `tox.ini` в корень репозитория каждой своей роли.
4. Создайте облегчённый сценарий для `molecule`. Проверьте его на исполнимость.
5. Пропишите правильную команду в `tox.ini` для того чтобы запускался облегчённый сценарий.
6. Запустите `docker` контейнер так, чтобы внутри оказались обе ваши роли.
7. Зайдти поочерёдно в каждую из них и запустите команду `tox`. Убедитесь, что всё отработало успешно.
8. Добавьте новый тег на коммит с рабочим сценарием в соответствии с семантическим версионированием.

После выполнения у вас должно получится два сценария molecule и один tox.ini файл в каждом репозитории. Ссылки на репозитории являются ответами на домашнее задание. Не забудьте указать в ответе теги решений Tox и Molecule заданий.

## Необязательная часть

1. Проделайте схожие манипуляции для создания роли logstash.
2. Создайте дополнительный набор tasks, который позволяет обновлять стек ELK.
3. В ролях добавьте тестирование в раздел `verify.yml`. Данный раздел должен проверять, что logstash через команду `logstash -e 'input { stdin { } } output { stdout {} }'`  отвечате адекватно.
4. Создайте сценарий внутри любой из своих ролей, который умеет поднимать весь стек при помощи всех ролей.
5. Убедитесь в работоспособности своего стека. Создайте отдельный verify.yml, который будет проверять работоспособность интеграции всех инструментов между ними.
6. Выложите свои roles в репозитории. В ответ приведите ссылки.

---
