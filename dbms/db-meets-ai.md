# Database Meets AI: A Survey

- [X. Zhou, C. Chai, G. Li and J. SUN, "Database Meets Artificial Intelligence: A Survey," in IEEE Transactions on Knowledge and Data Engineering, doi: 10.1109/TKDE.2020.2994641.](https://ieeexplore.ieee.org/document/9094012)

> 아직 정리 중

## AI for DB

### Learning-based database configuration

1. Knob tuning
    - DB는 튜닝할 수 있는 수많은 knob이 있고, 이 knob들을 매번 DBA가 튜닝하기는 어려움; Learning 기반 knob을 자동으로 튜닝해주는 다양한 연구 존재
    - e.g., 워크로드 기반 knob 튜닝 값 추천, 현재 워크로드에선 불필요한 knob 안내 등
    - D. V. Aken, A. Pavlo, G. J. Gordon, and B. Zhang. Automatic database management system tuning through large-scale machine learning. In SIGMOD 2017, pages 1009–1024, 2017.
    - G. Li, X. Zhou, and S. L. et al. Qtune: A query-aware database tuning system with deep reinforcement learning. VLDB, 2019.
    - J. Zhang, Y. Liu, K. Zhou, G. Li, Z. Xiao, B. Cheng, J. Xing, Y. Wang, T. Cheng, L. Liu, M. Ran, and Z. Li. An end-to-end automatic cloud database tuning system using deep reinforcement learning. In SIGMOD 2019, pages 415–432, 2019.

2. Index/view advisor
    - 적절한 index/view를 만들고 유지할 수 있도록 learning 기반 추천

3. SQL rewriter
    - Nested 쿼리를 join 쿼리로 바꿔주는 등 좀 더 효율적이고 더 나은 성능을 보이는 SQL 쿼리를 쓸 수 있도록 추천
    - e.g., Rule-based strategy (high-quality rule에 의존, cannot be scale to a large number of rules) -> deep reinforcing learning (to judiciously select the appropriate rules and apply the rules in a good order)

### Learning-based database optimization

1. Cardinality/cost estimation


## DB for AI
