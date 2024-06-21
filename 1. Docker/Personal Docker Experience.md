---
tags:
  - docker
  - linux
  - network
  - image
up: "[[1. Docker concept]]"
date: 2024-03-13
---
### Connecting Docker Desktop Containers To The Internet
1. در شرایطی هستیم که آی‌پی آدرس ویندوز ساب سیستم با آی‌پی آدرس مودم اینترنت متفاوت هستند و همدیگرو نمیبینند.
![[Pasted image 20240301140612.png]]

2. آدرس داکر سابنت را در ستینگ داکر دسکتاپ به آدرس شبکه مودم اینترنت تغییر میدهیم:
![[Pasted image 20240301140739.png]]

3. میبینیم که آدرس کارت شبکه WSL لینوکس داخل ویندوز همچنان متفاوت شبکه مودم اینترنت است
![[Pasted image 20240301140952.png]]
4. با بررسی کانتینرهای در حال اجرا میبینیم که آنها هم شبکه متفاوتی از مودم اینترنت دارند:
![[Pasted image 20240301141253.png]]
با ورود به کانتینتر مربوطه و تغییر ریزالور در سیستم عامل دومین سرور درست رو به سیستم اضافه میکنیم.
اضافه کردن nameserver 8.8.8.8 به /etc/resolve.conf 
```bash
echo nameserver 8.8.8.8 > /etc/resolve.conf
```
## مشکل DNS در DockerDesktop
من از docker desktop روی ویندوز برای یادگیری docker استفاده میکنم و از طریق WSL2 ubuntu استفاده میکنم.
در حالی که شبکه این ماشین از مودم اینترنت آی‌پی 192.168.1.102 را دریافت کرده WSL اما آی‌پی 172.17.71.95 دارد. و با همین تنظیمات به اینترنت متصل است.
![[Pasted image 20240313112255.png]]
اما وقتی یک کانتینر رو run میکنم resolver در رنج آی پی مودم (یعنی 192.168.1.7) در /etc/resolv.conf قرار میگیرد که امکان اتصال به اینترنت با این تنظیمات وجود ندارد
![[Pasted image 20240313112709.png]]
بعد از اجرای کانتینر با اضافه کردن یک nameserver 8.8.8.8 به resolv.conf به اینترنت متصل می شوم
اما مشکل اینجاست که با اجرای کانتینتر هایی مثل node امکان تغییر ریزالور را ندارم و نمیتوانم از داخل کانتینر به اینترنت وصل شوم. 
از طرفی برای ساخت image دستور تغییر ریزاولر را نمیتوان در همه Dockerfile ها قرار داد، چرا که برخی از اونها مثل node این امکان رو ندارند که در داخل آنها DNS را تغییر داد.
```bash
 Linux@:)bash~$: docker image build -t sushi2 .
[+] Building 0.4s (6/11)                         docker:default
 => [internal] load build definition from Dockerfile                   0.0s
 => => transferring dockerfile: 304B                                   0.0s
 => [internal] load metadata for docker.io/library/node:6-alpine       0.0s
 => [internal] load .dockerignore                                      0.0s
 => => transferring context: 2B                                        0.0s
 => CACHED [1/7] FROM docker.io/library/node:6-alpine                  0.0s
 => [internal] load build context                                      0.1s
 => => transferring context: 458.42kB                                  0.0s
 => ERROR [2/7] RUN echo nameserver 8.8.8.8 > /etc/resolv.conf
```
بهترین حالت برای تنظیمات شبکه کانتینر ها روی docker desktop چیست؟
بهترین حالت این است که Docker Engine رو جوری تنظیم کنیم که DNS صحیح به کانتینر ها اختصاص بدهد.
برای اینکار لازم است به تنظیمات داکر در ادرس زیر برویم و یک تنظیم DNS اضافه کنیم:
```bash
"dns": ["192.168.1.1","8.8.8.8","8.8.4.4"]
```

| OS and configuration | File location |
|---|---|
|Linux, regular setup|`/etc/docker/daemon.json`|
|Linux, rootless mode|`~/.config/docker/daemon.json`|
|Windows|`C:\ProgramData\docker\config\daemon.json`|

اما مشکل اینجا بود که هیچ کدام از آدرس های بالا در سیستم من وجود نداشت.
در ویندوز فولدر docker نبود و به جای آن DockerDesktop که در آن`daemon.json` وجود نداشت.
و در WSL Ubuntu آدرس `/etc/docker` وجود نداشت.
با جستجو در تنظیمات DockerDesktop بخش تنظیمات فایل config را پیدا کردم و تنظیم  DNS را اضافه کردم:
![[Pasted image 20240313203703.png]]

