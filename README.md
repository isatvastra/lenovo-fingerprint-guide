# lenovo-fingerprint-guide
# راهنمای نصب اثر انگشت Finger Print در لپ‌تاپ‌های لنوو

**English summary (Elan fingerprint reader on Lenovo laptops, Debian 13, TOD driver).** Some Elan match-on-chip fingerprint readers in Lenovo laptops are not handled correctly by the stock `elan` driver in Debian's `libfprint-2-2` (enrollment fails with `enroll-unknown-error` or `Timeout was reached`). This guide (written in Persian) describes one workaround: building the TOD (Touch OEM Driver) fork of libfprint and loading Lenovo's proprietary Elan plugin. It was tested on Debian 13 (x86_64) only, replaces system library files, and loads a closed-source binary into a root-run service. Read the warnings section first. No compatibility is guaranteed with any model or software version, and you alone are responsible for enabling fingerprint authentication on your system and for any consequences. Commands are in code blocks and can be copied as-is.

<div dir="rtl">

آخرین به‌روزرسانی: 2026-10-15

## هشدارهای مهم (پیش از شروع بخوانید)

این راهنما نصب رسمی و پشتیبانی‌شده نیست و تجربهٔ یک کاربر روی یک سیستم است. پیش از اجرای دستورها به این موارد توجه کنید:

۱. مرحلهٔ ۷ فایل‌های کتابخانهٔ سیستمی را با symlink جایگزین می‌کند. پیش از آن حتماً نسخهٔ پشتیبان بگیرید (در همان مرحله آمده است) و روش بازگشت را در بخش «بازگرداندن تغییرات» بخوانید.

۲. پلاگین اختصاصی لنوو یک باینری بستهٔ (closed-source) است و داخل سرویس fprintd، که با دسترسی root اجرا می‌شود، بارگذاری می‌شود. آن را فقط از منبع رسمی لنوو دریافت کنید و قابلیت اعتماد منبع را خودتان بسنجید. این فایل را در مخزن عمومی بازنشر نکنید.

۳. فعال‌کردن اثر انگشت برای `sudo` و صفحهٔ قفل، امنیت را نسبت به رمز عبور کاهش می‌دهد؛ اثر انگشت را نمی‌توان تغییر داد و جعل آن، هرچند دشوار، ممکن است. اگر مدل تهدید شما سخت‌گیرانه است، آن را برای `sudo` فعال نکنید.

۴. هیچ ضمانتی دربارهٔ سازگاری با مدل‌ها، نسخه‌های سیستم‌عامل یا نسخه‌های نرم‌افزارهای دیگر داده نمی‌شود. این راهنما فقط روی یک سیستم آزمایش شده است. درایور لنوو که در مرحلهٔ ۳ معرفی شده، برای سری‌های دیگری از لپ‌تاپ‌ها (ThinkPad E14/E15) و برای Ubuntu 22.04 منتشر شده است و کارکرد آن روی هر مدل دیگر باید به‌طور مستقل آزمایش شود.

۵. مسئولیت فعال‌سازی اثر انگشت روی سیستم شما، و هر پیامد آن (از جمله خطاهای نصب، ناسازگاری، از دست رفتن دسترسی یا کاهش امنیت)، کاملاً با خود کاربر است.

## مشخصات محیط آزمایش

| مورد | مقدار |
|---|---|
| لپ‌تاپ | Lenovo (مدل دقیق ذکر نشده است) |
| سیستم‌عامل | Debian 13، معماری x86_64 |
| حسگر | Elan، از نوع match-on-chip (تطبیق روی خود تراشه انجام می‌شود، نه توسط هاست) |
| شناسهٔ USB | `<VID:PID>` (با مرحلهٔ ۱ برای دستگاه خودتان پیدا کنید) |

نسخه‌های دیگر دبیان یا توزیع‌های دیگر آزمایش نشده‌اند.

## چرا راه‌حل ساده کار نکرد

