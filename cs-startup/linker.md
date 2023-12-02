---
layout: default
title: linker
nav_order: 2
parent: cs-startup
has_children: true
---



# 论文阅读
- 1 link time optimization
    - 1 Glek, Taras, and Jan Hubicka. "Optimizing real world applications with GCC link time optimization." arXiv preprint arXiv:1010.2196 (2010).
    - 2 Teresa Johnson, Mehdi Amini, and Xinliang David Li. ThinLTO: Scalable and Incremental LTO. In 2017 IEEE/ACM International Symposium on Code Generation and Optimization (CGO), pages 111–121, 2017.
    - 3 Maksim Panchenko, Rafael Auler, Bill Nell, and Guilherme Ottoni. Bolt: A Practical Binary Optimizer for Data Centers and Beyond. In 2019 IEEE/ACM International Symposium on Code Generation and Optimization (CGO), pages 2–14. IEEE, 2019.
    - 4 Google. PROPELLER: Profile Guided Optimizing Large Scale LLVM based Relinker. https://github.com/google/llvm-propeller, 2019. [Online; accessed 8-August-2022].
- 2 link time optimization for code size
    - 1 Liu, Gai, Umar Farooq, Chengyan Zhao, Xia Liu, and Nian Sun. "Linker code size optimization for native mobile applications." In Proceedings of the 32nd ACM SIGPLAN International Conference on Compiler Construction, pp. 168-179. 2023.
    - 2 Lee, Kyungwoo, Ellis Hoag, and Nikolai Tillmann. "Efficient profile-guided size optimization for native mobile applications." In Proceedings of the 31st ACM SIGPLAN International Conference on Compiler Construction, pp. 243-253. 2022.
    - 3 Chabbi, Milind, Jin Lin, and Raj Barik. "An experience with code-size optimization for production iOS mobile applications." In 2021 IEEE/ACM International Symposium on Code Generation and Optimization (CGO), pp. 363-377. IEEE, 2021.
    - 4 Quach, Anh, Aravind Prakash, and Lok Yan. "Debloating software through {Piece-Wise} compilation and loading." In 27th USENIX security symposium (USENIX Security 18), pp. 869-886. 2018.
    - 5 Ioannis Agadakos, Di Jin, David Williams-King, Vasileios P. Kemerlis, and Georgios Portokalidis. 2019. Nibbler: debloating binary shared libraries. In Proceedings of the 35th Annual Computer Security Applications Conference (ACSAC '19).
    - 6 Zhang, Haotian, Mengfei Ren, Yu Lei, and Jiang Ming. "One size does not fit all: security hardening of mips embedded systems via static binary debloating for shared libraries." In Proceedings of the 27th ACM International Conference on Architectural Support for Programming Languages and Operating Systems, pp. 255-270. 2022.
- 3 ELF and security
    - 1 Di Federico, Alessandro, Amat Cama, Yan Shoshitaishvili, Christopher Kruegel, and Giovanni Vigna. "How the {ELF} Ruined Christmas." In 24th USENIX Security Symposium (USENIX Security 15), pp. 643-658. 2015.
    - 2 Wenzl, Matthias, Georg Merzdovnik, Johanna Ullrich, and Edgar Weippl. "From hack to elaborate technique—a survey on binary rewriting." ACM Computing Surveys (CSUR) 52, no. 3 (2019): 1-37.

# 参考资料
- csapp教材第7章
- 《程序员的自我修养：链接、装载与库》[下载](https://awesome-programming-books.github.io/others/%E7%A8%8B%E5%BA%8F%E5%91%98%E7%9A%84%E8%87%AA%E6%88%91%E4%BF%AE%E5%85%BB%EF%BC%9A%E9%93%BE%E6%8E%A5%E3%80%81%E8%A3%85%E8%BD%BD%E4%B8%8E%E5%BA%93.pdf)