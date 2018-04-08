---
layout: article
title: "How Do You Install PHP 7+ on Windows?"
categories: posts
modified: 2018-04-07T12:00:00-04:00
tags: [php, windows, installer, msi]
comments: true
ads: false
---
This is just a quick note for other people like me who remember a time when the [PHP for Windows](https://windows.php.net/) site used to provide MSI installers.

Despite [outdated documentation](http://php.net/manual/fa/install.windows.installer.msi.php), it doesn't look like any MSI installer links show up anymore on the site. Browsing the [PHP archives](https://windows.php.net/downloads/releases/archives/), it appears that the last version for which an MSI installer was offered was 5.3.29 back in August 2014. Now the only option offered officially appears to be a ZIP archive you have to unpack and install manually, which is no fun.

If you're looking for a more recent version that's not quite as hands-on, I recommend the [Microsoft Web Platform installer](https://www.iis.net/downloads/microsoft/web-platform-installer). You simply select the verion of PHP you want, click through a few steps, and it's installed!

It may feel like bizzaro world that Microsoft offers options to install programming languages or tools (like MySQL!) that are not mainted by Microsoft themselves, but Microsoft has been getting friendlier with the open source community since as long as as 2011. 
