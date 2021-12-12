---
title: macOS openssl file not found while installing Ruby gems
tags: [ruby, ruby-gems, macOS, jekyll]
categories: [TIL, Ruby]
---

While installing Ruby gems on macOS, you might encounter an error like:
```bash
In file included from binder.cpp:20:
./project.h:119:10: fatal error: 'openssl/ssl.h' file not found
#include <openssl/ssl.h>
         ^~~~~~~~~~~~~~~
1 error generated.
make: *** [binder.o] Error 1

make failed, exit code 2

Gem files will remain installed in /Users/burak/.rvm/gems/ruby-3.0.1/gems/eventmachine-1.2.7 for inspection.
```

That was the case for me when I was trying to install `jekyll` gem using `gem install jekyll`.

Solution is to do:
```
gem install jekyll -- --with-cppflags=-I/opt/homebrew/opt/openssl/include
```
