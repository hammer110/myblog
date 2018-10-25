---
title: 万卡商城详情页SPU H5代码实现思路
date: 2018-05-23 17:24:15
---

> 思路构想
> 具体分析

### 思路构想

spu这个需求对数据结构有很重要的要求，当时和api同学经过多次商讨以后，最终定下来下文中的数据结构。大概的实现思路分为4种情况
<!-- more -->
1. 初始化的时候根据当前商品对应的spu，把api返回的对应数据标记为selected
2. 把选中的sku对应的productNos提取出来统一放到一个数组内维护如下
[
"pzsc1505285914560,pzsc1505285933659,pzsc1505285924156,pzsc1505285943396",
"pzsc1505285914560,pzsc1505285933659,pzsc1505285920795,pzsc1505285939779,pzsc1505285917619,pzsc1505285936763",

]
3. 用户点击某个sku的时候，拿到对应的productNos和selectArray其他索引的数据进行比对，看是否能匹配出来一个productNo 比如用户选中的是颜色维度内的银色（也就是索引为1）这个时候就会和selectArray内的索引非1的数组进行匹配

4. 异常情况

### 具体分析

数据结构

```
// api地址 https://sc.9fbank.com/wk/productdetailshowspujd
 {
   currentSpu: "金色、256G、iphone8plus",
   "spu": [{
			"dim": "1",
			"saleAttrList": [{
				"saleValue": "金色",
				"productNos": "pzsc1505285914560,pzsc1505285933659,pzsc1505285924156,pzsc1505285943396"
			}, {
				"saleValue": "银色",
				"productNos": "pzsc1505285920795,pzsc1505285939779,pzsc1505285930479,pzsc1505285949305"
			}, {
				"saleValue": "深空灰色",
				"productNos": "pzsc1505285917619,pzsc1505285936763,pzsc1505285927533,pzsc1505285946373"
			}],
			"saleName": "颜色"
		}, {
			"dim": "2",
			"saleAttrList": [{
				"saleValue": "64G",
				"productNos": "pzsc1505285914560,pzsc1505285933659,pzsc1505285920795,pzsc1505285939779,pzsc1505285917619,pzsc1505285936763"
			}, {
				"saleValue": "256G",
				"productNos": "pzsc1505285924156,pzsc1505285943396,pzsc1505285930479,pzsc1505285949305,pzsc1505285927533,pzsc1505285946373"
			}],
			"saleName": "内存"
		}, {
			"dim": "3",
			"saleAttrList": [{
				"saleValue": "iphone8",
				"productNos": "pzsc1505285914560,pzsc1505285924156,pzsc1505285920795,pzsc1505285930479,pzsc1505285917619,pzsc1505285927533"
			}, {
				"saleValue": "iphone8plus",
				"productNos": "pzsc1505285933659,pzsc1505285943396,pzsc1505285939779,pzsc1505285949305,pzsc1505285936763,pzsc1505285946373"
			}],
			"saleName": "型号"
		}],
 }
```
currentSpu是当前商品对应的spu中文名称
SPU是一个json数组，代表当前商品有几个维度(SPU)，saleName为当前维度的name，saleAttrList代表当前维度下面包含多少个属性，举例说明

```
{
 saleValue: "金色",  // 当前sku的name
 // 当前sku对应的商品id有哪些
 "productNos": "pzsc1505285914560,pzsc1505285933659,pzsc1505285924156,pzsc1505285943396"

}
```

#### 1. 初始化

```
let currentSpuCopy = currentSpu && currentSpu.split('、')
spu.map((val, idx) => {
    val.saleAttrList.map((_item, idx) => {
      if (_.indexOf(currentSpuCopy, _item.saleValue) > -1) {
        _item.selected = true
        res.data.curSkuProNo.push(_item.productNos)
      } else {
        _item.selected = false
      }
    })
})
// 计算默认选择的sku,其余的哪些sku不能选择
this.calceSkuNotChoose(res.data, currentSpuCopy)
```
currentSpu转换成数组，然后循环spu下面的saleAttrList，检查sku是否在currentSpuCopy数组内，如果包含的话，则该对象增加属性selected，并且把当前的sku对应的productNos放到curSkuProNo数组内

