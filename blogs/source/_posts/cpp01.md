---
title:  boost::multi_index 提供一种千人在线即时排行榜的设计思路
date: 2019-06-23 15:57:09
categories: C++
tags: C++
---
做游戏或金融后台开发，经常会遇到设计开发排行榜的需求。比如玩家的充值排行，战力排行等等。而这种排行基本都是即时更新的，快速排序对于单一类型排序可以满足需求，但是对于多种类的排序就很吃力，比如实现一个排行榜，有战力排序，有充值排序，如下图
<!--more-->
![1.jpg](1.jpg)
## 快速排序的缺陷
如果用快速排序实现，需要定义四种比较规则，而且qsort排序需要一段连续空间，如数组或者vector，为节约内存，每个元素存储玩家基本信息的指针。之后为每种排行类型定义单独的比较规则。代码如下
``` cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

struct PowerCmp;
struct ChargeCmp;
//玩家基本信息
class PlayrInfo{
public:
	int getPower()const{
		return u_power;
	}

	int getCharge()const{
		return u_charge;
	}
private:
	int u_ind; //玩家唯一id
	int u_power; //玩家战力
	int u_charge; //玩家充值数量
};

bool chargeCmp(PlayrInfo *a, PlayrInfo *b) 
{
	return a->getCharge() > b->getCharge();
}

bool powerCmp(PlayrInfo *a, PlayrInfo *b) 
{
	return a->getPower() > b->getPower();
}

int main(int argc, char* argv[])
{
	//分别实现排序
	vector<PlayrInfo*> vecPlayer;
	
	sort(vecPlayer.begin(),vecPlayer.end(),powerCmp);
	sort(vecPlayer.begin(),vecPlayer.end(),chargeCmp);
	getchar();
	return 0;
}
```
这么做有几个坏处
1 每个排行都要开辟一段连续的序列，即使存储指针也会造成空间的浪费。
2 qsort适合中小数量排序，当人数上千或万以上会造成瓶颈。但是可以通过插入排序优化。
下面提出一种新的结构来处理排序，boost::multi_index 实现了多索引结构，支持按照多个键值排序，可以极大减轻压力。
## multi_index 多索引处理排行榜
multi_index 是boost库提出的多索引结构表，可以设定多个主键进行排序，如战力，充值等等，但是必须要有一个唯一主键，我们将player的id设为唯一主键。
为了方便输出我们先完善下PlayerInfo类，重载输出运算符
``` cpp
//玩家基本信息
class PlayrInfo{
public:
    PlayrInfo(int id, int power, int charge):u_ind(id),u_charge(charge),u_power(power){}
	int getPower()const{
		return u_power;
	}

	int getCharge()const{
		return u_charge;
	}
private:
	int u_ind; //玩家唯一id
	int u_power; //玩家战力
	int u_charge; //玩家充值数量

	//重载输出
	friend std::ostream& operator<<(std::ostream& os,const boost::shared_ptr<PlayrInfo> & e)
	{
		//获取引用计数
		//cout << e.use_count();
		//获取原始指针
		//e.get()
		os<<e->u_ind<<" "<<e->u_power<<" "<<e->u_charge<<std::endl;
		return os;
	}
};
```
由于multi_index 各主键排序需要设置比较规则，默认是从小到大，int类型可以用greater<int>， less<int>等。
greater<int>从大到小, less<int>从小到大,也可以自己实现仿函数。
``` cpp
struct powerOperator
{
	bool operator()(int a, int b) const
	{
		return a > b;
	}
};
```
定义operator()一定要加上const，因为没有修改参数数据。
### multi_index 表定义
基于playerinfo实现multi_index表如下
``` cpp
struct by_id;
struct by_power;
struct by_charge;
typedef multi_index_container<
	boost::shared_ptr<PlayrInfo>, //插入的主体，当然可以是PlayrInfo本身
	indexed_by<
	//唯一主键，不可重复, std::greater<int> 为id规则从大到小，默认从小到大
	//tag<by_id>为标签名，可写可不写
	ordered_unique<tag<by_id> , const_mem_fun<PlayrInfo,int,&PlayrInfo::getId>>,

	//可重复的主键
	ordered_non_unique<tag<by_power>, const_mem_fun<PlayrInfo, int, &PlayrInfo::getPower>, powerOperator >,
	ordered_non_unique<tag<by_charge>, const_mem_fun<PlayrInfo, int, &PlayrInfo::getCharge>, greater<int>  >
	>
> PlayerContainer;
```
by_id， by_power等都是标签，只需要声明一个结构体，然后tag<标签名>放到索引里。当然可以不带tag，带tag是为了之后获取方便。
powerOperator是定义的战力比较规则，greater<int>定义的充值金额比较规则，是从大到小排序。
### multi_index 根据标签获取排序序列
multi_index表在数据插入的时候就根据各个主键排序规则进行排序，所以只需要按照主键取出，获得序列就是排好序的数据。

