# PageRank算法的Graphscope实现
---

## 一、问题定义
PageRank算法最初是用于对互联网页面的排序和排名的。它通过分析每个页面的链接关系，给每个页面一个权重或分数，以反映其重要性或受欢迎程度。PageRank的基本假设是，一个页面的重要性取决于指向该页面的其他页面的数量和这些页面的权重，权重高的页面会更有利于提高该页面的重要性。PageRank算法在搜索引擎中被广泛使用，可以帮助用户快速找到他们需要的信息。除了搜索引擎，PageRank算法还被用于社交网络分析、推荐系统和其他网络分析应用中。

---
## 二、基于Graphscope的分布式算法实现

### (一)、语义定义
根据PageRank的原理，在context中定义相关的参数，主要是是保存结果的$result$队列和马尔科夫链的相关参数。另外context中初始化消息管理器和片段。
``` tex
alpha：阻尼系数
step：迭代次数
max_round：最大迭代次数
tolerance：用于检查幂迭代法求解器的收敛性的误差容忍值，超出范围迭代将停止
dangling_vnum：没有出边的节点的数量
```

### (二)、关键函数定义
$PEval$函数完成对每个网站权重的初始化以及找出所有的悬空节点
``` C++
void PEval(const fragment_t& frag, context_t& ctx,
            message_manager_t& messages) {
    auto inner_vertices = frag.InnerVertices();

    size_t graph_vnum = frag.GetTotalVerticesNum();
    messages.InitChannels(thread_num());

    ctx.step = 0;
    double p = 1.0 / graph_vnum;

    // 为每个网页分配初始权重
    // 传递的消息记录PR(u)/L(u)
    ForEach(inner_vertices.begin(), inner_vertices.end(),
            [&ctx, &frag, p, &messages](int tid, vertex_t u) {
                ctx.result[u] = p;
                ctx.degree[u] =
                    static_cast<double>(frag.GetOutgoingAdjList(u).Size());
                if (ctx.degree[u] != 0.0) {
                    messages.SendMsgThroughOEdges<fragment_t, double>(
                        frag, u, ctx.result[u] / ctx.degree[u], tid);
                }
            });

    for (auto u : inner_vertices) {
        if (ctx.degree[u] == 0.0) {
        ++ctx.dangling_vnum;
        }
    }

    double dangling_sum =
        ctx.alpha * p * static_cast<double>(ctx.dangling_vnum);

    Sum(dangling_sum, ctx.dangling_sum);

    messages.ForceContinue();
}
```
需要注意的是在不同的$worker$会分别计算一个$dangling\_sum$，之后这个值作为$ctx.dangling\_sum$的增量，用于之后$IncEval$函数对于各个网页$rank$值的计算

``` C++
void IncEval(const fragment_t& frag, context_t& ctx,
               message_manager_t& messages) {
    auto inner_vertices = frag.InnerVertices();

    double dangling_sum = ctx.dangling_sum;

    size_t graph_vnum = frag.GetTotalVerticesNum();

    ++ctx.step;
    // process received ranks sent by other workers
    // 接收从其他worker发送来的rank数据
    {
        messages.ParallelProcess<fragment_t, double>(
            thread_num(), frag, [&ctx](int tid, vertex_t u, const double& msg) {
            ctx.result[u] = msg;
            ctx.pre_result[u] = msg;
            });
    }

    // 计算每个节点的出边的权重
    ForEach(inner_vertices.begin(), inner_vertices.end(),
            [&ctx](int tid, vertex_t u) {
                if (ctx.degree[u] > 0.0) {
                ctx.pre_result[u] = ctx.result[u] / ctx.degree[u];
                } else {
                ctx.pre_result[u] = ctx.result[u];
                }
            });

    double base = (1.0 - ctx.alpha) / graph_vnum + dangling_sum / graph_vnum;
    ForEach(inner_vertices.begin(), inner_vertices.end(),
            [&ctx, base, &frag](int tid, vertex_t u) {
                double cur = 0;
                if (frag.directed()) {
                auto es = frag.GetIncomingAdjList(u);
                for (auto& e : es) {
                    cur += ctx.pre_result[e.get_neighbor()];
                }
                } else {
                auto es = frag.GetOutgoingAdjList(u);
                for (auto& e : es) {
                    cur += ctx.pre_result[e.get_neighbor()];
                }
                }
                ctx.result[u] = cur * ctx.alpha + base;
            });

    double eps = 0.0;
    ctx.dangling_sum = 0.0;
    for (auto& v : inner_vertices) {
        if (ctx.degree[v] > 0.0) {
        eps += fabs(ctx.result[v] - ctx.pre_result[v] * ctx.degree[v]);
        } else {
        eps += fabs(ctx.result[v] - ctx.pre_result[v]);
        ctx.dangling_sum += ctx.result[v];
        }
    }
    double total_eps = 0.0;
    Sum(eps, total_eps);
    if (total_eps < ctx.tolerance * graph_vnum || ctx.step > ctx.max_round) {
        return;
    }

    ForEach(inner_vertices.begin(), inner_vertices.end(),
            [&ctx, &frag, &messages](int tid, vertex_t u) {
                if (ctx.degree[u] > 0) {
                messages.SendMsgThroughOEdges<fragment_t, double>(
                    frag, u, ctx.result[u] / ctx.degree[u], tid);
                }
            });

    double new_dangling = ctx.alpha * static_cast<double>(ctx.dangling_sum);
    Sum(new_dangling, ctx.dangling_sum);

    messages.ForceContinue();
}
```