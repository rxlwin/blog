---
title: 负载均衡算法 — 平滑加权轮询
date: 2018-12-02 20:45:12
tags:
- 算法
categories:
- 算法
- PHP
---

在 [负载均衡算法 — 轮询](https://www.fanhaobai.com/2018/11/load-balance-round-robin.html) 一文中，我们就指出了加权轮询算法一个明显的缺陷。即在某些特殊的权重下，加权轮询调度会生成不均匀的实例序列，这种不平滑的负载可能会使某些实例出现瞬时高负载的现象，导致系统存在宕机的风险。为了解决这个调度缺陷，就提出了 [平滑加权轮询](#) 调度算法。

![预览图](https://img1.fanhaobai.com/2018/11/load-balance-smooth-weighted-round-robin/fc16ba37-06b0-4193-9969-7541852dd46c.jpg)<!--more-->![预览图](https://img1.fanhaobai.com/2018/11/load-balance-smooth-weighted-round-robin/fc16ba37-06b0-4193-9969-7541852dd46c.jpg)

## 待解决的问题

为了说明平滑加权轮询调度的平滑性，使用以下 3 个特殊的权重实例来演示调度过程。

|    服务实例   | 	权重值 |
| ------------ | --------------|
|  192.168.10.1:2202 |	5 |
|  192.168.10.2:2202 | 1 |
|  192.168.10.3:2202 | 	1 |

我们已经知道通过 [加权轮询](https://www.fanhaobai.com/2018/11/load-balance-round-robin.html#%E5%8A%A0%E6%9D%83%E8%BD%AE%E8%AF%A2) 算法调度后，会生成如下不均匀的调度序列。

|   请求  |  选中的实例  |
| --------- | --------------|
|  1    |	192.168.10.1:2202  |
|  2    |   192.168.10.1:2202  |
|  3    | 	192.168.10.1:2202  |
|  4    | 	192.168.10.1:2202  |
|  5    | 	192.168.10.1:2202  |
|  6    | 	192.168.10.2:2202  |
|  7    | 	192.168.10.3:2202  |

接下来，我们就使用平滑加权轮询算法调度上述实例，看看生成的实例序列如何？

## 算法描述

假设有 N 台实例 S = {S1, S2, …, Sn}，配置权重 W = {W1, W2, …, Wn}，有效权重 CW = {CW1, CW2, …, CWn}。每个实例 i 除了存在一个配置权重 Wi 外，还存在一个当前有效权重 CWi，且 CWi 初始化为 Wi；指示变量 currentPos 表示当前选择的实例 ID，初始化为 -1；所有实例的配置权重和为 weightSum；

那么，调度算法可以描述为：
1、初始每个实例 i 的 **当前有效权重** CWi 为 **配置权重** Wi，并求得配置权重和 weightSum；
2、选出 **当前有效权重** [最大](#) 的实例，将 **当前有效权重** CWi 减去所有实例的 **权重和** weightSum，且变量 currentPos 指向此位置；
3、将每个实例 i 的 **当前有效权重** CWi 都加上 **配置权重** Wi；
4、此时变量 currentPos 指向的实例就是需调度的实例；
5、每次调度重复上述步骤 2、3、4；

上述 3 个服务，配置权重和 weightSum 为 7，其调度过程如下：

| 请求 | 选中前的当前权重 | currentPos |  选中的实例 | 选中后的当前权重 |
| --- | ------------- | -------- | ------------ | ----------- |
|  1   | {5, 1, 1}   | 0	| 192.168.10.1:2202 | {-2, 1, 1} |
|  2   | {3, 2, 2}   | 0	| 192.168.10.1:2202 | {-4, 2, 2} |
|  3   | {1, 3, 3}   | 1	| 192.168.10.2:2202 | {1, -4, 3} |
|  4   | {6, -3, 4}  | 0	| 192.168.10.1:2202 | {-1, -3, 4} |
|  5   | {4, -2, 5}  | 2	| 192.168.10.3:2202 | {4, -2, -2} |
|  6   | {9, -1, -1} | 0	| 192.168.10.1:2202 | {2, -1, -1} |
|  7   | {7, 0, 0}   | 0	| 192.168.10.1:2202 | {0, 0, 0}  |
|  8   | {5, 1, 1}   | 0	| 192.168.10.1:2202 | {-2, 1, 1} |

可以看出上述调度序列分散是非常均匀的，且第 8 次调度时当前有效权重值又回到 {0, 0, 0}，实例的状态同初始状态一致，所以后续可以一直重复调度操作。

> 此轮询调度算法思路首先被 Nginx 开发者提出，见 [phusion/nginx](https://github.com/phusion/nginx/commit/27e94984486058d73157038f7950a0a36ecc6e35) 部分。

## [代码实现](https://github.com/fan-haobai/load-balance/blob/master/Robin/SmoothWeightedRobin.php)

这里使用 PHP 来实现，源码见 [fan-haobai/load-balance](https://github.com/fan-haobai/load-balance/blob/master/Robin/SmoothWeightedRobin.php) 部分。

```PHP
class SmoothWeightedRobin implements RobinInterface
{
    private $services = array();

    private $total;

    private $currentPos = -1;

    public function init(array $services)
    {
        foreach ($services as $ip => $weight) {
            $this->services[] = [
                'ip'      => $ip,
                'weight'  => $weight,
                'current_weight' => $weight,
            ];
        }
        $this->total = count($this->services);
    }

    public function next()
    {
        // 获取最大当前有效权重实例的位置
        $this->currentPos = $this->getMaxCurrentWeightPos();

        // 当前权重减去权重和
        $currentWeight = $this->getCurrentWeight($this->currentPos) - $this->getSumWeight();
        $this->setCurrentWeight($this->currentPos, $currentWeight);

        // 每个实例的当前有效权重加上配置权重
        $this->recoverCurrentWeight();

        return $this->services[$this->currentPos]['ip'];
    }
}
```

其中，`getSumWeight()`为所有实例的配置权重和；`getCurrentWeight()`和 `setCurrentWeight()`分别用于获取和设置指定实例的当前有效权重；`getMaxCurrentWeightPos()`求得最大当前有效权重的实例位置，实现如下：

```PHP
public function getMaxCurrentWeightPos()
{
    $currentWeight = $pos = 0;
    foreach ($this->services as $index => $service) {
        if ($service['current_weight'] > $currentWeight) {
            $currentWeight = $service['current_weight'];
            $pos = $index;
        }
    }

    return $pos;
}
```

`recoverCurrentWeight()`用于调整每个实例的当前有效权重，即加上配置权重，实现如下：

```PHP
public function recoverCurrentWeight()
{
    foreach ($this->services as $index => &$service) {
        $service['current_weight'] += $service['weight'];
    }
}
```

需要注意的是，在配置`services`服务列表时，同样需要指定其权重：

```PHP
$services = [
    '192.168.10.1:2202' => 5,
    '192.168.10.2:2202' => 1,
    '192.168.10.3:2202' => 1,
];
```

## 数学证明

可惜的是，关于此调度算法严谨的数学证明少之又少，不过网友 [tenfy](https://tenfy.cn/2018/11/12/smooth-weighted-round-robin/) 给出的 [安大神](https://github.com/bigbuger) 证明过程，非常值得参考和学习。

### 证明权重合理性

假如有 n 个结点，记第 i 个结点的权重是 $x_i$，设总权重为 $S = x_1 + x_2 + … + x_n$。选择分两步：
1、为每个节点加上它的权重值；
2、选择最大的节点减去总的权重值；

n 个节点的初始化值为 [0, 0, …, 0]，数组长度为 n，值都为 0。第一轮选择的第 1 步执行后，数组的值为 $[x_1, x_2, …, x_n]$。

假设第 1 步后，最大的节点为 j，则第 j 个节点减去 S。
所以第 2 步的数组为 $[x_1, x_2, …, x_j-S, …, x_n]$。 执行完第 2 步后，数组的和为：
$x_1 + x_2 + … + x_j-S + … + x_n => x_1 + x_2 + … + x_n - S = S - S = 0$

由此可见，每轮选择第 1 步操作都是数组的总和加上 S，第 2 步总和再减去 S，所以每轮选择完后的数组总和都为 0。

假设总共执行 S 轮选择，记第 i 个结点选择 $m_i$ 次。第 i 个结点的当前权重为 $w_i$。 假设节点 j 在第 t 轮（t < S）之前，已经被选择了 $x_j$ 次，记此时第 j 个结点的当前权重为 $w_j = t \* x_j - x_j \* S = (t - S) \* x_j < 0$， 因为 t 恒小于 S，所以 $w_j < 0$。

前面假设总共执行 S 轮选择，则剩下 S-t 轮 j 都不会被选中，上面的公式 $w_j = (t - S) \* x_j + (S - t) \* x_j = 0$。 所以在剩下的选择中，$w_j$ 永远小于等于 0，由于上面已经证明任何一轮选择后，数组总和都为 0，则必定存在一个节点 k 使得 $w_k > 0$，永远不会再选中节点 j。

由此可以得出，第 i 个结点最多被选中 $x_i$ 次，即 $m_i <= x_i$。
因为 $S = m_1 + m_2 + … + m_n$ 且 $S = x_1 + x_2 + … + x_n$。 所以，可以得出 $m_i == x_i$。

### 证明平滑性

证明平滑性，只要证明不要一直都是连续选择那一个节点即可。

跟上面一样，假设总权重为 S，假如某个节点 i 连续选择了 t（$t < x_i$） 次，只要存在下一次选择的不是节点 i，即可证明是平滑的。

假设 $t = x_i - 1$，此时第 i 个结点的当前权重为 $w_i = t \* x_i - t \* S = (x_i - 1) \* x_i - (x_i - 1) \* S$。证明下一轮的第 1 步执行完的值 $w_i + x_i$ 不是最大的即可。

$w_i + x_i => (x_i - 1) \* x_i - (x_i - 1) \* S + x_i =>$
$x_i^2 - x_i \* S + S => (x_i - 1) \* (x_i - S) + x_i$

因为 $x_i$ 恒小于 S，所以 $x_i - S <= -1$。 所以上面：
$(x_i - 1) \* (x_i - S) + x_i <= (x_i - 1) \* -1 + x_i = -x_i + 1 + x_i = 1$

所以第 t 轮后，再执行完第 1 步的值 $w_i + x_i <= 1$。
如果这 t 轮刚好是最开始的 t 轮，则必定存在另一个结点 j 的值为 $x_j \* t$，所以有 $w_i + x_i <= 1 < 1 \* t < x_j \* t$。所以下一轮肯定不会选中 i。

## 总结

尽管，平滑加权轮询算法改善了加权轮询算法调度的缺陷，即调度序列分散的不均匀，避免了实例负载突然加重的可能，但是仍然不能动态感知每个实例的负载。

若由于实例权重配置不合理，或者一些其他原因加重系统负载的情况，平滑加权轮询都无法实现每个实例的负载均衡，这时就需要 [有状态](#) 的调度算法来完成。

<strong>相关文章 [»](#)</strong>

* [负载均衡算法 — 轮询](https://www.fanhaobai.com/2018/11/load-balance-round-robin.html) <span>（2018-11-29）</span>
