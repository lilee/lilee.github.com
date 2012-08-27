---
layout: post
title: "CEscapeString"
description: ""
category: 代码片段
tags: [cpp, string]
---

最近在项目中使用LOG打印调试信息日志时，有些需要打印的数据中含有不可显示的字符，在终端上就显示成了问号或者是乱码。这样非常不利于判断打印的调试信息中的数据是否正常。

但笔者最近在使用protobuf时，发现使用它的Message::DebugString的接口可以把不可显示的字符进行转义输出。protobuf中将这个转义函数叫做CEscapeString，意思是:
> escaping dangerous characters using C-style escape sequences
> (将危险的字符进行C风格的转义)

首先看看CEscapeString的签名
{% highlight cpp %}
// CEscapeString()
//    Copies 'src' to 'dest', escaping dangerous characters using
//    C-style escape sequences. This is very useful for preparing query
//    flags. 'src' and 'dest' should not overlap.
//    Returns the number of bytes written to 'dest' (not including the \0)
//    or -1 if there was insufficient space.
//
//    Currently only \n, \r, \t, ", ', \ and !isprint() chars are escaped.
int CEscapeString(const char* src, int src_len,
                  char* dest, int dest_len);
{% endhighlight %}

google代码中的注释非常详细，值得学习。注释中还解释说CEscapeString在preparing query flags时非常有用。也就是说，你通过gflags或者其他方式向命令行传递一些不可显示字符时就需要使用到这种方式。目前仅只对\n,\r,\t, ", ', \ 和不可显示字符做转义

概念了解了，实现很简单，所有码农都会写。这里还是贴出google的官方版本实现。可以窥探出CEscapeString实现的更多细节，这里不做过多解释

{% highlight cpp %}
int CEscapeInternal(const char* src, int src_len, char* dest,
                    int dest_len, bool use_hex, bool utf8_safe) {
  const char* src_end = src + src_len;
  int used = 0;
  bool last_hex_escape = false; // true if last output char was \xNN

  for (; src < src_end; src++) {
    if (dest_len - used < 2)   // Need space for two letter escape
      return -1;

    bool is_hex_escape = false;
    switch (*src) {
      case '\n': dest[used++] = '\\'; dest[used++] = 'n';  break;
      case '\r': dest[used++] = '\\'; dest[used++] = 'r';  break;
      case '\t': dest[used++] = '\\'; dest[used++] = 't';  break;
      case '\"': dest[used++] = '\\'; dest[used++] = '\"'; break;
      case '\'': dest[used++] = '\\'; dest[used++] = '\''; break;
      case '\\': dest[used++] = '\\'; dest[used++] = '\\'; break;
      default:
        // Note that if we emit \xNN and the src character after that is a hex
        // digit then that digit must be escaped too to prevent it being
        // interpreted as part of the character code by C.
        if ((!utf8_safe || static_cast<uint8>(*src) < 0x80) &&
            (!isprint(*src) ||
             (last_hex_escape && isxdigit(*src)))) {
          if (dest_len - used < 4) // need space for 4 letter escape
            return -1;
          sprintf(dest + used, (use_hex ? "\\x%02x" : "\\%03o"),
                  static_cast<uint8>(*src));
          is_hex_escape = use_hex;
          used += 4;
        } else {
          dest[used++] = *src; break;
        }
    }
    last_hex_escape = is_hex_escape;
  }

  if (dest_len - used < 1)   // make sure that there is room for \0
    return -1;

  dest[used] = '\0';   // doesn't count towards return value though
  return used;
}

int CEscapeString(const char* src, int src_len, char* dest, int dest_len) {
  return CEscapeInternal(src, src_len, dest, dest_len, false, false);
}
{% endhighlight %}

这段代码是从protobuf目录中的src/google/protobuf/stubs/strutil.cc中抠取出来的。strutil.cc中还包含很多字符串相关操作，而且都有非常优秀的实现。可将其抽取出来丰富你的CPP公共库