---
layout: post
title:  "对海量HTTP响应报文聚类分析——simhash算法的Python实现"
date:   2015-05-03 18:42:11
categories: BigData
comments: true
---

[上文](http://luoding.me/%E6%B5%B7%E9%87%8F%E6%95%B0%E6%8D%AE/2015/05/02/HttpMessageProc1/)我们对可能用到的相似性算法做了调研，最后暂定使用simhash作为本次相似性分类的算法。    
于是经过一番搜索后，决定不去造轮子，而是直接照搬现成的代码看看效果。  
但是毫不客气的说，现在不论是Google还是百度搜到的simhash的流程介绍都是有问题的（至少对新手来说），Python代码更是千篇一律，来源可能是github上的[python-hashes](https://github.com/sangelone/python-hashes/blob/master/hashes/simhash.py)。我不爽的原因很简单，因为我被坑了。。  
所以经过对算法流程重新学习，算法代码的review，各种内置hash算法的测试、以及不同hashbits和汉明距离阈值的测试。。。终于能用了。。。 \_(:3」∠)_   
算法流程的介绍与上文基本相同，如果不看也无所谓。我知道你们跟我一样复制代码就走人，出了问题才回头看流程的┑(￣Д ￣)┍  

#simhash算法流程

假设我有三个字符串，简单起见我们使用英文字符串，例子也是别人的例子  
* the cat sat on the mat  
* the cat sat on a mat  
* we all scream for ice cream  

####算法流程如下：  
1. 选择simhash的长度位数（二进制），请综合考虑存储成本以及数据集的大小，比如说64位  
2. 将simhash的各位初始化为0，得到64位的全0一维矩阵   
3. 提取原始文本中的特征，一般采用各种分词的方式。比如对于"the cat sat on the mat"，使用python常用的字符串分割split得到：['the', 'cat', 'sat', 'on', 'the', 'mat']  
4. 对分词后的word进行加权，这里方便起见，权重都为1.  
5. 计算各个word的hash值，这一步可以自由发挥，测试找到最快效果最好的算法。假设结果为[1001000]  
6. 对每个word的hash值的每一位乘以权重，如果是当前位为1则为结果为w，如果为0则结果为-w。因为我们权重为1，所以结果为[1,-1,-1,1,-1,-1,-1]  
7.  将所有word经过权重处理后的hash值按位相加，假设结果为[13,108,-22,-5,23,55]  
8.  对上一步的结果进行处理，如果该位大于等于0则记为1，如果小于0则记为0.  
9.  这样我们就得到了最终文本simhash值，结果为[1,1,0,0,1,1]  

####  算法流程图  
![simhash算法流程图](http://7xiprm.com1.z0.glb.clouddn.com/.1430579971731.png)  

#simhash算法匹配流程  
1. 得到两个文本的simhash值，假设为hash1（110011）和hash2（010001）  
2. 我们通过求汉明距离（Hamming distance）得到两hash 值的相似程度  
	hash1 ：  **1**100**1**1  
	hash2  ： **0**101**0**1  
	两个hash之间有2位不同，即汉明距离为2  
3. 通过测试得到汉明距离阈值，如果小于阈值则说明两文本为相似  


#simhash的Python实现  
代码是根据[sangelone](https://github.com/sangelone/python-hashes/blob/master/hashes/simhash.py) 的Python实现（也就是市面上流传最广的版本）修改而成  
修改如下：  
1. 减少了一些不必要的运算  
2. 增加的word权重设置  
3. 更加pythonic ；）  
4. 适用于multiprocessing的map操作  

```
#!/usr/bin/python
# coding=utf-8


class SimHash:
    def __init__(self, hashbits=128):
        self.hashbits = hashbits
        # 生成一个全1、长度为hashbits的二进制数
        self.all_1_bin = (1 << self.hashbits) - 1 

    def __call__(self, tokens):
        return self.get_hash(tokens)

    # 生成simhash值
    def get_hash(self, tokens):
        v = [0] * self.hashbits
        for token_hash, weight in self._proc_list_weight(tokens):
            for i in xrange(self.hashbits):
                bitmask = 1 << i
                if token_hash & bitmask:
                    v[i] += weight  # 查看当前bit位是否为1,是的话将该位+w
                else:
                    v[i] -= weight  # 否则的话,该位-w
        fingerprint = 0
        for i in xrange(self.hashbits):
            if v[i] >= 0:
                fingerprint += 1 << i
        return fingerprint  # 整个文档的fingerprint为最终各个位>=0的和

    # 求海明距离
    def calc_hamming_distance(self, hash1, hash2):
        x = (hash1 ^ hash2) & self.all_1_bin
        tot = 0
        while x:
            tot += 1
            x &= x - 1
        return tot

    # 对传入word列表进行处理，支持带权重（int或str）和不带权重（默认为1）
    def _proc_list_weight(self, tokens):
        if isinstance(tokens, list) or isinstance(tokens, tuple):
            if isinstance(tokens[0], list) or isinstance(tokens[0], tuple):
                if isinstance(tokens[1], int):
                    for token in tokens:
                        yield (self._string_hash(token[0]), token[1])
                else:
                    for token in tokens:
                        yield (self._string_hash(token[0]), int(token[1]))
            else:
                for token in tokens:
                    yield (self._string_hash(token), 1)

    # 针对source生成hash值 (一个可变长度版本的Python的内置散列)
    def _string_hash(self, source):
        if source == '':
            return 0
        else:
            x = ord(source[0]) << 7
            m = 1000003
            for c in source:
                x = ((x * m) ^ ord(c)) & self.all_1_bin
            x ^= len(source)
            if x == -1:
                x = -2
            return x  
            
if __name__ == '__main__':
    print '权重测试(int)'
    tokens1 = [['asdfasdf', 2], ['asdfa', 1], ['s', 0]]
    tokens2 = (('asdfasdf', 2), ('asdfa', 1), ('a', 0))
    test1 = SimHash()
    print test1.get_hash(tokens1)
    print test1.get_hash(tokens2)
    print test1.calc_hamming_distance(test1.get_hash(tokens1), test1.get_hash(tokens2))

    print '权重测试(str)'
    tokens1 = (['asdfasdf', '2'], ['asdfa', '1'], ['s', '0'])
    tokens2 = [('asdfasdf', '2'), ('asdfa', '1'), ('a', '0')]
    test2 = SimHash(128)
    print test2(tokens1)
    print test2(tokens2)
    print test2.calc_hamming_distance(test2.get_hash(tokens1), test2.get_hash(tokens2))

    print '无权重测试'
    str1 = 'asdfasdf asdfa a'
    str2 = 'asdfasdf asdfa b'
    test3 = SimHash(64)
    print test3.get_hash(str1.split())
    print test3.get_hash(str2.split())
    print test1.calc_hamming_distance(test3.get_hash(str1.split()), test3.get_hash(str2.split()))
```

###Reference：  
[1] [Github: sangelone/python-hashes](https://github.com/sangelone/python-hashes)   
