## Тестирование

Подготовительный этап:
- На хостовой машине выполнено `sudo sysctl -w vm.max_map_count=262144` (иначе не работает Elasticsearch)
- Почти во всех tasks и roles добавлена переменная `ansible_become: false`, чтобы игнорировалось `become: true` 
(иначе падают ошибки при выполнении на `podman`)
- Сценарий инициирован: `molecule init scenario --driver-name podman`
- Дополнительно в папке `molecule/default` создана подпапка `files` (туда происходит скачивание)
- Запуск производился с помощью `molecule --debug -vvv test --destroy=never` (`molecule destroy` выполнялся отдельно)
- На хост заходила с помощью `molecule login --host centos7`

Хосты:
- выполняется на `centos7` и `ubuntu` (`centos8` требует dnf по умолчанию, а у нас везде yum)
- в `platforms` также указан пустой `hostname`, иначе падает ошибка
- с `ubuntu` есть периодические проблемы со сменой пользователя (`su test`) - из-за `ansible_become: false` по-другому 
сменить пользователя не удаётся

`dependency`:
- роли подключаются через локальный git-репозиторий (т.к. в elasticsearch-role была необходима правка в handlers по
`when: ansible_virtualization_type != 'docker' and ansible_virtualization_type != 'podman'` - без условия по `podman` пытается запустить handler).
Для этого в `molecule.yml` в `provisioner` в `env` указана переменная `ANSIBLE_ROLES_PATH: /opt/ansible-testing` и закомментирован
блок с `dependency` и ссылкой на файл `requirements.yml`)

`prepare`:
- выполняется установка `ip route` (иначе в `hostvars` в `ubuntu` не появляется IP-адрес) и `Java` (для работы Elasticsearch)
- затем подключена роль `mnt-homeworks-ansible` для установки Elasticsearch и роль `kibana-role` для установки Kibana
- создан пользователь `test`, добавлен в `sudoers` (по шаблону), добавлены права на файлы Elasticsearch и выполнен запуск от имени этого пользователя
  (под рутом он не работает)
- добавлены права на файлы Kibana, выполнен запуск от имени пользователя `test`

`converge`:
- выполняются действия по установке `filebeat-role` с параметрами `ansible_become: false`, `filebeat_elastic_host: '{{ inventory_hostname }}'`,
`filebeat_kibana_host: '{{ inventory_hostname }}'`

`verify`:
- проверяется успешный http запрос Elasticsearch
- проверяется успешный http запрос Kibana
- проверяются логи filebeat

Правки в `filebeat-role`:
- В handlers добавлено условие по `ansible_virtualization_type != 'podman'`
- В шаблоне `filebeat.yml.j2` вместо хардкода в виде `el-instance` и `ki-instance` указаны переменные `filebeat_elastic_host` (хост с Elasticsearch) 
и `filebeat_kibana_host` (хост с Kibana).
В роли эти переменные вынесены в `defaults`, прописаны в README.md. В целях тестирования определяются как `inventory_hostname`


Итоговый ответ `verify` по `centos7`:
```
PLAY RECAP *********************************************************************
centos7                    : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

INFO     Verifier completed successfully.
```