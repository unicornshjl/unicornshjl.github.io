---
title: 2024 ICPC Kunming E题解
date: 2026-03-12
description: 2024 ICPC Kunming E题解
#image: 1.png
categories:
  - 算法
  - 高斯消元
  - 题解

tags:
  - ICPC
  - 2024
  - Regional
  - 线性代数
  - BFS
  - LCA
  - 交互题
  - 构造
  - 树
---

$2024\space ICPC\space Kunming\space E$

看到题目和数据范围 容易想到用高斯消元去求最终的解

关键在于如何判断 $YES$ 还是 $NO$ 也就是判断什么情况下有解

根据线性代数的知识 如果我们能够构造出来的方程中 **存在 $n$ 个线性无关的方程** 那么是有解的

需要注意的是 我们**需要把 $a[1] = 0$ 这个条件作为一个方程**放入所有构造的方程中 

而不是将其作为已知条件 去判断**是否存在 $n - 1$ 个线性无关的方程** 

因为可能这些方程能解出 $a[1] = 0$ 从而导致方程个数不够

距离为 $k$ 的点对 可以利用 $BFS$ 直接暴力求出 时间复杂度 $O(N^3)$

然后对这些点对去重 因为 $u$ 和 $v$ 之间的距离为 $k$ 而 $v$ 和 $u$ 的距离也为 $k$

接下来 将所有这些方程构造成一个系数矩阵 判断这个系数矩阵是否存在 $n$ 个主元列即可

如何构造？ 因为 $n$ 的范围很小 我们可以直接暴力利用 $LCA$ 来得到路径上的所有点 对应位置填 $1$ 即可构造出方程

需要注意的是我们在高斯消元的过程中会交换整行 导致后续得到询问的点对产生错误 

所以我们可以额外用一个 $line$ 数组记录真实行 以及用 $all$ 数组来表示点对的两点信息

交换整行的时候 我们也交换一下 $line$ 然后每次记录下当前主元列对应的行是多少即可

如果说存在某一个主元列上的数为 $0$ 那么就表明无法构成 $n$ 个线性无关的方程 直接返回 `false` 即可

判断完有解之后 就变得很简单了 因为我们存了对应的行号以及原始每一行的点对信息

只需要注意 如果线性无关组中有第 $1$ 行 那么需要额外处理 因为第 $1$ 行的方程是 $a[1] = 0$ 它只有一个单点

同时如果包含 $1$ 那么我们只需要询问 $n - 1$ 次 否则 需要询问 $n$ 次

然后按照输出格式输出点对、输入询问的值、构造出增广矩阵

最后 跑一遍增广矩阵的高斯消元 输出 $2 - n$ 的所有解即可

时间复杂度 $O(N^3)$

代码如下 $:$

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
constexpr int maxn = 1e5 + 10;
constexpr int N = 23;
constexpr int base = 29;
constexpr int eps = 1e-7;
constexpr ll inf = 1e15;
constexpr int intinf = 1e9 + 10;
constexpr int dx[] = { 1,-1,0,0 };
constexpr int dy[] = { 0,0,1,-1 };

int n, k;

vector<int> edge[maxn];
vector<pair<int, int>> pairs;

int a[251][251], val[maxn], cnt, f[maxn][N], dep[maxn], line[maxn];

bitset<251> b[251 * 251];

vector<int> lines;

struct node {
    int x, y;
} all[maxn];

void add(int u, int v) {
    edge[u].push_back(v);
}

bool TryGauss() {
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= cnt; j++) {
            if (j < i && b[j][j] == 1) continue;
            if (b[j][i] == 1) {
                swap(b[i], b[j]);
                swap(line[i], line[j]);
                lines.push_back(line[i]);
                break;
            }
        }
        if (b[i][i] == 0)
            return false;
        for (int j = 1; j <= cnt; j++) {
            if (j == i || b[j][i] == 0) continue;
            b[j] ^= b[i];
        }
    }
    return true;
}

