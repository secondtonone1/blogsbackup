---
title: eocs跨链合约区块数据结构设计与实现
date: 2019-03-15 12:01:47
categories: [区块链]
tags: [区块链eos]
---
## 核心数据结构分析
``` cpp
struct block_header {
    block_timestamp_type timestamp;
    account_name producer;
    //省略部分结构......
    static uint32_t num_from_id(const block_id_type& id);
}
```
结构体中timestamp是块打包好的时间戳，previous是前一个块的id，transaction_mroot表示所有交易的merkle root， action_root表示 action的merkle root。
num_from_id通过blockid找到blocknum。
<!--more-->
``` cpp
struct signed_block_header : block_header {
   signature producer_signature;
   EOSLIB_SERIALIZE_DERIVED(signed_block_header, block_header, (producer_signature))
};
```
struct signed_block_header :
该结构体继承了block_header，内部包含了producer_signature，提供了生产者的签名
struct block_header_state：
包括块基本的信息，blockid，blocknum，以及signed_block_header签名头信息。

struct block_header_with_merkle_path：
包括block_header_state 类型的block_header，以及由id序列构成的merkle_path，这个merkle_path中的blockid需要一个链接一个，不能间断，此外最后一个blockid需要指向block_header的前一个id。

struct action_receipt :
表示一个action的收据，包括接受者，摘要信息，授权顺序，接受顺序，编码顺序等。
用UML图表示上述各类之间关系:
![1.jpg](1.jpg)

## 核心api解析
``` cpp
uint32_t block_header::num_from_id(const block_id_type& id)
{
    return endian_reverse_u32(uint32_t(*reinterpret_cast<const uint64_t*>(id.hash)));
}
```
取出id的hash值，然后将高32位反转获得blocknum
``` cpp
checksum256 block_header::id() const {
auto result = sha256(*this);
*reinterpret_cast<uint64_t*>(result.hash) &= 0xffffffff00000000;
*reinterpret_cast<uint64_t*>(result.hash) += endian_reverse_u32(block_num());
return result;
}
```

计算id时，将自身256hash值去高32位，与之前获得的blocknum()高32位反转后相加，得到id号。
``` cpp
digest_type block_header::digest() const {
return sha256(*this);
}
```

获取摘要，就是将自己进行256hash得到摘要。
``` cpp
template <typename T>
checksum256 sha256(const T& value) {
auto digest = pack(value);
checksum256 hash;
::sha256(digest.data(), uint32_t(digest.size()), &hash);
return hash;
}
```

sha256是我们eocs团队自己封装的基于万能类型模板实现的hash函数，首先将value打包，然后将hash结果保存在变量hash中，返回此结果作为digest。
``` cpp
void block_header_state::validate() const {
auto d = sig_digest();
assert_recover_key(&d, (const char*)(header.producer_signature.data), 66, (const char*)(block_signing_key.data), 34);

eosio_assert(header.id() == id, "invalid block id"); // TODO: necessary?
}
```

该函数先计算摘要，然后根据摘要去除header的生产者签名信息和块签名key值。
接下来判断header的id是否和自己的id匹配。
``` cpp
producer_key block_header_state::get_scheduled_producer(block_timestamp_type t)const {
// TODO: block_interval_ms/block_timestamp_epoch configurable?
auto index = t.slot % (active_schedule.producers.size() * producer_repetitions);
index /= producer_repetitions;
return active_schedule.producers[index];
}
```

该函数获取轮询的生产者，producer_repetitions为重复的生产者数量，index除以该数量获取轮询的次数，进而根据轮询表里index索引取出对应的生产者。

``` cpp
uint32_t block_header_state::calc_dpos_last_irreversible()const {
vector<uint32_t> blocknums;
blocknums.reserve(producer_to_last_implied_irb.size());
for (auto& i : producer_to_last_implied_irb) {
blocknums.push_back(i.second);
}

if (blocknums.size() == 0) return 0;
std::sort(blocknums.begin(), blocknums.end());
return blocknums[(blocknums.size() - 1) / 3];
}
```

该函数计算最后一个不可逆区块num，首先定义了blocknums的序列，然后根据producer_to_last_implied_irb(包含不可逆区块的生产者map)，将对应的区块nums放入blocknums序列，之后，根据序列大小-1再除以三得到区块索引，就是最后的不可逆区块号。

到此为止

