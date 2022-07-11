---
layout: post
title: Golang中的gzip
categories: Programming
---

> 尝试将 struct 序列化为 json 后走 TCP 传输，在 bytes stream 上利用 json 实现 packets stream。  
> 联系 WebSocket 很容易想到可以对 json 文本进行压缩来做传输优化，就选择了广泛使用的流式压缩 gzip 来对 json 进行处理。  
> 写 Golang 测试的时候发现 gzip 压缩结果的 base64 编码中出现了大量重复的 `AAAA`，长度也比命令行中直接 `echo RAW_JSON_TEXT | gzip | base64` 得到的编码结果长了许多。  
> 好奇是什么原因导致了 Golang 的 gzip 与命令行里的 gzip 压缩结果出现这种差异，也借这个机会顺便了解一下 gzip 相关的知识。  

<!-- more -->

## original data
原始数据定义为 `var strList = []string{"hello", "world", "0123", "456", "789"}`，使用 `json.Encoder` 分五次 `Encode()`，最终压缩前的 json 文本为：
```
"hello"
"world"
"0123"
"456"
"789"
```

## gzip result
使用三种方式对原始数据进行压缩，再将压缩结果转换为 base64 表示，得到：

| method      | result length | result data                                                                                                                    |
| ----------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| cliGzip     | 68            | `H4sIAAAAAAAAA1PKSM3JyVfiUirPL8pJAdIGhkbGQMrE1AxImltYKnEBAHZoqCcjAAAA`                                                         |
| goGzipFlush | 124           | `H4sIAAAAAAAA/wAAAP//UspIzcnJV+ICAAAA//9SKs8vyklR4gIAAAD//1IyMDQyVuICAAAA//9SMjE1U+ICAAAA//9SMrewVOICAAAA//8BAAD//3ZoqCcjAAAA` |
| goGzip      | 80            | `H4sIAAAAAAAA/1LKSM3JyVfiUirPL8pJUeJSMjA0MlbiUjIxNVPiUjK3sFTiAgQAAP//dmioJyMAAAA=`                                             |

具体执行代码与详细结果说明如下：

### gzip in cli
```go
func cliGzip() string {
	buf := new(bytes.Buffer)
	enc := json.NewEncoder(buf)
	for _, str := range strList {
		err := enc.Encode(str)
		panicIf(err)
	}

	cmd := exec.Command("gzip")
	cmd.Stdin = buf
	res, err := cmd.CombinedOutput()
	panicIf(err)
	return base64.StdEncoding.EncodeToString(res)
}

// base64 output (length 68):
//   H4sIAAAAAAAAA1PKSM3JyVfiUirPL8pJAdIGhkbGQMrE1AxImltYKnEBAHZoqCcjAAAA
```

#### binary data
```
00000000: 1f8b 0800 0000 0000 0003 53ca 48cd c9c9  ..........S.H...
00000010: 57e2 522a cf2f ca49 01d2 0686 46c6 40ca  W.R*./.I....F.@.
00000020: c4d4 0c48 9a5b 582a 7101 0076 68a8 2723  ...H.[X*q..vh.'#
00000030: 0000 00                                  ...
```
{::options parse_block_html="true" /}
<details><summary markdown="span">gzip format details</summary>

* `1f8b`: ID1 ID2 (IDentification) (gzip magic number)
* `08`: CM (Compression Method) (deflate)
* `00`: FLG (FLaGs) (unset)
* `0000 0000`: MTIME (Modification TIME) (unset)
* `00`: XFL (eXtra FLags) (unset)
* `03`: OS (Operating System) (Unix)
* `53ca 48cd c9c9 57e2 522a cf2f ca49 01d2 0686 46c6 40ca c4d4 0c48 9a5b 582a 7101 00`: DATA (compressed blocks)
* `76 68a8 27`: CRC32 (CRC-32)
* `23 0000 00`: ISIZE (Input SIZE)

</details>
{::options parse_block_html="false" /}

