---
layout: post
title: "First week of GSoC"
date: 2021-06-15
author: "sn99"
comments: true
description: "A week into GSoC"
keywords: "gsoc, linux, rust, cpp, c, ffi, bindings"
---

So it has been 1 week since GSoC started, and a lot has happened since then (bad joke incoming ... about 7 days have
happened).

Let's start with a brief intro about my project: In short a rewrite of a pure C/C++ library into pure Rust. In long, we have
to rewrite [butteraugli](https://github.com/google/butteraugli) into pure Rust while making bindings for it too (well this
wasn't that long as I thought it would be, men huh).

Well the week started with me nuking my fedora because I was on beta before and had
a [sound bug](https://www.reddit.com/r/Fedora/comments/ntisfr/recent_pipewire_update_broke_audio/)
1-2 days before and what the hell, I don't have the patience to wait for the bug fix so well yeet it (I have never used
the word "yeet" before). It did come
back [5-6 days later](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/1269) but it was resolved soon after. So
overall a great start to the week. There was also firefox 89 and that is a whole other thing.

I also wrote [butteraugli-google-vs-libjxl PART 1](https://sn99.github.io/2021/06/07/butteraugli-google-vs-libjxl/), it
has most of it including few performance benchmarks and quotes taken from the original author of butteraugli from
his Twitter. The difference in different values is explained by him. Getting perf to work on docker is another story.

## Bindings

The first part of the project was of course the bindings. Rust, as far as I understand was not made to fight C/C++ but
rather work with it which is reflected in how well it works with it and the surrounding tooling. There
is [bindgen](https://github.com/rust-lang/rust-bindgen)
but it is mainly `C`, so as my mentor suggested I went with [cxx](https://github.com/dtolnay/cxx), the only difference
is that it was made with `C++` in mind rather than `C`.

So, now just call it, and it will make automatic bindings as bindgen does? Well, _N O_. CXX does not make automatic
bindings, so we have to do that manually. How hard can it be?

Well, it was not as hard as expected as the goal was to imitate what butteraugli does. I just wanted to run like
butteraugli, call a function `Run` from Rust and let it work the same way. After some clean-up of butteraugli we
were good to go (We now only have 2 files instead of 4).

```Rust
#[cxx::bridge]
mod ffi {
    extern "C++" {
        include!("butteraugli-sys/butteraugli-cc/butteraugli.h");

        // int Run(int argc, char *argv[]);
        unsafe fn Run(argc: i32, argv: *mut *mut c_char) -> i32;
    }
}
```

And it did not work, something about jpeg and png not being found by pkg-config in "build.rs" and the whole down the
rabbit-hole. At last, the solution like all other solutions was simple (make a `.cargo/config`).

The next problem was how to work `(int argc, char *argv[])` into Rust, but luckily Rust has `ffi::CString` and we were
able to [get it working](https://github.com/sn99/butteraugli-rs/blob/master/butteraugli-sys/src/lib.rs#L33), just the
same was the C++ program would run.

We will get back to bindings again as we want to have a function that takes 2 images and returns the `difference`
between them.

## Conversion to pure Rust

What a foolish mortal I was getting tricked by sweet promises of cxx, i.e, I will convert function by function make ffi
for them and then badabing badaboom we have a working pure Rust program (I might be wrong but for complex data
structures it quickly goes out of hand combined with how rust handles errors/strings/IO and everything in between).

![overview.svg](/resources/overview.svg)

So after trying that for a while and fighting the compiler why not just let cxx remain in the bindings only and work on
just Rust.

This looks so much more poetic:

```rust
pub fn open_image(path: PathBuf) -> DynamicImage {
    let image_open_result = image::open(path);

    let img = match image_open_result {
        Ok(file) => file,
        Err(error) => {
            eprintln!("{}", error);
            process::exit(1);
        }
    };
}
```

And unlike C++:

```c++
std::vector <Image8> rgb;
FILE *f = fopen(filename, "rb");
if (!f) {
   fprintf(stderr, "Cannot open %s\n", filename);
   exit(1);
}
unsigned char magic[2];
if (fread(magic, 1, 2, f) != 2) {
   fprintf(stderr, "Cannot read from %s\n", filename);
   exit(1);
}
if (magic[0] == 0xFF && magic[1] == 0xD8) {
   if (!ReadJPEG(f, &rgb)) {
      fprintf(stderr, "File %s is a malformed JPEG.\n", filename);
      exit(1);
   }
} else {
   if (!ReadPNG(f, &rgb)) {
      fprintf(stderr, "File %s is neither a valid JPEG nor a valid PNG.\n",
         filename);
      exit(1);
   }
}
fclose(f);
return rgb;
```

Error handling is just better in Rust (well image-rs error handling does most of the work). I realized late that I just
need to have a general idea of what they do and not try to do this `magic[0] == 0xFF && magic[1] == 0xD8` in Rust but
make the best solution in Rust (Result). The small thing that I should not have forgotten. **Rust is not C/C++**.

## What we have till now

- Bindings via [cxx](https://github.com/dtolnay/cxx)
- A test using [predicates](https://docs.rs/predicates/1.0.8/predicates/)
  and [assert_cmd](https://docs.rs/assert_cmd/1.0.5/assert_cmd/), it looks pretty good for now and compares images and
  matches the expected outcome we should get
- A CLI using [structopt](https://github.com/TeXitoi/structopt)
- A few functions converted to pure Rust

## Return of the bindings

Well well well, seems like we still have some more work to do, but lucky for us, it is simple. We now need to take 2
files and return the difference, so we add a function
in [`butteraugli.c`](https://github.com/sn99/butteraugli-rs/blob/b1ea3fb9741e6d680f0ac9c3c2b2c1f999a21936/butteraugli-sys/butteraugli-cc/butteraugli.cc#L2539)
, a ffi that looks like

```rust
unsafe fn diff_value(filename1: *mut c_char, filename2: *mut c_char) -> f64;
```

And Rust code that looks like:

```rust
pub fn butteraugli(filename1: String, filename2: String) -> f64 {
    unsafe {
        ffi::diff_value(
            CString::new(filename1).expect("").as_ptr() as *mut c_char,
            CString::new(filename2).expect("").as_ptr() as *mut c_char,
        )
    }
}
```

## Blockers

I ain't going to deny it, the blockers are my lack of experience in C++ (not Rust), almost all my projects have been in
Rust and I have never used C++ in a professional or a "big project" capacity. But luckily there are more than enough
resources, and I found out
about [builtin-dump-struct](https://clang.llvm.org/docs/LanguageExtensions.html#builtin-dump-struct) so all in all seems
everything is going alright.

You can find the current work at [Github](https://github.com/sn99/butteraugli-rs).

