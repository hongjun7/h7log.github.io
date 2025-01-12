---
title: "Accurate Sampling-Based Cardinality Estimation for Complex Graph Queries"
tags: query cardinality estimation
subtitle: This post provides a novel, strongly consistent method for accurately and efficiently estimating the cardinality of complex graph queries.
---
Accurate cardinality estimation (i.e., predicting the number of answers to a query) is a vital aspect of database systems, particularly in graph databases where queries often involve numerous joins and self-joins.

Recent studies identified that a sampling method based on the [WanderJoin algorithm](https://dl.acm.org/doi/pdf/10.1145/2882903.2915235) is highly effective in estimating cardinality for graph queries. However, this method has limitations, especially in handling complex queries and performing efficiently on large datasets.

This post explores a novel approach inspired by WanderJoin that can effectively handle complex graph queries and improve performance.

## Preliminaries

Let's consider the following SPARQL query.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/query1.png?raw=true" width="100%"> </p>

The nested **DISTINCT** operators were intended to reduce bindings for the $?mounting$ and $?motor$ variables, improving query efficiency. However, the RDF system had difficulty creating an efficient query plan due to inaccurate cardinality estimates for the **UNION** and **DISTINCT** subqueries, as well as their joins.

In summary, there are following challenges:

- Queries often involve multiple joins and self-joins.
- Data distributions can be skewed, leading to inaccurate or zero estimates.
- Cardinality estimation needs to account for complex graph operations like union, projection, and duplicate elimination. Here is the syntax and the semantics of the SPARQL.
  <p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/tab1.png?raw=true" width="100%"> </p>

Existing methods struggle with such complexities and may produce slow or inaccurate results on larger datasets.

## Related Approaches to Query Cardinality Estimation

### Principles of Sampling-based Cardinality Estimation

Most of sampling-based cardinality estimation methods can be viewed as variations of a [method introduced by Lipton and Naughton](ttps://dl.acm.org/doi/pdf/10.1145/298514.298540).

The approach involves partitioning the set of query results $ A $ from a database $ I $ into disjoint subsets $ A_1, A_2, ... , A_n $, allowing for efficient calculation of the size of each subset. By randomly selecting one subset and using its size to estimate the total cardinality( $ n \lvert A_i \rvert $ ), an unbiased estimate is obtained because $ A_1, ... , A_n $ are disjoint.

$$
( \sum_{i=1}^{n} n \lvert A_i \rvert )/n = \sum_{i=1}^{n} \lvert A_i \rvert = \lvert A \rvert
$$

The method is adaptable to various query types, including recursive path and datalog queries.  Moreover, estimation accuracy can be improved by computing the average of several samples, and a key question is how many samples should be taken. For certain classes of queries, it is possible to precompute the number of samples so the resulting estimate is within desired bounds. Alternatively, one can keep taking samples until the estimate falls within a confidence interval computed.

### The WanderJoin Algorithm

WanderJoin randomly selects facts from the database instance for each atom (query component) but ensures that these selections are not independent.

For example, let $ Q = AND(R(x, y), S(y, z), T(z, x)) $ be the example query.

When evaluating $ Q $, it first selects a fact from relation $R$, then matches facts from $S$ that join correctly with $R$, and finally matches $T$ to complete a valid query answer.

The order in which atoms are sampled critically determines the accuracy of the estimates produced by WanderJoin.

The advantage is its flexibility, as it doesn't need predefined schemas, making it ideal for unpredictable graph database queries.

[Li et al.](https://dl.acm.org/doi/pdf/10.1145/3284551) emphasize the need for an optimal atom order to improve cardinality estimates, proposing trial runs with all reasonable orders and selecting the least costly one for further trials. [Park et al.](https://personal.ntu.edu.sg/assourav/papers/SIGMOD-20-GCARE.pdf) integrate this into G-CARE as the **WJ** method, which averages 30 estimation attempts to find the final result. Orders are tested in a round-robin fashion, and the best one, based on variance, is used for the remaining runs.

The **WJ** method handles only conjunction queries, while [Li et al.](https://dl.acm.org/doi/pdf/10.1145/3284551)'s approach supports **AGGREGATE** queries but not **DISTINCT** queries or nested operators.


### Cardinality Estimation Methods Used in the G-CARE Framework

The G-CARE framework's cardinality estimation algorithms, as described by [Park et al.](https://personal.ntu.edu.sg/assourav/papers/SIGMOD-20-GCARE.pdf), are classified into two categories: **synopsis-based** and **sampling-based** methods.

#### Synopsis-based methods:

- Bounded Sketch (BS) : Divides relations into partitions, recording constants and max degree for each, and uses bounding formulas to estimate cardinality based on query structure.
- Characteristic Sets (C-Set) : Summarizes star-shaped structures in the database and estimates cardinality by decomposing queries into subqueries and combining results using independence assumptions.
- SumRDF : Merges graph vertices to summarize the graph and uses possible worlds semantics to estimate query cardinality with certainty bounds.

#### Sampling-based methods:

- $ J_{SUB} $ : Samples a fact matching the first query atom, evaluates the rest with this atom bound, similar to CS2 but samples for each estimate.
- Correlated Sampling (CS) : Uses hashing to create a database synopsis instead of sampling data.
- $ I_{MPR} $ : Adapts random walks to estimate the number of graphlets in a visible subgraph, tailored for graph matching and directed labelled graphs.

## Motivation

Consider queries $Q_1$, $Q_2$ and a database instance $I$ as follows.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/query2.png?raw=true" width="100%"> </p>

To apply the approach from the method proposed by Lipton and Naughton, we could partition $ I $ into $ I_i = \lbrace R(a, b_i), S(b_i, c) \rbrace $ for $ 1 ≤ i ≤ k $; cleary, $ ans_{I}(Q) = \bigcup_{i=1}^{k} ans_{I_i}(Q_2) $, as required. To estimate the cardinality of $ Q_2 $ in $ I $, we randomly choose $ i ∈ {1, ..., k} $ and return $ k \lvert ans_{I_i}(Q_2) \rvert $ as the estimate. Since $ I_i $ is much smaller than $ I $, computing $ \lvert ans_{I_i}(Q_2) \rvert $ is likely to be much faster than computing $ \lvert ans_{I}(Q_2) \rvert $.

Duplicate elimination reduces the number of answers in a way that can prevent effective partitioning. Indeed, $ ans_{I}(Q) ≠ \bigcup_{i=1}^{k} ans_{I_i}(Q_2) $, so the partitioning from the previous paragraph is not applicable. In fact, it is unclear how to partition $ I $ into $ I_1, ... , I_n $ so $ ans_{Q_1}(I) = \bigcup_{i=1}^{n} ans_{Q_1}(I_i) $ holds but computing $ \lvert ans_{I_i}(Q_2) \rvert $ is much faster than computing $ \lvert ans_I(Q_2) \rvert $.

The approach in the next chapter addresses these problems. In particular, it can process query $ Q_1 $ efficiently, and it is applicable even if $ Q_1 $ is conjoined with another query.

Overall there are two main issues:

- Existing methods cannot handle complex queries with arbitrary operator nesting
- Partitioning techniques struggle with duplicate elimination

## Strongly Consistent Cardinality Estimator for Complex Queries

This chapter explains the cardinality estimation approach by outlining the query evaluation, underlying intuitions, the basic algorithm with its optimisation, and practical considerations.

### Query Evaluation via Sideways Information Passing

Standard query evaluation typically proceeds in a bottom-up manner. For instance, when evaluating a query such as $ Q = Q_1 $ AND $ Q_2 $, the results of $ Q_1 $ and $ Q_2 $ are computed individually before joining the two results.

However, this process can be optimised using a technique known as ***sideway information passing***. This technique allows one operator in a query plan to pass potential variable bindings to other operators, enabling them to eliminate irrelevant tuples early on. By doing so, the evaluation becomes more efficient without needing to fully compute the entire result set. Such techniques have been applied to various types of queries, including relational, RDF, recursive, and visual query processing.

In particular, the procedure evalI(Q, σ) uses a variant of sideways information passing to evaluate a query $ Q $ over a database instance $ I $. While this algorithm can be practical, the paper's main point is conceptual: the cardinality estimation algorithm can be seen as "sampling the loops" of evalI. Essentially, the algorithm functions similarly to the magic sets transformation but is adapted for more complex, nested queries. For example, when evaluating $ Q = Q_1 $ AND $ Q_2 $, the algorithm enumerates all the answers for $ Q_1 $ and then evaluates $ Q_2 $ in the context of each answer.

The algorithm facilitates sideways information passing through the substitution $ σ $, which provides context for subqueries that are evaluated prior to $ Q $. Only answers compatible with this context substitution are produced. The algorithm is presented as a generator, where each invocation of $eval_{I}(Q, σ)$ provides an iterator, and the output keyword adds one substitution to the iterator result. Additionally, the algorithm is structured to process one tuple at a time, which, while potentially incurring random access costs, can be offset by the benefits of sideways information passing, especially in RAM-based databases.

This sideways information passing significantly reduces the number of tuples processed and ensures the evaluation is efficient, even for complex queries. The algorithm also incorporates optimisation techniques like nested loop joins, duplicate removal, and minimisation strategies to handle a wide range of query types.

The paper demonstrates that the algorithm is practical in specific cases and that its prototype implementation effectively evaluates queries with these optimisations in place.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/alg1.png?raw=true" width="100%"> </p>

### Principles for Estimating the Cardinality of Complex Queries

The WanderJoin algorithm proposes an optimization technique called "sampling the loops" which, unlike traditional algorithms that iterate over all matcher combinations, selects just one pair of matchers to generate an answer. This selective process leverages sideways information passing, significantly reducing the complexity required to approximate the full result set of a query. Several examples illustrate WanderJoin's versatility across query types, such as UNION, MINUS, and DISTINCT, where partial sampling provides unbiased estimates.

For instance, in **UNION** queries, sampling a single matcher and its candidate count yields an unbiased result, with repeated runs converging toward the actual cardinality.

In **MINUS** queries, only valid matches are selected, enhancing accuracy.

With **DISTINCT** queries, the algorithm avoids duplicates by choosing unique matches within unions, thereby minimizing estimation bias.

This approach effectively reduces variance by averaging over multiple sampling runs, ultimately converging on highly accurate cardinality estimates. Notably, it achieves efficient and reliable results without exhaustive data examination.

### The Basic Cardinality Estimation Approach

**Algorithm 2** is designed to estimate the cardinality of query results by leveraging a sampling-based method. Taking as inputs a database instance, a query, and a context substitution, the algorithm returns a structured triple [ω, β, c] representing sampled outcomes, a substitution satisfying the query conditions, and a cardinality estimate. In cases where no answer is found, the algorithm signals failure. This process incorporates loop sampling, enabling random selections from a sample space optimised via fixed probability distributions.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/alg2.png?raw=true" width="100%"> </p>

For **DISTINCT** queries, the algorithm maintains a global mapping of substitutions to ensure unique outcomes, mirroring the structure of a Horvitz–Thompson estimator. The approach is unbiased and consistent, allowing conjunctions, disjunctions, and filters to be managed through recursive calls that account for both successful estimates and failures. The algorithm’s general structure can be extended to graph query languages and aggregation functions, though certain complex queries, like nested aggregations, may present additional challenges. Further refinements are planned to incorporate more advanced query forms, such as Conjunctive Regular Path Queries (CRPQs).

### Optimising the Basic Approach

There is an improved version of **Algorithm 2** that reduces variance and computation time by refining the sampling process. Instead of maintaining Horvitz–Thompson estimations for all calls, the new approach uses unbiased but simplified estimates, especially for **UNION** and conjunctive queries. By sampling without replacement in certain cases, it achieves broader sample coverage and lower estimator variance. **Algorithm 3** aggregates independent estimates for query components, offering faster, more consistent cardinality estimates while retaining accuracy for various query structures.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/alg3.png?raw=true" width="100%"> </p>

### Practical Considerations

There are key optimisations to make **Algorithm 2** and **Algorithm 3** more practical:

1. **Order Enumeration:** Determining an optimal order for atoms in a conjunction (e.g., **AND** clauses) can reduce variance. Rather than testing all permutations, only select a subset where the choice of the first atom significantly impacts the estimation, reducing computational effort for cases like star queries.

2. **Single Order Selection:** To further decrease variance, **Algorithm 4** uses simple data statistics to order atoms by minimising sample space choices. Although this ordering can be imprecise, Algorithms 2 and 3 compensate with accurate cardinality estimates.

3. **Dynamic Stopping Condition:** To avoid excessive runs, a dynamic stopping criterion adjusts based on variance and q-error targets, ensuring an efficient yet reliable number of runs for cardinality estimation.

4. **Partitioned Sample Space:** Sample space is partitioned into fixed-size blocks to better capture relevant data without excessive computation, particularly in complex queries.

5. **Combining Algorithms:** For complex queries that return zero estimates, a fallback from **Algorithm 2** to the optimised **Algorithm 3** ensures more accurate results without overloading simpler queries.

6. **Dependency-Directed Backtracking:** When an atom fails to match, the algorithm backtracks to earlier matches that determined variable bindings, avoiding redundant searches and enhancing efficiency in complex joins.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/alg4.png?raw=true" width="100%"> </p>

## Integrating Cardinality Estimation into Query Planning

Accurate cardinality estimates can greatly enhance query performance, particularly for complex queries. **Algorithm 5** introduces a dynamic programming-based query planner that builds efficient query orders by minimising costly cross-products and reducing unnecessary substitutions.

It evaluates each atom’s placement within a query to identify the order with the lowest cumulative cost based on cardinality estimates. This approach dynamically selects the most efficient order at each stage, significantly reducing query evaluation times while adding only minimal overhead. Experimental results confirm that this planner produces more optimised query plans, showcasing the value of data-informed orderings in streamlining query processing.

<p align="center"> <img src="https://github.com/hongjun7/logs/blob/main/_posts/image/2024-10-29-Accurate-Sampling-Based-Cardinality-Estimation/alg5.png?raw=true" width="100%"> </p>

## Reference

- [Hu, Pan, and Boris Motik. &#34;Accurate Sampling-Based Cardinality Estimation for Complex Graph Queries.&#34; ACM Transactions on Database Systems 49.3 (2024): 1-46.](https://dl.acm.org/doi/pdf/10.1145/3689209)
- [Li, Feifei, et al. &#34;Wander join: Online aggregation via random walks.&#34; Proceedings of the 2016 International Conference on Management of Data. 2016.](https://dl.acm.org/doi/pdf/10.1145/2882903.2915235)
- [F. Li, B. Wu, K. Yi, and Z. Zhao. 2019. Wander join and XDB: Online aggregation via random walks. ACM Trans. Datab. Syst. 44, 1 (2019), 2:1–2:41.](https://dl.acm.org/doi/pdf/10.1145/3284551)
- [Y. Park, S. Ko, S. S. Bhowmick, K. Kim, K. Hong, and W.-S. Han. 2020. G-CARE: A framework for performance benchmarking of cardinality estimation techniques for subgraph matching. In Proceedings of the 41st ACM SIGMOD International Conference on Management of Data (SIGMOD’20). ACM Press, 1099–1114.](https://personal.ntu.edu.sg/assourav/papers/SIGMOD-20-GCARE.pdf)
- [Lipton, R. J., & Naughton, J. F. (1990, April). Query size estimation by adaptive sampling. In Proceedings of the ninth ACM SIGACT-SIGMOD-SIGART symposium on Principles of database systems (pp. 40-46).](https://dl.acm.org/doi/pdf/10.1145/298514.298540)