# Анализ производительности распределённой вычислительной системы

Приложение для анализа содержит в себе:
- сервис обработки запросов (сервис HTTP-роутера);
- сервис контроля прав доступа (сервис ACL);
- сервис хранения данных (СУБД).

Для упрощения задачи из инфраструктуры системы исключены БД прав доступа и кеш представленные на рисунке


# Цели проекта
 - использовать систему Prometheus для сбора метрик системы;
 - использовать средство визуализации Grafana для анализа производительности системы;
 - проводить анализ производительности системы с заданной архитектурой (см. рисунок).

# Запуск инфраструктуры
Запуск осуществляется командой: 

`docker-compose -f docker-compose.yml up -d`  

Запустите инфраструктуру и убедитесь, что все сервисы запущены и работоспособны, используя команду `docker ps`.

# Настройка визуализатора Grafana

Для настройки визуализатора Grafana надо посетить http://localhost:3000/login и ввести данные, определённые в переменных `GF_SECURITY_ADMIN_USER` и `GF_SECURITY_ADMIN_PASSWORD` из `docker-compose.yaml`. Затем в настройках потребуется добавить новый источник данных типа Prometheus, указав в качестве URL его внутренний адрес: http://prometheus:9090.

# USE-метрики подсистемы

USE-метрики - показания по утилизации и сатурации для работающей серверной инфраструктуры. Агент `node-exporter` производит чтение показаний из ProcFS, а Prometheus собирает данные в собственную TSDB с периодом, указанным в файле конфигурации `prometheus.yml` (5 секунд). Доступ к TSDB осуществляется через веб-интерфейс по адресу http://localhost:9090/graph. Веб-интерфейс предлагает строку быстрого поиска, использующую [язык запросов временных рядов](https://prometheus.io/docs/prometheus/latest/querying/basics/).


В таблице  приводятся запросы к системе Prometheus на языке временных рядов для получения USE-метрик подсистем: процессора, оперативной памяти, жёсткого диска и сети.

  **Важно!** *В таблице приводятся примеры запросов, но ни в коем случае не единственно верный вариант получения USE-метрик.*
  
  |        | Формула утилизации                | Формула сатурации |
| ------------- |:------------------| :-----|
| Процессор     | 1 - avg(irate(node_cpu_seconds_total{job="node-exporter",mode="idle"}[1m]))    | sum(node_load1{job="node-exporter"}) |
| Память     | 1 - sum(node_memory_MemFree_bytes{job="node-exporter"} + node_memory_Cached_bytes{job="node-exporter"} + node_memory_Buffers_bytes{job="node-exporter"}) / sum(node_memory_MemTotal_bytes{job="node-exporter"}) |   sum(rate(node_vmstat_pgpgin{job="node-exporter"}[1m]) + rate(node_vmstat_pgpgout{job="node-exporter"}[1m])) |
| Диск  | 1 - max(node_filesystem_avail_bytes{job="node-exporter"}) / max(node_filesystem_size_bytes{job="node-exporter"})         |    sum(rate(node_disk_reads_completed_total[1m]) + rate(node_disk_writes_completed_total[1m])) |
| Сеть | sum(rate(node_network_receive_bytes_total{job="node-exporter",device!="lo"}[1m]))  sum(rate(node_network_transmit_bytes_total{job="node-exporter",device!="lo"}[1m])) | sum(rate(node_network_receive_drop_total{job="node-exporter",device!="lo"}[1m]))  sum(rate(node_network_transmit_drop_total{job="node-exporter",device!="lo"}[1m])) |

# RED-метрики сервисов

