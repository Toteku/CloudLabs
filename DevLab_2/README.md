# Отчет Лаб_2 
## Цель работы
* Написать “плохой” Dockerfile, в котором есть не менее трех “bad practices” по написанию докерфайлов
* Написать “хороший” Dockerfile, в котором эти плохие практики исправлены
* Описать, в чем заключаются изменения 
## Ход работы 
Создали "плохой" Dockerfile 
```dockerfile

FROM ubuntu:latest

RUN apt-get update && apt-get install -y golang

COPY . /app

WORKDIR /app

RUN go get ./...

CMD go run app.go
```
"Хороший" 

```dockerfile

FROM golang:1.20

COPY . /app

WORKDIR /app

RUN go mod init app && go mod tidy

RUN go build -o app .

CMD ["./app"]
```


## В чем различие?
* __FROM ubuntu:latest__ - ооо..даааа.. Та самая надоедливая надпись, которая выскакивает при каждом запуске. Проблема в том, что мы явно не указываев версию ubuntu, из-за чего могут быть изменения в выстроенных зависимостях. Таким образом, сборка обернется сбоем. __FROM golang:1.20__ явно обращается к версии, что является good_practice
* __RUN apt-get update && apt-get install -y golang__ - скачиваем golang, но не указываем версию. В исправленной версии данная строчка не нужна, так как у нас уже используется образ Go.
* __RUN go get ./...__ - попытка установить зависимости, что является bad_practice. Вместо этого используем __go mod__ 

Приступим к Compose файлам. 
"Плохой"

```yaml
version: '3'
services:
  app:
    image: myapp:latest
    build: .
    ports:
      - "80:80"
    restart: always

```

"Хороший"
```yaml
version: '3.8'  

services:
  app:
    image: myapp:latest
    build:
      context: .
      dockerfile: Dockerfile 
    ports:
      - "8080:8080"  
    restart: unless-stopped 
    environment:              
      - ENV=production
      - DEBUG=false  
    volumes:                  
      - ./app:/app:ro  
      - app_data:/data  
    depends_on: 
      - db
    networks:  
      - app-net

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: strong_password
    volumes:
      - db_data:/var/lib/mysql  
    networks:
      - app-net

networks:
  app-net:
    driver: bridge

volumes:
  app_data: {}
  db_data: {}  


```

## Больше текста? - Да, но правильного
* Указывается явный путь к Dockerfile 
* Убираем always, что предотвращает бесконечного цикла. Вместо этого используем unless-stopped, который не даст перезапускать все снова, и снова, и снова, и......
* В исправленной версии мы добавили хранение данных вне контейнера. Данные, созданные или измененные внутри контейнера, будут потеряны, если контейнер будет удален или перезапущен
* __depends_on__ гарантирует, что контейнер приложения не будет запущен до того, как контейнер базы данных будет готов



