---
layout:     post
title:      ApprovalTests的成功实践
subtitle:   
date:       2020-7-17
author:     孙继峰
header-img: img/th.jpg
catalog: true
tags:
	- 测试
---

年初去 ThoughtWorks 参加道长的[镀金玫瑰重构实践](https://www.jianshu.com/p/360e735cd3d5)里面接触了一个有趣的重构工具--[ApprovalTests](https://github.com/approvals)



不了解这个工具的可以看一下这几个视频:

- [镶金玫瑰-重构-Part 1_3 -建立安全网](https://www.bilibili.com/video/BV1BJ411N7aD/)
- [镶金玫瑰-重构-Part 2_3- 使用“条件语句上移对逻辑进行重构](https://www.bilibili.com/video/BV1BJ411N7qN/)
- [镶金玫瑰-重构-Part 3_3 - 使用“多态”代替“条件语句”](https://www.bilibili.com/video/BV1BJ411N7yE/)



**一句话说明它的好处: 可在没有测试、不懂业务的情况下去重构**



### 理想的实践

##### 背景

项目初期, 查询完全走数据库, 担心上线后数据库撑不住, 所以想加缓存扛一下. 要加缓存, 就有数据一致性, 那如何保证我加完缓存之前和加缓存之后数据是一直的呢?

##### 解决方案





---



### 理想总是美好的

ApprovalTest仅仅适用于有返回值的方法, 对于没有返回值的方法没有一点办法

``` java
    public void updateGoodsPrice(GoodsInfoPriceDTO goodsInfoPriceDTO) throws ApiException {

        if (goodsInfoPriceDTO == null || goodsInfoPriceDTO.getShippingRate() == null) {
            throw new ApiException(GoodsCodeEnum.GOODS_PRICE_MODIFIED_FAILED);
        }

        GoodsInfoPriceDTO oldGoodsInfo = getGoodsInfoPriceByGid(goodsInfoPriceDTO.getGid());
        GoodsInfoDO goodsInfoUpdate = new GoodsInfoDO();
        BeanUtils.copyProperties(goodsInfoPriceDTO, goodsInfoUpdate);
        goodsInfoUpdate.setShippingRate(null);

        Integer customsDuties = goodsInfoPriceDTO.getCustomsDuties() == null
                ? oldGoodsInfo.getCustomsDuties() : goodsInfoPriceDTO.getCustomsDuties();
        Long goodsPrice = goodsInfoPriceDTO.getGoodsPrice() == null
                ? oldGoodsInfo.getGoodsPrice() : goodsInfoPriceDTO.getGoodsPrice();
        //总价
        Long price = (long) customsDuties + goodsPrice + (long) goodsInfoPriceDTO.getShippingRate();
        goodsInfoUpdate.setPrice(price);

        //商品价格信息更新需记价格log
        if (!goodsPrice.equals(oldGoodsInfo.getGoodsPrice())) {
            goodsInfoUpdate.setLastPrice(oldGoodsInfo.getGoodsPrice())
                    .setPriceModifiedTime(LocalDateTime.now());
            //价格更改log
            GoodsPriceModifiedLogDO logDO = new GoodsPriceModifiedLogDO();
            logDO.setGid(goodsInfoPriceDTO.getGid())
                    .setLastGoodsPrice(oldGoodsInfo.getGoodsPrice())
                    .setGoodsPrice(goodsInfoUpdate.getGoodsPrice());
            priceModifiedLogMapper.insert(logDO);
        }

        log.info("商品更新价格，旧： {}, 新： {}", JsonUtil.toJson(oldGoodsInfo), JsonUtil.toJson(goodsInfoUpdate));
        update(goodsInfoUpdate, new LambdaQueryWrapper<GoodsInfoDO>()
                .eq(GoodsInfoDO::getGid, goodsInfoPriceDTO.getGid()));
        goodsRedisService.updateGoodsByGid(goodsInfoPriceDTO.getGid());
    }
```



上面的方法是一个电商项目的一隅, 大意就是传入商品gid, 和这个商品关税、运费、Sku最低价、Sku最高价, 根据这些来更新数据库中的商品价格, 进而触发生成改价日志, 触发缓存更新.

```GoodsInfoPriceDTO``` 代码如下

```java
@Data
@Accessors(chain = true)
public class GoodsInfoPriceDTO {

    private String gid;

    /**
     * 关税
     */
    private Integer customsDuties;

    /**
     * 商品原价
     */
    private Long goodsPrice;

    /**
     * 最高价格
     */
    private Long maxPrice;

    /**
     * 最低价格
     */
    private Long minPrice;

    /**
     * 商品来源渠道 1:国内现货 2:国外现货 3:代购 4:现货
     */
    private Integer goodsChannelSource;

    /**
     * 运费，计算总价
     */
    private Integer shippingRate;
```

是典型的[贫血模型](https://www.ituring.com.cn/article/25)

