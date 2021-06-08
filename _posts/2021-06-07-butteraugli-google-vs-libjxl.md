---
layout: post 
title: "Butteraugli libjxl vs Google PART 1"
date: 2021-06-07 
author: "sn99"
comments: true 
description: "A comparison between JPEG XL and Google implementation of Butteraugli"
keywords: "libjxl, google, butteraugli"
---

## First glance comparison

We can see that [`libjxl/lib/jxl/butteraugli/`](https://github.com/libjxl/libjxl/tree/main/lib/jxl/butteraugli)
contains:

```shell
butteraugli
├── butteraugli.cc
└── butteraugli.h
```

Compared to Google's [`butteraugli/butteraugli/`](https://github.com/google/butteraugli/tree/master/butteraugli):

```shell
butteraugli
├── Makefile
├── butteraugli.cc
├── butteraugli.h
└── butteraugli_main.cc
```

A library vs standalone implementation.

## Comparison between functions

Noe let's dwell a bit deeper shall we. For a small function like `AmplifyRangeAroundZero`; in

[`libjxl/lib/jxl/butteraugli/`](https://github.com/libjxl/libjxl/tree/main/lib/jxl/butteraugli):

```c++
template<class D, class V>
HWY_INLINE V

AmplifyRangeAroundZero(const D d, const double kw, const V x) {
    const auto w = Set(d, kw);
    return IfThenElse(x > w, x + w, IfThenElse(x < Neg(w), x - w, x + x));
}
```

and [`butteraugli/butteraugli/`](https://github.com/google/butteraugli/tree/master/butteraugli):

```c++
static BUTTERAUGLI_INLINE float AmplifyRangeAroundZero(float w, float x) {
    return x > w ? x + w : x < -w ? x - w : 2.0f * x;
}
```

[`IfThenElse`](https://communityviz.city-explained.com/communityviz/s360webhelp/Formulas/Function_library/IfThenElse_function.htm)
is nothing but syntactic sugar over chained `? :`
> If {Condition A} and {Condition B} are both false but {Condition C} is true, then return {Value C}, etc.

And both of them do the same thing i.e. `Make area around zero more important (2x it until the limit).`

There are also some functions that are identical, for example:

```c++
std::vector<float> ComputeKernel(float sigma) {
    const float m = 2.25;  // Accuracy increases when m is increased.
    const double scaler = -1.0 / (2.0 * sigma * sigma);
    const int diff = std::max<int>(1, m * std::fabs(sigma));
    std::vector<float> kernel(2 * diff + 1);
    for (int i = -diff; i <= diff; ++i) {
        kernel[i + diff] = std::exp(scaler * i * i);
    }
    return kernel;
}
```

So what does it mean ? Most of the changes have been around converting butteraugli to a part of jpeg xl (libjxl) and
improving code readability and incorporating in into jxl's workflow

### Which one is the latest ?

The last commit on [`libjxl/lib/jxl/butteraugli/`](https://github.com/libjxl/libjxl/tree/main/lib/jxl/butteraugli) was
on May 25, 2021 vs Mar 20, 2019
for [`butteraugli/butteraugli/`](https://github.com/google/butteraugli/tree/master/butteraugli)

## Building/Compiling butteraugli

It seems both of them target Ubuntu/Debian based systems, but we will be working on Fedora, let's see how much they
differ and what can be done.

### [`libjxl/lib/jxl/butteraugli/`](https://github.com/libjxl/libjxl/tree/main/lib/jxl/butteraugli)

1. `git clone https://gitlab.com/wg1/jpeg-xl.git --recursive`

Now almost all the packages are available in Fedora albeit with a change in name, for example `libpng-dev` becomes
`libpng-devel` and so on. But it still does not build for god knows how many reasons.

**Docker to rescue**

Installing docker on fedora is [very](https://linuxconfig.org/how-to-install-docker-on-fedora-linux-system)
[very](https://docs.docker.com/engine/install/fedora/) easy. And
the [performance of Docker is near-native](http://domino.research.ibm.com/library/cyberdig.nsf/papers/0929052195DD819C85257D2300681E7B/$File/rc25482.pdf)
and lot better than using a VM.

2. After installing Docker, `sudo docker pull gcr.io/jpegxl/jpegxl-builder`
3. `sudo docker run -it --rm \
   --user root:root \
   -v $HOME/jpeg-xl:/jpeg-xl -w /jpeg-xl \
   gcr.io/jpegxl/jpegxl-builder bash`

We need `root` to [install perf](https://www.fosslinux.com/7069/installing-and-using-perf-in-ubuntu-and-centos.htm)

4. Finally, `CC=clang-7 CXX=clang++-7 ./ci.sh opt`

No more dependency hell using Docker so yay ?

### [`butteraugli/butteraugli/`](https://github.com/google/butteraugli/tree/master/butteraugli)

Well the instructions say to use bazel but then bazel does not have Fedora as 1st class, and it maintained by community,
instead we will build butteraugli directly.

1. Inside the `../butteraugli` just type `make` and voila we are done.

## Usage

1a.png                      |           1b.png
:--------------------------:|:--------------------------:
![1a.png](/resources/1a.png) | ![1b.png](/resources/1b.png)

`perf stat build/tools/butteraugli_main sample/1a.png sample/1b.png` gives:

```shell
../lib/extras/codec_png.cc:496: PNG: no color_space/icc_pathname given, assuming sRGB
../lib/extras/codec_png.cc:496: PNG: no color_space/icc_pathname given, assuming sRGB
112.6187591553
3-norm: 36.247990

 Performance counter stats for 'build/tools/butteraugli_main sample/1a.png sample/1b.png':

        511.888365      task-clock:u (msec)       #    1.040 CPUs utilized          
                 0      context-switches:u        #    0.000 K/sec                  
                 0      cpu-migrations:u          #    0.000 K/sec                  
             56622      page-faults:u             #    0.111 M/sec                  
        1626170411      cycles:u                  #    3.177 GHz                    
          17544518      stalled-cycles-frontend:u #    1.08% frontend cycles idle   
         863154666      stalled-cycles-backend:u  #   53.08% backend cycles idle    
        3508284724      instructions:u            #    2.16  insn per cycle         
                                                  #    0.25  stalled cycles per insn
         251453202      branches:u                #  491.227 M/sec                  
           7982904      branch-misses:u           #    3.17% of all branches        

       0.492104909 seconds time elapsed
```

`./butteraugli sample/1a.png sample/1b.png` gives:

```shell
133.019547

 Performance counter stats for './butteraugli sample/1a.png sample/1b.png':

          5,245.69 msec task-clock:u              #    1.000 CPUs utilized          
                 0      context-switches:u        #    0.000 K/sec                  
                 0      cpu-migrations:u          #    0.000 K/sec                  
            57,878      page-faults:u             #    0.011 M/sec                  
    21,313,735,734      cycles:u                  #    4.063 GHz                    
        31,603,794      stalled-cycles-frontend:u #    0.15% frontend cycles idle   
    11,809,642,370      stalled-cycles-backend:u  #   55.41% backend cycles idle    
    54,301,550,018      instructions:u            #    2.55  insn per cycle         
                                                  #    0.22  stalled cycles per insn
     4,953,100,365      branches:u                #  944.223 M/sec                  
        23,344,803      branch-misses:u           #    0.47% of all branches        

       5.247477878 seconds time elapsed

       5.186435000 seconds user
       0.059993000 seconds sys
```

There is definitely a difference in results although minor and a hit in performance (might be due to security issues of
perf in
docker [link](https://stackoverflow.com/questions/44745987/use-perf-inside-a-docker-container-without-privileged))

# From the author of Butteraugli:

> Butteraugli vs Butteraugli(jpeg xl)
>
> From Jyrki Alakuijala butteraugli's xyb is likely more accurate,
> because of asymptotic log behaviour for high intensity values (instead of raising to a power),
> jpeg xl's xyb modeling is going to be substantially faster to computer, because gamma is exactly three there.
>
> Shouldn't matter in the end. All butteraugli's have been calibrated to the same reference corpus with pretty good results. The reference corpus contains 2500 image pairs in the area of 0.6--1.3 max butteraugli values. Earlier butterauglis may be slightly more accurate, later butterauglis more practical and compatible with compression. Later butterauglis can be slightly better at detecting faint larger scale degradations as they do recursive 2x multiscaling of top of the Laplacian pyramid, not just a single run of four levels of Laplacian pyramid.
>
> I, as the author of butteraugli, use the latest butteraugli in jpeg xl for my use.


