## 从程序入口到开始仿真

main.c: `main` calls `cp_evloop`.

frontend/control.c: `cp_evloop` $\rightarrow$ `doblock` $\rightarrow$ `docommand` $\rightarrow$ `command->co_argfn`, `command` is `spcp_coms[i]`, type `comm *`.

include/ngspice/cpdefs.h: defines type `comm`.

frontend/command.c: defines `spcp_coms`, for "run" command `co_argfn` is `com_run`.

frontend/runcoms.c: `com_run` $\rightarrow$ `dosim` $\rightarrow$ `if_run`

frontend/spiceif.c: `if_run` *do a run of the circuit*

## 瞬态仿真的主函数

frontend/spiceif.c: `if_run` 中, 真正开始求解是调用 `ft_sim->doAnalyses`.

include/ngspice/ifsim.h: 定义了 `*ft_sim` 的类型 `IFsimulator`.

main.c: `main` 中调用 `SIMinit` 将 `ft_sim` 初始化为 `&SIMinfo`.

ngspice.c: 定义了 `SIMinfo`, `SIMinfo.doAnalyses` 指向 `CKTdoJob`.

spicelib/analysis/cktdojob.c: `CKTdoJob` 中根据任务类型调用 `analInfo[i]->an_func` 来求解.

spicelib/analysis/analysis.h: 定义了 `*analInfo[i]` 的类型 `SPICEanalysis`.

spicelib/analysis/analysis.c: 定义了 `analInfo` 数组，对应瞬态仿真的元素是 `TRANinfo`.

spicelib/analysis/transetp.c: 定义了 `TRANinfo`,  `TRANinfo.an_func` 指向 `DCtran`.

spicelib/analysis/dctran.c: `DCtran` is *subroutine to do DC TRANSIENT analysis*.

## 具体求解的线性方程组

spicelib/analysis/dctran.c: `DCtran` 中每个时间步调用 `CKTdump` 输出时间点信息 (这个函数只有与上次输出时间点间隔足够才会真的输出), 并调用 `NIiter` 求解.

maths/ni/niiter.c: *This subroutine performs the actual numerical iteration. It uses the sparse matrix stored in the circuit struct along with the matrix loading program, the load data, the convergence test function, and the convergence parameters*. `NIiter` 中的每个迭代步会调用 `CKTload` 加载需要求解的方程组, 包括系数矩阵和右端项. 之后调用重排序, LU 分解, 求解等等.

spicelib/analysis/cktload.c: `CKTload` 的参数类型 `CKTcircuit`，其成员 `(SMPmatrix *) CKTmatrix` 保存了系数矩阵, `(double *) CKTrhs` 保存了右端项. 在这里修改代码导出线性方程组是比较好的选择.

include/ngspice/cktdefs.h: 定义了 `CKTcircuit`.

include/ngspice/smpdefs.h: `SMPmatrix` 的实际类型是 `MatrixFrame`, 但由于接口实现隔离, 无法直接访问. 只能通过提供的函数来操作 `SMPmatrix`, 相当于搞了个 C++ 的类, 如 `SMPaddElt` 添加元素,  `SMPsolve` 求解线性方程组,  `SMPmatSize` 得到大小,  `SMPnewMatrix` 创建对象... 有两个函数 `SMPprint` 和 `SMPprintRHS` 可以用于打印系数矩阵和右端项.