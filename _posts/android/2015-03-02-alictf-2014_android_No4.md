---
layout: post
title:  阿里ctf-2014 android 第四题
category: android
tags: reverse
keywords: 
description: 
---

# evilapk400分析

### 一、JNI_Onload
调用`RegisterNatives`注册了两个函数`attachBaseContext`和`onCreate`，这一步是将java层声明的两个native函数和so里面的原生函数对应起来。

```c
//Application与ContentProvider的初始化次序是这样的：

Application:attachBaseContext()	//最早执行，然后是
ContentProvider:onCreate()	//然后是
Application:onCreate()
````
ContentProvider是delvik虚拟机加载dex的框架类，在Application类里面声明`attachBaseContext`则会重载框架的同名函数，实现在运行dex的`onCreate`之前运行代码，实现对dex的篡改等操作。

这里需要额外说明的是RegisterNatives要用到的一个数据结构（数组）：

```c
typedef struct {  
    char *name;  
    char *signature;  
    void *fnPtr;  
} JNINativeMethod;  
```
函数的注册对应关系就通过这个结构来体现。
下面`parse_dex`里面调用libdvm.so中的`dvm_dalvik_system_DexFile`也是`JNINativeMethod`结构的数组。这个也是所谓反射调用`openDexFile`的原理。

### 二、attachBaseContext
重头戏是经过里面的一系列准备后，调用`parse_dex`函数，实现了dex的解密、解压缩以及dex_code的解密、解压缩、组装。

### 三、parse_dex
这个是还原dex的核心函数，函数很大，不过很多是为了应对不同android版本和cpu体系的。我们只关注arm体系和android 4.0以上的相关代码。以下是自己整理的函数基本逻辑：

```c
if( ali::isDelvik )
{
	//1.这一步处理cls.jar的解密和解压缩，还原出一个dex内存镜像
    if ( ali::sdk_int > 13 )		//api等级>13，android版本比较高
	{
		ali::EncFile::openWithHeader
		...
	}
	else
	{
		//低版本android的处理代码
	}

	//2.这一步调用delvik框架加载上面解密出的dex内存镜像
	if ( ali::sdk_int > 13 )
    {
      v45 = dlopen("libdvm.so", 1);
      v46 = (JNINativeMethod *)dlsym(v45, "dvm_dalvik_system_DexFile");
      v55 = 0;
      lookup(v46, "openDexFile", "([B)I", &v55);
	  ...
      ((void (*)(void))v55)();
	  ...
    }
    else
    {
      v33 = dlopen("libdvm.so", 1);
      v34 = dlsym(v33, "dvmDexFileOpenPartial");
      v35 = dlsym(v33, "dexCreateClassLookup");
	  ...
    }

	//3.这一步是为上面还原出的dex替换正确的`class_def_item.class_data_item.encoded_method.code_off`
    std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v94, &ali::g_filePath, "/juice.data");
    if ( !ali::dex_juicer_patch((ali *)v29, v52, (unsigned int)v96, v49) )
    {
		//正确还原dex，退出函数
		...
    }
	//还原错误处理，退出函数
	...
}
else
{
	//其他cpu体系的还原代码
	...
}
```

ali在这里使用了两个技巧。
1. [**反射调用openDexFile加载dex**](http://blog.csdn.net/androidsecurity/article/details/9674251)
&#160;&#160;&#160;&#160;主要看里面链接的英文pdf。英文好的，可以搜索相应的视频。
2. **dalvik虚拟机，类结构和函数代码分别加载**
&#160;&#160;&#160;&#160;对于dalvik虚拟机，它在解析dex文件的时候会且仅会把所有的class_def_item结构加载到内存，而只有在使用到某个类的方法的时候，才会具体加载这个方法的代码。主要是为了缩短加载速度，加快apk的启动速度。
&#160;&#160;&#160;&#160;这个原理可以参考一个看雪上的例子[运行时自篡改dalvik字节码的原理解析](http://bbs.pediy.com/showthread.php?t=176732)。

总结一下：
1. `parse_dex`的第1、2步解密、解压cls.jar为dex内存镜像。但这个dex的`encoded_method.code_off`所指向的`code_item`都是同一个`throw new RuntimeException();`函数的实现代码根本就没有在cls.jar中。
2. `parse_dex`的第三步解密、解压juice.data(这个文件里面保存的就是真实的method code_item),并且要完成对cls.dex中`encoded_method.code_off`的修改。

### 四、`parse_dex`的第1、2步
分析第1、2步还是比较简单的，主要在`ali::EncFile::openWithHeader`中。

### `ali::EncFile::openWithHeader`伪代码：
```c
  v10 = open("/Path/cls.jar", 0);
  v11 = fstat(v10, &buf);
  if ( v11 )
  {
    v8 = (int)"debug";
    v9 = "fstat failed";
    goto LABEL_5;
  }
  v13 = buf.st_blksize;
  *v6 = buf.st_blksize;
  *(_DWORD *)v4 = v13;
  v14 = (int)mmap(0, *v6, 3, 2, v10, 0);
  *((_DWORD *)v4 + 1) = v14;
  close(v10);
  ...

  //rc4解密函数
  ali::decryptRc4((ali *)v14, (const unsigned __int8 *)v14, (unsigned __int8 *)v6, v18);
  。。。
  do
  {
    v23 = 8 * v11++;
    v24 = (unsigned __int64)*(_BYTE *)(v22++ + 1) << v23;
    v21 += v24;
  }
  while ( v11 != 8 );	//这个循环是取dex镜像的header.filesize字段，将little-ending转成DWORD。
  _android_log_print(3, "debug", "unpackSize: %u", v21);
  *(_DWORD *)v4 = v7 + v21;
  v25 = (ali *)mmap(0, v7 + v21, 3, 34, -1, 0);
  ...

  //解压缩函数
  v30 = LzmaDecode(v26, &v33, v14 + 13, (int)&v32, v14, 5, 1, (int)&v34, (int)&off_54028);
  ...
}
```
实际上整个过程分为两个部分：
1、 rc4解密：`ali::decryptRc4`

```c
  v7 = (int)calloc(0x18u, 1u);
  v8 = (const unsigned __int8 *)v7;
  if ( v7 )
  {
    if ( calc_rc4_key(v7) )		//这个函数是sub_28854，函数名是根据功能标注的。
    {
      v9 = (ali::CryptoRc4 *)operator new(0x108u);
      ali::CryptoRc4::CryptoRc4(v9, v8, 0x18u); // ali::CryptoRc4 类构造函数
      (*(void (__fastcall **)(_DWORD, _DWORD, _DWORD, _DWORD))(*(_DWORD *)v9 + 12))(v9, v4, v5, v6);// ali::CryptoRc4::decrypt(...)
      (*(void (__fastcall **)(_DWORD))(*(_DWORD *)v9 + 4))(v9);// ali::CryptoRc4 类析构函数
    }
  }
  else
  {
    _android_log_print(6, "debug", "invalid input...");
  }
  operator delete((void *)v8);
