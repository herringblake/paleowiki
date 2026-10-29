# Docker Service Port Map

## _!!DEPRECATED!!_

This information is no longer valid now that we have kubernetes deployed.

Services running in docker on this host and the ports they are using

| Service Name | External Port | Internal Port | Notes |
|--------------|---------------|---------------|-------|
| Kiwix        |          8082 |        8080  | |
| Kafka        | 19092 | 19092 | |
| Zookeeper    | 2181 | 2181 | |
| schemaregistry | 18085 | 8085 | |
| kafkaexample-app | 15454 | 5454 |  |
| kafdrop | 19000 | 9000 |  |
| mariadb | 3306 | 3306 |  |
| jellyfin | 8096 | 8096  |  | 
| MongoDB | 27017 | 27017 |  |
| plantuml | 8881 | 8080 |  |
| moov.io | 8882 | 8080 | needs docker-compose file |
| postgres | 5432 | 5432 |  | 
| otterwiki | 8083 | 3000 |  |
