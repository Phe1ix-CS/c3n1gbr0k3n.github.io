---
title: "阿里云CTF"
description: ""
date: 2023-04-23T02:21:47+08:00
lastmod: 2023-04-23T02:21:47+08:00
draft: false
images: []
type: docs
weight: 1
toc: true
---

## Reverse\_字节码跳动

### 0x01 分析

这道题是一个js字节码分析的题目，根据run.sh可以知道题目删除了js源码，保留了字节码文件 `flagchecker_bytecode.txt` ，以及一个jsc二进制文件。既然有字节码文件，那么我们直接来分析该文件。在这之前我也没有接触过js字节码逆向，所以我先写一个测试文件并输出字节码来对比分析，再结合网上的部分文章，大概可以看懂字节码。

测试js文件为

```javascript
function aaa(string, j) {
        for(let i = 0; i < string.length; i++) {
                string[i] |= j;
        }
}


function main() {
        let string = "ctf{test}";
        aaa(string, 20);
        console.log(string);
}
main()
```

然后执行命令

```bash
./node --print-bytecode test.js > test.txt
```

直接搜字符串 `ctf{test}`。

![f4054b2b09fca5e777b7fcb465588a6e](images/f4054b2b09fca5e777b7fcb465588a6e.png)  

然后就直接找到了main函数，如果直接去搜 `SharedFunctionInfo main` 。

![65e01228c9966c22df36c58be65e8f46](images/65e01228c9966c22df36c58be65e8f46.png)  

发现有一个对类似于对函数进行注册的地方，里面还能看到aaa函数，接下来我们对比源码来分析字节码

```javascript
// main函数
Bytecode Age: 0
  147 S> 0x3f079242086e @    0 : 13 00             LdaConstant [0]
// 将常量池中的0读取到累加器中，常量池就是最下面那个FixedArray，这里就是加载ctf{test}
         0x3f0792420870 @    2 : c3                Star0 
// 将累加器中的值给r0
  161 S> 0x3f0792420871 @    3 : 17 02             LdaImmutableCurrentContextSlot [2]
// 猜测是加载aaa函数到累加器
         0x3f0792420873 @    5 : c2                Star1 
// 将累加器值给r1
         0x3f0792420874 @    6 : 0d 14             LdaSmi [20]
// 立即值20加载到累加器
         0x3f0792420876 @    8 : c0                Star3 
// 20给r3
  161 E> 0x3f0792420877 @    9 : 62 f9 fa f7 00    CallUndefinedReceiver2 r1, r0, r3, [0]
// 调用函数 r1(r0, r3)
  179 S> 0x3f079242087c @   14 : 21 01 02          LdaGlobal [1], [2]
// 加载常量池的1 console
         0x3f079242087f @   17 : c1                Star2 
// 将其给r2
  187 E> 0x3f0792420880 @   18 : 2d f8 02 04       LdaNamedProperty r2, [2], [4]
// 常量池的2为log，所以为加载console.log到累加器
         0x3f0792420884 @   22 : c2                Star1 
// 给r1
  187 E> 0x3f0792420885 @   23 : 5d f9 f8 fa 06    CallProperty1 r1, r2, r0, [6]
// 调用方法 r1(r2, r0)，这里r2相当于this指针，r0是开头的字符串
         0x3f079242088a @   28 : 0e                LdaUndefined 
  200 S> 0x3f079242088b @   29 : a8                Return 
Constant pool (size = 3)
0x3f0792420811: [FixedArray] in OldSpace
 - map: 0x26392a3c12c1 <Map>
 - length: 3
           0: 0x3f07924207a1 <String[9]: #ctf{test}>
           1: 0x1127121a1619 <String[7]: #console>
           2: 0x319f33ce8e69 <String[3]: #log>
```

接着是aaa函数

