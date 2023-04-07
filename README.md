# UCAS-CACT-LAB-REPORTS
## Introduction
This is our lab reports for UCAS compiler seminar 2021-2022 Spring. 

This is toy compiler from a subset of C to RISCVGC, and we implemented some naive data-flow based optimizations / algorithms in the backend:
+ constant fold and propagation
+ live variable analysis
+ register allocation
+ ...


Most of our designs and implementation details are just in the reports and we hope them can be helpful to your practice!

## Recommend Reading Materials
+ https://pku-minic.github.io/online-doc/#/(compiler cookbook from PKU)
+ https://cs.nju.edu.cn/changxu/2_compiler/index.html(compiler cookbook from NJU and targets at MIPS)
+ [Enginering  a Compiler](https://www.elsevier.com/books/engineering-a-compiler/cooper/978-0-12-815412-0)
+ [SSA book](https://pfalcon.github.io/ssabook/latest/book-v1.pdf) (if you want to do some SSA-based optimization and have enough time)
+ https://zhuanlan.zhihu.com/p/25042028(a very helpful answer collection)
