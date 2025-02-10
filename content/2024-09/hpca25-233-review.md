+++
+++
# #233-Pfeife: Automatic Pipeline Parallelism for PyTorch

本文焦点是想透明的用pipeline parallelism来解决GPU内存不足以支撑大模型(训练还是推理?). 目标是降本增效.

核心就两点: pipeline parallelism是新的么? 透明是新的么? 效果是好的么(成本以及性能)? 如何体现的自动?

看figure 9发现本文的主要竞争对手是尤洋的Colossal-AI.
