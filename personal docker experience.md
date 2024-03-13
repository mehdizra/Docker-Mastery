---
tags:
  - docker
  - linux
  - network
  - image
up: "[[1. Docker concept]]"
date: 2024-03-13
---
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
 mmzare@:)Linux~$: docker image build -t sushi2 .
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
curl [https://raw.githubusercontent.com/docker/docker-ce/master/components/cli/contrib/completion/bash/docker](https://raw.githubusercontent.com/docker/docker-ce/master/components/cli/contrib/completion/bash/docker) -o /etc/bash_completion.d/docker.sh
```

Logout and login again.