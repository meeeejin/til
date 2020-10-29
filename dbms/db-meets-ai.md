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
    - 다른 columns/tables 간에 상관 관계에 기반한 optimized plan을 제공하기 위해 deep NN를 사용; For cost/cardinality estimiatoin
    - A. Kipf, T. Kipf, B. Radke, V. Leis, P. A. Boncz, and A. Kemper. Learned cardinalities: Estimating correlated joins with deep learning. In CIDR 2019, 2019.
    - J. Ortiz, M. Balazinska, J. Gehrke, and S. S. Keerthi. Learning state representations for query optimization with deep reinforcement learning. In DEEM@SIGMOD, pages 4:1–4:4, 2018.

2. Join order selection
    - Deep reinforcement learning에 기반해 good plan을 자동으로 선택

3. End-to-end optimizer
    - Deep NN에 기반해 SQL 쿼리를 최적화하는 learning-based optimizer 등장
    - R. C. Marcus, P. Negi, H. Mao, C. Zhang, M. Alizadeh, T. Kraska, O. Papaemmanouil, and N. Tatbul. Neo: A learned query optimizer. PVLDB, 12(11):1705–1718, 2019
    - C. Wu, A. Jindal, S. Amizadeh, H. Patel, W. Le, S. Qiao, and S. Rao. Towards a learning optimizer for shared clouds. PVLDB, 12(3):210–222, 2018

### Learning-based database design

1. Learned indexes
    - Index 크기 감소; Indexing 성능 향상
    - T. Kraska, A. Beutel, and E. H. C. et al. The case for learned index structures. In SIGMOD, pages 489–504, 2018.

2. Learned data structure design
    - 실험 환경 (HW 종류, 어플리케이션의 RW 비율 등)에 따라 적절한 data structure가 다름; 매 환경에 맞춰 설계하는 건 어려움
    - 적절한 data structure를 추천해주는 data inference engine 제안
    - S. Idreos, N. Dayan, W. Qin, M. Akmanalp, S. Hilgard, A. Ross, J. Lennon, V. Jain, H. Gupta, D. Li, and Z. Zhu. Design continuums and the path toward self-designing key-value stores that
know and learn. In CIDR, 2019.

3. Learning-based transaction management
    - Transaction을 예측하고 스케줄링하는 기술 등장
    - 기존 데이터 패턴을 학습하고 미래 워크로드를 예측해 conflict rates <=> concurrency 간 밸런스를 맞춤
    - L. Ma, D. V. Aken, and A. H. et al. Query-based workload forecasting for self-driving database management systems. In SIGMOD 2018, pages 631–645, 2018.
    - Y. Sheng, A. Tomasic, T. Sheng, and A. Pavlo. Scheduling OLTP transactions via machine learning. CoRR, abs/1903.02990, 2019

### Learning-based database monitoring

- To determine when and how to monitor which database metrics
- H. Kaneko and K. Funatsu. Automatic database monitoring for process control systems. In IEA/AIE 2014, pages 410–419, 2014.
- H. Grushka-Cohen, O. Biller, O. Sofer, L. Rokach, and B. Shapira. Diversifying database activity monitoring with bandits. CoRR, abs/1910.10777, 2019.

### Learning-based database security

1. Sensitive data discovery
    - ML을 사용해 자동으로 sesitive data 탐지

2. Anomaly detection
    - DB activity를 감시하고 취약점 탐지
    - Z. Lin, X. Li, and X. Kuang. Machine learning in vulnerability databases. In ISCID 2017, pages 108–113, 2017.

3. Accesss control
    - 다양한 data access action을 자동으로 예측하여 data leak 방지
    - M. L. Goyal and G. V. Singh. Access control in distributed heterogeneous database management systems. Computers & Security, 10(7):661–669, 1991.

4. SQL injection attacks
    - 유저의 행동을 파악해 SQL injection attack 식별
    - P. Tang, W. Qiu, Z. Huang, H. Lian, and G. Liu. SQL injection behavior mining based deep learning. In ADMA 2018, pages 445–454, 2018.
    - H. Zhang, B. Zhao, H. Yuan, J. Zhao, X. Yan, and F. Li. SQL injection detection based on deep belief network. In CSAE 2019, pages 20:1–20:6, 2019.

## DB for AI
