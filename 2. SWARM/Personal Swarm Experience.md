---
tags:
  - docker
  - automation
  - swarm
up: "[[1. SWARM Concepts]]"
date: 2024-05-17
---
## multi server
برای در اختیار داشتن چندین node و تست سوارم سعی کردم چندین WSL با لینوکس های مختلف روی کامپیوتر شخصی نصب کنم اما علی رغم متفاوت بودن ورژن لینوکس ها آنها ایزوله نبودند و با ایجاد تغییر در هر کدام از آنها، تمامی لینوکس ها تغییر می کردند.
## Docker Swarm App Assignment
برای انجام assignment اول در بخش [[4. Overlay Multi-Host Networking]] با توجه به اینکه صرفا از یک نٌد و کانفیگ docker desktop استفاده می کردم، تمامی سرویس های مورد نظر را روی همین یک نٌد راه اندازی کردم.
[[4. Overlay Multi-Host Networking#docker swarm app assignment]]
> [!idea] نکته جالب استفاده از `docker service`  به جای `docker run command` در این است که در سرویس، داکر مسئولیت برقرار بودن کانتینر ها را به عهده میگیرد. پس از خاموش و روشن کردن سیستم و اجرای مجدد docker desktop سرویس ها مجددا راه اندازی و فعال شدند. و این در نوع خودش بسیار جالب بود.
> داکر سرویس به این صورت کار میکند که اگر برنامه یا بخشی از کار دچار اختلال شود، آن را از نو راه اندازی میکند و آن قدر این فرایند را تکرار میکند که سرویس به درستی پاسخگو باشد.

## Docker Stack Secret Assignment
در این  assignment که در بخش [[6. Secrets Storage]] یک compose-file را آماده کردیم و باید secret را از قبل تعریف کرده و در compose-file فراخوانی کنیم.
برای فراخوانی کافیست در مقابل کلید `external` مقدار `true` را وارد کنیم. به شرط آنکه secret با نامی که در این فایل معرفی میکنیم (در اینجا `mydbpass`) را از قبل ایجاد کرده باشیم.
فایل YAML را به این صورت آماده کردم:
```yml
services:

  drupal:
    image: mehdi/drupal
    ports:
      - "8080:80"
    volumes:
      - drupal-modules:/var/www/html/modules
      - drupal-profiles:/var/www/html/profiles       
      - drupal-sites:/var/www/html/sites      
      - drupal-themes:/var/www/html/themes
 
  postgres:
    image: postgres
    secrets:
      - mydbpass
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/mydbpass
    volumes:
      - drupal-data:/var/lib/postgresql/data

volumes:
  drupal-data:
  drupal-modules:
  drupal-profiles:
  drupal-sites:
  drupal-themes:

secrets:
  mydbpass:
    external: true
```

اگر secret مورد نظر را تعریف نکرده باشیم در هنگام راه اندازی stack با ارور مواجه می شویم:
```bash
Linux@:)bash$: docker stack deploy -c docker-compose.yml myDrupal
Creating network myDrupal_default
service postgres: secret not found: mydbpass

Linux@:)bash$: docker service ls
ID        NAME      MODE      REPLICAS   IMAGE     PORTS
Linux@:)bash$: docker stack ls
NAME      SERVICES
Linux@:)bash$:
```

### نکته آموزشی
با کانفیگ های بالا سرویس ها راه اندازی می شدند اما سرویس `postgres` نمیتوانست تسک و کانتینر مورد نیازش را ایجاد کند و با بررسی لاگ متوجه شدم که آدرس معرفی شده در کلید `environment` با ارور `No such file or directory` مواجه شده است:
```bash
Linux@:)bash$: docker service logs myDrupal_postgres
myDrupal_postgres.1.tmhhpnnw8qd7@docker-desktop    | /usr/local/bin/docker-entrypoint.sh: line 21:  /run/secrets/mydbpass: No such file or directory
```
متوجه شدم یک فاصله در ابتدای این آدرس قرار دارد که نباید داشته باشد. یعنی اشتباها به این صورت تعریف شده:
```yml
    environment:
      - POSTGRES_PASSWORD_FILE= /run/secrets/mydbpass
```