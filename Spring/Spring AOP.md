### 一、基础

AOP术语

- 通知(Advice):在连接点处增强的方法
- 连接点(JoinPoint):连接点是在应用执行过程中能够插入切面(Aspect)的一个点
- 切点(PointCut):指通知(Advice)所要织入(Weaving)的具体位置
- 切面(Aspect):切面是通知和切点的结合
- 引入(Introduction):引入允许我们向现有的类添加新的方法或者属性
- 织入(Weaving):将增强处理添加到目标对象中，并创建一个被增强的对象，这个过程就是织入