```javascript
// aaa函数
   40 S> 0x3f079242096e @    0 : 0c                LdaZero 
// 加载0到累加器
         0x3f079242096f @    1 : c3                Star0 
// 给r0
   54 S> 0x3f0792420970 @    2 : 2d 03 00 00       LdaNamedProperty a0, [0], [0]
// 读取第一个参数a0的length属性到累加器
   45 E> 0x3f0792420974 @    6 : 6c fa 02          TestLessThan r0, [2]
// 判断r0是否小于累加器
         0x3f0792420977 @    9 : 98 19             JumpIfFalse [25] (0x3f0792420990 @ 34)
// false则跳转25个距离
   85 S> 0x3f0792420979 @   11 : 0b fa             Ldar r0
// 累加器设置为r0
   92 E> 0x3f079242097b @   13 : 2f 03 03          LdaKeyedProperty a0, [3]
// 相当于a0[r0]
         0x3f079242097e @   16 : c0                Star3 
// 给r3
         0x3f079242097f @   17 : 0b 04             Ldar a1
// 加载第二个参数a1给累加器
   98 E> 0x3f0792420981 @   19 : 3e f7 05          BitwiseOr r3, [5]
// 累加器和r3进行异或运算
   95 E> 0x3f0792420984 @   22 : 34 03 fa 06       StaKeyedProperty a0, r0, [6]
// 将累加器的值设置给 a0[r0]
   63 S> 0x3f0792420988 @   26 : 0b fa             Ldar r0
// 载入r0到累加器
         0x3f079242098a @   28 : 50 08             Inc [8]
// 到下一个单元
         0x3f079242098c @   30 : c3                Star0 
// 给r0
   28 E> 0x3f079242098d @   31 : 88 1d 00          JumpLoop [29], [0] (0x3f0792420970 @ 2)
// 跳转循环
         0x3f0792420990 @   34 : 0e                LdaUndefined 
  111 S> 0x3f0792420991 @   35 : a8                Return 
Constant pool (size = 1)
0x3f0792420921: [FixedArray] in OldSpace
 - map: 0x26392a3c12c1 <Map>
 - length: 1
           0: 0x26392a3c4d41 <String[6]: #length>
```

有了这个基础后我们再来看题目的字节码

![ad0b40af2977bfb23912b17aeec012a4](images/ad0b40af2977bfb23912b17aeec012a4.png)  

可以看到应该用到了main、aaa和ccc函数，这三个函数顺序应该就是main->aaa->ccc

