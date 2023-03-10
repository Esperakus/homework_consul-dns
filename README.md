# Домашняя работа "Consul cluster для service discovery и DNS"

Цель работы - добавить в стенд веб приложения из прошлых ДЗ Service Discovery nginx с помощью кластера Consul. Точка входа - Haproxy, в качестве бэкенда в ней - кластер Consul из 3х нод с настроенным DNS, который проверяет наличие сервисов Nginx на 2х виртуальных машинах. Nginx в свою очередь балансируют запросы к веб-приложениям (простейшее приложение на go) на 2 бэкендах.

Данный репозиторий содержит:

- Манифесты terraform для создания инфраструктуры проекта:
  - виртуальная машина с Haproxy
  - 3 ноды кластера Consul в режиме сервера
  - 2 воркера nginx, которые в свою очередь настроены на простейшую балансировку трафика в бэкенды веб приложения
  - 2 воркера бэкенда, на которых в systemd запущено простейшее приложение на go, слушающее порт 8090. При запросе отдаёт в зависимости от uri имя бэкенда (чтобы понять, на какой бэкенд прилетел запрос из Nginx), версию БД, либо статику в виде картинки
  - 1 iscsi target, раздающий диск со статикой в бэкенды
  - 1 экземпляр БД Postgresql 13 c базой test и пользователем БД test, чтоб принимать запросы от бэкенда
  - и, наконец, виртуалка с установленным ансиблем, чтоб развернуть вышеупомянутые роли. Выступает так же в роли Jump host проекта

- Роли ansible для приведения виртуальных машин в проекте в требуемое состояние.

При разворачивании стенда создаются ВМ с параметрами:
- 2 CPU;
- 2 GB RAM;
- 10 GB диск;
- операционная система almalinux 8;

Для разворачивания стенда необходимо:

1. Заполнить значение переменных cloud_id, folder_id и iam-token в файле **variables.tf**.

2. Инициализировать рабочую среду Terraform:

```
$ terraform init
```
В результате будет установлен провайдер для подключения к облаку Яндекс.

3. Запустить разворачивание стенда:
```
$ terraform apply
```
В выходных данных будут показаны все внешние и внутренни ip адреса. 
Для проверки работы стенда необходимо открыть status страницу haproxy (*http://{external_ip_address_haproxy}:7000*) и веб-консоль consul'a (*http://{external_ip_address_haproxy}:8500*). На 80м порте можно посмотреть работу приложения (*http://external_ip_address_haproxy/* , *http://external_ip_address_haproxy/db* , *http://external_ip_address_haproxy/image*)

С джамп-хоста можно ходить по ssh по всем машинам внутри проекта по их внутренним ip адресам или hostname (nginx0, nginx1, backend0, backend1, db, iscsi).

Если останавливать по одной службы nginx на балансировщиках nginx0, nginx1 можно увидеть, что запросы всё равно идут к бэкендам благодаря consul Service Discovery и consul DNS, который отдаёт в haproxy только рабочие сервера nginx.

На странице статуса haproxy будет наглядно видно, что при остановке nginx на одном из серверов его статус в haproxy будет меняться на Maintanence и запросы на него ходить не будут. В веб-консоли consul в этом случае будет видно, что сервис на одной из нод nginx недоступен. При повторном запуске nginx он включается в балансировку haproxy и снова принимает запросы.