درایور متن‌باز `elan` که در بستهٔ `libfprint-2-2` موجود در مخازن دبیان قرار دارد، روی این حسگر خاص به‌درستی کار نکرد. تلاش برای enroll با این درایور در آزمایش من با خطای `enroll-unknown-error` یا `Timeout was reached` پایان یافت. مشکلات مشابهی در جامعهٔ لینوکس گزارش شده است (برای نمونه تیکت شمارهٔ ۳۹ در مخزن `iafilatov/libfprint`)، اما پیش از اتکا به آن، وضعیت فعلی گزارش و هماهنگی مدل حسگر خود را خودتان بررسی کنید.

راه‌حلی که برای من جواب داد، استفاده از معماری **TOD (Touch OEM Driver)** در libfprint بود: نسخه‌ای فورک‌شده از libfprint که اجازه می‌دهد باینری اختصاصی (proprietary blob) شرکت Elan، که در بستهٔ درایور رسمی لنوو منتشر شده است، به‌عنوان پلاگین بارگذاری شود.

---

## مرحلهٔ ۱: شناسایی سخت‌افزار

```bash
lsusb | grep -iE "synaptics|goodix|validity|elan"
```

خروجی مورد انتظار به این شکل است (مقادیر در این نمونه جایگزین شده‌اند):

```
Bus <BUS> Device <DEV>: ID <VID:PID> Elan ...
```

شناسهٔ `<VID:PID>` دستگاه خود را یادداشت کنید؛ در مراحل بعد به آن نیاز دارید.

## مرحلهٔ ۲: نصب بستهٔ پایه fprintd

```bash
sudo apt install fprintd libpam-fprintd
fprintd-list $USER
```

اگر پیام `User has no fingers enrolled` را دیدید، یعنی دستگاه شناسایی شده و آماده است.

## مرحلهٔ ۳: دانلود درایور رسمی لنوو

بستهٔ درایور را از صفحهٔ رسمی لنوو دانلود و اکسترکت کنید:

```
https://support.lenovo.com/us/en/downloads/ds560939-elan-fingerprint-driver-for-ubuntu-2204-thinkpad-e14-gen-4-e15-gen-4
```

این صفحه برای ThinkPad E14/E15 Gen 4 و Ubuntu 22.04 است؛ پیش از دانلود بررسی کنید که هنوز در دسترس است و مدل شما را پوشش می‌دهد. فایل مورد نیاز داخل بسته `libfprint-2-tod1-elan.so` است. مسیر آن را با این دستور پیدا کنید:

```bash
find ~/Downloads -iname "libfprint-2-tod1-elan.so"
```

## مرحلهٔ ۴: نصب ابزارهای کامپایل

```bash
sudo apt install meson ninja-build gcc g++ libglib2.0-dev libgusb-dev \
  libgirepository1.0-dev libnss3-dev libgudev-1.0-dev gtk-doc-tools libudev-dev
```

## مرحلهٔ ۵: کلون و بیلد نسخهٔ TOD از libfprint

```bash
cd ~
git clone https://gitlab.freedesktop.org/3v1n0/libfprint.git
cd libfprint
git tag -l '*tod1*'
```

از فهرست تگ‌ها، یک نسخهٔ `+tod1` را انتخاب کنید. در آزمایش من تگ `v1.94.9+tod1` بود؛ اگر در فهرست شما وجود نداشت، نزدیک‌ترین نسخهٔ موجود را به‌جای آن بگذارید. نسخه‌های دیگر آزمایش نشده‌اند.

```bash
git checkout tags/v1.94.9+tod1
meson setup builddir
cd builddir
meson compile
```

### مشکل احتمالی: خطای `Dependency "udev" not found`

در Debian 13 ممکن است فایل `udev.pc` مربوط به pkg-config همراه `libudev-dev` نصب نشود. ابتدا نصب بستهٔ زیر را امتحان کنید؛ ممکن است فایل مورد نیاز را فراهم کند (این روش را نویسنده آزمایش نکرده است، چون خودش از ساخت دستی فایل استفاده کرده است):

```bash
sudo apt install systemd-dev
pkg-config --exists udev && echo "FOUND" || echo "NOT FOUND"
```