void Gauss() {
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n; j++) {
            if (j < i && a[j][j] == 1) continue;
            if (a[j][i] == 1) {
                swap(a[i], a[j]);
                break;
            }
        }
        for (int j = 1; j <= n; j++) {
            if (j == i || a[j][i] == 0) continue;
            for (int k = 1; k <= n + 1; k++)
                a[j][k] ^= a[i][k];
        }
    }
}

int get_lca(int u, int v) {
    if (dep[u] < dep[v]) swap(u, v);
    int k = dep[u] - dep[v];
    for (int i = 13; i >= 0; i--)
        if ((k >> i) & 1)
            u = f[u][i];
    if (u == v) return u;
    for (int i = 13; i >= 0; i--)
        if (f[u][i] != f[v][i])
            u = f[u][i], v = f[v][i];
    return f[u][0];
}

void dfs(int u, int fa) {
    f[u][0] = fa;
    dep[u] = dep[fa] + 1;
    for (int i = 1; i < 14; i++)
        f[u][i] = f[f[u][i - 1]][i - 1];
    for (auto v : edge[u]) {
        if (v == fa) continue;
        dfs(v, u);
    }
}

void bfs() {
    for (int i = 1; i <= n; i++) {
        queue<pair<int, int>> q; q.push({ i,0 });
        vector<bool> vis(n + 1);
        for (int j = 1; j <= n; j++) vis[j] = 0;
        vis[i] = 1;
        while (!q.empty()) {
            int u = q.front().first, dis = q.front().second;
            if (dis == k) {
                int x = u, y = i;
                if (x > y) swap(x, y);
                pairs.push_back({ x,y });
            }
            q.pop();
            for (auto v : edge[u]) {
                if (!vis[v]) {
                    vis[v] = 1;
                    q.push({ v,dis + 1 });
                }
            }
        }
    }
    sort(pairs.begin(), pairs.end());
    int len = unique(pairs.begin(), pairs.end()) - pairs.begin();
    pairs.resize(len);
}

void solve(int id) {
    cin >> n >> k;
    for (int i = 1; i < n; i++) {
        int u, v;
        cin >> u >> v;
        add(u, v); add(v, u);
    }
    bfs();
    dfs(1, 0);
    b[++cnt].set(1);
    all[cnt].x = 1; all[cnt].y = -1;
    line[cnt] = cnt;
    for (auto [x, y] : pairs) {
        int u = x, v = y, lca = get_lca(x, y);
        cnt++;
        line[cnt] = cnt;
        while (u != lca) {
            b[cnt].set(u);
            u = f[u][0];
        }
        while (v != lca) {
            b[cnt].set(v);
            v = f[v][0];
        }
        b[cnt].set(lca);
        all[cnt].x = x; all[cnt].y = y;
    }

    if (!TryGauss()) {
        cout << "NO" << endl;
        return;
    }

    bool flag = false;
    for (int i = 1; i <= n; i++)
        if (lines[i - 1] == 1)
            flag = true;
    cout << "YES" << endl;
    if (flag) cout << "? " << n - 1 << " ";
    else cout << "? " << n << " ";
    for (int i = 1; i <= n; i++) {
        int line = lines[i - 1];
        if (line == 1) continue;
        int x = all[line].x, y = all[line].y;
        cout << x << " " << y << " ";
    }
    cout << endl;
    for (int i = 1; i <= n; i++) {
        int line = lines[i - 1];
        if (line == 1) {
            a[i][1] = 1; a[i][n + 1] = 0;
        }
        else {
            int val;
            cin >> val;
            int u = all[line].x, v = all[line].y;
            int lca = get_lca(u, v);
            while (u != lca) {
                a[i][u] = 1;
                u = f[u][0];
            }
            while (v != lca) {
                a[i][v] = 1;
                v = f[v][0];
            }
            a[i][lca] = 1;
            a[i][n + 1] = val;
        }
    }
    Gauss();
    cout << "! ";
    for (int i = 2; i <= n; i++)
        cout << a[i][n + 1] << ' ';
    cout << endl;
}

signed main() {
    ios::sync_with_stdio(false); cin.tie(nullptr);
    int T = 1;
    for (int i = 1; i <= T; i++)
        solve(i);
    return 0;
}
```