---
layout: tech_post
title:  " C/C++ 中STL的总结"
date:   2019-12-12 11:51:36 -0300
catalogue: Programming
tags: 中文 
description: C/C++ 容器的应用和总结
permalink: /chinese/C_Cpp_STL.html
---
## 目录
* TOC
{:toc}


## 序列容器Sequential container

list,queue,vector,deque

list: 非连续存储空间，插入/删除效率高，支持双向操作pop_back/push_back,pop_front/push_front。  
vector: 连续存储空间，插入/删除效率低，仅支持队尾操作push_back/pop_back  
deque: 连续存储空间，


（1）如果你需要高效的随即存取，而不在乎插入和删除的效率，使用vector
（2）如果你需要大量的插入和删除，而不关心随机存取，则应使用list
（3）如果你需要随机存取，而且关心两端数据的插入和删除，则应使用deque


### vector

在C++中vector 容器是最基础和通用的STL容器之一
vector的基本操作包括
push_back();
pop_back()

支持通过iterator进行操作：
insert:  
- vector.insert(pos,elem);   //在pos位置插入一个elem元素的拷贝，返回新数据的位置。
- vector.insert(pos,n,elem);   //在pos位置插入n个elem数据，无返回值。
- vector.insert(pos,beg,end);   //在pos位置插入[beg,end)区间的数据，无返回值 

erase:  
- vector.clear(); //移除容器的所有数据
- vec.erase(beg,end);  //删除[beg,end)区间的数据，返回下一个数据的位置。
- vec.erase(pos);    //删除pos位置的数据，返回下一个数据的位置。

## 无序关联容器Associative container

 在c++中有4种关联容器：set,multiset,map,multimap
（顺序）关联容器是基于对树的结构，无序关联容器是基于对哈希表的结构(hash table)


###  unordered_map

unordered_map内部实现了一个哈希表，因此其元素的排列顺序是杂乱的，无序的。

<B> Map </B>  
优点：
- 有序性，这是map结构最大的优点，其元素的有序性在很多应用中都会简化很多的操作
- 红黑树，内部实现一个红黑书使得map的很多操作在lgn的时间复杂度下就可以实现，因此效率非常的高  

缺点：
- 空间占用率高，因为map内部实现了红黑树，虽然提高了运行效率，但是因为每一个节点都需要额外保存父节点，孩子节点以及红/黑性质，使得每一个节点都占用大量的空间
适用处：对于那些有顺序要求的问题，用map会更高效一些



<B>Unordered_map</B>  
优点：
- 因为内部实现了哈希表，因此其查找速度非常的快时间复杂度O(1)  

缺点：
- 哈希表的建立比较耗费时间
适用处:对于查找问题，unordered_map会更加高效一些，因此遇到查找问题，常会考虑一下用unordered_map






#### unordered_map中常用方法与操作

{% highlight c linenos %}

#include<iostream>
#include<map>
#include<string>
 
using namespace std;
 
int main()
{
	
	
	// 构造函数
	map<string, int> dict;
	
	/* C++11 中使用{初始化} 
	map<string,int> dict ={ {"apple",2 }, {"orange",3} }; 
 	*/
	
	
	// 插入数据的三种方式
	dict.insert(pair<string,int>("apple",2));
	dict.insert(map<string, int>::value_type("orange",3));
	dict["banana"] = 6;
 
	// 判断是否有元素
	if(dict.empty())
		cout<<"该字典无元素"<<endl;
	else
		cout<<"该字典共有"<<dict.size()<<"个元素"<<endl;
 
	// 遍历
	map<string, int>::iterator iter;
	for(iter=dict.begin();iter!=dict.end();iter++)
		cout<<iter->first<<ends<<iter->second<<endl;
 
	// 查找
	if((iter=dict.find("banana"))!=dict.end()) //  返回一个迭代器指向键值为key的元素，如果没找到就返回end()
		cout<<"已找到banana,其value为"<<iter->second<<"."<<endl;
	else
		cout<<"未找到banana."<<endl;
 
	if(dict.count("watermelon")==0) // 返回键值等于key的元素的个数，map和unordered_map中因为键值为1所以count返回为1或0。
		cout<<"watermelon不存在"<<endl;
	else
		cout<<"watermelon存在"<<endl;
	
	pair<map<string, int>::iterator, map<string, int>::iterator> ret;
	ret = dict.equal_range("banana"); // 查找键值等于 key 的元素区间为[start,end)，指示范围的两个迭代器以 pair 返回
	cout<<ret.first->first<<ends<<ret.first->second<<endl;
	cout<<ret.second->first<<ends<<ret.second->second<<endl;
 
	iter = dict.lower_bound("boluo"); // 返回一个迭代器，指向键值>=key的第一个元素。
	cout<<iter->first<<endl;
	iter = dict.upper_bound("boluo"); // 返回一个迭代器，指向值键值>key的第一个元素。
	cout<<iter->first<<endl;
	return 0;
}
{% endhighlight %}

## (顺序)关联容器

### map 

与unordered_map对比： map内部实现了一个红黑树，该结构具有自动排序的功能，因此map内部的所有元素都是有序的，红黑树的每一个节点都代表着map的一个元素，因此，对于map进行的查找，删除，添加等一系列的操作都相当于是对红黑树进行这样的操作，故红黑树的效率决定了map的效率。与unordered_map的详细对比见[unordered_map]()