```javascript
 // main函数
 1207 S> 0x3711a3be0a56 @    0 : 21 00 00          LdaGlobal [0], [0]
// 加载常量池的process
         0x3711a3be0a59 @    3 : c2                Star1 
// 给r1
 1215 E> 0x3711a3be0a5a @    4 : 2d f9 01 02       LdaNamedProperty r1, [1], [2]
// 获取r1.[1]即process.argv，也就是flag
         0x3711a3be0a5e @    8 : c2                Star1 
// 给r1
         0x3711a3be0a5f @    9 : 0d 02             LdaSmi [2]
// 累加器加载为2
 1219 E> 0x3711a3be0a61 @   11 : 2f f9 04          LdaKeyedProperty r1, [4]
// r1[2]到累加器
         0x3711a3be0a64 @   14 : c3                Star0 
// 给r0
 1270 S> 0x3711a3be0a65 @   15 : 96 23             JumpIfToBooleanFalse [35] (0x3711a3be0a88 @ 50)
         0x3711a3be0a67 @   17 : 17 03             LdaImmutableCurrentContextSlot [3]
// 猜测是加载aaa到累加器
         0x3711a3be0a69 @   19 : c2                Star1 
// 给r1
 1279 E> 0x3711a3be0a6a @   20 : 61 f9 fa 06       CallUndefinedReceiver1 r1, r0, [6]
// 调用函数 r1(r0) aaa(flag)
         0x3711a3be0a6e @   24 : c2                Star1 
// 将返回值给r1
         0x3711a3be0a6f @   25 : 11                LdaTrue 
 1286 E> 0x3711a3be0a70 @   26 : 6a f9 08          TestEqual r1, [8]
// 判断返回值是否为true
         0x3711a3be0a73 @   29 : 98 15             JumpIfFalse [21] (0x3711a3be0a88 @ 50)
 1305 S> 0x3711a3be0a75 @   31 : 21 02 09          LdaGlobal [2], [9]
// 为true就执行console.log("Right!")
         0x3711a3be0a78 @   34 : c1                Star2 
 1313 E> 0x3711a3be0a79 @   35 : 2d f8 03 0b       LdaNamedProperty r2, [3], [11]
         0x3711a3be0a7d @   39 : c2                Star1 
         0x3711a3be0a7e @   40 : 13 04             LdaConstant [4]
         0x3711a3be0a80 @   42 : c0                Star3 
 1313 E> 0x3711a3be0a81 @   43 : 5d f9 f8 f7 0d    CallProperty1 r1, r2, r3, [13]
         0x3711a3be0a86 @   48 : 89 13             Jump [19] (0x3711a3be0a99 @ 67)
 1349 S> 0x3711a3be0a88 @   50 : 21 02 09          LdaGlobal [2], [9]
// 接下来代码就是console.log("Wrong!")
         0x3711a3be0a8b @   53 : c1                Star2 
 1357 E> 0x3711a3be0a8c @   54 : 2d f8 03 0b       LdaNamedProperty r2, [3], [11]
         0x3711a3be0a90 @   58 : c2                Star1 
         0x3711a3be0a91 @   59 : 13 05             LdaConstant [5]
         0x3711a3be0a93 @   61 : c0                Star3 
 1357 E> 0x3711a3be0a94 @   62 : 5d f9 f8 f7 0f    CallProperty1 r1, r2, r3, [15]
         0x3711a3be0a99 @   67 : 0e                LdaUndefined 
 1378 S> 0x3711a3be0a9a @   68 : a8                Return 
Constant pool (size = 6)
0x3711a3be09e1: [FixedArray] in OldSpace
 - map: 0x2546cb1c12c1 <Map>
 - length: 6
           0: 0x39ea5930f9e9 <String[7]: #process>
           1: 0x20e3b754b6d9 <String[4]: #argv>
           2: 0x26784b2e1619 <String[7]: #console>
           3: 0x0ef123a28e69 <String[3]: #log>
           4: 0x3711a3be0961 <String[6]: #Right!>
           5: 0x3711a3be0979 <String[6]: #Wrong!>
```

接下来是aaa函数

