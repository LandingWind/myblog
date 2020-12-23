# 并查集算法概念及实现


记录一下很早就学过的并查集 最简洁而优雅的数据结构之一 

主要用于解决一些元素分组的问题

## 核心思想
通过一维数组，index表示当前元素编号，value记录当前元素的上级元素，通过递归查找到关系网的根元素

这里要注意的是，为了避免查找效率低下，每次在查找根元素的时候进行路径压缩，也就是将路径上的元素的上级元素全部更改为根元素，这样减少了递归层级

### 数组记录元素关系
开始的时候`key=value`表示每个元素的爸爸是自己本身
![relation](http://qiniustorage.joyinn.top/20200207111128.png "relation")
然后比如`2`和`3`从属于`1` 那么就会有如下关系`arr[2] = 1`以及`arr[3] = 1`
![relation2](http://qiniustorage.joyinn.top/20200207111846.png "relation2")

### 路径压缩
并查集的数据储存模式可能会导致从属链过长 而导致查询效率低下 
![route](http://qiniustorage.joyinn.top/20200207112259.png "route")
比如上面的图在查询`2`的根节点的过程中 完全可以将这条路径上的所有节点直接指向`4` 这样就优化了数据结构 (换个角度讲就是减少了树的深度)

## 实现代码
将并查集**Disjoint Set Union**写成一个单独的**class**方便调用
```java
class DSU {
    public int parent[];
    public DSU(int n) {
        this.parent = new int[n];
        for(int i=0;i<n;i++) parent[i] = i;
    }
    public int find(int x) {
        if(x!=parent[x]) parent[x] = find(parent[x]);
        return parent[x];
    }
    public void join(int x, int y) {
        parent[find(x)] = find(y);
    }
    public void print() {
        for(int i=0;i<parent.length;i++) {
            System.out.println(i+": "+parent[i]);
        }
    }
}
```
这里对`int find(int x)`简单做一下解释

`x`表示待查找的元素 假如它并不是根元素 则调用`find(parent[x])`递归找到根元素 同时将根元素赋值给`parent[x]`(这样就实现了路径的压缩) 返回的当然是`parent[x]` 因为只有它是在被更新的

## LeetCode练习

上手练几道题巩固知识

### #959 由斜杠划分区域

{{< admonition quote "题目" true>}}

在由 1 x 1 方格组成的 N x N 网格 grid 中，每个 1 x 1 方块由 /、\ 或空格构成。这些字符会将方块划分为一些共边的区域。
（请注意，反斜杠字符是转义的，因此 \ 用 "\\" 表示。）
返回区域的数目。

{{< /admonition >}}

```java
class Solution {
    public int regionsBySlashes(String[] grid) {
        int N = grid.length;
        DSU dsu = new DSU(4*N*N);
        for (int i=0;i<N;i++) {
            for (int j=0;j<N;j++) {
                char ch = grid[i].charAt(j);
                int base = (i*N + j)*4;
                // join in a cell
                if(ch == ' ') {
                    dsu.join(base,base+1);
                    dsu.join(base, base+2);
                    dsu.join(base, base+3);
                } else if (ch == '/') {
                    dsu.join(base,base+3);
                    dsu.join(base+1,base+2);
                } else if (ch == '\\') {
                    dsu.join(base,base+1);
                    dsu.join(base+2,base+3);
                } else {
                    System.out.println("input error!");
                    return 0;
                }
                // join with other cells
                if(i > 0) {
                    dsu.join(base, base-4*N+2);
                }
                if(j > 0) {
                    dsu.join(base+3, base-4+1);
                }
            }
        }
        // get res
        int res = 0;
        for (int i = 0; i < 4 * N * N; i++) {
            if (dsu.find(i) == i)
                res++;
        }
        return res;
    }
}
```

### #990 等式方程的可满足性

{{< admonition quote "题目" true>}}

给定一个由表示变量之间关系的字符串方程组成的数组，每个字符串方程 equations[i] 的长度为 4，并采用两种不同的形式之一："a==b" 或 "a!=b"。在这里，a 和 b 是小写字母（不一定不同），表示单字母变量名。

只有当可以将整数分配给变量名，以便满足所有给定的方程时才返回 true，否则返回 false。 

**示例 1**

输入：["a==b","b!=a"]

输出：false

解释：如果我们指定，a = 1 且 b = 1，那么可以满足第一个方程，但无法满足第二个方程。没有办法分配变量同时满足这两个方程。

**示例 2**

输出：["b==a","a==b"]

输入：true

解释：我们可以指定 a = 1 且 b = 1 以满足满足这两个方程。

**示例 3**

输入：["a==b","b==c","a==c"]

输出：true

**示例 4**

输入：["a==b","b!=c","c==a"]

输出：false

**示例 5**

输入：["c==c","b==d","x!=z"]

输出：true

**提示**

`1 <= equations.length <= 500`

`equations[i].length == 4`

`equations[i][0]` 和 `equations[i][3]` 是小写字母

`equations[i][1]` 要么是 `=`，要么是 `!`

`equations[i][2]` 是 `=`

{{< /admonition >}}

```java
class Solution {
    public boolean equationsPossible(String[] equations) {
        DSU dsu = new DSU(26);
        for (String equation : equations) {
            if(equation.charAt(1) == '=') {
                int x = equation.charAt(0) - 'a';
                int y = equation.charAt(3) - 'a';
                dsu.join(x, y);
            }
        }
        for (String equation : equations) {
            if(equation.charAt(1) == '!') {
                int x = equation.charAt(0) - 'a';
                int y = equation.charAt(3) - 'a';
                if(dsu.find(x) == dsu.find(y)) return false;
            }
        }
        return true;
    }
}
```

### #1202 交换字符串中的元素

{{< admonition quote "题目" detail>}}

给你一个字符串 s，以及该字符串中的一些「索引对」数组 pairs，其中 pairs[i] = [a, b] 表示字符串中的两个索引（编号从 0 开始）

你可以**任意多次交换**在 pairs 中任意一对索引处的字符。

返回在经过若干次交换后，s 可以变成的按字典序最小的字符串。

**示例 1**

输入：s = "dcab", pairs = [[0,3],[1,2]]

输出："bacd"

解释：

交换 s[0] 和 s[3], s = "bcad"

交换 s[1] 和 s[2], s = "bacd"

**示例 2**

输入：s = "dcab", pairs = [[0,3],[1,2],[0,2]]

输出："abcd"

解释：

交换 s[0] 和 s[3], s = "bcad"

交换 s[0] 和 s[2], s = "acbd"

交换 s[1] 和 s[2], s = "abcd"

**示例 3**

输入：s = "cba", pairs = [[0,1],[1,2]]

输出："abc"

解释：

交换 s[0] 和 s[1], s = "bca"

交换 s[1] 和 s[2], s = "bac"

交换 s[0] 和 s[1], s = "abc"

**提示**

`1 <= s.length <= 10^5`

`0 <= pairs.length <= 10^5`

`0 <= pairs[i][0], pairs[i][1] < s.length`

s 中只含有小写英文字母

{{< /admonition >}}


```java
import java.util.*;

public class SwapString {
    public static void main(String[] args) {
        Scanner in = new Scanner(System.in);
        SwapString a = new SwapString();

        String s = in.nextLine();
        int n = in.nextInt();
        List<List<Integer>> pairs = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            ArrayList<Integer> pair = new ArrayList<>();
            pair.add(in.nextInt());
            pair.add(in.nextInt());
            pairs.add(pair);
        }
        String res = a.smallestStringWithSwaps(s, pairs);
        System.out.println("res: " + res);
    }

    public String smallestStringWithSwaps(String s, List<List<Integer>> pairs) {
        int len = s.length();
        DSU dsu = new DSU(len);
        for (List<Integer> pair : pairs) {
            int x = pair.get(0);
            int y = pair.get(1);
            dsu.join(x, y);
        }
//         System.out.println(dsu);
        // force union
        int temp;
        HashSet<Integer> hs = new HashSet<>();
        for (int i = 0; i < len; i++) {
            temp = dsu.find(i);
            hs.add(temp);
        }
        StringBuilder strb = new StringBuilder(s);
        for (Integer item : hs) {
            System.out.println("sort item: " + item);
            // collect item
            List<Character> ls = new ArrayList<>();
            List<Integer> pos = new ArrayList<>();
            for (int i = 0; i < len; i++) {
                if (dsu.parent[i] == item) {
                    ls.add(s.charAt(i));
                    pos.add(i);
                }
            }
            // sort
            Collections.sort(ls, Comparator.comparingInt(Character::charValue));
            System.out.println(ls);
            for (int i=0;i<ls.size();i++) {
                strb.setCharAt(pos.get(i), ls.get(i));
            }
        }

        return strb.toString();
    }
}
```

### #1319 连通网络的操作次数

{{< admonition quote "题目" true>}}

用以太网线缆将 n 台计算机连接成一个网络，计算机的编号从 0 到 n-1。线缆用 connections 表示，其中 connections[i] = [a, b] 连接了计算机 a 和 b

网络中的任何一台计算机都可以通过网络直接或者间接访问同一个网络中其他任意一台计算机

给你这个计算机网络的初始布线 connections，你可以拔开任意两台直连计算机之间的线缆，并用它连接一对未直连的计算机。请你计算并返回使所有计算机都连通所需的最少操作次数。如果不可能，则返回 -1

**示例 1**

输入：n = 4, connections = [[0,1],[0,2],[1,2]]

输出：1

解释：拔下计算机 1 和 2 之间的线缆，并将它插到计算机 1 和 3 上。

**示例 2**

输入：n = 6, connections = [[0,1],[0,2],[0,3],[1,2],[1,3]]

输出：2

**示例 3**

输入：n = 6, connections = [[0,1],[0,2],[0,3],[1,2]]

输出：-1

解释：线缆数量不足。

**示例 4**

输入：n = 5, connections = [[0,1],[0,2],[3,4],[2,3]]

输出：0

**提示**

`1 <= n <= 10^5`

`1 <= connections.length <= min(n*(n-1)/2, 10^5)`

`connections[i].length == 2`

`0 <= connections[i][0], connections[i][1] < n`

`connections[i][0] != connections[i][1]`

没有重复的连接。

两台计算机不会通过多条线缆连接。

{{< /admonition >}}

```java
class Solution {
    public int makeConnected(int n, int[][] connections) {
        int len = connections.length;
        if(len+1 < n) return -1;
        DSU dsu = new DSU(n);
        for(int i=0;i<len;i++) {
            dsu.join(connections[i][0], connections[i][1]);
        }
        // dsu.print();
        int ans = 0;
        for(int i=0;i<n;i++) {
            if(i == dsu.find(i)) ans++;
        }
        // System.out.println("ans: "+ans);
        return ans-1;
    }
}
```
