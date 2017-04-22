---
title:  how to set your linux's timezone
comments: true
date: 2017-02-16 08:16:22
tags: [linux]
categories: 技术分享
---



> my blog post time seems not correct these days. but time is fetched from the internet, so it is verly likely caused by an incorrect timezone setting of the linux server.


type a few commands to see which timezone is set.
```
pi@raspberrypi:~/hexo-blog $ date
2017年  3月  4日 星期六 13:47:09 UTC
pi@raspberrypi:~/hexo-blog $ date -R
Sat, 04 Mar 2017 13:47:14 +0000
```

opps, the timezone is +0, but we are in china, the timezone shoud be +8

then use `tzselect` to set the correct timezone now!


```
pi@raspberrypi:~/hexo-blog $ sudo tzselect
Please identify a location so that time zone rules can be set correctly.
Please select a continent, ocean, "coord", or "TZ".
 1) Africa
 2) Americas
 3) Antarctica
 4) Arctic Ocean
 5) Asia
 6) Atlantic Ocean
 7) Australia
 8) Europe
 9) Indian Ocean
10) Pacific Ocean
11) coord - I want to use geographical coordinates.
12) TZ - I want to specify the time zone using the Posix TZ format.
#? 5
Please select a country whose clocks agree with yours.
 1) Afghanistan		  18) Israel		    35) Palestine
 2) Armenia		  19) Japan		    36) Philippines
 3) Azerbaijan		  20) Jordan		    37) Qatar
 4) Bahrain		  21) Kazakhstan	    38) Russia
 5) Bangladesh		  22) Korea (North)	    39) Saudi Arabia
 6) Bhutan		  23) Korea (South)	    40) Singapore
 7) Brunei		  24) Kuwait		    41) Sri Lanka
 8) Cambodia		  25) Kyrgyzstan	    42) Syria
 9) China		  26) Laos		    43) Taiwan
10) Cyprus		  27) Lebanon		    44) Tajikistan
11) East Timor		  28) Macau		    45) Thailand
12) Georgia		  29) Malaysia		    46) Turkmenistan
13) Hong Kong		  30) Mongolia		    47) United Arab Emirates
14) India		  31) Myanmar (Burma)	    48) Uzbekistan
15) Indonesia		  32) Nepal		    49) Vietnam
16) Iran		  33) Oman		    50) Yemen
17) Iraq		  34) Pakistan
#? 9
Please select one of the following time zone regions.
1) Beijing Time
2) Xinjiang Time
#? 1

The following information has been given:

	China
	Beijing Time

Therefore TZ='Asia/Shanghai' will be used.
Local time is now:	Sat Mar  4 21:46:15 CST 2017.
Universal Time is now:	Sat Mar  4 13:46:15 UTC 2017.
Is the above information OK?
1) Yes
2) No
#? 1

You can make this change permanent for yourself by appending the line
	TZ='Asia/Shanghai'; export TZ
to the file '.profile' in your home directory; then log out and log in again.

Here is that TZ value again, this time on standard output so that you
can use the /usr/bin/tzselect command in shell scripts:
Asia/Shanghai
```

let's see if the time is correct now:

```
pi@raspberrypi:~/hexo-blog $ date
2017年  3月  4日 星期六 13:47:09 UTC
pi@raspberrypi:~/hexo-blog $ date -R
Sat, 04 Mar 2017 13:47:14 +0000
```

still incorrect! may be we should follow the instruction above, say we have to set TZ in the .profile and logout/login.


```
pi@raspberrypi:~/hexo-blog $ echo "TZ='Asia/Shanghai'; export TZ" >> ~/.profile 
pi@raspberrypi:~/hexo-blog $ exit
logout

Connection closed by foreign host.

Disconnected from remote host(pi) at 21:48:00.

Type `help' to learn how to use Xshell prompt.
[d:\~]$ 

Connecting to 192.168.1.10:22...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.


The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Mar  4 12:09:50 2017 from 192.168.1.111
pi@raspberrypi:~ $ date
2017年  3月  4日 星期六 21:48:13 CST
```

ok, we are done!