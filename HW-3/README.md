# Домашнее задание 3

В качестве CMS используется Wordpress, установленный на хосте 10.0.0.5 (публичный адрес 87.242.118.70), там же настроены экспортеры для сбора метрик
## Стек CMS

Nginx отдает статистику через UNIX-сокет /var/run/nginx-status.sock по пути /nginx_status

PHP-FPM отдает статистику по tcp://127.0.0.1:9001/status

Для забора статистики используем отдельного пользователя в БД, подключаясь к стандартному порту 3306
## Настроенные экспортеры
Node Exporter - сбор основных метрик системы

NGINX Exporter - сбор метрик с NGINX

PHP-FPM Exporter - сбор метрик сервиса php-fpm

MYSQLD Exporter - сбор статистики с БД

Blackbox exporter - выполнение HTTP-чеков по корню главной страницы сайта

Exporter exporter - проксирование прочих экспортеров, чтоб избежать необходимости выставлении порта для каждого отдельно, слушает порт 9999

## AlerManager
Настроен на хосте 10.0.0.6, слушает порт 9093, публично не прокинут

Настроены уведомления в два telegram канала: для алертов уровня critical и уровня warning.

Алерт уровня critical срабатывает на отличие от 200 кода ответа с blackbox-экспортера.
Алерт уровня warning срабатывает на низкое значение аптайма сервиса БД,что может свидетельствовать о его перезапуске.

Примеры алертов в канале:

<img width="627" alt="image" src="https://github.com/user-attachments/assets/f4582b41-e550-4f3a-a137-ebed23e89754" />
<img width="658" alt="image" src="https://github.com/user-attachments/assets/d6e6e232-a4a1-4ef1-a1e7-866b0c51bf72" />



## VictoriaMetrics
Настроен на хосте 10.0.0.6 (публичный адрес http://213.171.28.17:8428)

Хранит метрики в течение 14 дней

## Prometheus
Настроен на хосте 10.0.0.6 (публичный адрес http://213.171.28.17:9090)

Все подключения к экспортерам выполняются через проксирующий Exporter exporter по порту 9999

Для выбора целевого экспортера на наблюдаемой машине используется параметр module
## Структура репозитория
Две Ansible роли для установки экспортеров и настройки Prometheus: roles/exporters и roles/prometheus

Роль exporters скачивает и устанавливает все указанные экспортеры в виде сервисов, конфигурационные файлы находятся в roles/exporters/templates

Роль alertmanager устанавливает alertmanager. Файл с настройками приемников алертов и матчинга по severity: roles/alertmanager/templates/alertmanager.yml.j2.

Роль vm скачивает, устанавливает и настраивает VictoriaMetrics. Файл с настройкой сервиса roles/vm/templates/vm.service.j2. В нем описаны параметры запуска, в том числе устанавливается retention период в 14 дней.

Роль prometheus скачивает, устанавливает и конфигурирует сам Prometheus. Основной конфигурационный файл - roles/prometheus/templates/prometheus.yml.j2, в нем описано подключение ко всем настроенным ранее экспортерам, кроме того указана отправка метрик в запущенный на том же порту VM на порту 8428, при помощи external_labels дописываем метку site: prod.

Для работы алертинга добавлен файл с правилами генерации алертов: roles/prometheus/templates/rules.yml.j2

## Запуск решения

Установить зависимости: pip3 install -r -requirements.txt (Опционально делать это в виртуальном окружении, чтоб не "запачкать" свою систему)

Настроить инвентарь в inventory.yml, вписав нужные адреса хостов

Добавить приватный ключ, указать его в ansible.cfg

Запустить установку экспортеров: ansible-playbook -i inventory.yml --ask-vault-pass exporters.yml

В ходе установки ввести пароль от ansible-vault (требуется для конфигурирования подключения mysqld exporter к базе данных)

Запустить установку Prometheus: ansible-playbook -i inventory.yml --ask-vault-pass prometheus.yml

Перейти на страницу http://213.171.28.17:9090 для доступа к prometheus, на страницу http://213.171.28.17:8428/vmui для доступа к VictoriaMetrics
