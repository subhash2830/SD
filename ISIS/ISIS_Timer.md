---
uid: 
title: ISIS
aliases: 
topic: Timer
date: 
tags:
  - protocol/isis
status: 
priority:
---




tag : ISIS

- [ ] ISIS hello interval 
	->  10 sec(default) for non DIS
	->	Hello interval/3 for DIS

- [ ]  ISIS hold interval
	->  3 * hello interval

- [ ]  DIS interval
-  Dis hello interval is normal Hello interval  / 3

```
Interface level cmd 
isis hello-interval 10  > hello timer change
isis hello-multiplier 4 > hold time change 
```

```
change hello interval to msec
- []  "isis hello-interval minimal" ---- set hold down time to 1 sec
- []  "isis hello-multiplier 4"  ------- set hello interval to 250 msec

```









 


 
 