# 深入理解Java Binder 和 MessageQueue

本章重点分析两个基础点:  
- Binder系统在Java世界如何布局和工作  
- MessageQueue的新职责  

## Java层中Binder架构分析
  Java层中Binder实际上也是一个C/S架构, 并且在其类的命名上尽量和Native层保持一致,因此可以认为,Java层的Binder架构是Native层Binder架构的一个镜像.