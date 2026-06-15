# Enterprise Database Administration Runbook: Herd & DBngin Ecosystem

**Target Platforms:** MySQL & PostgreSQL (Windows + Laravel Herd Environment)
**Role Context:** Database Administrator (DBA) / Developer Operations Manual
**Classification:** Internal Technical Reference

---

## 1. Executive Summary & Architecture Overview

ဤ Local development environment သည် **Laravel Herd** ကို အပေါ့ပါးဆုံး visual PHP engine runtime အဖြစ်လည်းကောင်း၊ **DBngin** ကို micro-isolated database binary controller အဖြစ်လည်းကောင်း ပေါင်းစပ်အသုံးပြုထားသော စနစ်ဖြစ်ပါသည်။

ထို့အပြင် phpMyAdmin နှင့် Adminer တို့ကို Herd ၏ internal Nginx server layer တွင် `.test` domain များအဖြစ် virtual site များပြုလုပ်ကာ (ဥပမာ - `phpmyadmin.test`) Map လုပ်ထားပါသည်။ ဤစနစ်သည် host OS တွင် service ပွပွကြီး ဖြစ်မနေစေဘဲ ထိရောက်မြန်ဆန်သော database UI workspace ကို ရရှိစေပါသည်။

```text
+-----------------------------------------------------------------------+
|                       LARAVEL HERD LOCAL NETWORK                      |
|                           (Local .test domains)                       |
+-----------------------------------+-----------------------------------+
                                    |
          +-------------------------+-------------------------+
          |                                                   |
          ▼                                                   ▼
+----------------───+                               +----------------───+
|   MySQL Engine    |                               | PostgreSQL Engine |
|   (via DBngin)    |                               |   (via DBngin)    |
| Port: 3306 / 3307 |                               | Port: 5432 / 5433 |
+---------┬---------+                               +---------┬---------+
          ▲                                                   ▲
          │ (config.inc.php)                                  │ (Driver Mapping)
          │                                                   │
+---------┴---------+                               +---------┴---------+
|    phpMyAdmin     |                               |      Adminer      |
|  (Hosted in Herd) │                               |  (Hosted in Herd) │
| http://           |                               | http://           |
| phpmyadmin.test   |                               | adminer.test      |
+-------------------+                               +-------------------+

```

---

## 2. PART I: MySQL Administration via phpMyAdmin (Herd Setup)

### Step 1: Network Socket & Port Discovery

DBngin သည် system service conflict များကို ရှောင်ရှားရန်အတွက် default port `3306` မအားပါက နောက်ဆက်တွဲ port များ (`3307`, `3308` စသည်) ကို အလိုအလျောက် ပြောင်းလဲသတ်မှတ်ပေးတတ်ပါသည်။

1. **DBngin** control panel ကို ဖွင့်ပါ။
2. လက်ရှိ run နေသော MySQL engine ၏ **Port** နံပါတ်ကို မှတ်သားထားပါ (ဥပမာ- `3307`)။
3. Service အခြေအနေ **ON** ဖြစ်နေကြောင်း သေချာပါစေ။

### Step 2: Herd Paths & Virtual Site Deployment

