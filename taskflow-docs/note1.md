TaskFlow中的基本元素

1. Atom：Atom是TaskFlow中的最小单位元素，是一个抽象类，也是Task类与Retry类的基类，声明了`execute()`抽象方法，用于执行一个动作。
2. Task：Task是一个继承了Atom类的抽象类，表示任务流中的一个任务，是不同类型任务类的基类。
3. Flow：Flow是一个抽象类，表示任务流中不同Task之间的关联关系，规定了Task类的执行与回滚顺序。TaskFlow为Flow提供了三种实现方式：graph_flow表示图流、linear_flow表示线性流、unordered_flow表示无序流。
4. Retry：Retry是一个继承了Atom类的抽象类，表示任务流在有错误发生时，如何进行重试操作，是不同类型重试类的基类。
5. Engine：Engine是一个抽象类，表示真正执行任务流的对象，是不同类型引擎类的基类。Engine的具体实现类提供了Flow载入方法、并且驱动Flow中Task对象的执行。



