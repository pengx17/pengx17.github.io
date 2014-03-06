---
layout: post
keywords: blog
description: blog
title: "I don't know C"
categories: [C, CPP]
tags: [Archive]
group: archive
icon: file-o
---

C is purer than C++. It does not have so many obscure features and ambiguous grammar. I understand these facts and thought there would be nothing more to learn about the C language itself. This was almost true in my mind until I met some open-source projects coded in C, i.e., x264 and ffmpeg. In this article, I will not talk about the x264 techniques but only the C language.

A colleague poked me yesterday and asked how to read the array structure below, which was originally found in x264 implementation. I edited it for explanation:

```cpp
int16_t (*mv[2][2])[2];
```

For me this kind of presentation of array structure declaration was seen rarely. I paused for a few seconds and recalled a [spirial rule][1] I had learnt in college (probably 5 years ago). Back to that time I did not pay much attention to that because I could not understand it due to lack of coding experience. I did not manage to decipher it in a way both of us could understand at first and thus I went through the _spirial rule_.

So to speak in spirial rule, we may draw it in such way:

                         +-----------+
                         | +---+     |
                         | ^   |     |
                int16_t (*mv[2][2])[2];
                 ^       ^     |     |
                 |       +-----+     |
                 +-------------------+

In speaking, it could be explained in the following English statement:

> `mv` is a 2x2 2D array of pointers to `int16_t[2]`.

It may still unclear to understand. I extend it in this way:

> `mv` is a 2x2 2D array. Each of the array element is a pointer. Each pointer is pointing to one `int16_t[2]` element.

For now I think it will not be that wired to see why x264 accesses `mv` in, for example `mv[0][1][6376][1]`, patterns.

------

In debugging we found that x264 sometimes use negative indexes in an array. e.g.:

```cpp
int t = some_random_int_array[-1]; 
```

it is like why the hell can indexes are negative? However it turns out to be totally legal and not like python, negative indexes indicate elements before the first element of the array. This is because the pattern `array[idx]` is equivalent to `*(array + idx)`. [This SO thread][2] explains and quotes the following from C99 §6.5.2.1/2:

> The deﬁnition of the subscript operator [] is that E1[E2] is identical to (*((E1)+(E2))).

-----

The story of learning new facts wen on and then I met [designated initializers][3] but I do not want to repeat every details of the specification here. As an short example in ffmpeg, I saw this:

```cpp
AVCodec ff_libx264_encoder = {
    .name             = "libx264",
    .long_name        = NULL_IF_CONFIG_SMALL("libx264 H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10"),
    .type             = AVMEDIA_TYPE_VIDEO,
    .id               = AV_CODEC_ID_H264,
    .priv_data_size   = sizeof(X264Context),
    .init             = X264_init,
    .encode2          = X264_frame,
    .close            = X264_close,
    .capabilities     = CODEC_CAP_DELAY | CODEC_CAP_AUTO_THREADS,
    .priv_class       = &x264_class,
    .defaults         = x264_defaults,
    .init_static_data = X264_init_static,
};
```
I can guess what is the dot variable name is about, but did not ever imagine C can do something like this!


These all kinds of both new/old facts refreshed my attitude towards C. I knew C++ is a language hard to master all the details, but I have always underestimated C as well. Language is evolving itself all the time even for C.


------

#### References
[The ``Clockwise/Spiral Rule''][1]

[Negative array indexes in c][2]

[Designated Initializers][3]

[1]: http://c-faq.com/decl/spiral.anderson.html
[2]: http://stackoverflow.com/questions/3473675/negative-array-indexes-in-c
[3]: http://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html