----

**计算当前sku哪些是不可以选择**

> 第一步我们标记了哪些属性是需要处于选中状态的，现在我们需要标记出来哪些属性是不可以被选中的

简单说明一下，一个sku有3种状态分别是
1: 选中也就是标红
2: 一般状态也就是这个sku当前可以被点击
3: 当前sku按钮处于置灰状态也就是当前sku对应的productNos匹配不出来一个productNo，如果匹配不出来一个productNo的话，也就是相当于匹配不出来一个商品

```
calceSkuNotChoose (spuData, currentSpuCopy) {
      const { spu, curSkuProNo } = spuData
      spu.map((val, idx) => {
        val.saleAttrList.map((_item, index) => {
          let curSkuProNoList = []
          if (_.indexOf(currentSpuCopy, _item.saleValue) > -1) {
            return false
          }
          let productNos = _item.productNos.split(',')
          // 把当前选中的sku对应的数组存起来，以为二维数组的形式存放
          curSkuProNo.map((v, i) => {
            if (idx == i) {
              return false
            }
            curSkuProNoList.push(v.split(','))
          })
          // 只有当前属性和所有选中的sku能匹配出来productNo，判定为可以选择
          let intersectionArr = _.intersection(productNos, ...curSkuProNoList)[0]
          if (!(intersectionArr && intersectionArr.length > 0)) {
            _item.isNotExit = true
          }
        })
      })
      return spuData
    }
```
其实上述代码主要实现思路是当前sku对应productNos和当前对应的sku对应的productNos匹配,看是否可以匹配出最少一个productNo，如果匹配不出来，当前维度整体标红，代表当前维度和当前选中的sku匹配不出来一个productNo 

**重点**：此时的sku对应的productNos和上文的curSkuProNo不能混用，这个也就是为什么此处需要单独做一次循环，因为此时需要排除当前索引的sku。举一个例子：金色对应的productnos不能继续和当前维度的别的颜色的sku进行匹配，因为这样是没意义，so他只能和其他维度的sku进行匹配。


#### 2. 用户点击

思路分析：
  用户点击的时候，会把对应的productNos替换到curSkuProNo数组对应的位置上，然后把curSkuProNo转成二维数组，然后取交集 1: 匹配出来一个productNo, 首先拿着最新的curSkuProNo，判断出来每个sku和当前的这个数组是否有交集，如果有的话代表是可以点击的，否则置灰 2: 匹配不出来一个productNo, 那么当前sku变为选中状态，其余spu维度都清空

```
let arrSkuProNoList = []  // 所有选中sku对应的productNo存放再该数组中和curSkuProNo作用类似
let isChooseSelected = false
val.isClearUserChoose = false  // 判断是否显示红色tips的标志
// 如果没有交集，则把当前数组赋值给选中的sku数组列表
if (!(intersectionArr && intersectionArr.length > 0)) {
 arrSkuProNoList.push(spu[spuIndex].saleAttrList[skuIndex].productNos.split(','))
} else {
    curSkuProNo.map((v, i) => {
      if (idx == i || v == '') {
        return false
      }
      arrSkuProNoList.push(v.split(','))
    })
}
```

intersectionArr代表的是当前用户选中的所有sku是否可以匹配出来一个productNo，如果匹配不出来的话，把当前用户选中的sku对应的productNo 赋值给arrSkuProNoList，这种情况下arrSkuProNoList数组内只会存放一个维度的数据