После запуска разработанных сервисов наблюдение за поведением системы осуществляется посредством стандартного HTTP-обработчика `/metrics`. Обработчик создаётся автоматически при запуске библиотеки. [Prometheus Go client library](https://github.com/prometheus/client_golang)

|        | Requests               | Errors | Duration |
| ------------- |:------------------|:-----|:-----|
| acl.Identity     | rate(request_total{handler="/identity",job="acl",method="GET"}[1m])    | rate(errors_total{handler="/identity",job="acl",method="GET"}[1m]) | duration_seconds{handler="/identity",job="acl",method="GET",status="200",quantile="0.99"}  duration_seconds{handler="/identity",job="acl",method="GET",status="200",quantile="0.95"}  duration_seconds{handler="/identity",job="acl",method="GET",status="200",quantile="0.9"}|
| router.addEntity    | rate(request_total{handler="/entity",job="router",method="POST"}[1m]) |   rate(errors_total{handler="/entity",job="router",method="POST"}[1m]) | duration_seconds{handler="/entity",job="router",method="POST",status="200",quantile="0.99"}  duration_seconds{handler="/entity",job="router",method="POST",status="200",quantile="0.95"}  duration_seconds{handler="/entity",job="router",method="POST",status="200",quantile="0.9"} |
| rounter.listEntities  | rate(request_total{handler="/entities",job="router",method="GET"}[1m])         |  rate(errors_total{handler="/entities",job="router",method="GET"}[1m])  | duration_seconds{job="router",method="GET",status="200",quantile="0.99"}  duration_seconds{job="router",method="GET",status="200",quantile="0.95"}  duration_seconds{job="router",method="GET",status="200",quantile="0.9"} |

Обратите внимание, что для определения задержки (Duration) в таблице приводится целых три метрики, а не одна, как для запросов (Requests) и ошибок (Errors). Значение временного ряда — это случайная величина, и анализировать её целесообразно, используя статистический аппарат.

### Процесс введения нескольких метрик для определения задержки на конкретном примере. 

Пусть на заводе работает 10 человек. Девять из них получают зарплату 100 рублей, а один — 1000 рублей. Узнаем, насколько финансово благополучны сотрудники.

Если рассчитать среднюю зарплату, получим, что каждый сотрудник получает по 190 рублей. Но в действительности 90% рабочих таких денег не видят. Если же рассчитать медианное значение, то среднее значение как раз таки составит 100 рублей. Для рассматриваемого примера эта оценка более правдива и лучше описывает ситуацию.


### Способы оценки задержки ответов сервиса.

Анализировать все временные ряды, которые создаёт сервис, обрабатывающий, например, 100,000 и более запросов в секунду, бессмысленно, так как это потребует слишком много ресурсов. Медианное значение в этом случае также не представляет ценности, ведь 50% запросов будут обрабатываться медленнее. Поэтому при анализе задержки сервисов принято рассматривать более высокие перцентили, чем медианный:  90-й (p.9), 95-й (p.95), 99-й (p.99), 99.99-й (p.9999). В таблице приводятся формулы для расчёта задержки для перцентилей p.99, p.95 и p.9. Выбор перцентиля обычно определяется требованиями надсистемы.

# Нагрузочное тестирование


# TODO
- вставить картинку с архитектурой приложения
- написать сервис обработки запросов 
- написать сервис контроля прав доступа
- написать сервис хранения данных
- описать архитектуру исследуемой системы композицией сервисов acl, router, mysql в docker-compose.yaml
- сконфигурировать Prometheus в файле prometheus.yaml
- нагрузочное тестирование с помощью утилиты wrk и дополнить соответствующий раздел
- Создать набор RED-метрик для методов `Exec`, `Query` и `QueryRow` структуры `sql.Db` пакета `"database/sql"`, используя [библиотеку системы Prometheus](https://github.com/prometheus/client_golang)
- *Изменить нагрузочный тест для `router.addEntity` таким образом, чтобы число ошибок было менее 1%. Оценить изменения нагрузочного профиля
- *Придумать решение для обеспечения контроля производительности подсистемы, относящейся к сервису хранения данных, используя возможности [Percona PMM](https://www.percona.com/doc/percona-monitoring-and-management/deploy/server/docker.html)
- *Попытаться расширить архитектуру системы, дописать некоторые недостающие сервисы и провести анализ производительности. В качестве дополнительного сервиса использовать СУБД для хранения прав ACL — ранее описаны права доступа прямо в коде функции `acl.Identity`

lesson4 go-backend-2  course