```
&#160;&#160;&#160;&#160;rc4的秘钥很有意思，`calc_rc4_key`函数是将原apk的classes.dex的crc32值与一个硬编码的0x18字节的字符串```.text:0002885A                 ADD             R3, PC  ; "YRq&rxh6Nsbh^W1nI5RfZzJZ"```进行每4字节的DWORD亦或运算得到的。

2、 lzma解压缩：`LzmaDecode`

函数名在这两步中提供了足够的信息。上面的伪代码只能看一个大概，参数的分析结果是存在错误的，参数如何传递，dex文件镜像在内存中的位置，需要调试时看ARM代码确定。
这个时候如果在`openDexFile`函数加载完dex后dump出文件，那么baksmali的结果就是只有class_def而没有正确method_code的结果。

### 五、`parse_dex`的第三步：修补dex
`ali::dex_juicer_patch`

```c
ali::dex_juicer_patch(...)
{
  _android_log_print(3, "debug", "enter dex_juicer_patch %p, %d", this, a2);
  v6 = (ali::EncFile *)operator new(0xCu);	// 相当于 new ali::EncFile，v6就是this指针。可能高级语言new一个类对象的时候是一条语句，自动调用构造函数，但反编译层面是两条语句。（实际上是编译器自动生成构造函数的调用）
  ali::EncFile::EncFile(v6, (const char *)v4);  // ali::EncFile::EncFile(v6, "/Path/juice.data") 
                                                // 构造函数只是调用strndup复制了第二个参数，并将复制后的地址保存在v6+8的位置。v6+8应该是ali::EncFile类中的私有变量 char* filename。

  v7 = ali::EncFile::open(v6, &ali::juiceMem, &ali::juiceLength);// 这个函数内部实际上调用ali::EncFile::openWithHeader,只是参数R3=0。
  if ( ali::juiceMem == (unsigned __int8 *)-1 )	//ali::juiceMem实际上是由ali::EncFile::openWithHeader中的mmap返回的，所以这里和-1比较，判断是否正确分配了内存空间。
  {
	//debug error
	...
    result = -1;
  }
  else
  {
    v10 = 0;
    _android_log_print(3, "debug", "mapped juice to %p with size %d", ali::juiceMem, ali::juiceLength);
    v11 = 0;
    v12 = 0;
    v13 = *(_DWORD *)ali::juiceMem;		//这里开始的一段涉及到juice文件的结构，下面解释。
    v14 = *((_DWORD *)ali::juiceMem + 1);
    v19 = ali::juiceMem + 8;
    if ( v13 > 0x1FC00000 )
      v15 = -1;
    else
      v15 = 4 * v13;
    v18 = v14;
    ali::orgOffset = operator new[](v15);	//这个保存原来Dex中用来干扰的method_code_off(uleb128解码后的）。
    while ( v10 != v13 )
    {
      v12 += uleb128_decode((int **)&v19);
      v16 = uleb128_decode((int **)&v19);
      v17 = ali::orgOffset;
      v20 = (ali *)((char *)v5 + v12);
      v11 += v16;
      *(_DWORD *)(v17 + 4 * v10++) = uleb128_decode((int **)&v20);
      uleb128_encode(v5, v12, &ali::juiceMem[v11] + v18 - (unsigned __int8 *)v5);
    }
    result = 0;
  }
  return result;
}
/*
说明：
```

`sub_27A9C uleb128_decode`

```c
//函数通过传递一个二级指针作为参数，可以将字节数组移动后的当前位置返回出来。
int uleb128_decode( uint** p )
{
	ubyte* b = *p;
	int offset = b[0] & 7f;
	if(b[0] > 7f)
	{
		offset |= (b[1] & 7f) << 7;
		if(b[1] > 7f)
		{
			offset |= (b[2] & 7f) << 14;
			if(b[2] > 7f)
			{
				offset |= (b[3] & 7f) << 21;
				if(b[3] > 7f)
				{
					offset |= (b[4] & 7f) << 28;
					*p = b+5;
				}
				else
					*p = b+4;
			}
			else
				*p = b+3;
		}
		else
			*p = b+2;
	}
	else
		*p = b+1;

	return offset;
}
```

`sub_27AEE uleb128_encode`

```c
/*参数说明：
pDex: 内存中Dex镜像的地址，patch的目标文件。
method_code_off：Dex中class_def_item.class_data_item.encoded_method.code_off，patch保存目标函数代码地址的偏移。
delta：juicer_method_code_addr - pDex，juicer中真实函数代码的地址和pDex的差。这个值就是要经过uleb128编码后放到pDex+method_code_off里面，从而修复Dex文件函数代码指向错误的问题。

说明：ali这里为了简单，对Dex文件中def_item.class_data_item.encoded_method.code_off字段的长度使用了uleb128的最大长度5个字节的固定值。这简化了本函数的修补过程。
*/
void uleb128_encode( ubyte* pDex, uint method_code_off, int delta)
{
.text:00027AEE                 ORN.W           R3, R2, #0x7F // R3 = R2 | (~0x7f)
.text:00027AF2                 STRB            R3, [R0,R1]
.text:00027AF4                 ADD             R1, R0
.text:00027AF6                 LSRS            R3, R2, #7	// 带符号右移7位
.text:00027AF8                 ORN.W           R3, R3, #0x7F
.text:00027AFC                 STRB            R3, [R1,#1]
.text:00027AFE                 LSRS            R3, R2, #0xE	// 带符号右移7位14
.text:00027B00                 ORN.W           R3, R3, #0x7F
.text:00027B04                 STRB            R3, [R1,#2]
.text:00027B06                 LSRS            R3, R2, #0x15	// 带符号右移21位
.text:00027B08                 LSRS            R2, R2, #0x1C	// 带符号右移28位
.text:00027B0A                 ORN.W           R3, R3, #0x7F
.text:00027B0E                 STRB            R2, [R1,#4]
.text:00027B10                 STRB            R3, [R1,#3]
.text:00027B12                 BX              LR
}
```

通过分析以上三个函数，可以知道`decoded_juice.data`内存镜像的文件结构：


![图](/public/img/2015-03-02-msc-2015_android_No4_juicedata_layout.png)


文件结构的前面部分实际上是两个索引，成对出现。一个是确定需要修补的code_off字段相对于Dex文件起始位置的偏移。一个就是确定相对应的函数实现代码，相对于juice文件起始位置的偏移。
两者分别加上各自文件的内存加载首地址，然后作差，得到的就是Dex文件code_off字段真实需要的函数实现代码的地址。这个过程在函数`sub_27AEE uleb128_encode`中完成。

需要说明的是，网上有些文章教导在dex_juice_patch执行完毕后，dump出dex文件后面的大块内存区域，然后baksmali的方法是完全依靠运气的。这个运气就是juice_mapaddr > dex_mapaddr，既要求juice的内存加载位置在dex的后面，如果在前面这种方法就不行了。而我的虚拟机里面就是出现在前面的。

### 六、脱壳程序
为了避免去写rc4和lzma的代码，也避免自己优化dex（程序有openDexFile进行）。所以比较好的一个修复点是从内存中dump出cls.dex和decoded_juice.data。
这个还是比较好实现的，就是decoded_juice.data的长度，需要在恰当的地方下断点获取这个值，当然这个值也会在debug调试信息中输出，可以在DDM中查看。

```c
#include <stdlib.h>
#include <stdio.h>

#define DexLength 0x019cf4
#define JuiceLength 0x16098

int read_uleb128(unsigned char** p)
{
	unsigned char* b = *p;
	int offset = b[0] & 0x7f;
	if(b[0] > 0x7f)
	{
		offset |= (b[1] & 0x7f) << 7;
		if(b[1] > 0x7f)
		{
			offset |= (b[2] & 0x7f) << 14;
			if(b[2] > 0x7f)
			{
				offset |= (b[3] & 0x7f) << 21;
				if(b[3] > 0x7f)
				{
					offset |= (b[4] & 0x7f) << 28;
					*p = b+5;
				}
				else
					*p = b+4;
			}
			else
				*p = b+3;
		}
		else
			*p = b+2;
	}
	else
		*p = b+1;

	return offset;
}

void write_uleb128(unsigned char* p, int num)
{
	p[0] = (num & 0x7f) | 0x80;
	p[1] = ((num >> 7) & 0x7f) | 0x80;
	p[2] = ((num >> 14) & 0x7f) | 0x80;
	p[3] = ((num >> 21) & 0x7f) | 0x80;
	p[4] = ((num >> 28) & 0x7f);
}


int _tmain(int argc, _TCHAR* argv[])
{
	FILE* fp1 = fopen("cls.dex", "rb");
	FILE* fp2 = fopen("decoded_juice.data", "rb");

	unsigned char* p = (unsigned char*) malloc(DexLength + JuiceLength);

	int i = 0;
	for(; i<DexLength; i++)
		p[i] = getc(fp1);
	for(; i<DexLength+JuiceLength; i++)
		p[i] = getc(fp2);

	int method_count = *((int*) (p+DexLength));
	int code_off = *((int*) (p+DexLength+4)) + DexLength;
	unsigned char* offblock_pointer = p+DexLength+8;
	unsigned char* dexcode_off = p;

	for(i=0; i<method_count; i++)
	{
		dexcode_off += read_uleb128(&offblock_pointer);
		code_off += read_uleb128(&offblock_pointer);
		write_uleb128(dexcode_off, code_off);
	}

	FILE* fp3 = fopen("evilapk400.dex", "wb");
	for(i=0; i<DexLength+JuiceLength; i++)
		putc(p[i], fp3);

	free(p);
	fclose(fp3);
	fclose(fp2);
	fclose(fp1);
	return 0;
}
```

这段代码没有修正dex的filesize字段，所以修补过的evilapk400.dex直接反编译成java是有问题的。这时候可以使用大招`baksmali` `smali`。然后再dexdecompile。当然这里如果是用dex2jar反编译，会出错。根据出错函数的名称，在.smali文件中找到所有这个函数，删除之，就能反编译了。

### 七、寻找题目的秘钥
&#160;&#160;&#160;&#160;这个题目脱壳之后的dex代码也是经过**代码混淆**的，不过代码不是特别多，混淆强度也不大。主要是**类名、函数名、变量名**都改成了无意义的名称，而且函数名都改的一样。这样只能通过参数和返回值类型来分析到底调用了哪一个函数。而变量名在不同的使用地方也都相同，导致反编译器的时候出现识别变量类型的错误。这个确实对分析造成了一定困扰，需要识别变量的正确类型，进而识别函数调用。
&#160;&#160;&#160;&#160;dex2jar可能是我的版本较低，就算修复了.smali反编译成功，在处理这种**代码混淆**的时候，也产生了部分关键函数不能正确分析的错误。
&#160;&#160;&#160;&#160;不多说了，直接上破解算法代码吧：

```java
public class cipher {

	/**
	 * @param args
	 */
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		String target = "000A0A0A0A0202AA5458D715704493D8E6B9BD38F8B6BE0E";
		String keyString = "1F98CEAB209770EFA875C245853ECE761F98CEAB209770EF";
		
		IvParameterSpec random = new javax.crypto.spec.IvParameterSpec(str2bytes("000A0A0A0A0202AA"));
		Key key =  new javax.crypto.spec.SecretKeySpec(str2bytes(keyString), "DESede");
		Cipher cipher;
		try {
			cipher = javax.crypto.Cipher.getInstance("DESede/CBC/PKCS5Padding");
	        cipher.init(Cipher.DECRYPT_MODE, key, random);
	        byte[] b = cipher.doFinal(str2bytes("5458D715704493D8E6B9BD38F8B6BE0E"));
	        System.out.println(new String(b, "UTF-8"));
		}  catch (Exception e) {
            e.printStackTrace();
		}
        
	}

    public static byte[] str2bytes(String p9)
    {
        byte[] v1 = p9.getBytes();
        int v2 = v1.length;
        byte[] v3 = new byte[(v2 / 2)];
        int v0_1 = 0;
        while (v0_1 < v2) {
            v3[(v0_1 / 2)] = ((byte) Integer.parseInt(new String(v1, v0_1, 2), 16));    //字符串是16进制数值的表示,转换成相应的数字字节码
            v0_1 += 2;
        }
        return v3;
    }
    }
```
target和keyString的值取自bg.png，不过不是直接取里面的字节，而是经过了字节到对应的hexString的转换后的值。
特别需要注意的一点是Edit.getText()方法获取的字符串编码是**`UTF-8`**的，让String的构造函数自动识别解码后的字符串的话，就会当成unicode解析，从而在最后的破解门口遇到障碍。结果是：**`日天@土侸`**


### 八、还有没有分析的，属于需要学习理解的内容
1. delvik虚拟机openDexFile加载dex的原理
2. dex加载后替换原来的Application，让delvik运行的机制。

虽然对于本次破解来说，这个不是必要的，但是理解了整个过程，对于掌握和分析其他加壳技术很重要。因为可以迅速确定解码后的dex文件的出现时机，既内存dump的时机。

[题目下载](/public/download/AliCrackme_4.apk)