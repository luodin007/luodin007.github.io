---
layout: post
title:  "对海量HTTP响应报文聚类分析——针对HTTP报文优化simhash"
date:   2015-05-16 23:23:41
categories: Bigdata
comments: true
---


[上文](http://luoding.me/bigdata/2015/05/03/HttpMessageProc2/)已经实现了simhash，但是经过测试，并没有达到预期的分类效果。  
simhash作为Google用来网页去重的算法，对于长文本效果很不错，而且配合抽屉原理的hash匹配后效率高的惊人。 
但是simhash的所有优点对我来说并不一定有用，通过测试发现的问题如下：     
1. http返回报文中夹杂了许多无意义的信息，并不适合直接进行文本相似度计算   
2. 经过数据清洗的报文长度过小，simhash算法对于小文本效果并不显著，可能存在误报问题   
3. 因为我需要判断的是是否归为一类组件，而不是简单的判断是否相同。 

经过不断的测试和修改，解决方案如下：  
1. **对HTTP报文进行数据清洗**  
	HTTP报文包括HTTP响应头（header）和响应包体（html）两部分，如果header与html保存在一个文件中，我们需要提前对其进行分割。对我我们来说header中的包含Server、X-Powered-By等有利于组件的信息。而html中包含大量网页信息，这对于判断是否是一个组件的帮助远小于带来的副作用。所以第一步就是要对报文数据进行清洗，只留下对我们有效的数据。  
	* **header清洗**  
		header中的常见项在[wiki中可以查到](http://en.wikipedia.org/wiki/List_of_HTTP_header_fields)，我们并不需要对每一项进行了解，只需要知道哪些对我们没有帮助，哪些是我们识别中经常会使用的。 
比如说下面的项虽然对浏览器有用，但是对我们识别组件并没有帮助，所以我们将其从header中除去，减少我们处理的文本量。 

		|    Key   	|    Value 						|
	| :-------- | :-----------------------------:|
	| date 		| Fri, 13 Mar 2015 16:01:14 GMT |
	| age		|   22 							|
	| content-length|   1636			|
	| last-modified|   Thu, 11 Apr 2013 19:48:27 GMT	|
	| expires	|   Thu, 01 Jan 1970 00:00:00 GMT	|
	| location	|   https://192.168.1.58:4343/ 							|
	| content-type|   text/html; charset=utf-8	|
	| connection	|   close	|
	| pragma|   no-cache|
	| cache-control|   private|
	| accept-ranges|   bytes|
	下列的项对于我们识别组件至关重要，很多情况下正则识别的就是这些项的值。
		|    Key   	|    Value 						|
	| :-------- | :-----------------------------|
	| server	| cloudfront 					 |
	| x-		| X-Cache: Error from cloudfront |
	| www-authenticate| Basic realm="Multi-Homing Gateway Administration Tools" |
	| via 		| 1.1 69138579f0e00411ece41ff78ec07fb6.cloudfront.net (CloudFront) |

* **html清洗**  
	对于html中有很多powered by等有效信息，但是对于我们来说提取难度过大，所以我们只对容易提取的html结构（title和meta）等进行提取，至于网页正文中的内容并不做处理。而且为了减少正则匹配代价，我们对html长度小于200（暂定）并不做清洗，直接分词加入较低权重即可。  

2. **调整hashbits长度**  
	hashbits就是simhash的fingerprint的二进制长度，理论上hashbits长度越长其fingerprint中包含文本内容的特征越丰富，也就是两个相似的文本更加相似，两个不相似的文本的区别更明显。  
	但是hashbit越长就需要更多储存hash的空间，而且汉明距离计算时需要更多次计算。所以我们需要一个在匹配结果能接受情况下最小的hashbits。[Google论文](http://www.wwwconference.org/www2007/papers/paper215.pdf)中提到的hashbits长度为64，这对于爬虫页面去重所需的长文本匹配效果很棒。  
	经过测试64bit的fingerprint的对http报文相似性分类的效果并不令人满意，很多是一类组件的报文被分到多个簇中（召回率），有一部分明显不是同一组件的报文夹杂在簇中（准确率）。所以经过测试，我选取了一个相对能够接受的长度——**128**（判断过程过于主观，而且可能只适用于本项目情况，所有测试数据不会整理上传，之后可能引入较科学的评估方法）。  
	在hashbits为128时虽然还存在召回率低的问题，但是准确率基本保证，在抽样中几乎见不到有误报情况。召回率低点无所谓，只要结果足够精准，多分几个簇可以用后期的算法进行解决。  

3. **调整汉明距离阈值**  
	汉明距离是我们判断两个hash的相似程度的参数，当汉明距离小于某一特定值则我们认为两个文本相似。Google论文中提到的阈值为3，也就是hash汉明距离小于等于3都可以认为文本相同。但是对于我们这个项目并不适用，所以需要重新找到适合自己的阈值。  
	经过测试（测试1-20所有的阈值结果）以后，我觉得（也就是没理论基础）汉明距离为10时（hashbits为128）结果的误报和分类效果可以接受。  

4. **对关键词调整权重**   
	因为simhash是基于权重的，所以我们要对越能表达特征的特征更多的权重，而那种可有可无的可能是特征关键词较低的权重，根本无用的关键词在清洗时直接删除所以不存在零权重的关键词。 
	比如说，header中的server是识别组件中常用的参数，所以我给server一个压倒性的权重，为了有server特征时能够第一时间体现在hash中，并且防止特征被其他无关的关键词权重覆盖。至于具体设置多少我现在还处于一个测试状态，所以就不摆了。  


### 总结  
经过清洗加权、调整hashbits和汉明距离阈值以后，我们的simhash算法基本上能够胜任http报文的分类工作，之后就需要解决大量数据的汉明距离匹配问题。    


###Reference：  
[1] [List_of_HTTP_header_fields - wikipedia](http://en.wikipedia.org/wiki/List_of_HTTP_header_fields)   
[2] [Detecting Near-Duplicates for Web Crawling](http://www.wwwconference.org/www2007/papers/paper215.pdf)   