---
title: 2025 北京市赛 H题解
date: 2026-03-12
description: 2025 北京市赛 H题解
#image: 1.png
categories:
  - 算法
  - 高斯消元
  - 题解

tags:
  - ICPC
  - 2025
  - 北京市赛
  - 线性代数
  - 构造
---

根据 $C_{i,j} = \oplus _{k = 1} ^ m A_{i,k} \& B_{k,j}$ 在已知 $C$ 和 $A$ 的情况下要求出 $B$ 我们考虑如何处理该式子

原式子即为 $C_{i,j} = A_{i,1} \& B_{1,j} \oplus A_{i,2} \& B_{2,j} \oplus \dots \oplus A_{i,m} \& B_{m,j}$ 其中 $1\le i \le n$ 同时 $1\le j \le p$

这就是说 如果我我们**把第 $j$ 列视为未知数集** 那么可以构造出 $n$ 个方程 这 $n$ 个方程的系数和解全部已知

而且 **每一列的值都是相互独立的** 第 $j$ 列的值与第 $j'$ 列并不想管

那么我们不妨就一列一列处理 构造方程进行高斯消元 只要有一行无解就输入 `No` 有解的情况下去得到一组解

考虑第 $j$ 列的情况下 构造方程是简单的 

由于 $\&$ 的性质 如果 $A_{i,k}$ 为 $1$ 那么 $B_k$ (由于列已经固定 此处省略了 $j$) 系数为 $1$ 否则为 $0$ 等号右边的值显然为 $C_{i,j}$

那么我们就可以得出以下代码

```cpp
#include<algorithm>
#include<bitset>
#include<climits>
#include<cmath>
#include<cstdio>
#include<cstdlib>
#include<cstring>
#include<iomanip>
#include<iostream>
#include<map>
#include<queue>
#include<set>
#include<string>
#include<unordered_map>
#include<unordered_set>
#include<vector>
#include<random>

using namespace std;

#define int long long
typedef long long ll;
typedef long double ld;
typedef unsigned long long ull;

constexpr ll mod = 998244353;

#define lson(x) (x << 1)
#define rson(x) (x << 1 | 1)

constexpr int maxn = 1e5 + 10;
constexpr int N = 23;
constexpr int base = 29;
constexpr int eps = 1e-7;
constexpr ll inf = 1e15;
constexpr int intinf = 1e9 + 10;
constexpr int dx[] = { 1,-1,0,0 };
constexpr int dy[] = { 0,0,1,-1 };


int n, m, p, a[1010][1010], cnt, c[1010][1010], M, ans[1010][1010];

bitset<1010> b[1001];

int get_pos(int x, int y) {
    return (x - 1) * p + y;
}

void Gauss() {
    for (int i = 1; i <= M; i++) {
        for (int j = 1; j <= M; j++) {
            if (j < i && b[j][j] == 1) continue;
            if (b[j][i]) {
                swap(b[j], b[i]);
                break;
            }
        }
        if (b[i][i] == 0) continue;
        for (int j = 1; j <= M; j++)
            if (j != i && b[j][i])
                b[j] ^= b[i];
    }
}

void solve(int id) {
    cin >> n >> m >> p;
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            cin >> a[i][j];
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= p; j++) 
            cin >> c[i][j];
    M = max(m, n);
    for (int j = 1; j <= p; j++) {
        for (int i = 1; i <= M; i++) b[i].reset();
        for (int i = 1; i <= n; i++) {
            for (int k = 1; k <= m; k++)
                b[i][k] = a[i][k];
            b[i][M + 1] = c[i][j];
        }
        Gauss();
        bool flag = true;
        for (int i = 1; i <= M; i++)
            if (b[i][i] == 0 && b[i][M + 1] == 1) {
                flag = false; break;
            }
        if (flag == false) {
            cout << "No\n";
            return;
        }
        for (int i = m; i >= 1; i--) {
            if (b[i][i] == 0) {
                ans[i][j] = 0;
                continue;
            }
            int res = b[i][M + 1];
            for (int k = i + 1; k <= m; k++)
                if (b[i][k])
                    res ^= ans[k][j];
            ans[i][j] = res;
        }
    }
    cout << "Yes\n";
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= p; j++)
            cout << ans[i][j] << " \n"[j == p];
}

signed main() {
    ios::sync_with_stdio(false); cin.tie(nullptr);
    int T = 1;
    //cin >> T;
    for (int i = 1; i <= T; i++)
        solve(i);
    return 0;
}
```



