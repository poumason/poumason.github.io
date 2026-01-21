---
title: 學習 Ansys RSM 執行 fluent 模擬流體與力學
date: 2025-11-28 22:39:26
categories: 3D
tags:
    - Fluent
    - Ansys
---
過去使用 Ansys 在跑專案的時候都是靠 CPU，隨著 GPU 越來越越多的討論，開始思考要怎麼使用它。加上專案參與的人越多，是時候研究一下 Ansys RSM(remote solve managment) 的機制了。
<!-- more-->
## Remote Solve Management

### 如何使用 Python 執行 Fluent 的專案
請先安裝 [ansys-fluent-core](https://pypi.org/project/ansys-fluent-core/)


## 如何搭建 Slurm
[Slurm](https://slurm.schedmd.com/documentation.html)

# References
- [Starting Ansys Fluent](https://ansyshelp.ansys.com/public/account/secured?returnurl=//Views/Secured/corp/v251/en/flu_ug/flu_ug_startramp.html)
- [Starting the Fluent GPU Solver](https://ansyshelp.ansys.com/public/account/secured?returnurl=//Views/Secured/corp/v251/en/flu_ug/flu_ug_sec_gpu_solver_starting.html%23flu_ug_sec_gpu_solver_comnd_line)
- [Commands for ARC Job Management](https://ansyshelp.ansys.com/public/account/secured?returnurl=//Views/Secured/corp/v251/en/wb_rsm/rsm_comm_basic.html)
- [Remote Solve Manager R2025 R2 Tutorials](https://ansyshelp.ansys.com/public//Views/Secured/corp/v252/en/pdf/Remote_Solve_Manager_Tutorials_2025_R2.pdf)
- [Ansys Remote Solve Manager User's Guide](https://ansyshelp.ansys.com/public/Views/Secured/corp/v252/en/pdf/Ansys_Remote_Solve_Manager_Users_Guide.pdf)
- [Submitting Solutions to Remote Solve Manager](https://ansyshelp.ansys.com/public/account/secured?returnurl=//Views/Secured/corp/v251/en/wb2_help/wb2h_submit_solutions_rsm.html)
- [Using RSM Job Submission Capabilities](https://ansyshelp.ansys.com/public/account/secured?returnurl=///Views/Secured/corp/v251/en/act_cust_wb/ch05s03s03.html)
- [Running Fluent Under Slurm](https://ansyshelp.ansys.com/public/account/secured?returnurl=//Views/Secured/corp/v252/en/flu_lm/slurm_sec_overview.html)
- [Submitting a Fluent Job from the Command Line](https://ansyshelp.ansys.com/public/account/secured?returnurl=/Views/Secured/corp/v242/en/flu_lm/flu_lsf_parallel_lsf_sumbit_fluent.html)
- [PyFluent cheat sheet](https://fluent.docs.pyansys.com/version/stable/_static/cheat_sheet.pdf)
- [Slurm 安裝流程](https://hackmd.io/@stargazerwang/S1t60i6lF)
- [Deploy slurm on ubuntu 18.04](https://medium.com/jacky-life/deploy-slurm-on-ubuntu-18-04-33abb6e902c4)