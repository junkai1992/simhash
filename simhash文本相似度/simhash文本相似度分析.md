# SimHash原理

## 1.SimHash背景

- SimHash算法来自于 GoogleMoses Charikar发表的一篇论文“detecting near-duplicates for web crawling” ，其主要思想是降维， 将高维的特征向量映射成低维的特征向量，通过两个向量的Hamming Distance（汉明距离）来确定文章是否重复或者高度近似。 
- **Hamming Distance**： 又称汉明距离，在信息论中，两个等长字符串之间的汉明距离是两个字符串对应位置的不同字符的个数。也就是说，它就是将一个字符串变换成另外一个字符串所需要替换的字符个数。例如：1011101 与 1001001 之间的汉明距离是 2（异或）。至于我们常说的字符串编辑距离则是一般形式的汉明距离。这样通过比较文档之间simHash值的汉明距离，可以获取他们相似度。

## 2.如何比较两篇文章相似度呢？

- 方法1：通过分词(这里用到jieba)，得到一系列特征向量， 然后计算特征向量之间的距离（可以计算它们之间的欧氏距离、海明距离或者夹角余弦等等），从而通过距离的大小来判断两篇文章的相似度。 
- 方法2： 另外一种方案是传统hash，我们考虑为每一个web文档通过hash的方式生成一个指纹 （finger print）。

## 3.实现流程

- **分词**：对一段语句，进行分词，得到有效特征向量，然后给每一个特征向量设置1-5个级别代表权重（如果给定是一个文本特征向量可以通过TF-IDF）。
- **hash**：通过hash函数计算各个特征向量的hash值。hash值为二进制数0 1 组成的n-bit签名。比如 “茶壶”的hash值为100101，“饺子”的hash值为101011.
- **加权**：在hash值的基础上，给所有特征向量进行加权，即`W = Hash * weight`，且遇到1则hash值和权值正相乘，遇到0则hash值和权值负相乘。例如给“茶壶”的hash值“100101”加权得 到：`W= 100101*4 = 4 -4 -4 4 -4 4`，给“饺子”的hash值“101011”加权得到：`W=101011*5 = 5 -5 5 -5 5 5`，其余特征向量类似此般操作。
- **合并**： 将上述各个特征向量的加权结果累加，变成只有一个序列串。拿前两个特征向量举例，例如“茶壶”的“4 -4 -4 4 -4 4”和“饺子”的“5 -5 5 -5 5 5”进行累加，得到“4+5 -4+-5 -4+5 4+-5 -4+5 4+5”，得到“9 -9 1 -1 1”。 
- **降维**：对于n-bit签名的累加结果，如果大于0则置1，否则置0，从而得到该语句的simhash值，最后我们便可以根据不同语句simhash的海 明距离来判断它们的相似度。例如把上面计算出来的“9 -9 1 -1 1 9”降维（某位大于0记为1，小于0记为0），得到的01串为：“1 0 1 0 1 1”，从而形成它们的simhash签名。

![image-20191214112044185](https://github.com/junkai1992/junK/blob/master/simhash%E6%96%87%E6%9C%AC%E7%9B%B8%E4%BC%BC%E5%BA%A6/images/image-20191214112044185.png) 

## 4.python代码中的实现：

```python
import json
import jieba
import jieba.analyse
import numpy as np

class Simhash(object):
    def simhash(self,content):
        keylist = []
	    #jieba分词
        seg = jieba.cut(content)
        #去除停用词永祥
        jieba.analyse.set_stop_words("stopwords.txt")
        #得到前20个分词和tf-idf权值
        keywords = jieba.analyse.extract_tags("|".join(seg),topK=20,withWeight=True,allowPOS=())
        a = 0
        for feature,weight in keywords:
            weight = int(weight * 20)
            feature = self.string_hash(feature)
            temp = []
            for i in feature:
                if i == "1":
                    temp.append(weight)
                else:
                    temp.append(-1*weight)
            keylist.append(temp)
        list1 = np.sum(np.array(keylist),axis=0)
        if keylist == []:
            return "00"
        simhash = ""
        #降维处理
        for i in list1:
            if i>0:
                simhash += "1"
            else:
                simhash += "0"
        return simhash
    def string_hash(self,source):
        if source == "":
            return 0
        else:
            x = ord(source[0]) << 7
            m = 1000003
            mask = 2 ** 128 - 1
            for c in source:
                x = ((x * m) ^ ord(c)) & mask
            x ^= len(source)
            if x == -1:
                x = -2
            x = bin(x).replace('0b', '').zfill(64)[-64:]
        return str(x)
        
#         x=str(bin(hash(source)).replace('0b','').replace('-','').zfill(64)[-64:])
#         return x 
```

- 相似度比较

```python
def hammingDis(s1,s2):
    t1 = "0b" + s1
    t2 = "0b" + s2
    n = int(t1,2) ^ int(t2,2)
    i = 0
    while n:
        n &= (n-1)
        i += 1
    print(i)
    if i <= 18:
        print("文本相似")
    else:
        print("文本不相似")
if __name__ == "__main__":
    text1 = open("article1.txt","r",encoding="utf-8")
    text2 = open("article2.txt","r",encoding="utf-8")
    hammingDis(text1,text2)
    text1.close()
    text2.close()
```

## 5. simhash 模块

-  simhash算法的主要思想是降维，将高维的特征向量映射成一个f-bit的指纹(fingerprint)，通过比较两篇文章的f-bit指纹的Hamming Distance来确定文章是否重复或者高度近似。 

- 下载

  ```python
  pip install simhash
  ```

- 代码实现

  ```python
  from simhash import Simhash
  def simhash_similarity(text1,text2):
      a_simhash = Simhash(text1)
      b_simhash = Simhash(text2)
      print(a_simhash.value)
      print(b_simhash.value)
      max_hashbit = max(len(bin(a_simhash.value)),len(bin(b_simhash.value)))
      print(max_hashbit)
      
      #汉明距离
      distince = a_simhash.distance(b_simhash)
      print(distince)
      similar = distince/max_hashbit
      return similar
  if __name__ == "__main__":
      text1 = open("article1.txt","r",encoding="utf-8")
      text2 = open("article2.txt","r",encoding="utf-8")
      similar=simhash_similarity(text1,text2)
      #相相似度
      print(similar)
      text1.close()
      text2.close()
  ```

- 参考资料：

  - https://blog.csdn.net/lihaitao000/article/details/52355704
  - https://blog.csdn.net/madujin/article/details/53152619
  - https://leons.im/posts/a-python-implementation-of-simhash-algorithm/ 

  - https://github.com/leonsim/simhash 



