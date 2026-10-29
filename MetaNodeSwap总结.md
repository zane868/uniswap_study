# MetaNodeSwap总结
MetaNodeSwap没有实现UniswapV3的全部功能，主要实现了集中流动性的核心价值"把有限的资金集中用于自己选定的价格范围"

## 价格定义
- UniswapV3的价格区间定义在仓位上，资金池确定手续费等级和价格刻度间距（tickSpacing），仓位的价格上下界需要符合这个间距
- MetaNodeSwap的价格区间定义在资金池上，仓位只能在池子的范围内定义

## 资金池
- UniswapV3：`token0+token1+fee`唯一确定一个资金池
- MetaNodeSwap：`token0+token1+tickLower+tickUpper+fee`唯一确定一个资金池

## 仓位
- UniswapV3：每个仓位可以选择自己的价格范围，上下界必须符合池子的价格刻度间距
- MetaNodeSwap：仓位的价格范围受所属资金池限制

## 兑换
- UniswapV3：兑换通过指定池子或路由执行,单池内按当前活跃流动性L计算，分段兑换,跨池兑换由路由组合完成
- MetaNodeSwap：每个池子只有一个固定价格区间，单池兑换使用该池子的总流动性L计算，取消了分段兑换

## 总结
MetaNodeSwap将价格范围固定在池子上,同池仓位共用区间,采用V3风格的流动性数学和手续费记账,省略了同池不同区间仓位叠加及跨Tick切换活跃流动性的完整机制