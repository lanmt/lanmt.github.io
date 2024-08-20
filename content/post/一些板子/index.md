---
title: 一些板子
date: 2024-08-18
description: 背一背准备机试
categories:
    - algorithm
    - 保研
---
## 树状数组
```cpp
#define lowbit(x) (x&(-x))
int tree[maxn];
int query(int i) {
    int ans = 0;
    for (;i;i -= lowbit(i)) {
        ans += tree[i];
    }
    return ans;
}
void update(int i, int x) {
    for (;i < maxn;i += lowbit(i)) {
        tree[i] += x;
    }
}
```
## trie树
```cpp
int trie[maxn][26];
int cnt[maxn];
int idx = 0;
void insert(string str) {
    int now = 0;
    for (auto c : str) {
        if (!trie[now][c - 'a'])trie[now][c - 'a'] = ++idx;
        now = trie[now][c - 'a'];
    }
    //标记字符串末尾
    cnt[now]++; 
}
int query(string str) {
    int now = 0;
    for (auto c : str) {
        if (!trie[now][c - 'a'])return 0;
        now = trie[now][c - 'a'];
    }
    return cnt[now];
}
```
## lca
```cpp
int fa[maxn][20];
int dep[maxn];
vector<int>G[maxn];

//预处理每个节点向上跳2^i次的位置
void dfs(int u, int f) {
    fa[u][0] = f;
    dep[u] = dep[f] + 1;
    for (int i = 1;i <= 19;i++) {
        fa[u][i] = f[fa[u][i - 1]][i - 1];
    }
    for (auto v : G[u]) {
        if (v == f)continue;
        dfs(v, u);
    }
}

int lca(int u, int v) {
    if (v == u)return u;
    if (v > u)swap(u, v);
    for (int i = 19;i >= 0;i--) {
        if (dep[fa[u][i]] >= dep[v]) u = fa[u][i];
    }
    if (v == u)return u;
    for (int i = 19;i >= 0;i--) {
        if (fa[u][i] != fa[v][i]) {
            u = fa[u][i];
            v = fa[v][i];
        }
    }
    return fa[u][0];
}
```
## 线性筛
```cpp
int primes[maxn];// 0 2 3 5 7...
//为1被筛掉
bitset<maxn>vis;
int pp = 0;

void shai() {
    vis[1] = 1;
    for (int i = 2;i <= maxn;i++) {
        if (!vis[i])primes[++pp] = i;
        for (int j = 1;primes[j] * i < maxn;j++) {
            vis[primes[j] * i] = 1;
            // 如果i是质数，primes[j]==i的时候break
            if (i % primes[j] == 0)break;
        }
    }
}
```
## dijkstra
```cpp
//还好看了一眼，前两天睿抗国赛的时候搓出来了，美美国一嘿嘿


//在存图的时候，v表示终点，w表示边权
//在跑dij的时候，v表示当前的点，w表示距离起点的距离
struct node {
    int v, w;
    bool operator<(node const n)const {
        return w > n.w;
    }
};
vector<node>G[maxn];
int len[maxn];
int vis[maxn];
void dij(int s) {
    for (int i = 1;i < maxn;i++)len[i] = inf;
    len[s] = 0;
    priority_queue<node>pq;
    pq.push({ s,0 });
    while (!pq.empty()) {
        auto tmp = pq.top();pq.pop();
        if (vis[tmp.v])continue;
        vis[tmp.v] = 1;
        for (auto [v, w] : G[tmp.v]) {
            if (vis[v])continue;
            if (len[v] >= len[tmp.v] + w) {
                len[v] = len[tmp.v] + w;
                pq.push({ v,len[v] });
            }
        }
    }
}
```
## 组合数
```cpp

//f[i][j]=f[i−1][j]+f[i−1][j−1]  n方求组合数
const int maxn = 200010;
const int mod = 1e9 + 7;
int fac[maxn + 1], inv[maxn + 1];
int quickPow(int a, int b) {
    a %= mod;
    int ans = 1;
    while (b) {
        if (b & 1)
            ans = (ans * a) % mod;
        b >>= 1;
        a = (a * a) % mod;
    }
    return ans;
}
void init() {
    fac[0] = 1;
    for (int i = 1;i <= maxn;i++)fac[i] = fac[i - 1] * i % mod;
    inv[maxn] = quickPow(fac[maxn], mod - 2);
    for (int i = maxn - 1;i >= 0;i--)inv[i] = inv[i + 1] * (i + 1) % mod;
}
int C(int n, int m) {
    if (m > n)return 0;
    return fac[n] * inv[m] % mod * inv[n - m] % mod;
}
```
## kmp
```cpp
//kmp[i]表示模式串中i位置对应的最大相等前后缀的 前缀的末尾
int kmp[maxn];
string a, b;
void getKmp() {
    int l = 0, r = 1;
    while (r < b.size()) {
        while (l && b[l] != b[r]) l = kmp[l - 1];
        if (b[l] == b[r])l++;
        kmp[r] = l;
        r++;
    }
}
void find() {
    int na = 0, nb = 0;
    while (na < a.size()) {
        while (nb && a[na] != b[nb])nb = kmp[nb - 1];
        if (a[na] == b[nb])nb++;
        na++;
        if (nb == b.size()) {
            cout << na - nb + 1 << endl;
            nb = kmp[nb - 1];
        }
    }
}
```
## 马拉车
略
## 字符串hash
```cpp
const int mod = 1e9+7;
const int b = 133331;
int p[maxn], hash[maxn];
string s = "#";
void init() {
    p[0] = 1;
    for (int i = 1;i < maxn;i++)p[i] = p[i - 1] * b % mod;
    for (int i = 1;i < s.size();i++)hash[i] = (hash[i - 1] * b % mod + s[i]) % mod;
}
int cut(int l, int r) {
    return ((hash[r] - hash[l] * p[r - l] % mod) % mod + mod) % mod;
}
```
## 最大字段和
```cpp
int n;
int a[maxn];
int ma() {
    // f表示选中当前的数最大的和，或者是0
    int m = 0, f = 0;
    for (int i = 1;i <= n;i++) {
        f = max((int)0, f + a[i]);
        m = max(m, f);
    }
    
    /*
    如果必须要选一个
    if (m == 0)return *max_element(a + 1, a + n + 1);
    */
    return m;
}
```
## exgcd
```cpp
void exgcd(int a, int b, int& x, int& y) {
    (!b) && (x = 1) && (y = 0);
    if (!b)return;
    exgcd(b, a % b, y, x);
    y -= (a / b) * x;
}
gcd = __gcd(a, b);
dx = b / gcd;
dy = a / gcd;
tx = x0 * c / gcd;
ty = y0 * c / gcd;
//x = tx + dx * k
//y = ty - dy * k
```