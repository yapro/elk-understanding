# ELK understanding

```shell
docker-compose up
```

### Filebeat -> Logstash -> Elasticsearch

Пример того, что происходит с лог-записью используя текущую конфигурацию (logstash.conf):

| оригинал | в Elasticsearch |
|---|---|
| my php-fpm-1 | {"message":"my php-fpm-1"} |
| {"message":"my php-fpm-2"} | {"message": "{\"message\":\"my php-fpm-2\"}", "message_json": {"message": "my php-fpm-2"} } |
| {"message":"{\"inner\":\"my php-fpm-3\"}"} | {"message" : "{\"message\":\"{\\\"inner\\\":\\\"my php-fpm-3\\\"}\"}", "message_json": {"message": "{\"inner\":\"my php-fpm-3\"}"} } |

Ради понимания концепции в таблице появление дополнительных полей игнорируется (они не указаны).

## Logstash

Убедимся, что Logstash работает (будут выведены настройки):
```shell
curl localhost:9600/_node/stats?pretty
```
Если выдает следующее:
```text
curl: (56) Recv failure: Connection reset by peer
```
Значит, он еще не запустился, нужно подождать, запускается долго (N минут).

## Elasticsearch

Смотрю статус elasticsearch:
```shell
curl -XGET http://127.0.0.1:9200/_cluster/health?pretty=true
```
Можно минуя logstash добавить в elasticsearch запись с _index: website, c _type: blog и _id: 123
```shell
curl -XPUT 'localhost:9200/website/blog/123?pretty' -H 'Content-Type: application/json' -d'
{
"title": "My first blog entry",
"text":  "Just trying this out...",
"date":  "2014/01/01"
}
'
```
Как положить данные: https://www.elastic.co/guide/en/elasticsearch/guide/current/index-doc.html

Смотрю индексы в elasticsearch:
```shell
curl -XGET http://127.0.0.1:9200/_cat/indices?v

health status index               uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   website             nVpIJHIlTiy0NSEumG3adw 5   1   1          0            5.4kb      5.4kb
```
Если данных нет, то будут выведены заголовки таблицы, что-то вроде:
```shell
health status index uuid pri rep docs.count docs.deleted store.size pri.store.size
```
Смотрю records в elasticsearch в индексе website:
```shell
curl -XGET "http://127.0.0.1:9200/website/_search?pretty=true&scroll=1m&q=*:*"
```
Для логов из logstash:
```shell
curl -XGET "http://127.0.0.1:9200/my-logstash-index/_search?pretty=true&scroll=1m&q=*:*"
```
Подробнее:
- https://kb.objectrocket.com/elasticsearch/how-to-return-all-documents-from-an-index-in-elasticsearch
- https://stackoverflow.com/questions/8829468/elasticsearch-query-to-return-all-records

Удаляю все records в индексе logstash-2021.05.27:
```shell
curl -X DELETE "127.0.0.1:9200/logstash-2021.05.27?pretty"
```

### Logstash + input.file

Чтобы проверить работу вычитывания строк из файла logstash-input.txt, нужно по факту начала работы добавить строки:
```json
my string-1
{"message":"my string-2"}
{"message":"{\"inner\":\"my string-3\"}"}
{"message":"my check that logstash.filter verify type=json from logstash-input.txt", "type":"json"}
{"message":"my check that logstash.filter verify type=type_my_own from logstash-input.txt", "type":"type_my_own"}
```

### Logstash + input.beats

Когда в phpfpm.txt появляется строка "ttttttt", тогда в logstash.filter приходит сообщение (см. в логах filter received):
```json
{
    "event": {
        "source": "/tmp/phpfpm.txt",
        "offset": 54,
        "input": {
            "type": "log"
        },
        "prospector": {
            "type": "log"
        },
        "beat": {
            "hostname": "278558e78891",
            "name": "278558e78891",
            "version": "6.8.15"
        },
        "log": {
            "file": {
                "path": "/tmp/phpfpm.txt"
            }
        },
        "message": "ttttttt",
        "host": {
            "name": "278558e78891"
        },
        "@version": "1",
        "@timestamp": "2022-01-20T15:42:51.723Z",
        "tags": [
            "beats_input_codec_plain_applied"
        ]
    }
}
```
После того как фильтрация применилась, сообщение логируется в аутпуте (см. output received) + отправляется в Elasticsearch

Если что-то не получилось, попробуйте почитать https://habr.com/ru/post/165059/ и https://habr.com/ru/post/451264/

## Не проверено

Как протестировать tcp
```shell
echo '{"my":"jsontext"}' | nc localhost 31311
```
Как протестировать udp
```shell
echo -n '{"my":"jsontext"}' | nc -w0 -u 127.0.0.1 12201

echo -n '{"my":"jsontext"}' | nc -ul 127.0.0.1 -p 12201

echo -n '{ "version": "1.1", "host": "example.org", "short_message": "A short message", "level": 5, "_some_info": "foo", "type": "json" }' | nc -w0 -u 127.0.0.1 12201

echo -e '{"version": "1.1","host":"example.org","short_message":"Short message","full_message":"Backtrace here\n\nmore stuff","level":1,"_user_id":9001,"_some_info":"foo","_some_env_var":"bar"}\0' | nc -w 1 127.0.0.1 12201

echo -e '{"version": "1.1","host":"example.org","short_message":"Short message","full_message":"Backtrace here\n\nmore stuff","level":1,"_user_id":9001,"_some_info":"foo","_some_env_var":"bar"}\0' | nc -w -u 1 127.0.0.1 12201


echo '{"version": "1.1","host":"example.org","short_message":"A short message that helps you identify what is going on","full_message":"Backtrace here\n\nmore stuff","level":1,"_user_id":9001,"_some_info":"foo","_some_env_var":"bar"}' | gzip | nc -u -w 1 127.0.0.1 12201

curl -X POST -H 'Content-Type: application/json' -d '{ "version": "1.1", "host": "example.org", "short_message": "A short message", "level": 5, "_some_info": "foo" }' 'http://127.0.0.1:12201/gelf'

```
Подробнее:
- https://docs.graylog.org/en/4.0/pages/gelf.html
- https://docs.graylog.org/docs/gelf#example-payload
- https://qastack.ru/server/591758/send-echo-message-to-graylog2-via-gelf-tcp-12201-port