```javascript
  757 S> 0x3711a3be0cde @    0 : 21 00 00          LdaGlobal [0], [0]
         0x3711a3be0ce1 @    3 : c0                Star3 
// 加载Buffer对象到r3
  768 E> 0x3711a3be0ce2 @    4 : 2d f7 01 02       LdaNamedProperty r3, [1], [2]
         0x3711a3be0ce6 @    8 : c0                Star3 
// Buffer.from => r3
         0x3711a3be0ce7 @    9 : 0b f7             Ldar r3
  757 E> 0x3711a3be0ce9 @   11 : 68 f7 03 01 04    Construct r3, a0-a0, [4]
// 将flag字符串转换为数组
         0x3711a3be0cee @   16 : c3                Star0 
  792 S> 0x3711a3be0cef @   17 : 2d fa 02 06       LdaNamedProperty r0, [2], [6]
// 获取flag长度
         0x3711a3be0cf3 @   21 : c0                Star3 
         0x3711a3be0cf4 @   22 : 0d 2b             LdaSmi [43]
  799 E> 0x3711a3be0cf6 @   24 : 6a f7 08          TestEqual r3, [8]
// 判断长度是否达到43
         0x3711a3be0cf9 @   27 : 97 04             JumpIfTrue [4] (0x3711a3be0cfd @ 31)
  806 S> 0x3711a3be0cfb @   29 : 12                LdaFalse 
  819 S> 0x3711a3be0cfc @   30 : a8                Return 
  835 S> 0x3711a3be0cfd @   31 : 21 00 00          LdaGlobal [0], [0]
         0x3711a3be0d00 @   34 : c0                Star3 
  846 E> 0x3711a3be0d01 @   35 : 2d f7 03 09       LdaNamedProperty r3, [3], [9]
         0x3711a3be0d05 @   39 : c0                Star3 
         0x3711a3be0d06 @   40 : 0d 2b             LdaSmi [43]
         0x3711a3be0d08 @   42 : bf                Star4 
         0x3711a3be0d09 @   43 : 0b f7             Ldar r3
  835 E> 0x3711a3be0d0b @   45 : 68 f7 f6 01 0b    Construct r3, r4-r4, [11]
// 用Buffer.alloc创建一个新数组
         0x3711a3be0d10 @   50 : c2                Star1 
  942 S> 0x3711a3be0d11 @   51 : 21 00 00          LdaGlobal [0], [0]
         0x3711a3be0d14 @   54 : bf                Star4 
  949 E> 0x3711a3be0d15 @   55 : 2d f6 01 02       LdaNamedProperty r4, [1], [2]
         0x3711a3be0d19 @   59 : c0                Star3 
         0x3711a3be0d1a @   60 : 13 04             LdaConstant [4]
         0x3711a3be0d1c @   62 : be                Star5 
         0x3711a3be0d1d @   63 : 13 05             LdaConstant [5]
         0x3711a3be0d1f @   65 : bd                Star6 
  949 E> 0x3711a3be0d20 @   66 : 5e f7 f6 f5 f4 0d CallProperty2 r3, r4, r5, r6, [13]
// 通过Buffer.from(常量池4, "hex") 将十六进制字符串转换为buffer
         0x3711a3be0d26 @   72 : c1                Star2 
 1056 S> 0x3711a3be0d27 @   73 : 17 02             LdaImmutableCurrentContextSlot [2]
// 这个应该是ccc函数
         0x3711a3be0d29 @   75 : c0                Star3 
         0x3711a3be0d2a @   76 : 19 fa f6          Mov r0, r4
         0x3711a3be0d2d @   79 : 19 f9 f5          Mov r1, r5
         0x3711a3be0d30 @   82 : 19 f8 f4          Mov r2, r6
 1063 E> 0x3711a3be0d33 @   85 : 5f f7 f6 03 0f    CallUndefinedReceiver r3, r4-r6, [15]
// 调用ccc函数，r3(r4, r5, r6) 即ccc(flag数组, 空白数组, 密文数组)
 1087 S> 0x3711a3be0d38 @   90 : a8                Return 
Constant pool (size = 6)
0x3711a3be0c69: [FixedArray] in OldSpace
 - map: 0x2546cb1c12c1 <Map>
 - length: 6
           0: 0x3adf78836561 <String[6]: #Buffer>
           1: 0x2546cb1c4999 <String[4]: #from>
           2: 0x2546cb1c4d41 <String[6]: #length>
           3: 0x3adf78836c31 <String[5]: #alloc>
           4: 0x3711a3be0bb1 <String[86]: #3edd7925cd6e04ab44f25bef57bc53bd20b74b8c11f893090fdcdfddad0709100100fe6a9230333234fbae>
           5: 0x3adf78838741 <String[3]: #hex>
```

最后是ccc函数