但是这段代码却 `TLE` 了 我们尝试分析复杂度

我们每次都对某一列进行高斯消元 即使使用了 `bitset` 优化 复杂度仍然为 $O(\frac{N^4}{w})$ 复杂度爆了

然后我们观察发现对于每一列 **构造出的方程的系数都是固定的，只有等号右边的值变化** 

($B_{k,j}$ 前的系数永远为 $A_{i,k}$ 并不会随着列数 $j$ 的变化而变化 而等号右边的值为 $C_{i,j}$ 会随着 $j$ 变化而变化)

也就意味着每次构造出的方程组的**系数矩阵完全相同** 而我们高斯消元的过程中**消元只对增广矩阵中的系数进行消元**

那么我们就可以把所有列一次性消元 具体地说 我们先将系数全部填入矩阵中去 然后对于第 $j$ 列我们将它的值$C_{i,j}$ 填入矩阵的第 $i$ 行第$j + M$ 列 (其中 $M$ 为 $max(n,m)$)  然后在高斯消元时一起消元即可

消元完以后 我们对于每一列还是去分别构造解 如果有一列发现无解 就输出 `No` 否则输出最终答案即可

整体时间复杂度为 $O(\frac{N^3}{w})$ 可以通过本题

代码如下 

```cpp
#include<algorithm>
#include<bitset>
#include<climits>
#include<cmath>
#include<cstdio>
#include<cstdlib>
#include<cstring>
#include<iomanip>
#include<iostream>
#include<map>
#include<queue>
#include<set>
#include<string>
#include<unordered_map>
#include<unordered_set>
#include<vector>
#include<random>

using namespace std;

//#define int long long
typedef long long ll;
typedef long double ld;
typedef unsigned long long ull;

constexpr ll mod = 998244353;

constexpr int maxn = 1e5 + 10;
constexpr int N = 23;
constexpr int base = 29;
constexpr int eps = 1e-7;
constexpr ll inf = 1e15;
constexpr int intinf = 1e9 + 10;
constexpr int dx[] = { 1,-1,0,0 };
constexpr int dy[] = { 0,0,1,-1 };


int n, m, p, a[1010][1010], cnt, c[1010][1010], M, ans[1010][1010];

bitset<2010> b[1001];

void Gauss() {
    for (int i = 1; i <= M; i++) {
        for (int j = 1; j <= M; j++) {
            if (j < i && b[j][j] == 1) continue;
            if (b[j][i]) {
                swap(b[j], b[i]);
                break;
            }
        }
        if (b[i][i] == 0) continue;
        for (int j = 1; j <= M; j++)
            if (j != i && b[j][i])
                b[j] ^= b[i];
    }
}

void solve(int id) {
    cin >> n >> m >> p;
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            cin >> a[i][j];
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= p; j++) 
            cin >> c[i][j];
    for (int i = 1; i <= n; i++) 
        for (int k = 1; k <= m; k++)
            b[i][k] = a[i][k];
    M = max(n, m);
    for (int i = 1; i <= n; i++)
        for (int k = 1; k <= p; k++)
            b[i][M + k] = c[i][k];
    Gauss();
    bool flag = true;
    for (int i = 1; i <= M; i++) {
        if (b[i][i] == 0) {
            for (int j = 1; j <= p; j++)
                if (b[i][M + j]) {
                    flag = false; break;
                }
        }
    }
    if (flag == false) {
        cout << "No\n";
        return;
    }
    for (int j = 1; j <= p; j++){
        for (int i = m; i >= 1; i--) {
            if (b[i][i] == 0) {
                ans[i][j] = 0;
                continue;
            }
            int res = b[i][M + j];
            for (int k = i + 1; k <= m; k++)
                if (b[i][k])
                    res ^= ans[k][j];
            ans[i][j] = res;
        }
    }
    cout << "Yes\n";
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= p; j++)
            cout << ans[i][j] << " \n"[j == p];
}

signed main() {
    ios::sync_with_stdio(false); cin.tie(nullptr);
    int T = 1;
    //cin >> T;
    for (int i = 1; i <= T; i++)
        solve(i);
    return 0;
}
```