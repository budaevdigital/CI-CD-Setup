## Подключение к удалённому серверу

Для того чтобы запустить обновлённый проект на боевом сервере, сервер должен:

-   скачать с Docker Hub обновлённый образ проекта;
-   остановить и удалить запущенный контейнер с проектом;
-   запустить контейнер из обновлённого образа.

При выполнении этих задач «вручную» разработчик соединяется по ssh с сервером и отправляет на сервер команды `docker pull ...`, `docker stop ...` и `docker run ...`. При работе с GitHub Actions эти команды будут выполняться из инструкции _workflow_.

Подключение к удалённому серверу — востребованная задача на GitHub Actions, и для автоматизации этого процесса есть специальный _action_ для выполнения ssh-команд.

Ваш боевой сервер получит команды не от вас, а от «незнакомого» сервера GitHub Actions_._ Чтобы боевой сервер позволил установить соединение, добавьте на сервер GitHub Actions ssh-ключ для подключения к боевому серверу.

Чтобы никто не мог получить доступ к вашему приватному ключу на GitHub Actions, сохраните ключ в Secrets.

1  Скопируйте приватный ключ с компьютера, имеющего доступ к боевому серверу:

```bash
cat ~/.ssh/id_rsa
```

2  На GitHub Actions сохраните ключ в **Secrets**, в переменную `SSH_KEY`.

Помимо ключа для доступа к серверу нужны _username_ и адрес хоста. Тоже сохраните их в **Secrets**:

-   в _secret_-переменную `USER` сохраните имя пользователя для подключения к серверу;
-   в _secret_-переменную `HOST` сохраните IP-адрес вашего сервера;
-   если при создании ssh-ключа вы использовали фразу-пароль, то сохраните её в _secret_-переменную `PASSPHRASE`.

Добавьте в _workflow_ ещё один _job_ — в нём будут инструкции для скачивания на боевой сервер обновлённого образа и запуска контейнера:

```yaml
deploy:
    runs-on: ubuntu-latest
    needs: build_and_push_to_docker_hub
    steps:
    - name: executing remote ssh commands to deploy
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USER }}
        key: ${{ secrets.SSH_KEY }}
        passphrase: ${{ secrets.PASSPHRASE }} # Если ваш ssh-ключ защищён фразой-паролем
        script: |
          # Выполняет pull образа с DockerHub
          sudo docker pull <имя-пользователя>/<имя-репозитория>
          #остановка всех контейнеров
          sudo docker stop $(sudo docker ps -a -q)
          sudo docker run --rm -d -p 5000:5000 <имя-пользователя>/<имя-репозитория> 
``` 

По инструкции `docker run` из докер-образа будет собран и запущен контейнер на порте 5000 — порт указан в параметре `-p`. Параметр `-d` устанавливает, что контейнер будет запущен в фоновом режиме, а параметр `--rm` определяет, что при остановке контейнер будет автоматически удалён.


## Подготовка сервера

Остановите службу `nginx`:

 ```bash
sudo systemctl stop nginx
``` 

Установите `docker`:

```BASH
sudo apt install docker.io
``` 

Установите `docker-compose`, с этим вам поможет [официальная документация](https://docs.docker.com/compose/install/).
    
Скопируйте файлы _docker-compose.yaml_ и _nginx/default.conf_ из вашего проекта на сервер в _home/<ваш_username>/docker-compose.yaml_ и _home/<ваш_username>/nginx/default.conf_ соответственно.

> ! Эти файлы всегда копируются вручную. Обычно они настраиваются один раз и изменения в них вносятся крайне редко.

Добавьте в `Secrets GitHub Actions` переменные окружения для работы базы данных; отредактируйте инструкции _workflow_ для задачи `deploy`:

```YAML
deploy:
    runs-on: ubuntu-latest
    needs: build_and_push_to_docker_hub
    steps:
      - name: executing remote ssh commands to deploy
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USER }}
          key: ${{ secrets.SSH_KEY }}
          passphrase: ${{ secrets.PASSPHRASE }}
          script: |
            sudo docker-compose stop
            sudo docker-compose rm web
            touch .env
            echo DB_ENGINE=${{ secrets.DB_ENGINE }} >> .env
            echo DB_NAME=${{ secrets.DB_NAME }} >> .env
            echo POSTGRES_USER=${{ secrets.POSTGRES_USER }} >> .env
            echo POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }} >> .env
            echo DB_HOST=${{ secrets.DB_HOST }} >> .env
            echo DB_PORT=${{ secrets.DB_PORT }} >> .env
            sudo docker-compose up -d 
```

У вас уже должен быть настроен **docker-compose** для трёх контейнеров — `web`, `db` и `nginx`.

Проверьте настройки файла `docker-compose.yaml` он должен разворачивать контейнер `web`, используя образ, который вы создали на Docker Hub.

 В файле _settings.py_ укажите [значения по умолчанию](https://docs.python.org/3/library/os.html#os.getenv) для переменных из env-файла. Это нужно для того, чтобы тесты на платформе прошли успешно. Обратите внимание, что значения по умолчанию должны быть валидными.
    
Хороший пример:
    
```PYTHON
'ENGINE': os.getenv('DB_ENGINE', default='django.db.backends.postgresql')
```

Плохой пример — без значения по умолчанию:

```PYTHON
'ENGINE': os.getenv('DB_ENGINE')
```


Плохой пример — с невалидным значением по умолчанию:

```PYTHON
'ENGINE': os.getenv('DB_ENGINE', default='engine')
```

Добавьте в файл _README.md_ [бейдж](https://docs.github.com/en/free-pro-team@latest/actions/managing-workflow-runs/adding-a-workflow-status-badge), который будет показывать статус вашего _workflow_.





## Отправка отчёта

Остался финальный шаг. Вместо того чтобы отслеживать выполнение _workflow_ в GitHub Actions, можно настроить бота: пусть он отправляет оповещение об успешном завершении процесса.

У вас очень кстати есть готовый бот. Хватит ему бездельничать.

В маркетплейсе `GitHub Actions` есть [специальный скрипт для отправки уведомлений в Телеграм](https://github.com/marketplace/actions/telegram-message-notify).

Добавьте ещё один шаг в _workflow_:

```YAML
send_message:
  runs-on: ubuntu-latest
  needs: deploy
  steps:
  - name: send message
    uses: appleboy/telegram-action@master
    with:
      to: ${{ secrets.TELEGRAM_TO }}
      token: ${{ secrets.TELEGRAM_TOKEN }}
      message: ${{ github.workflow }} успешно выполнен! 
``` 

Зайдите в **Settings → Secrets** в вашем репозитории и добавьте ещё две переменные:

-   чтобы бот отправил сообщение именно вам, в переменной `TELEGRAM_TO` сохраните ID своего телеграм-аккаунта. Узнать свой ID можно у бота @_userinfobot_;
-   в переменной `TELEGRAM_TOKEN` сохраните токен вашего бота. Получить этот токен можно у бота @_BotFather_.

Если вы хотите отправлять сообщение в Slack или куда-то ещё — найдите нужный _action_ в маркетплейсе GitHub Actions и подключите его по инструкции.