```javascript
// ccc函数
  173 S> 0x3711a3be1216 @    0 : 00 0d aa 00       LdaSmi.Wide [170]
         0x3711a3be121a @    4 : c3                Star0 
// r0 = 170
  188 S> 0x3711a3be121b @    5 : 0c                LdaZero 
  189 E> 0x3711a3be121c @    6 : 23 00 00          StaGlobal [0], [0]
  194 S> 0x3711a3be121f @    9 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be1222 @   12 : c1                Star2 
         0x3711a3be1223 @   13 : 0d 13             LdaSmi [19]
  194 E> 0x3711a3be1225 @   15 : 6c f8 04          TestLessThan r2, [4]
// 循环19个字符
         0x3711a3be1228 @   18 : 98 2a             JumpIfFalse [42] (0x3711a3be1252 @ 60)
  274 S> 0x3711a3be122a @   20 : 21 00 02          LdaGlobal [0], [2]
  273 E> 0x3711a3be122d @   23 : 2f 03 06          LdaKeyedProperty a0, [6]
  265 E> 0x3711a3be1230 @   26 : 38 fa 08          Add r0, [8]    
// a0[i]+170
  277 E> 0x3711a3be1233 @   29 : 44 33 09          AddSmi [51], [9]    
// a0[i]+170+51
  285 E> 0x3711a3be1236 @   32 : 00 4c ff 00 05 00 BitwiseAndSmi.Wide [255], [5]
         0x3711a3be123c @   38 : c3                Star0    
// r0 = (a0[i]+170+51) & 0xff
  305 S> 0x3711a3be123d @   39 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be1240 @   42 : c0                Star3
         0x3711a3be1241 @   43 : 0b fa             Ldar r0
  308 E> 0x3711a3be1243 @   45 : 34 04 f7 0a       StaKeyedProperty a1, r3, [10]    
// a1[i] = (a0[i]+51) & 255
  200 S> 0x3711a3be1247 @   49 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be124a @   52 : 50 0c             Inc [12]
// i加上一个单元
  200 E> 0x3711a3be124c @   54 : 23 00 00          StaGlobal [0], [0]
  183 E> 0x3711a3be124f @   57 : 88 30 00          JumpLoop [48], [0] (0x3711a3be121f @ 9)
  337 S> 0x3711a3be1252 @   60 : 0d 55             LdaSmi [85]
         0x3711a3be1254 @   62 : c2                Star1   
// r1 = 85
  352 S> 0x3711a3be1255 @   63 : 0d 13             LdaSmi [19]
  353 E> 0x3711a3be1257 @   65 : 23 00 00          StaGlobal [0], [0]
  359 S> 0x3711a3be125a @   68 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be125d @   71 : c1                Star2
         0x3711a3be125e @   72 : 0d 2b             LdaSmi [43]
  359 E> 0x3711a3be1260 @   74 : 6c f8 0d          TestLessThan r2, [13]
// 从19循环到43
         0x3711a3be1263 @   77 : 98 34             JumpIfFalse [52] (0x3711a3be1297 @ 129)
  383 S> 0x3711a3be1265 @   79 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be1268 @   82 : c0                Star3
  403 E> 0x3711a3be1269 @   83 : 21 00 02          LdaGlobal [0], [2]
  402 E> 0x3711a3be126c @   86 : 2f 03 10          LdaKeyedProperty a0, [16]
  394 E> 0x3711a3be126f @   89 : 38 f9 0f          Add r1, [15]    
// a0[i]+r1
  407 E> 0x3711a3be1272 @   92 : 00 4c ff 00 0e 00 BitwiseAndSmi.Wide [255], [14]
  386 E> 0x3711a3be1278 @   98 : 34 04 f7 12       StaKeyedProperty a1, r3, [18]   
//a1[i] = a0[i]
  442 S> 0x3711a3be127c @  102 : 21 00 02          LdaGlobal [0], [2]
  441 E> 0x3711a3be127f @  105 : 2f 04 16          LdaKeyedProperty a1, [22]
  436 E> 0x3711a3be1282 @  108 : 3f f9 15          BitwiseXor r1, [21]
// a1[i] ^ r1
  446 E> 0x3711a3be1285 @  111 : 00 4c ff 00 14 00 BitwiseAndSmi.Wide [255], [20]
         0x3711a3be128b @  117 : c2                Star1  
// r1 = a1[i] ^ r1
  365 S> 0x3711a3be128c @  118 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be128f @  121 : 50 18             Inc [24]    
// i增加一个单元
  365 E> 0x3711a3be1291 @  123 : 23 00 00          StaGlobal [0], [0]
  347 E> 0x3711a3be1294 @  126 : 88 3a 00          JumpLoop [58], [0] (0x3711a3be125a @ 68)
  531 S> 0x3711a3be1297 @  129 : 00 0d 9f 00       LdaSmi.Wide [159]
  540 E> 0x3711a3be129b @  133 : 6a f9 19          TestEqual r1, [25]
         0x3711a3be129e @  136 : 98 32             JumpIfFalse [50] (0x3711a3be12d0 @ 186)
  563 S> 0x3711a3be12a0 @  138 : 0c                LdaZero 
  564 E> 0x3711a3be12a1 @  139 : 23 00 00          StaGlobal [0], [0]
  569 S> 0x3711a3be12a4 @  142 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be12a7 @  145 : c1                Star2
         0x3711a3be12a8 @  146 : 0d 2b             LdaSmi [43]
// 循环43，比对密文和加密后的flag
  569 E> 0x3711a3be12aa @  148 : 6c f8 1a          TestLessThan r2, [26]
         0x3711a3be12ad @  151 : 98 21             JumpIfFalse [33] (0x3711a3be12ce @ 184)
  604 S> 0x3711a3be12af @  153 : 21 00 02          LdaGlobal [0], [2]
  603 E> 0x3711a3be12b2 @  156 : 2f 05 1b          LdaKeyedProperty a2, [27]    // a2[i]
         0x3711a3be12b5 @  159 : c1                Star2 
  614 E> 0x3711a3be12b6 @  160 : 21 00 02          LdaGlobal [0], [2]
  613 E> 0x3711a3be12b9 @  163 : 2f 04 1d          LdaKeyedProperty a1, [29]    // a1[i]
  607 E> 0x3711a3be12bc @  166 : 6a f8 1f          TestEqual r2, [31]
         0x3711a3be12bf @  169 : 97 04             JumpIfTrue [4] (0x3711a3be12c3 @ 173)
  636 S> 0x3711a3be12c1 @  171 : 12                LdaFalse 
  649 S> 0x3711a3be12c2 @  172 : a8                Return 
  575 S> 0x3711a3be12c3 @  173 : 21 00 02          LdaGlobal [0], [2]
         0x3711a3be12c6 @  176 : 50 20             Inc [32]
  575 E> 0x3711a3be12c8 @  178 : 23 00 00          StaGlobal [0], [0]
  558 E> 0x3711a3be12cb @  181 : 88 27 00          JumpLoop [39], [0] (0x3711a3be12a4 @ 142)
  682 S> 0x3711a3be12ce @  184 : 11                LdaTrue 
  694 S> 0x3711a3be12cf @  185 : a8                Return 
  705 S> 0x3711a3be12d0 @  186 : 12                LdaFalse 
  718 S> 0x3711a3be12d1 @  187 : a8                Return 
Constant pool (size = 1)
0x3711a3be11c9: [FixedArray] in OldSpace
 - map: 0x2546cb1c12c1 <Map>
 - length: 1
           0: 0x2546cb1c9159 <String[1]: #i>
```

所以可以容易知道加密算法分为两部分，第一部分是0~18字节的算法

```javascript
let x = 170
for(let i = 0; i < 19; i++) {
    newbuf[i] = (flag[i] + x + 51) & 0xff;
    x = newbuf[i];
}
```

第二部分是剩下的字节

```javascript
let y = 85;
for(let i = 19; i < 43; i++) {
    newbuf[i] = (flag[i] + y) & 0xff;
    y = newbuf[i] ^ y;
}
```

所以很容易得到逆向脚本

```python
string = "3edd7925cd6e04ab44f25bef57bc53bd20b74b8c11f893090fdcdfddad0709100100fe6a9230333234fbae"
li = []
for i in range(0, len(string), 2):
    li.append(int(string[i]+string[i+1], 16))

x = 170
for i in range(0, 19):
    print(chr((li[i]-x-51)&0xff), end="")
    x = li[i]
    
y = 85
for i in range(19, len(li)):
    print(chr((li[i]-y)&0xff), end="")
    y = li[i]^y
```

![642c7060efcfe7f45f75b7986e8deb87](images/642c7060efcfe7f45f75b7986e8deb87.png)  

### 0x02 参考文章

[理解 V8 的字节码「译」 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/28590489)
