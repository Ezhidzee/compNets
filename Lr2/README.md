# Отчет по лабораторной работе №2
## Тема: Loki + Zabbix + Grafana

### Ход работы

### Подготовка файлов конфигурации

docker-compose.yml

```yml
version: '3'
services:
  nextcloud:
    image: nextcloud:29.0.6
    container_name: nextcloud
    ports:
      - "8080:80"
    volumes:
      - nc-data:/var/www/html/data

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - nc-data:/opt/nc_data # Монтируем ту же папку, где лежат логи Nextcloud
      - ./promtail_config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:11.2.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    command: /run.sh

  postgres-zabbix:
    image: postgres:15
    container_name: postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
    volumes:
      - zabbix-db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "zabbix"]
      interval: 10s
      retries: 5
      start_period: 5s

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-6.4-latest
    container_name: zabbix-back
    ports:
      - "10051:10051"
    depends_on:
      - postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      DB_SERVER_HOST: postgres-zabbix

  zabbix-web-nginx-pgsql:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-6.4-latest
    container_name: zabbix-front
    ports:
      - "8082:8080"
    depends_on:
      - postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      DB_SERVER_HOST: postgres-zabbix
      ZBX_SERVER_HOST: zabbix-back

volumes:
  nc-data:
  zabbix-db:
```

promtail_config.yml

```yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: nextcloud_logs
      __path__: /opt/nc_data/*.log
```

template.yml

```yml
zabbix_export:
  version: '6.4'
  template_groups:
    - uuid: a571c0d144b14fd4a87a9d9b2aa9fcd6
      name: Templates/Applications
  templates:
    - uuid: a615dc391a474a9fb24bee9f0ae57e9e
      template: 'Test ping template'
      name: 'Test ping template'
      groups:
        - name: Templates/Applications
      items:
        - uuid: a987740f59d54b57a9201f2bc2dae8dc
          name: 'Nextcloud: ping service'
          type: HTTP_AGENT
          key: nextcloud.ping
          value_type: TEXT
          trends: '0'
          preprocessing:
            - type: JSONPATH
              parameters:
                - $.body.maintenance
            - type: STR_REPLACE
              parameters:
                - 'false'
                - healthy
            - type: STR_REPLACE
              parameters:
                - 'true'
                - unhealthy
          url: 'http://{HOST.HOST}/status.php'
          output_format: JSON
          triggers:
            - uuid: a904f3e66ca042a3a455bcf1c2fc5c8e
              expression: 'last(/Test ping template/nextcloud.ping)="unhealthy"'
              recovery_mode: RECOVERY_EXPRESSION
              recovery_expression: 'last(/Test ping template/nextcloud.ping)="healthy"'
              name: 'Nextcloud is in maintenance mode'
              priority: DISASTER
```

### Запускаем compose файл

![alt text](images/image.png)

### Инициализация Nextcloud

![alt text](<images/image copy.png>)

### Проверка генерации логов

![alt text](<images/image copy 2.png>)

### Настройка мониторинга

![alt text](<images/image copy 3.png>)

### Настройка доступа

![alt text](<images/image copy 4.png>)

### Создание хоста в Zabbix

![alt text](<images/image copy 5.png>)

### Проверка данных

![alt text](<images/image copy 6.png>)

### Визуализация

![alt text](<images/image copy 7.png>)

### Подключение Loki

![alt text](<images/image copy 8.png>)

![alt text](<images/image copy 9.png>)

### Подключение Zabbix

![alt text](<images/image copy 10.png>)

### Запрос Loki

![alt text](<images/image copy 11.png>)

![alt text](<images/image copy 12.png>)

### Запрос Zabbix

![alt text](<images/image copy 13.png>)

### Создание Дашбордов

![alt text](<images/image copy 14.png>)

![alt text](<images/image copy 15.png>)


# Ответы на вопросов

1.  **Чем SLO отличается от SLA?**
    *   SLA - это внешнее, юридическое соглашение с клиентом, описывающее гарантированный уровень услуг и штрафы за их невыполнение.
    *   SLO - это внутренняя цель команды инженеров по надежности сервиса (например, "мы хотим, чтобы сайт отвечал быстрее 200мс в 99% случаев"). SLO помогает соблюдать SLA.

2.  **Чем отличается инкрементальный бэкап от дифференциального?**
    *   Инкрементальный: Сохраняет только данные, изменившиеся с момента последнего любого бэкапа (полного или инкрементального). Занимает меньше места, но дольше восстанавливается (нужна вся цепочка).
    *   Дифференциальный: Сохраняет данные, изменившиеся с момента последнего полного бэкапа. Занимает больше места, чем инкрементальный, но быстрее восстанавливается (нужен только полный + последний дифференциальный).

3.  **В чем разница между мониторингом и observability?**
    *   Мониторинг отвечает на вопрос "Всё ли работает?" и "Здорова ли система?". Он показывает известные метрики и алертит, когда что-то сломалось.
    *   Observability отвечает на вопрос "Почему это сломалось?". Она позволяет понять внутреннее состояние системы на основе внешних данных (логи, трейсы, метрики) и помогает диагностировать неизвестные ранее проблемы.