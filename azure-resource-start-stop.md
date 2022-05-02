# Azure resource를 start / stop 하는 API 존재여부


## Virtual Machine

있음. (내용은 생략)


## Databases

가능함을 확인한 데이터베이스들:
- Azure SQL Virtual Machine
  + Virtual Machine 에 SQL Server를 설치하고 쓰는 방식인 듯
  + 즉 같은 이름의 Virtual Machine이 존재하며 Virtual Machine을 껐다켜는 방식으로 제어 가능.
- Azure Database for MySQL
- Azure Database for MySQL flexible server
- Azure Database for MariaDB
- Azure Database for Postgresql flexible server


지원안되는 데이터베이스들 (2022년 3월 현재 확인) :
- Azure SQL managed instances
- Azure Database for PostgreSQL servers - single server
- Azure Cosmos DB
- SQL databases (SQL servers 와 짝을 이루어 구성하는 DB인데 상세한 실체는 잘 모름)


파악되지 않은 데이터베이스
- Document DB 들 일체


