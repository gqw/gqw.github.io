---
date: 2020-11-18
title: "C++序列化和反序列化工具"
linkTitle: "C++序列化和反序列化工具"
author: asura9527 ([@gqw](https://gqw.github.io))
description:
  std::stringstream, std::strstream, std::iostream, std::streambuf 使用技巧


menu:
  sidebar:
    name: C++序列化和反序列化工具
    identifier: c++-serialize-deserialize
    parent: C++
    weight: 3
hero: serialization_hero.png
---

在开发时我们经常会遇到需要序列号和反序列化得操作。比如将程序配置导出下次打开再从配置中读取，通常你可以将信息保存成json、xml或ini格式，让后再解析出来。更常见的情形是我们通过网络发送和接收数据，也涉及到序列化和反序列化操作，当让你可以使用protobuf、msgpack等序列化库。但是如果你只是个简单程序，或者序列化和反序列化的业务不是很重，不想搞得那么复杂有什么简单的方法吗？

## 最直接的方法

直接按字节拷贝，见代码:

```cpp
#include <string>

bool encode(const std::string& s, int32_t a, int32_t b,
	int32_t c, std::string& buf) {
	int32_t d = static_cast<int32_t>(s.length());

	if (buf.size() < (sizeof(int32_t) * 4 + d))
		return false;

	std::size_t pos = 0;
	memcpy(buf.data() + pos, &a, sizeof(int32_t));
	pos += sizeof(int32_t);
	memcpy(buf.data() + pos, &b, sizeof(int32_t));
	pos += sizeof(int32_t);
	memcpy(buf.data() + pos, &c, sizeof(int32_t));
	pos += sizeof(int32_t);
	memcpy(buf.data() + pos, &d, sizeof(int32_t));
	pos += sizeof(int32_t);
	memcpy(buf.data() + pos, s.data(), d);
	return true;
}

bool decode(const std::string& buf, int32_t& a, int32_t& b,
	int32_t& c, std::string& out) {
	if (buf.size() < (sizeof(int32_t) * 4))
		return false;

	std::size_t pos = 0;
	memcpy(&a, buf.data() + pos, sizeof(int32_t));
	pos += sizeof(int32_t);
	memcpy(&b, buf.data() + pos, sizeof(int32_t));
	pos += sizeof(int32_t);
	memcpy(&c, buf.data() + pos, sizeof(int32_t));
	pos += sizeof(int32_t);
	int32_t d = 0;
	memcpy(&d, buf.data() + pos, sizeof(int32_t));
	pos += sizeof(int32_t);

	out = std::string(buf.data() + pos, d);
	return true;
}


int main()
{
	std::string buf;
	buf.resize(1024);
	encode("test", 1, 2, 3, buf);
	std::string out;
	int a = 0, b = 0, c =0;
	decode(buf, a, b, c, out);
	return 0;
}
```

或者简单封装下：

```cpp
struct TestData {
	int32_t a =0;
	int32_t b = 0;
	int32_t c = 0;
	std::string s;

	bool encode(std::string& buf) {
        ...
	}

	bool decode(const std::string& buf) {
		...
	}
};

TestData in{ 1, 2, 3, "test" };
std::string buf;
buf.resize(1024);
in.encode(buf);

TestData out;
out.decode(buf);
```

## 使用stringstream

可以看到上面的代码还是比较繁琐的，不仅需要手动计算字符索引位置，还要直接使用memcpy函数。对于这个问题c++的stringstream可以简化不少代码：

```cpp
struct TestData {
	int32_t a =0;
	int32_t b = 0;
	int32_t c = 0;
	std::string s;

	bool encode(std::string& buf) {
		int32_t d = static_cast<int32_t>(s.length());

		std::ostringstream ss;
		ss.write((const char*)&a, sizeof(a));
		ss.write((const char*)&b, sizeof(b));
		ss.write((const char*)&c, sizeof(c));
		ss.write((const char*)&d, sizeof(d));
		ss << s;
		buf = ss.str();
		return ss.good();
	}

	bool decode(const std::string& buf) {
		std::istringstream ss(buf);
		int32_t d = 0;
		ss.read((char*)&a, sizeof(a));
		ss.read((char*)&b, sizeof(b));
		ss.read((char*)&c, sizeof(c));
		ss.read((char*)&d, sizeof(d));
		ss >> s;
		return ss.good();
	}
};
```
可以明显的看出代码比前面的要简洁优雅多了，但是这里面却有个比较严重的问题，无论是encode还是decode都需要构建stringstream，而stringstream会在内部构建一个新的streambuf对象，无论是输入还是输出都需要一次拷贝操作。

## 我们还有strstream

```cpp
struct TestData {
	int32_t a =0;
	int32_t b = 0;
	int32_t c = 0;
	std::string s;

	bool encode(std::string& buf) {
		int32_t d = static_cast<int32_t>(s.length());

		std::ostrstream ss(buf.data(), buf.length());
		ss.write((const char*)&a, sizeof(a));
		ss.write((const char*)&b, sizeof(b));
		ss.write((const char*)&c, sizeof(c));
		ss.write((const char*)&d, sizeof(d));
		ss << s;
		return ss.good();
	}

	bool decode(const std::string& buf) {
		std::istrstream ss((char*)buf.data(), buf.length());
		ss.seekp(buf.length()); // 设置写入位置（写入位置>读取位置才代表有数据可读）

		int32_t d = 0;
		ss.read((char*)&a, sizeof(a));
		ss.read((char*)&b, sizeof(b));
		ss.read((char*)&c, sizeof(c));
		ss.read((char*)&d, sizeof(d));
		ss >> s;
		return ss.good();
	}
};
```
stringstream使用的人比较多，但是知道strstream的就少的多了。strstream可以使用已有的内存做rdbuf, 但是很不幸的是这么好用的工具竟然被c++标准给废弃了<sup>[[1]](#ref_1)</sup><sup>[[2]](#ref_2)</sup>。

## 最后大招，自己造轮子

```cpp
struct serialization_streambuf : public std::streambuf
{
	serialization_streambuf(char* s, std::size_t n)
	{
		setp(s, s, s + n); // set write buffer pointer
		setg(s, s, s + n); // set read buffer pointer
	}
};

struct TestData {
	int32_t a =0;
	int32_t b = 0;
	int32_t c = 0;
	std::string s;

	bool encode(std::string& buf) {
		int32_t d = static_cast<int32_t>(s.length());

		serialization_streambuf osrb(buf.data(), buf.length());
		std::ostream ss(&osrb);

		ss.write((const char*)&a, sizeof(a));
		ss.write((const char*)&b, sizeof(b));
		ss.write((const char*)&c, sizeof(c));
		ss.write((const char*)&d, sizeof(d));
		ss << s;
		return ss.good();
	}

	bool decode(const std::string& buf) {
		serialization_streambuf osrb((char*)buf.data(), buf.length());
		std::istream ss(&osrb);

		int32_t d = 0;
		ss.read((char*)&a, sizeof(a));
		ss.read((char*)&b, sizeof(b));
		ss.read((char*)&c, sizeof(c));
		ss.read((char*)&d, sizeof(d));
		ss >> s;
		return ss.good();
	}
};
```
可以看到，其实也很简单。就是模仿strstream为iostream构建streambuf就行了。这样不仅享受到了iostream类给我们提供的便利，也克服了stringstream需要内存拷贝的缺陷。

# 参考引用

[1]  <a id="ref_1" href="https://en.cppreference.com/w/cpp/io/strstream" >std::strstream</a>`[EB/OL].`  https://en.cppreference.com/w/cpp/io/strstream

[2]  <a id="ref_2" href="https://stackoverflow.com/questions/2820221/why-was-stdstrstream-deprecated" >Why was std::strstream deprecated?</a>`[EB/OL].`  https://stackoverflow.com/questions/2820221/why-was-stdstrstream-deprecated