我们先按照id获取，然后打印输出结果
``` cpp
auto& ids = con.get<by_id>();
copy(ids.begin(), ids.end(), ostream_iterator<boost::shared_ptr<PlayrInfo> >(cout));
```
结果如下，可以看出是按照id从小到大排序输出的。
``` cmd
1 1231 10000
2 22222 2000
3 19999 222222
```
那我们按照战力获取
``` cpp
auto& powers = con.get<by_power>();
copy(powers.begin(), powers.end(), ostream_iterator<boost::shared_ptr<PlayrInfo> >(cout));
```
结果能看出是按照战力从大到小输出
``` cpp
2 22222 2000
3 19999 222222
1 1231 10000 
```
### multi_index 删除数据
删除元素如果是根据唯一主键删除，可以直接删除，如果是非唯一主键，需要查出所有记录并删除。
删除唯一主键为2的玩家
``` cpp
auto &itfind = ids.find(2);
	
	if (itfind != ids.end()){
		ids.erase(itfind);
		cout << "after erase: "<<endl;
		copy(con.begin(), con.end(), ostream_iterator<boost::shared_ptr<PlayrInfo> >(cout));
	}
```
结果
``` cmd
after erase:
1 1231 10000
3 19999 222222
```
插入一条战力重复的数据，并且遍历删除所有战力为1231的玩家
``` cpp
con.insert(boost::make_shared<PlayrInfo>(4, 1231, 10000));
auto &beginit = powers.lower_bound(1231);
auto & endit = powers.upper_bound(1231);
for( ; beginit != powers.end()&& beginit != endit; ){
		cout << "...................." <<endl;
		cout << *beginit;
		beginit = powers.erase(beginit);
}
	
copy(powers.begin(), powers.end(), ostream_iterator<boost::shared_ptr<PlayrInfo> >(cout));
```
结果
``` cmd
....................
1 1231 10000
....................
4 1231 10000
3 19999 222222
```
lower_bound 获取的是战力为1231的玩家迭代器的起始值，upper_bound获取的是战力为1231的玩家的最大值的下一个元素。

### multi_index 修改数据
修改数据可以用replace，也可以用modify
replace 失败不会删除条目，但是二次copy造成效率低下 
modify失败会删除对应条目，容易暴力误删除，但是效率高
replace 找到战力为19999的玩家，并且替换为新的数据。
``` cpp
	auto piter = powers.find(19999);
	if(piter != powers.end() ){
		auto newvalue = boost::make_shared<PlayrInfo>(100,200,300);
		powers.replace(piter, newvalue);
		copy(powers.begin(), powers.end(), ostream_iterator<boost::shared_ptr<PlayrInfo> >(cout));
	}
```
下面用modify修改数据
``` cpp
piter = powers.find(200);
	if(piter != powers.end()){
		auto newp = 1024;
		powers.modify(piter, [&](boost::shared_ptr<PlayrInfo> & playptr)->void{
			playptr->setPower(newp) ;
		});
		copy(powers.begin(), powers.end(), ostream_iterator<boost::shared_ptr<PlayrInfo> >(cout));
	}
```
modify 第一个参数和replace一样，都是要修改的迭代器，第二个参数是一个函数对象，参数类型为表中元素类型。
到目前为止，multi_index介绍完毕，用该多索引结构可以高效实现多级排序，非常适用于即时排行榜。
源码下载
[https://github.com/secondtonone1/boost-multi_index-](https://github.com/secondtonone1/boost-multi_index-)
我的公众号，谢谢关注
![wxgzh.jpg](wxgzh.jpg)