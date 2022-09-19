---
title: C++三方库
top_img: transparent
date: 2022-09-19 21:57:30
updated: 2022-09-19 21:57:30
tags:
  - Linux
  - c++
categories: C++
keywords:
description: 一些好用的c++库
---

## fmt

```bash
git clone --depth=1 https://github.com/fmtlib/fmt.git

cd fmt/
mkdir build
cd build
cmake ..
make -j

sudo make install
# g++ test.cpp -o debug -lfmt
```

## fmtlog

好像只能使用静态库。

```bash
git clone https://github.com/MengRao/fmtlog.git

cd fmtlog
git submodule init
git submodule update
./build.sh

# g++ test.cpp -o debug -lfmt -lfmtlog-static
```

## hiredis

```bash
git clone --depth=1 https://github.com/redis/hiredis.git

cd hiredis
make

sudo make install
# g++ test.cpp -o debug -lhiredis
```

## redis-plus-plus

```bash
git clone --depth=1 https://github.com/sewenew/redis-plus-plus.git

cd redis-plus-plus
mkdir build
cd build
cmake -DREDIS_PLUS_PLUS_CXX_STANDARD=17 ..
make

sudo make install
# g++ test.cpp -o debug -lhiredis -lredis++
```