اگر هنوز `NOT FOUND` دیدید، فایل را دستی بسازید. نسخه را از خروجی pkg-config برمی‌دارد تا به نسخهٔ سیستم شما وابسته باشد:

```bash
mkdir -p ~/local-pkgconfig

cat > ~/local-pkgconfig/udev.pc << EOF
prefix=/usr
exec_prefix=\${prefix}
udevdir=\${exec_prefix}/lib/udev
udevrulesdir=\${udevdir}/rules.d

Name: udev
Description: udev
Version: $(pkg-config --modversion libudev)
Libs:
Cflags:
EOF

export PKG_CONFIG_PATH=~/local-pkgconfig:$PKG_CONFIG_PATH
pkg-config --exists udev && echo "FOUND" || echo "NOT FOUND"
```

**نکته:** این `export` فقط در همان نشست ترمینال معتبر است. اگر بعداً نیاز به بیلد مجدد شد، باید دوباره این خط را اجرا کنید یا آن را به `~/.bashrc` اضافه کنید.

بعد از رفع این مشکل، دوباره از پوشهٔ `~/libfprint` اجرا کنید:

```bash
cd ~/libfprint
rm -rf builddir
meson setup builddir
cd builddir
meson compile
```

خروجی موفق باید در انتها چیزی شبیه به `Linking target` نشان دهد و در فهرست Drivers نام `elanmoc` و `elan` دیده شود.

## مرحلهٔ ۶: نصب کتابخانهٔ کامپایل‌شده

**حتماً از داخل پوشهٔ `~/libfprint/builddir` اجرا شود**، وگرنه خطای `Install data not found` می‌دهد:

```bash
cd ~/libfprint/builddir
sudo meson install
```

فایل‌های کلیدی که نصب می‌شوند:

```
/usr/local/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0
/usr/local/lib/x86_64-linux-gnu/libfprint-2-tod.so.1
```

## مرحلهٔ ۷: جایگزینی کتابخانهٔ سیستمی با نسخهٔ تازه‌ساخته‌شده

این مرحله فایل‌های متعلق به بستهٔ رسمی را تغییر می‌دهد. ابتدا از آن‌ها نسخهٔ پشتیبان بگیرید:

```bash
mkdir -p ~/libfprint-backup
for f in libfprint-2.so.2.0.0 libfprint-2-tod.so.1; do
  [ -e /usr/lib/x86_64-linux-gnu/$f ] && sudo cp -a /usr/lib/x86_64-linux-gnu/$f ~/libfprint-backup/
done
ls -l ~/libfprint-backup
```

سپس symlinkها را بسازید:

```bash
sudo ln -sf /usr/local/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0 \
  /usr/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0

sudo ln -sf /usr/local/lib/x86_64-linux-gnu/libfprint-2-tod.so.1 \
  /usr/lib/x86_64-linux-gnu/libfprint-2-tod.so.1
```

## مرحلهٔ ۸: نصب پلاگین اختصاصی Elan (فایل دانلودشده از لنوو)

```bash
sudo mkdir -p /usr/local/lib/x86_64-linux-gnu/libfprint-2/tod-1
sudo cp ~/Downloads/libfprint-2-tod1-elan.so \
  /usr/local/lib/x86_64-linux-gnu/libfprint-2/tod-1/
sudo chown root:root /usr/local/lib/x86_64-linux-gnu/libfprint-2/tod-1/libfprint-2-tod1-elan.so
sudo chmod 644 /usr/local/lib/x86_64-linux-gnu/libfprint-2/tod-1/libfprint-2-tod1-elan.so
```

تعیین مالک root و دسترسی 644 مهم است، چون این فایل در سرویسی با دسترسی root بارگذاری می‌شود و نباید برای کاربر عادی قابل نوشتن باشد.

## مرحلهٔ ۹: بازخوانی کش لینکر و ری‌استارت سرویس

```bash
sudo ldconfig
sudo systemctl restart fprintd.service
```

## مرحلهٔ ۱۰: ثبت اثر انگشت

