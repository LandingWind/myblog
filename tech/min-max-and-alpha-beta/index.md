# Min-Max算法及Alpha-Beta剪枝


适用于所有情况完全已知的游戏，并且是零和游戏。
零和游戏（Zero-sum Game）：意思就是你死我活，一方的胜利代表另一方的失败，比如，象棋，五子棋等。 完全信息（Perfect Information）：玩家知道之前所有的步骤。象棋就是完全信息，因为玩家是交替着落子，且之前的步骤都能在棋盘上体现，但是石头剪子布就不是。

这样的游戏可以用树状图表示：MAX表示自己，MIN表示对手；Utility是对着一种情况给出的分数（对自己而言越高越好）

![](https://i.loli.net/2020/12/26/25mVeYwD6CBXITh.jpg "minmax")

那么自己希望最大化这一utility，对手希望最小化这一utility，所以可以通过结果反推某一步应该采取的选择。

![](https://i.loli.net/2020/12/26/Lj4VviFS5cdZJ3N.jpg "step1")

![](https://i.loli.net/2020/12/26/4npYIBkPti13qAv.jpg "step2")

![](https://i.loli.net/2020/12/26/TPBDU5KYiZkStcA.jpg "step3")

![](https://i.loli.net/2020/12/26/DOkltoeCxNwXc6G.jpg "step4")

由于情况数太多需要采取剪枝算法（Alpha-Beta）来减少运算量：

每个MAX结点设置一个目前已知下界alpha，每个MIN节点设置一个目前已知上界beta。当计算一个MIN结点时，如果它的beta值小于等于其父结点的alpha值，则可以立即停止此结点的计算（alpha剪枝）；当计算一个MAX结点时，如果它的alpha结点大于等于其父结点的beta值，也可以立即停止此结点的计算（beta剪枝）
