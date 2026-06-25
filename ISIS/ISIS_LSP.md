---
uid:
title:
alias:
topic:
date:
tags:
status:
priority:

---
Basically contain IP prefix , mask ,metric and adjacent neighbors

LSPs Acknowledge by either by  CSNP and PSNP

LSP max lifetime  Default : 20 min 
```
command "lsp-max-lifetime"
```

LSP refresh interval    Default : 15 min 
```
lsp-refresh-interval"
```

What is zero age lifetime of LSP  : 60 Sec  can not changed of modified

	If a router does not refresh its LSP after the refresh interval and the LSP ages    on to zero remaining lifetime.   after that grace period of 60sec is given and then LSP is purged