# 实时流的状态管理

在前面的章节中，关于实时流，我们关注更多的是对"流"的控制，比如流的DAG设计，流的反向压力，流的分散聚合（Fork/Join）等等。
但是这些都只是关注了流的执行过程。而流在执行过程中，必然会涉及到状态的管理。本章就来讨论与流有关的状态管理。