```bash
fprintd-enroll -f right-index-finger $USER
```

باید چندین بار (معمولاً ۱۲ مرحله) پیام `Enroll result: enroll-stage-passed` را ببینید و در پایان:

```
Enroll result: enroll-completed
```

توصیه می‌شود انگشت دوم را هم به‌عنوان پشتیبان ثبت کنید:

```bash
fprintd-enroll -f left-index-finger $USER
```

## مرحلهٔ ۱۱: تست صحت (Verify)

```bash
fprintd-verify $USER
```

انگشت را روی سنسور بگذارید؛ باید پیام `verify-match (done)` نمایش داده شود.

برای تست انگشت خاص:

```bash
fprintd-verify -f right-index-finger $USER
```

برای دیدن فهرست انگشت‌های ثبت‌شده:

```bash
fprintd-list $USER
```

## مرحلهٔ ۱۲: فعال‌سازی در PAM (برای sudo، صفحهٔ قفل، ورود گرافیکی)

```bash
sudo pam-auth-update
```

از منوی تعاملی، گزینهٔ مربوط به fingerprint authentication را فعال کنید. دربارهٔ پیامدهای امنیتی این کار، به هشدار شمارهٔ ۳ در ابتدای راهنما مراجعه کنید.

### تست اولیهٔ ایمن

پیش از تست روی صفحهٔ قفل یا صفحهٔ ورود گرافیکی (که در صورت خطا ممکن است شما را از سیستم قفل کند)، اول در ترمینال با یک دستور بی‌خطر امتحان کنید:

```bash
sudo ls
```

باید به‌جای درخواست رمز عبور، پیام درخواست اسکن اثر انگشت نمایش داده شود.

**نکتهٔ امنیتی مهم:** پیش از قفل‌کردن صفحه برای اولین‌بار، حتماً یک ترمینال یا نشست SSH دیگر باز نگه دارید تا در صورت بروز مشکل، امکان بازگشایی سیستم را داشته باشید.

---

## بازگرداندن تغییرات

اگر بخواهید به حالت اولیه برگردید:

```bash
sudo pam-auth-update   # گزینهٔ fingerprint را غیرفعال کنید
sudo rm -f /usr/local/lib/x86_64-linux-gnu/libfprint-2/tod-1/libfprint-2-tod1-elan.so
sudo apt install --reinstall libfprint-2-2
```

اگر در مرحلهٔ ۷ فایل `libfprint-2-tod.so.1` از قبل در `~/libfprint-backup` نبود، یعنی symlink آن را شما ساخته‌اید و می‌توانید آن را حذف کنید:

```bash
sudo rm -f /usr/lib/x86_64-linux-gnu/libfprint-2-tod.so.1
sudo ldconfig
sudo systemctl restart fprintd.service
```

---

## بخش عیب‌یابی (Troubleshooting)

### خطای `EnrollStart failed: Timeout was reached`

معمولاً به‌خاطر مدیریت توان USB (autosuspend) یا قفل‌ماندن device handle در سرویس fprintd است.

```bash
sudo systemctl restart fprintd.service
sudo dmesg | tail -n 40
```

برای غیرفعال‌کردن autosuspend، ابتدا مسیر دستگاه خود را در sysfs پیدا کنید (در دستور زیر `<VID>` بخش اول شناسهٔ USB شماست):

```bash
grep -l "<VID>" /sys/bus/usb/devices/*/idVendor
```

سپس در مسیر پیداشده (که شکل `/sys/bus/usb/devices/<PORT>/` دارد) اجرا کنید:

```bash
echo on | sudo tee /sys/bus/usb/devices/<PORT>/power/control
```

اگر مشکل ادامه داشت:

```bash
sudo systemctl stop fprintd.service
sudo pkill -f fprintd
sudo systemctl start fprintd.service
```

### خطای `Not Authorized: net.reactivated.fprint.device.enroll` (مشکل Polkit)

اگر در `journalctl -u fprintd` این خط دیده شود، مشکل از سیاست polkit است، نه از سخت‌افزار یا درایور.