```
// 如果spu的纬度和当前循环到的纬度一样，则直接return 回去
// 维度一样不做比较
if (spuIndex == idx) {
  return false
}
// 首先判断当前选中的几个sku是否有交集如果没有,则其余的纬度都显示error信息
if (!(intersectionArr && intersectionArr.length > 0)) {
  val.isClearUserChoose = true
  _item.selected = false
}
```

循环spu的json数组，维度一样话直接return，如果所有选中的sku没有匹配出来productNo, 那么isClearUserChoose标记为true(在每个维度后面显示请选择spu name), selected标记为false

```
// 判断当前sku与已经选中的数据是否有交集(已经选中的sku，需要排除当前纬度)
    let curProNo = _.intersection(...arrSkuProNoList, saleAttrList)[0]
    /**
     * 如果selected是true代表是当前选中状态
     * 需要判断用户选中的这个sku对应productNo，
     * 是否在其他spu的sku对应的productno中有交集
     * 如果没有则这个属性的外边框变成虚线，然后提示用户请选择属性
     */
    // 当前sku对应的productNos
    let isSelected = !!(_item.selected && (curProNo && curProNo.length > 0))
    _item.selected = isSelected
    if (_item.selected) {
      isChooseSelected = _item.selected
    }
    // 当前属性需要和除本纬度以外所有选中的属性进行取交集操作，如果能取出来那代表可以选择
    if (curProNo) {
      _item.isNotExit = false
    } else {
      _item.isNotExit = true
      _item.selected = false
    }
    val.saleAttrList.splice(index, 1, _item)
  })

  // 如果当前纬度没有选中的状态，那么判定当前需要显示红色error信息, 并且底部的btn置灰不能点击
  if (!isChooseSelected && idx != spuIndex) {
    this.currentParams.curSkuProNo.splice(idx, 1, "")
    val.isClearUserChoose = true
    isGrayBtnFlag = true
  }
  if (val.isClearUserChoose && !isUserChooseNotExit) {
    isUserChooseNotExit = true
  }
```

上述代码和isUserChooseNotExit为false的时候表示当前用户选中的sku可以正常匹配出来一个productNO。

```
if (!isUserChooseNotExit) {
  // sku取交集，然后重新渲染页面
  const productNos = this.intersectionAArr()
  // 取出来数组交集的productNo通知父级组件
  this.emitParentInfo(productNos)
  // this.isMoneyCalced = false
}
```

上述代码代表匹配出来一个productno以后，通知父级组件，重新调用接口渲染页面

**需要着重说的是**
const productNos = this.intersectionArr()

这个方法内取交集是分为两次，第一次是判断当前用户选中的状态是否可以匹配出来一个商品，然后根据这个条件来对每个spu进行不同的处理流程。虽然这个方法this.intersectionArr()也是取交集，但是这一次是等数据都稳定以后，何为数据稳定以后。比如我现在有3个维度，粉笔为颜色，内存，型号。我当前只选中了2个维度 颜色和内存，第一次取交集这个时候可能能取成功，但是用户没有选中所有维度，所以我们在最后的时候再一次取交集，来从用户选中的维度数和是否匹配出来productNo来做最后验证，代码如下

```
// 该函数是在提交代码的时候取出来交集用的
intersectionArr () {
  let curSpu = []
  this.currentParams.curSkuProNo.map((val, index) => {
    curSpu[index] = val.split(',')
  })
  // 判断当前选中的几个spu是否都有值
  const chooseNum = this.checkOneSku()
  // 数组取交集 如果选中的数组只有刚才用户点击的有值，那么直接把用户选中的spu返回去，否则取交集
  const productNos = _.intersection(...curSpu)[0]
  if (chooseNum == 1) {
    return productNos
  } else {
    return [pruductNoList.split(',')]
  }
}
```


#### 总结

以上是我在做万卡商城spu的时候的思路总结，其实有一部分代码可以抽取成公共方法，一些流程上的优化。
最后再强调一下spu这种类似的需求对后台返回的数据有一个比较强的依赖性的


