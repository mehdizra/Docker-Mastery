از وبلاگ [احمد رفیعی](https://virgool.io/@rafiee/%D8%AF%D8%B1-%D9%85%D8%B3%DB%8C%D8%B1-%D8%AF%D9%88%D8%A7%D9%BE%D8%B3-%D8%A7%DB%8C%D9%86%D8%A8%D8%A7%D8%B1-%D8%AF%D9%88%D8%B1-%D9%88-%D8%A8%D8%B1%DB%8C-%D9%87%D8%A7%DB%8C-%D8%AF%D8%A7%DA%A9%D8%B1-%D9%82%D8%B3%D9%85%D8%AA-%D9%87%D9%81%D8%AA%D9%85-cnal7vujndki)
### **چطوری سرویس داکرمون رو کانفیگ کنیم؟**

سرویس داکر همانند سرویس‌های دیگه داخل لینوکس قابل سفارشی‌سازی و کانفیگ است. توی داکر میتونیم مواردی از قبیل گزینه های زیر رو کانفیگ کنیم.

- Change Root Directory
- Set Mirror Registry
- Change Docker Subnet Range
- Change Docker Port Bind
- Change Docker Default Driver

برای کانفیگ کردن داکر سه تا راه وجود داره. راه اول از طریق اضافه کردن فایل کانفیگ به مسیر زیر هست.

**/etc/systemd/system/docker.service.d/override.conf**

که مثلا فایلی رو با نام override.conf توی این مسیر میذاریم که داخلش با فرمتی شبیه سرویس فایل هایی که برای سرویس های لینوکسی که با systemd اجراشون میکنیم، کانفیگ هامون رو مینویسیم و بعد از آن باید daemon رو ریلود کنیم و داکر رو ری‌استارت کنیم. این روش دقیقا روش کانفیگ با استفاده از systemd فایل هست که برای همه‌ی سرویس‌های لینوکس کاربرد دارد.

راه دوم برای کانفیگ کردن داکر هم از طریق تغییر کانفیگ daemon که در قالب json توی فایل **etc/docker/daemon.json/** است قرار می‌دهیم. معمولا این فایل وجود نداره و باید ایجادش کنیم. برای اعمال کانفیگ‌هامون فقط لازمه تو این روش سرویس داکر رو ریستارت کنیم.

راه سوم هم استفاده از وریبل‌های محلی است که با استفاده از آنها می‌تونیم سرویس داکر رو کانفیگ کنیم.