**هشدار:** قانون زیر همهٔ عملیات fprintd را برای آن کاربر بدون هیچ پرسشی مجاز می‌کند. فقط برای تست و بسیار کوتاه‌مدت استفاده کنید و بلافاصله پس از ثبت اثر انگشت آن را حذف کنید. اگر مشکل در یک نشست راه‌دور رخ می‌دهد، ابتدا بررسی کنید که آیا مشکل از همان نشست است و نه از قانون پیش‌فرض.

```bash
sudo nano /etc/polkit-1/rules.d/45-fprintd.rules
```

محتوا (به‌جای `YOUR_USERNAME` نام کاربری خود را بنویسید):

```javascript
polkit.addRule(function(action, subject) {
    if (action.id.indexOf("net.reactivated.fprint") == 0 && subject.user == "YOUR_USERNAME") {
        return polkit.Result.YES;
    }
});
```

سپس:

```bash
sudo systemctl restart polkit
```

و پس از اتمام کار، حذف قانون:

```bash
sudo rm /etc/polkit-1/rules.d/45-fprintd.rules
sudo systemctl restart polkit
```

### خطای `enroll-unknown-error`

اگر با درایور استاندارد `elan` رخ دهد، ممکن است نشانهٔ عدم پشتیبانی این مدل حسگر توسط آن درایور باشد؛ راه‌حل در آزمایش من، مسیر کامل نصب TOD در همین سند بود (مراحل ۳ تا ۹).

### مشکل بعد از `apt upgrade`

اگر بعد از یک آپدیت سیستمی بستهٔ رسمی `libfprint-2-2` دوباره جایگزین شود و اثر انگشت از کار افتاد و دوباره `enroll-unknown-error` گرفتید، کافی است دو symlink مرحلهٔ ۷ را دوباره بسازید (نیازی به تکرار پشتیبان‌گیری نیست):

```bash
sudo ln -sf /usr/local/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0 \
  /usr/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0

sudo ln -sf /usr/local/lib/x86_64-linux-gnu/libfprint-2-tod.so.1 \
  /usr/lib/x86_64-linux-gnu/libfprint-2-tod.so.1

sudo ldconfig
sudo systemctl restart fprintd.service
```

توصیه می‌شود پیش از هر `apt upgrade`، بررسی کنید که آیا `libfprint-2-2` در فهرست بسته‌های قابل‌ارتقا هست یا نه:

```bash
apt list --upgradable | grep libfprint
```

---

## خلاصهٔ فایل‌ها و مسیرهای کلیدی

| مورد | مسیر |
|---|---|
| سورس libfprint (نسخهٔ TOD) | `~/libfprint` |
| کتابخانهٔ اصلی نصب‌شده | `/usr/local/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0` |
| کتابخانهٔ TOD | `/usr/local/lib/x86_64-linux-gnu/libfprint-2-tod.so.1` |
| پلاگین اختصاصی Elan | `/usr/local/lib/x86_64-linux-gnu/libfprint-2/tod-1/libfprint-2-tod1-elan.so` |
| پشتیبان کتابخانه‌های سیستمی | `~/libfprint-backup` |
| فایل pkg-config دستی برای udev (در صورت نیاز) | `~/local-pkgconfig/udev.pc` |
| قانون polkit سفارشی (فقط برای تست) | `/etc/polkit-1/rules.d/45-fprintd.rules` |

## مجوز

متن این راهنما تحت مجوز [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) منتشر می‌شود. libfprint تحت مجوز خودش (LGPL-2.1 یا بالاتر) و پلاگین اختصاصی Elan تحت مجوز لنوو است و در این مخزن توزیع نمی‌شود. این راهنما «همان‌گونه که هست» و بدون هیچ‌گونه ضمانت صریح یا ضمنی، از جمله ضمانت سازگاری با دستگاه یا نسخهٔ خاصی، ارائه می‌شود. مسئولیت اجرای دستورها و فعال‌سازی اثر انگشت روی سیستم، و هر پیامد آن، با خود کاربر است.

</div>

