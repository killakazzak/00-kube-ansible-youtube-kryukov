# Ansible playbook для установки тестового кластера k8s

Плейбук проверяется на наборе [приложений](https://github.com/BigKAA/youtube/tree/master/1.31).

| k8s ver         | Distributive    | CRI             | Notes           |
|-----------------|-----------------|-----------------|-----------------|
| 1.32            | Debian 12 | CRI-O 1.32   | *Не стартует harbor.* |
| 1.31            | Ubuntu 22.04.4 LTS | CRI-O 1.31   |  *Не стартует harbor. Не монтируется emptyDir.* |
| 1.3x            | Ubuntu 22.04.4 LTS | containerd 1.7.12  | С последним обновление должен работать. *На проверке* |
| 1.31.2          | Rocky Linux 9.4 | containerd 1.7.23-3.1 | **OK** |
| 1.31            | Rocky Linux 8.10 | CRI-O 1.31 | *Не стартует harbor. Не монтируется emptyDir.*  |
| 1.31            | Rocky Linux 8.10 | containerd 1.6.32 | **OK** *Приложения пока не тестил* |
| 1.30            | Rocky Linux 8.10 | containerd 1.6.32 | **OK** |
| 1.31            | Debian 12 | containerd.io 1.7.21 | Кластер стартует. Не работает metallb. |
| 1.30            | Debian 12 | containerd.io 1.7.21 | **Ok** |

Остальные дистрибутивы проверю, когда до них руки дойдут.

Поддерживает:

- Kubernetes v1.31.
- Установку одной или несколько control nodes.
- HA доступ к API kubernetes.
- CRI-O.
- calico.
- В KubeProxyConfiguration установлены параметры для работы Metallb.
- nodelocaldns - кеширующий DNS сервер на каждой ноде кластера.

## Установка ansible

Так получилось, что у меня в WSL2 стоит Ubuntu:

```shell
python3 -m venv venv
. ~/venv/bin/activate
pip3 install "ansible-core<2.17"
```

Или используем контейнер с Ansible:

```shell
docker pull alpine/ansible:2.17.0
mkdir -p ~/.ansible
alias ansible-playbook="docker run -ti --rm -v ~/.ssh:/root/.ssh -v ~/.ansible:/root/.ansible -v $(pwd):/apps -w /apps alpine/ansible:2.17.0 ansible-playbook"
ansible-playbook --version
```

Генерируем ssh ключ:

```shell
ssh-keygen
```

Копируем ключики в виртуальные машины из [hosts.yaml](hosts.yml):

 ```shell
ssh-copy-id root@control1.kryukov.local
ssh-copy-id root@control2.kryukov.local
ssh-copy-id root@control3.kryukov.local
ssh-copy-id root@worker1.kryukov.local
ssh-copy-id root@worker2.kryukov.local
ssh-copy-id root@worker3.kryukov.local
```

## Конфигурационные параметры

- [Инвентори](hosts.yaml).
- [Общая конфигурация](group_vars/k8s_cluster).

## Установка

### k8s с одной control node

В [инвентори](hosts.yaml) в группе `k8s_masters` необходимо указать только один хост.

```shell
ansible-playbook install-cluster.yaml
```

### k8s с несколькими control nodes

В [инвентори](hosts.yaml) в группе `k8s_masters` необходимо указать **нечётное количество
control nodes**.

```shell
ansible-playbook install-cluster.yaml
```

### k8s c HA

Используются haproxy и keepalived.

![ha cluster](images/ha_cluster.jpg)

В конфигурационном файле определите параметры доступа к API :

- `ha_cluster_virtual_ip` - виртуальный IP адрес.
- `ha_cluster_virtual_port` - порт. Не должен быть равен 6443.

## Удалить кластер

```shell
ansible-playbook reset.yaml
```

**Внимание!!!** Скрипт удаляет **все** нестандартные цепочки и чистит все стандартные цепочки.

## Апдейт кластера

Изменяете версию кластера в `group_vars\k8s_cluster` и запускаете апдейт.

```shell
ansible-playbook upgrade.yaml
```

## Utils playbook

Playbook с утилитами. [Обычный набор утилит](https://github.com/BigKAA/youtube/tree/master/1.31), который я ставлю в тестовых кластерах. Раньше ставил руками. Надоело, решил автоматизировать.

```shell
ansible-playbook services/06-utils.yaml
```

## Сервисные функции

Сервисные функции находятся в директории `services`