## مشکل DNS در WSL
**در این مرحله سیستم دچار مشکل شد و متوجه نشدم از تغییرات من بوده یا از آپدیتی که از صبح آلارم نصب آن را اطلاع میداد**
بنابراین DockerDesktop و WSL را از روی سیستم حذف کرده و مجددا نصب کردم.
و این بار مشکل DNS روی WSL به وجود آمد که با هر بار ری استارت `resolv.config` تغییر میکرد که با جستجو به راهکار زیر رسیدم.
**1. Turn off generation of `/etc/resolv.conf`**
Using your Linux prompt, (I'm using Ubuntu), modify (or create) /etc/wsl.conf with the following content
```
[network]
generateResolvConf = false
```
(Apparently there's a bug in the current release where any trailing whitespace on these lines will trip things up.)

**2. Restart the WSL2 Virtual Machine**
Exit all of your Linux prompts and run the following Powershell command
```
wsl --shutdown
```

**3. Create a custom `/etc/resolv.conf`**
Open a new Linux prompt and cd to `/etc`
If `resolv.conf` is soft linked to another file, remove the link with
```
rm resolv.conf
```
Create a new `resolv.conf` with the following content
```
nameserver 1.1.1.1
```

**4. Restart the WSL2 Virtual Machine**
Same as step #2

**5. Start a new Linux prompt.**

## مشکل Autocomplete Docker
پس از حل مشکل DNS در WSL و نصب مجدد DockerDesktop در زمان کار با Docker متوجه شدم که autocomplete برای دستورات Docker کار نمیکند و کمتر کسی در فضای اینترنت توضیح درستی در مورد آن داده بود که [یکی از این توضیحات](https://ismailyenigul.medium.com/enable-docker-command-line-auto-completion-in-bash-on-centos-ubuntu-5f1ac999a8a6) کار کرد:

Install bash-completion package

on CentOS/RedHat
```
# yum -y install bash-completion
```

on Ubuntu
```
apt-get update  
apt-get install bash-completion -y
```

```
If you installed docker-ce-cli package, it already ships with bash-completion files, you don't need to run the following commands.# ==dpkg -L docker-ce-cli |grep completion==  
/usr/share/bash-completion  
/usr/share/bash-completion/completions  
/usr/share/bash-completion/completions/docker  
/usr/share/fish/vendor_completions.d  
/usr/share/fish/vendor_completions.d/docker.fish  
/usr/share/zsh/vendor-completions  
/usr/share/zsh/vendor-completions/_docker
```

Download bash completion file from [https://github.com/docker/docker-ce/blob/master/components/cli/contrib/completion/bash/docker](https://github.com/docker/docker-ce/blob/master/components/cli/contrib/completion/bash/docker) into /etc/bash_completion.d
```
curl https://raw.githubusercontent.com/docker/docker-ce/master/components/cli/contrib/completion/bash/docker -o /etc/bash_completion.d/docker.sh
```

Logout and login again.

## Named Volume experience 
در حین تست استفاده از -v برای نامگذاری volume در کانتینر از یک volume در دو کانتینر استفاده کردم. و جالب بود که با تغییر فایل از داخل هر کانتینر در کانتینر دوم هم تغییر مشاهده می شود.
این یعنی دو کانتینر میتوانند همزمان روی یک فایل کار کنند:
سوال: باید عوارض این موضوع را بررسی کنم؟
```bash
mmzare@:DLinux~$:)docker container run -d --name database -e POSTGRES_HOST_AUTH_METHOD=trust -v db:/var/lib/postgresql/data postgres
bcf90681854526648da3c283122ccee041537f41eb92cea8a12211ee86b8a551
mmzare@:DLinux~$:)docker container run -d --name database2 -e POSTGRES_HOST_AUTH_METHOD=trust -v db:/var/lib/postgresql/data postgres
44da0b807bb1add72f11c82336138b3bb44f9ba1d25dc94e06e64c0a9c03e0a7
mmzare@:DLinux~$:)docker exec -it database bash
root@bcf906818545:/# touch /var/lib/postgresql/data/test.txt
root@bcf906818545:/# exit
exit
mmzare@:DLinux~$:)docker exec -it database2 bash
root@44da0b807bb1:/# ls /var/lib/postgresql/data/test.txt
/var/lib/postgresql/data/test.txt
root@44da0b807bb1:/# echo hello >> /var/lib/postgresql/data/test.txt
root@44da0b807bb1:/# exit
exit
mmzare@:DLinux~$:)docker exec database cat /var/lib/postgresql/data/test.txt
hello
mmzare@:DLinux~$:)docker exec database2 cat /var/lib/postgresql/data/test.txt
hello
```

## Docker-compose Assignment 1
ابتدا فایل YAML را اینگونه نوشتم:
```bash
services:
  drupal:
    image: drupal
    ports:
      - '8080:80'
    volumes:
      - /var/www/html/modules
      - /var/www/html/profiles
      - /var/www/html/themes
  postgres:
    image: postgres
    environment:
      - POSTGRES_PASSWORD=123456c
```
که پس از اجرا ارور دریافت کردم
پس از بررسی متوجه شدم که volume ها را باید خارج از سرویس ها هم تعریف کنم:
```bash
services:
  drupal:
    image: drupal
    ports:
      - '8080:80'
    volumes:
      - drupal-modules:/var/www/html/modules
      - drupal-profiles:/var/www/html/profiles      
      - drupal-sites:/var/www/html/sites      
      - drupal-themes:/var/www/html/themes
  postgres:
    image: postgres
    environment:
      - POSTGRES_PASSWORD=123456
  
volumes:
  drupal-modules:
  drupal-profiles:
  drupal-sites:
  drupal-themes:
```
پس از اجرای فایل بالا به مرحله نصب Drupal می رسیم با توجه به اینکه اسم سرویس در Docker-compose به عنوان DNS استفاده می شود باید اسم سرویسی که برای دیتابیس انتخاب کردیم (در اینجا postgres) را در Database Name و Host قرار دهیم. علاوه بر آن چون یوزرنیم تعریف نکردیم دوباره باید از همان اسم سرویس استفاده کنیم.
![[Pasted image 20240416172056.png]]
## Docker-compose Assignment 2
فایل YAML را به این شکل نوشتم:
```yaml
services:
  drupal-demon:
    build:
      context: .
      dockerfile: Dockerfile
    image: custom-drupal
    ports:
    - '8080:80'

  postgres:
    image: postgres
    environment:
      - POSTGRES_PASSWORD=123456
    volumes:
      - drupal-data:/var/lib/postgresql/data
volumes:
  drupal-data:
```

فایل Dockerfile را به این شکل نوشتم:
```dockerfile
# create your custom drupal image here, based of official drupal
FROM mehdi/drupal:latest
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
WORKDIR /var/www/html/themes
RUN git clone --branch 8.x-4.x --single-branch --depth 1 https://git.drupalcode.org/project/bootstrap.git / 
&& chown -R www-data:www-data bootstrap
WORKDIR /var/www/html
```
اما متوجه شدم که ایمیج drupal که من از آن استفاده میکنم به جای `bash` از `sh` به عنوان SHELL استفاده میکند و دستورات من در حین اجرا دچار خطا می شوند. با جستجو به این راهکار رسیدم که دستورات را با دستور `bin/bash` تلفیق کنم:
```dockerfile
# create your custom drupal image here, based of official drupal
FROM mehdi/drupal:latest
RUN ["/bin/bash", "-c", "apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*"]
WORKDIR /var/www/html/themes
RUN ["/bin/bash", "-c", "git clone --branch 8.x-4.x --single-branch --depth 1 https://git.drupalcode.org/project/bootstrap.git && chown -R www-data:www-data bootstrap"]
WORKDIR /var/www/html
```

> [!hint]  نهایتا سایت بالا آمد اما تم مربوطه روی آن نصب نمیشد چون ورژنهای سازگار باهم نبودند
> 

### Assignment Answer:
```Dockerfile
FROM drupal:9
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    git \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /var/www/html/themes
RUN git clone --branch 8.x-4.x --single-branch --depth 1 https://git.drupalcode.org/project/bootstrap.git \
    && chown -R www-data:www-data bootstrap
WORKDIR /var/www/html
```

```Yaml
services:
  drupal:
    image: custom-drupal
    build: .
    ports:
      - "8080:80"
    volumes:
      - drupal-modules:/var/www/html/modules
      - drupal-profiles:/var/www/html/profiles      
      - drupal-sites:/var/www/html/sites      
      - drupal-themes:/var/www/html/themes
  postgres:
    image: postgres:14
    environment:
      - POSTGRES_PASSWORD=mypasswd
    volumes:
      - drupal-data:/var/lib/postgresql/data
volumes:
  drupal-data:
  drupal-modules:
  drupal-profiles:
  drupal-sites:
  drupal-themes:
```
**نکته**: کلید`build: .` به این معناست که داکر فایل را از فولدر جاری پیدا کن و ایمیج را بساز