1. Official [phpMyAdmin Downloads Portal](https://www.phpmyadmin.net/downloads/) မှ `phpMyAdmin-all-languages.zip` ကို ဒေါင်းလုဒ်ဆွဲပါ။
2. ၎င်းကို သင်၏ Herd paths အောက်ရှိ folder တစ်ခုအနေဖြင့် ဖြည်ချပါ (ဥပမာ- `C:\Users\YourUser\Herd\phpmyadmin\`)။
3. Laravel Herd သည် ၎င်း folder အမည်ကိုယူ၍ `[http://phpmyadmin.test](http://phpmyadmin.test)` ဆိုပြီး local domain အလိုအလျောက် သတ်မှတ်ပေးမည် ဖြစ်သည်။

### Step 3: Configuration Storage Parameter Injection

phpMyAdmin ၏ default setup ဖိုင်တွင် port သတ်မှတ်ချက် ကနဦး၌ ပါဝင်မလာပါ။ ထို့ကြောင့် အောက်ပါအတိုင်း manual ဖြည့်စွက်ပေးရန် လိုအပ်သည်။

1. `phpmyadmin` folder ထဲရှိ `config.sample.inc.php` ကို **`config.inc.php`** ဟု အမည်ပြောင်းလဲပါ။
2. ဖိုင်ကို ဖွင့်ပြီး Server parameters အပိုင်းကို အောက်ပါအတိုင်း ပြင်ဆင်/ဖြည့်စွက်ပါ-

```php
/* Server parameters */
$cfg['Servers'][$i]['host'] = '127.0.0.1';     // Windows named pipes conflict မဖြစ်စေရန် localhost အစား ပြောင်းလဲပါ
$cfg['Servers'][$i]['port'] = '3307';          // သင့် DBngin ၏ တိကျသော Port နံပါတ်ကို ဤနေရာတွင် ထည့်ပါ
$cfg['Servers'][$i]['compress'] = false;
$cfg['Servers'][$i]['AllowNoPassword'] = true; // DBngin ၏ default password အလွတ်ပေါ်လစီအတွက် true ပြောင်းပါ

```

### Step 4: Authentication & Login Workflow

1. Browser တွင် `[http://phpmyadmin.test](http://phpmyadmin.test)` သို့ သွားပါ။
2. **Username** နေရာတွင် `root` ဟု ရိုက်ထည့်ပါ။
3. **Password** အကွက်ကို လုံးဝအလွတ် (Blank) ထားခဲ့ပြီး "Login" ကို နှိပ်ပါ။

---

## 3. PART II: PostgreSQL Administration (Herd Alternative)

> **Architectural Note:** phpMyAdmin သည် PostgreSQL wire protocol ကို support မလုပ်ပါ။ အကယ်၍ PostgreSQL ကို အသုံးပြုမည်ဆိုပါက Laravel Herd ပေါ်တွင် **Adminer** ကိုသုံးပြီး web-GUI တစ်ခုကို လွယ်ကူစွာ ဖန်တီးနိုင်သည်။

### Step 1: Engine Mapping

1. **DBngin** တွင် PostgreSQL runtime active ဖြစ်နေကြောင်း သေချာပါစေ။
2. ၎င်း၏ သတ်မှတ် port (PostgreSQL ၏ default မှာ `5432` ဖြစ်ပြီး၊ conflict ဖြစ်ပါက `5433` သို့ ပြောင်းလေ့ရှိသည်) ကို မှတ်သားပါ။
3. Default user သည် `postgres` ဖြစ်ပြီး password မှာ အလွတ် ဖြစ်သည်။

### Step 2: Deploying Adminer Web Console via Herd

Adminer သည် single PHP file ဖြစ်ပြီး multi-driver support ပေးနိုင်သဖြင့် Postgres အတွက် အကောင်းဆုံး phpMyAdmin အစားထိုး ဝဘ်ကွန်ဆိုး ဖြစ်သည်။

1. Herd ၏ site paths အောက်တွင် folder အသစ်တစ်ခု ဆောက်ပါ (ဥပမာ- `C:\Users\YourUser\Herd\adminer\`)။
2. Official ဆိုက်မှ `adminer-4.x.x.php` script ကို ဒေါင်းလုဒ်လုပ်ပြီး အထက်ပါ folder ထဲသို့ **`index.php`** အမည်ဖြင့် သိမ်းဆည်းပါ။
3. Herd က ၎င်းကို `[http://adminer.test](http://adminer.test)` အဖြစ် domain map အလိုအလျောက် လုပ်ပေးမည် ဖြစ်သည်။

### Step 3: Connection String Mapping

1. Browser တွင် `[http://adminer.test](http://adminer.test)` သို့ သွားပါ။
2. Login Form တွင် အောက်ပါအတိုင်း ဖြည့်စွက်ပါ-

| Property | Value Specification |
| --- | --- |
| **System** | Dropdown မှ `PostgreSQL` ကို ရွေးချယ်ပါ |
| **Server** | `127.0.0.1:5432` *(သင့် DBngin မှတ်သားထားသော port အတိုင်း ထည့်ပါ)* |
| **Username** | `postgres` |
| **Password** | *[လုံးဝအလွတ် ထားခဲ့ပါ]* |
| **Database** | `postgres` |

---

### Option B: Deeply Integrated Desktop Workflow via TablePlus

ဝဘ် UI အပြင် သင်သည် Desktop Application အသုံးပြုရသည်ကို ပိုနှစ်သက်ပါက **TablePlus** ကို အသုံးပြုနိုင်ပါသည်။ DBngin နှင့် TablePlus သည် developer တစ်ဖွဲ့တည်းမှ ဖန်တီးထားခြင်းဖြစ်သောကြောင့် အလိုအလျောက် ချိတ်ဆက်လုပ်ဆောင်ပေးပါသည်။

1. သင်၏စက်တွင် **TablePlus** ကို Install လုပ်ထားပါ။
2. **DBngin** ကို ဖွင့်ပါ။
3. PostgreSQL service ၏ ဘေးနားရှိ **Arrow (Open)** ခလုတ်လေးကို နှိပ်လိုက်ပါ။
4. Config များ (port, user, password) အားလုံး အလိုအလျောက် pre-fill ဖြစ်သွားပြီး TablePlus တွင် connection ချက်ချင်း တက်လာမည် ဖြစ်ပါသည်။ မည်သည့် manual setup မှ ထပ်လုပ်ရန် မလိုအပ်ပါ။