#### compressed blocks
```
00000000: ________ ________ ________ ________ ________ ________  ......
00000006: ________ ________ ________ ________ 01010011 11001010  ....S.
0000000c: 01001000 11001101 11001001 11001001 01010111 11100010  H...W.
00000012: 01010010 00101010 11001111 00101111 11001010 01001001  R*./.I
00000018: 00000001 11010010 00000110 10000110 01000110 11000110  ....F.
0000001e: 01000000 11001010 11000100 11010100 00001100 01001000  @....H
00000024: 10011010 01011011 01011000 00101010 01110001 00000001  .[X*q.
0000002a: 00000000 ________ ________ ________ ________ ________  .vh.'#
00000030: ________ ________ ________                             ...
```
{::options parse_block_html="true" /}
<details><summary markdown="span">deflate format details</summary>

* `1`: BFINAL (last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `10011000`: Code (104 `h`)
* `10010101`: Code (101 `e`)
* `10011100`: Code (108 `l`)
* `10011100`: Code (108 `l`)
* `10011111`: Code (111 `o`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `01010010`: Code (34 `"`)
* `10100111`: Code (119 `w`)
* `10011111`: Code (111 `o`)
* `10100010`: Code (114 `r`)
* `10011100`: Code (108 `l`)
* `10010100`: Code (100 `d`)
* `0000001`: Length (3)
* `00101_1`: Distance (8)
* `01100000`: Code (48 `0`)
* `01100001`: Code (49 `1`)
* `01100010`: Code (50 `2`)
* `01100011`: Code (51 `3`)
* `0000001`: Length (3)
* `00101_0`: Distance (7)
* `01100100`: Code (52 `4`)
* `01100101`: Code (53 `5`)
* `01100110`: Code (54 `6`)
* `0000001`: Length (3)
* `00100_1`: Distance (6)
* `01100111`: Code (55 `7`)
* `01101000`: Code (56 `8`)
* `01101001`: Code (57 `9`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `0000000`: PADDING (until byte boundary)

</details>
{::options parse_block_html="false" /}

### gzip in go (flush after jsonEncode)
```go
func goGzipFlush() string {
	buf := new(bytes.Buffer)
	w := gzip.NewWriter(buf)
	w.Flush() // send gzip header immediately
	enc := json.NewEncoder(w)
	for _, str := range strList {
		err := enc.Encode(str)
		panicIf(err)
		w.Flush() // useful in network connection
	}
	w.Close()
	return base64.StdEncoding.EncodeToString(buf.Bytes())
}

// base64 output (length 124):
//   H4sIAAAAAAAA/wAAAP//UspIzcnJV+ICAAAA//9SKs8vyklR4gIAAAD//1IyMDQyVuICAAAA//9SMjE1U+ICAAAA//9SMrewVOICAAAA//8BAAD//3ZoqCcjAAAA
```

#### binary data
```
00000000: 1f8b 0800 0000 0000 00ff 0000 00ff ff52  ...............R
00000010: ca48 cdc9 c957 e202 0000 00ff ff52 2acf  .H...W.......R*.
00000020: 2fca 4951 e202 0000 00ff ff52 3230 3432  /.IQ.......R2042
00000030: 56e2 0200 0000 ffff 5232 3135 53e2 0200  V.......R215S...
00000040: 0000 ffff 5232 b7b0 54e2 0200 0000 ffff  ....R2..T.......
00000050: 0100 00ff ff76 68a8 2723 0000 00         .....vh.'#...
```
{::options parse_block_html="true" /}
<details><summary markdown="span">gzip format details</summary>

* `1f8b`: ID1 ID2 (IDentification) (gzip magic number)
* `08`: CM (Compression Method) (deflate)
* `00`: FLG (FLaGs) (unset)
* `0000 0000`: MTIME (Modification TIME) (unset)
* `00`: XFL (eXtra FLags) (unset)
* `ff`: OS (Operating System) (unknown)
* `0000 00ff ff52 ca48 cdc9 c957 e202 0000 00ff ff52 2acf 2fca 4951 e202 0000 00ff ff52 3230 3432 56e2 0200 0000 ffff 5232 3135 53e2 0200 0000 ffff 5232 b7b0 54e2 0200 0000 ffff 0100 00ff ff`: DATA (compressed blocks)
* `76 68a8 27`: CRC32 (CRC-32)
* `23 0000 00`: ISIZE (Input SIZE)

</details>
{::options parse_block_html="false" /}

#### compressed blocks
```
00000000: ________ ________ ________ ________ ________ ________  ......
00000006: ________ ________ ________ ________ 00000000 00000000  ......
0000000c: 00000000 11111111 11111111 01010010 11001010 01001000  ...R.H
00000012: 11001101 11001001 11001001 01010111 11100010 00000010  ...W..
00000018: 00000000 00000000 00000000 11111111 11111111 01010010  .....R
0000001e: 00101010 11001111 00101111 11001010 01001001 01010001  *./.IQ
00000024: 11100010 00000010 00000000 00000000 00000000 11111111  ......
0000002a: 11111111 01010010 00110010 00110000 00110100 00110010  .R2042
00000030: 01010110 11100010 00000010 00000000 00000000 00000000  V.....
00000036: 11111111 11111111 01010010 00110010 00110001 00110101  ..R215
0000003c: 01010011 11100010 00000010 00000000 00000000 00000000  S.....
00000042: 11111111 11111111 01010010 00110010 10110111 10110000  ..R2..
00000048: 01010100 11100010 00000010 00000000 00000000 00000000  T.....
0000004e: 11111111 11111111 00000001 00000000 00000000 11111111  ......
00000054: 11111111 ________ ________ ________ ________ ________  .vh.'#
0000005a: ________ ________ ________                             ...
```
{::options parse_block_html="true" /}
<details><summary markdown="span">deflate format details</summary>

* `0`: BFINAL (not last block)
* `00`: BTYPE (no compression)
* `00000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN
* `0`: BFINAL (not last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `10011000`: Code (104 `h`)
* `10010101`: Code (101 `e`)
* `10011100`: Code (108 `l`)
* `10011100`: Code (108 `l`)
* `10011111`: Code (111 `o`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `0`: BFINAL (not last block)
* `00`: BTYPE (no compression)
* `000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN
* `0`: BFINAL (not last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `10100111`: Code (119 `w`)
* `10011111`: Code (111 `o`)
* `10100010`: Code (114 `r`)
* `10011100`: Code (108 `l`)
* `10010100`: Code (100 `d`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `0`: BFINAL (not last block)
* `00`: BTYPE (no compression)
* `000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN
* `0`: BFINAL (not last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `01100000`: Code (48 `0`)
* `01100001`: Code (49 `1`)
* `01100010`: Code (50 `2`)
* `01100011`: Code (51 `3`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `0`: BFINAL (not last block)
* `00`: BTYPE (no compression)
* `000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN
* `0`: BFINAL (not last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `01100100`: Code (52 `4`)
* `01100101`: Code (53 `5`)
* `01100110`: Code (54 `6`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `0`: BFINAL (not last block)
* `00`: BTYPE (no compression)
* `000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN
* `0`: BFINAL (not last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `01100111`: Code (55 `7`)
* `01101000`: Code (56 `8`)
* `01101001`: Code (57 `9`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `0`: BFINAL (not last block)
* `00`: BTYPE (no compression)
* `000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN
* `1`: BFINAL (last block)
* `00`: BTYPE (no compression)
* `00000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN

</details>
{::options parse_block_html="false" /}

### gzip in go (no manual flush)
```go
func goGzip() string {
	buf := new(bytes.Buffer)
	w := gzip.NewWriter(buf)
	enc := json.NewEncoder(w)
	for _, str := range strList {
		err := enc.Encode(str)
		panicIf(err)
	}
	w.Close()
	return base64.StdEncoding.EncodeToString(buf.Bytes())
}

// base64 output (length 80):
//   H4sIAAAAAAAA/1LKSM3JyVfiUirPL8pJUeJSMjA0MlbiUjIxNVPiUjK3sFTiAgQAAP//dmioJyMAAAA=
```

#### binary data
```
00000000: 1f8b 0800 0000 0000 00ff 52ca 48cd c9c9  ..........R.H...
00000010: 57e2 522a cf2f ca49 51e2 5232 3034 3256  W.R*./.IQ.R2042V
00000020: e252 3231 3553 e252 32b7 b054 e202 0400  .R215S.R2..T....
00000030: 00ff ff76 68a8 2723 0000 00              ...vh.'#...
```
{::options parse_block_html="true" /}
<details><summary markdown="span">gzip format details</summary>

* `1f8b`: ID1 ID2 (IDentification) (gzip magic number)
* `08`: CM (Compression Method) (deflate)
* `00`: FLG (FLaGs) (unset)
* `0000 0000`: MTIME (Modification TIME) (unset)
* `00`: XFL (eXtra FLags) (unset)
* `ff`: OS (Operating System) (unknown)
* `52ca 48cd c9c9 57e2 522a cf2f ca49 51e2 5232 3034 3256 e252 3231 3553 e252 32b7 b054 e202 0400 00ff ff`: DATA (compressed blocks)
* `76 68a8 27`: CRC32 (CRC-32)
* `23 0000 00`: ISIZE (Input SIZE)

</details>
{::options parse_block_html="false" /}

#### compressed blocks
```
00000000: ________ ________ ________ ________ ________ ________  ......
00000006: ________ ________ ________ ________ 01010010 11001010  ....R.
0000000c: 01001000 11001101 11001001 11001001 01010111 11100010  H...W.
00000012: 01010010 00101010 11001111 00101111 11001010 01001001  R*./.I
00000018: 01010001 11100010 01010010 00110010 00110000 00110100  Q.R204
0000001e: 00110010 01010110 11100010 01010010 00110010 00110001  2V.R21
00000024: 00110101 01010011 11100010 01010010 00110010 10110111  5S.R2.
0000002a: 10110000 01010100 11100010 00000010 00000100 00000000  .T....
00000030: 00000000 11111111 11111111 ________ ________ ________  ...vh.
00000036: ________ ________ ________ ________ ________           '#...
```
{::options parse_block_html="true" /}
<details><summary markdown="span">deflate format details</summary>

* `0`: BFINAL (not last block)
* `01`: BTYPE (fixed Huffman codes)
* `01010010`: Code (34 `"`)
* `10011000`: Code (104 `h`)
* `10010101`: Code (101 `e`)
* `10011100`: Code (108 `l`)
* `10011100`: Code (108 `l`)
* `10011111`: Code (111 `o`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `01010010`: Code (34 `"`)
* `10100111`: Code (119 `w`)
* `10011111`: Code (111 `o`)
* `10100010`: Code (114 `r`)
* `10011100`: Code (108 `l`)
* `10010100`: Code (100 `d`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `01010010`: Code (34 `"`)
* `01100000`: Code (48 `0`)
* `01100001`: Code (49 `1`)
* `01100010`: Code (50 `2`)
* `01100011`: Code (51 `3`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `01010010`: Code (34 `"`)
* `01100100`: Code (52 `4`)
* `01100101`: Code (53 `5`)
* `01100110`: Code (54 `6`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `01010010`: Code (34 `"`)
* `01100111`: Code (55 `7`)
* `01101000`: Code (56 `8`)
* `01101001`: Code (57 `9`)
* `01010010`: Code (34 `"`)
* `00111010`: Code (10 `\n`)
* `0000000`: End Of Block (0)
* `1`: BFINAL (last block)
* `00`: BTYPE (no compression)
* `000`: PADDING (until byte boundary)
* `00000000 00000000`: LEN
* `11111111 11111111`: NLEN

</details>
{::options parse_block_html="false" /}

## analysis
* 单个 gzip 文件的内容可以简单划分为 header，body，footer 三部分，header 存储有部分压缩参数/原始文件信息，footer 主要是检查原始数据用的校验码，实际压缩数据都保存在 body 中。
* gzip header 中的 CM 标记了压缩时使用的压缩方法，RFC 1952 中表示当前只有 deflate (CM=8) 一种，0-7 的取值尚未被使用。
* deflate 的每个 block 可以指定不同的压缩方式，包括“不压缩/固定霍夫曼编码/动态霍夫曼编码”三种，程序通常会在预估不同压缩结果后选择相对优化的方式。
* RFC 1951 中 Packing into bytes 讲的 bits 打包顺序有点绕，三条规则可以解释为：
  1. 向 block bytes 内写入时，总是从 byte 的最低有效位开始；
  2. 读取 data elements 时，除了霍夫曼编码外都是从 data element 的最低有效位开始；
  3. 读取 data elements 时，霍夫曼编码是从 data element 的最高有效位开始。
  
  以 cliGzip 的压缩结果中 compressed blocks 的前两个字节 `53ca` 为例：
  ```
  01010011 11001010
         ^
         BFINAL // 第一个字节首位是 `1`，对应的 BFINAL 含义是当前 block 为最后一个 block；
  01010011 11001010
        ^
        BTYPE LSB // 第一个字节第二位是 `1`，对应的 BTYPE 最低位为 `1`；
  01010011 11001010
       ^
       BTYPE MSB  // 第一个字节第三位是 `0`，对应的 BTYPE 最高位为 `0`，所以 BTYPE 为 `01`，表示当前 block 使用固定霍夫曼编码；
  01010011 11001010
      ^
      Code Bit 1 (MSB) // 第一个字节第四位是 `0`，对应的霍夫曼编码最高位为 `0`，累计为 `0`；
  ...skip...
  01010011 11001010
  ^
  Code Bit 5 // 第一个字节第八位是 `0`，对应的霍夫曼编码第五位为 `0`，累计为 `01010`；
  01010011 11001010
                  ^
                  Code Bit 6 // 第二个字节首位是 `0`，对应的霍夫曼编码第六位为 `0`，累计为 `010100`；
  ...skip...
  01010011 11001010
                ^
                Code Bit 8 // 第二个字节第三位是 `0`，对应的霍夫曼编码第八位为 `0`，累计为 `01010010`，在固定霍夫曼编码表中对应 ASCII 34 `"`；
  ```

## comparison
* cliGzip，goGzipFlush 和 goGzip 三种方式得到的 gzip 压缩结果中 header 和 footer 差别不大。因为是直接对数据流进行压缩，所以 header 中文件相关的 FLG 和 MTIME 都没有设置，只有 OS 标志有区别：在同样的 Linux 环境下 goGzipFlush 和 goGzip 将其设置为 unknown，而 cliGzip 会将其设置为 Unix。
* 三种结果的主要区别在 gzip 的 body 部分，虽然都是 deflate compressed blocks，但 block 的组装方式大不相同：
  * cliGzip: body 中只有一个 block 且标记为 last block，使用固定霍夫曼编码压缩数据；
  * goGzipFlush: body 中包含多个 block，使用固定霍夫曼编码压缩数据，每次 Flush 操作都会新增一个空白 block 来对齐 bytes 边界，Close 操作会新增一个空白 block 并标记为 last block，每个空白 block 约占 4-5 个字节；
  * goGzip: body 中包含两个 block，第一个 block 使用固定霍夫曼编码压缩数据，第二个是 Close 操作新增的空白 block，被标记为 last block；

## conclusion
* base64 的 `AAAA` 转成二进制后就是一串 0，Golang 这边调用 Flush 等价于 zlib.Z_SYNC_FLUSH，会新增一个无压缩的空白 block 用于对齐 bytes 边界，重复调用 Flush 就会出现重复的 `AAAA`。
* 如果数据压缩时有明确的结束时间，也不要求压缩后立即写入，例如压缩文件，那么等待结束时的 Close 对齐 bytes 即可，无需手动 Flush。
* 如果将数据压缩用在了 TCP 连接上，既不确定连接什么时候结束，又期望压缩完一段数据后能立即发送出去，那么在每次压缩后都需要手动 Flush，因为基于字节流的 TCP 在传输时需要完整的 bytes。
* cliGzip 的 compressed blocks 中可以明显看到 `<length, distance>` 组合表明压缩生效，但 goGzip 的 compressed blocks 中没有看到预期效果，原因是 Golang 的 deflate 实现增大了压缩字串的[最小长度](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/compress/flate/deflate.go;l=43;drc=bc92fe8b340c302db970fe8a42c8fcf6ccf7141a)，使用 `aaaa,aaaa` 作为原始数据进行压缩可以验证。
* 原始数据内容长度较短，且无明显规律，因此三种方式都是自动使用了固定霍夫曼编码压缩数据。动态霍夫曼编码会在 block 的 BTYPE 之后写入预生成且优化过的霍夫曼编码，解析时会比固定霍夫曼编码稍复杂。
* gzip 相当于是在 deflate 的基础上加了一层壳，并提供了多个 gzip 文件直接拼接的格式支持。单纯用来压缩数据的话并不需要保存原始文件信息，也就不需要这层壳了，因此开头部分提到的 json stream 直接使用 Golang 的 `compress/flate` 即可。

---
## reference
* [RFC 1951: DEFLATE Compressed Data Format Specification](https://www.rfc-editor.org/rfc/rfc1951.html)
* [RFC 1952: GZIP File Format Specification](https://www.rfc-editor.org/rfc/rfc1952.html)
* [Stack Overflow: The structure of Deflate compressed block](https://stackoverflow.com/questions/32419086/the-structure-of-deflate-compressed-block/32419490#32419490)
