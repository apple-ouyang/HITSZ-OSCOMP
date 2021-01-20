[不上班行不行](https://www.cnblogs.com/startkey/p/10678173.html)

[[差量更新系列1\]BSDiff算法学习笔记_add_ada的博客-CSDN博客_bsdiff算法原理](https://blog.csdn.net/add_ada/article/details/51232889)

# （1）BSDiff算法包含哪些基本步骤？

**步骤1.**是所有差量更新算法的瓶颈，时间复杂度为O(nlogn),空间复杂度为O(n)，n为old文件的长度。BSDiff采用 Faster suffix sorting方法获得一个字典序，使用了类似于快速排序的二分思想，使用了bucket，I，V三个辅助数组。最终得到一个数组I，记录了以前缀分组的各个字符串组的最后一个字符串在old中的开始位置

**步骤2.**是BSDiff产生patch包的核心部分，详细描述如下：

![1](https://img-blog.csdn.net/20160424124812650?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![2](https://img-blog.csdn.net/20160424124859572?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![3](https://img-blog.csdn.net/20160424125102547?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**步骤3.**将diff string 和extrastring 以及相应的控制字用zip压缩成一个patch包。

![4](https://img-blog.csdn.net/20160424125148292?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

可以看出在用zip压缩之前的patch包是没有节约任何字符的，但diff strings可以被高效的压缩，故BSDiff是一个很依赖于压缩与解压的算法！

# （2）BSDiff算法的时间、空间复杂度分别是？并据此推断该算法的瓶颈；

BSDiff

时间复杂度 O(nlogn) 空间复杂度 O(n)

BSPatch

时间复杂度 O(n+m)  空间复杂度 O(n+m)

字典序排序算法

压缩算法

# （3）为什么BSDiff算法不适用于压缩文件的差分以及小内存设备？

压缩文件的结构依赖于不同的压缩算法, 压缩文件的new和old文件有重复的部分可能较少

bsdiff算法会产生较大的空间占用

# （4）目前是否存在适用于压缩文件和小内存设备的差分算法？它们和BSDiff算法相比，做了哪些改进？

[一种小内存设备系统升级的差分算法的制作方法](http://www.xjishu.com/zhuanli/55/201811416681.html)

# （5）尝试提出BSDiff算法的优化思路和优化方法；

找更好的压缩算法

字典排序更好

[More on bsdiff and delta compression](http://richg42.blogspot.com/2015/11/more-on-bsdiff.html)

# （6）出错回滚的原理、常用方法分别是？对比这些方法的优缺点。

[数据库原理--第十章：数据库恢复技术](https://godway.work/2019/06/17/Database10/)

[『浅入深出』MySQL 中事务的实现 - 面向信仰编程](https://draveness.me/mysql-transaction/)

方法1：数据转储（俗称备份）

定期将系统复制到其他存储介质上保存起来。转储操作十分耗费时间和资源，不能频繁进行

转储可以分为静态转储和动态转储

- 静态转储：在操作系统没有进程运行时进行的转储操作，转储期间不能对操作系统有任何的修改。
	- 得到的是一个一致性副本
	- 不能热插拔
- 动态转储：转储期间允许操作系统进行操作
	- 必须把转储期间操作系统进行的活动记录下来，建立日志文件

方法2：日志文件

对于以记录为单位的日志文件，其需要登记的内容包括：

- 各个事务的开始标记
- 各个事务的结束标记
- 各个事务的所有更新操作

日志文件的作用：

- 事务故障恢复和系统故障恢复必须用日志文件
- 在动态转储方式中必须建立日志文件，后备副本和日志文件结合起来才能有效的恢复数据库
- 在静态转储中也可以建立日志文件，用于加速数据库恢复过程

为保证数据库是可恢复的，登记日志文件的两条原则：

- **登记的次序严格按并发事务执行的时间次序**
- **必须先写日志文件，后写数据库**

恢复：UNDO list和REDO list， 时间戳

升级之前进行备份。将patch包的升级过程记录下来







# 算法解析

查看代码之后可以知道`bsdiff_internal`函数是关键。

```C++
static int bsdiff_internal(const struct bsdiff_request req)
{
	// I: 已经排好的字典序
	int64_t *I, *V;
	int64_t scan, pos, len;
	// lastoffset: lastoffset为new和old的偏移量,如果在old中的内容A在new中可以找到，
	// 			   而且A+lastoffset=new中的A，则认为old和new中的A相同。
	int64_t lastscan, lastpos, lastoffset;
	// oldscore代表相同内容的len
	// scsc代表new中开始和old中比较是否相同开始的位置
	int64_t oldscore, scsc;
	// 而old中开始的位置是scsc+lastoffset。lenf代表扩展前缀，lenb代表扩展后缀。
	int64_t s, Sf, lenf, Sb, lenb;
	int64_t overlap, Ss, lens;
	int64_t i;
	uint8_t *buffer;
	uint8_t buf[8 * 3];

	if ((V = req.stream->malloc((req.oldsize + 1) * sizeof(int64_t))) == NULL)
		return -1;
	I = req.I;
	// 任务1：生成后缀数组
	qsufsort(I, V, req.old, req.oldsize);
	req.stream->free(V);

	buffer = req.buffer;

	/* Compute the differences, writing ctrl as we go */
    // scan: new中要查询的字符
	scan = 0;
    // len: 匹配的长度
	len = 0;
    // pos: 代表old中相匹配的字符
	pos = 0;
	lastscan = 0;
	lastpos = 0;
	lastoffset = 0;
    // 任务2：循环处理新文件数据，找到代码操作导致二进制数据相差字节多于 8bytes 的偏移点
	while (scan < req.newsize)
	{
		oldscore = 0;

		for (scsc = scan += len; scan < req.newsize; scan++)
		{
            // 新版本文件和老版本文件都从数据开头开始，通过二分法，在整个后缀数组 I 中找到与新版本数据匹配最长的长度 len 和数组编号 pos。
            // 返回的数组编号即为在老版本文件中的偏移。
			len = search(I, req.old, req.oldsize, req.new + scan, 
                         req.newsize - scan, 0, req.oldsize, &pos);
			// 计算出当前偏移的 old 数据与 new 数据相同的字节个数，再与 len 比较
			for (; scsc < scan + len; scsc++)
				if ((scsc + lastoffset < req.oldsize) &&
					(req.old[scsc + lastoffset] == req.new[scsc]))
					oldscore++;
			// 如果相差小于 8 则继续 for 循环。
            // 相差小于 8，可以认为插入的数据较少，没必要切换 old 数据的 offset，每切换一次就需要进行一次 diff 和 extra 处理。
			if (((len == oldscore) && (len != 0)) ||
				(len > oldscore + 8))
				break;

			if ((scan + lastoffset < req.oldsize) &&
				(req.old[scan + lastoffset] == req.new[scan]))
				oldscore--;
		};
		// 对上一个位置到新位置之间的数据进行处理
		if ((len != oldscore) || (scan == req.newsize))
		{
			s = 0;
			Sf = 0;
			lenf = 0;
            // 计算 diff string 的长度
			for (i = 0; (lastscan + i < scan) && (lastpos + i < req.oldsize);)
			{
				if (req.old[lastpos + i] == req.new[lastscan + i])
					s++;
				i++;

                // 由于任务2中找的是差异较大的点，因此差异较大的部分就是该段数据的末尾数据，从头开始比较，通过以下判断可以近似找出类似的最长字符串。
                /** Sf*2-lenf 和 s*2-i 都是等式： a*2 - b。
* Sf*2-lenf可以理解为 上一组s和i，s*2-i计算出的值。
* s*2 - i, i和s的增长步长都为1，也就是i走两步，s走一步就可以维持s*2 - i结果不变，
如果结果要增加，也就是s增加的频率要>50%，即后续增加的数据超过50%的数据需要是相等的。*/
				if (s * 2 - i > Sf * 2 - lenf)
				{
					Sf = s;
					lenf = i;
				};
			};
			// 获取与下一段数据近似相同数据的长度
			lenb = 0;
			if (scan < req.newsize)
			{
				s = 0;
				Sb = 0;
				for (i = 1; (scan >= lastscan + i) && (pos >= i); i++)
				{
					if (req.old[pos - i] == req.new[scan - i])
						s++;
					if (s * 2 - i > Sb * 2 - lenb)
					{
						Sb = s;
						lenb = i;
					};
				};
			};
            /* 上面lenf的值即为diff string的长度，该段数据剩余部分即可认为是extra string，
            这样长度相减即可获得extra string的长度;但是保存的diff string是新老数据的相差值，
            可以更好的被压缩，而extra string保存的就是原始数据，压缩层度不高，
            为了减少extea string的长度，采取了将部分extra string与下一段数据近似相同的数据遗留下来，
            在下一段数据可以充当diff string。
            scan等于newsize，就没有下一段数据了，因此这里需要判断scan小于newsize。
            */
            
			// 上面两段数据可能重叠，处理重叠
			if (lastscan + lenf > scan - lenb)
			{
				overlap = (lastscan + lenf) - (scan - lenb);
				s = 0;
				Ss = 0;
				lens = 0;
				for (i = 0; i < overlap; i++)
				{
					if (req.new[lastscan + lenf - overlap + i] ==
						req.old[lastpos + lenf - overlap + i])
						s++;
					if (req.new[scan - lenb + i] ==
						req.old[pos - lenb + i])
						s--;
					if (s > Ss)
					{
						Ss = s;
						lens = i + 1;
					};
				};

				lenf += lens - overlap;
				lenb -= lens;
			};
			// 如果上面两段数据存在重叠，说明不存在extra string，重叠部分可以划分给当前diff string，也可以划分给下一段diff string，这里根据相似数据分布获取到lens值。
			offtout(lenf, buf);
			offtout((scan - lenb) - (lastscan + lenf), buf + 8);
			offtout((pos - lenb) - (lastpos + lenf), buf + 16);

			/* Write control data */
			if (writedata(req.stream, buf, sizeof(buf)))
				return -1;

			/* Write diff data */
			for (i = 0; i < lenf; i++)
				buffer[i] = req.new[lastscan + i] - req.old[lastpos + i];
			if (writedata(req.stream, buffer, lenf))
				return -1;

			/* Write extra data */
			for (i = 0; i < (scan - lenb) - (lastscan + lenf); i++)
				buffer[i] = req.new[lastscan + lenf + i];
			if (writedata(req.stream, buffer, (scan - lenb) - (lastscan + lenf)))
				return -1;
			// 保存偏移
			lastscan = scan - lenb;
			lastpos = pos - lenb;
			lastoffset = pos - scan;
            /*
            lenf值即为diff string的长度；
            (scan-lenb)-(lastscan+lenf)即为extra string的长度。
            diff string保存的是相差值，可以很好被压缩。
            */
		};
	};

	return 0;
}
```
