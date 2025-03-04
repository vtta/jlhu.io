+++
+++
- [ ] ~~[Harry Xu](https://web.cs.ucla.edu/~harryxu/)~~
- [ ] [Tianyin Xu](https://tianyin.github.io/)
- [ ] [Huaicheng Li](https://huaicheng.github.io/#research)

---

## Huaicheng Li

### [ASPLOS25]Melody



### [ASPLOS23]Pond

### Draft

```
Dear Professor Li,

My name is Junliang Hu, and I am a fourth-year PhD student in the Department of Computer Science and Engineering at The Chinese University of Hong Kong, advised by Professor Ming-Chang Yang. I am writing to inquire about the possibility of joining your research group as a visiting student with co-advisorship for up to one year, starting in the spring or summer term of this year. My visit would be fully funded through my department, Prof. Yang's support, and a CUHK overseas research grant for which I am currently applying.

My research focuses on optimizing modern virtualized cloud environments through the integration of heterogeneous memory technologies, including CXL memory, persistent memory, and RDMA and addresses critical challenges in performance, capacity, and cost-effectiveness. A core motivation driving my work is advancing cloud infrastructure to enable tangible societal impact—a vision that deeply resonates with your interests in innovating system support for modern hardware.

I have published two papers at SOSP/OSDI and submitted another manuscript in December. My recent work introduces tiered memory support to hypervisors, which I plan to extend to pooled datacenter environments by developing a tiered pooling architecture that combines CXL memory and transitional legacy RDMA for serverless systems. My systems research journey has evolved from implementing experimental operating systems and key-value stores during my undergraduate years to my current work optimizing the Linux kernel and modern Rust-based hypervisors. I believe that my technical background and research interests align well with your group’s efforts.

Your pioneering research has particularly inspired me, especially Pond, which is the first to introduce CXL-based pooling system design for real world datacenter to the academia, and Melody, which provided in-depth details across different CXL setups. Your seminar at CUHK two years ago left a lasting impression and further motivated me to explore this collaboration opportunity.

During my visit, I aim to complete my proposed project and develop it into a publication for a top-tier systems conference. I would also be excited to contribute to your ongoing projects on modern and emerging hardware.

I would greatly appreciate the chance to discuss this opportunity further. I have attached my CV, and additional information about my work is available at https://jlhu.io. Thank you for your time and consideration.

Best regards,
Junliang Hu 胡俊良
PhD Candidate, Department of Computer Science and Engineering
The Chinese University of Hong Kong
Email: jlhu@cse.cuhk.edu.hk
Homepage: jlhu.io <https://jlhu.io/>
```



---

## Harry

> Systems for Resource-Disaggregated Datacenters ([OSDI'20](https://web.cs.ucla.edu/~harryxu/papers/wang-osdi20.pdf), [PLDI'22-a](https://web.cs.ucla.edu/~harryxu/papers/mako-pldi22.pdf), [OSDI'22-best-paper](https://web.cs.ucla.edu/~harryxu/papers/memliner-osdi22.pdf), [NSDI'23-a](https://web.cs.ucla.edu/~harryxu/papers/canvas-nsdi23.pdf), [NSDI'23-d](https://web.cs.ucla.edu/~harryxu/papers/hermit-nsdi23.pdf), [NSDI'24-a](https://web.cs.ucla.edu/~harryxu/papers/midas-nsdi24.pdf), [OSDI'24-a](https://web.cs.ucla.edu/~harryxu/papers/drust-osdi24.pdf), [OSDI'24 -b](https://web.cs.ucla.edu/~harryxu/papers/atlas-osdi24.pdf))

### [OSDI20]Semeru

Add DM support for JVM through 1) a unified memory abstraction; 2) distributed GC; and 3) kernel swapping support.

### [PLDI22]Mako

A GC for DM that offload tracing and evacuation to memory servers allowing concurrent execution with mutator threads on compute servers.

### [OSDI22BP]MemLiner

Futher GC for DM optimizations that 1) reduces local working set and 2) improves prefetching; through "lining up" memory accesses.

### [NSDI23]Canvas

A swap system for DM that 1) isolates swap path for remote memory applications;  2) has finer prefetching; and 3) has better RDMA scheduling.

### [NSDI23]Hermit

A swap system for DM that 1) has adaptive, feedback-directed asynchrony

### [NSDI24]Midas

Address the concept of "soft state/memory", which is non-essential for correctness.

Improve kernel's idle memory handling and allow it to be used for storing soft state

### [OSDI24]DRust

DSM that make use of rust's ownership to constrains read/write order and improve coherence performance.

### [OSDI24]Atlas

Make runtime-level object-based DM perform better than kernel-level page-based DM.

> System Design for Emerging (Datacenter) Hardware ([PLDI'19](https://web.cs.ucla.edu/~harryxu/papers/wang-pldi19.pdf), [PLDI'20](https://web.cs.ucla.edu/~harryxu/papers/genc-pldi20.pdf), [ASPLOS'21](https://web.cs.ucla.edu/~harryxu/papers/jaaru-asplos21.pdf), [PLDI'21-a](https://web.cs.ucla.edu/~harryxu/papers/jportal-pldi21.pdf), [ASPLOS'22-a](https://web.cs.ucla.edu/~harryxu/papers/yashme-asplos22.pdf), [ASPLOS'22-b](https://web.cs.ucla.edu/~harryxu/papers/heterogen-asplos22.pdf), [PLDI'22-b](https://web.cs.ucla.edu/~harryxu/papers/psan-pldi22.pdf), [OSDI'22-b](https://web.cs.ucla.edu/~harryxu/papers/upgradvisor-osdi22.pdf))

### [PLDI19]Panthera

Hybrid memory systems in the language runtime that manages placement and migration.

### [PLDI20]Crafty

Ensure consistency and atomicity on PMEM with HTM

### [ASPLOS21]Jaaru

Crash consistency model checker for PMEM

### [PLDI12]JPortal

Support Intel PT for high level languages through a JVM profiling tools

### [ASPLOS22]Yashme

A dector for persistency race for PMEM

### [ASPLOS22]HeteroGen

An EDA tool that optimize the HLS compilation time from C code

### [PLDI22]PSan

Persistency bug decector tool for PMEM

### [OSDI22]Upgradvisor

better static analsys and dynamic tracing for detecting backwards compatible software updates.

### 套瓷信

修改版:

```
Subject: Inquiry about Visiting Student Opportunity in Building Future Cloud Infrastructure – Junliang Hu @ CUHK (jlhu.io)

To: harryxu@cs.ucla.edu
From: jlhu@cse.cuhk.edu.hk

Dear Professor Xu,

I hope you had a wonderful Lunar/Chinese New Year. My name is Junliang Hu, and I am a fourth-year PhD student in the Department of Computer Science and Engineering at The Chinese University of Hong Kong. I am writing to inquire about the possibility of joining your research group as a visiting student for up to one year, starting in the spring or summer term of this year. My visit would be supported by my department and supervisor, and I am also in the process of applying for my university’s overseas research grant.

My research focuses on enhancing modern virtualized cloud environments through the integration of heterogeneous memory technologies, including persistent memory, CXL.mem, and RDMA, to address challenges related to performance, capacity, and cost. Since the beginning of my PhD journey, I have been driven by the goal of conducting research that makes a tangible impact on society by advancing the cloud infrastructures behind virtually every online interaction.

To date, I have published two papers at SOSP/OSDI, with another manuscript submitted last December. My latest work involved introducing tiered memory support to hypervisors, and I plan to extend this approach to disaggregated datacenters by incorporating tiered memory into serverless systems. My passion for systems research has grown over the years, from experimenting with toy operating systems and key–value stores during my undergraduate studies to navigating millions of lines of code, and debugging and enhancing the Linux kernel and modern Rust-based hypervisors in my recent work. I believe that my background aligns well with your group’s efforts in advancing infrastructural systems.

I have been impressed by your group’s contributions to infrastructural software, including your work on language runtimes, addressing performance, scheduling, consistency, and correctness, since your early academic and industry days. Your recent projects on kernel-level support for disaggregated memory and the integration of emerging heterogeneous memory technologies, such as persistent memory, into cloud environments are especially inspiring.

During my visit, I aim to complete my current project and consolidate the results into a publication for a top-tier systems conference. I would also be excited to contribute to your ongoing projects on disaggregated datacenters and emerging hardware.

I would be very grateful for the opportunity to discuss this possibility with you further. I have attached my CV for your reference, and my homepage is available at https://jlhu.io. Thank you very much for your time and consideration. I look forward to your response.

Best regards,

Junliang Hu
Department of Computer Science and Engineering
The Chinese University of Hong Kong
jlhu@cse.cuhk.edu.hk
```

Original:

```
Subject: Inquiry about Visiting Student Opprotunity in Building Furture Cloud Infra - Junliang Hu @ CUHK jlhu.io
To: harryxu@cs.ucla.edu
From: jlhu@cse.cuhk.edu.hk

Hi Prof. Xu,

Happy a belated lunar/chinese new year! I am Junliang Hu, a 4th year PhD student in computer science and engineering department at The Chinese University of Hong Kong. I am writing to inquire about the possibility of joining your research group as a visiting student up to one year starting from spring/summer this year. I will be supported by my department and my supervisor and I'm also applying for university's overseas research grant.

My research focus is on empowering the morden virtualized cloud environment with heterogeneous memory techonogies, including persistent memory, CXL.mem, and RDMA to address performance, capacity, and cost concerns.
My goal since day one of my PhD journey is to do the research that will finally have an impact to the great public through the modern cloud infrastructure that is behind essentially every click.

I have 2 SOSP/OSDI publications with maybe one more, which is just submitted last december.
My latest work was about introducing tiered memory support to hypervisors and my  plan for the next step is to move a step up and try to introduce tierd memory for disaggrageted datacenters through modifications of serverless systems.
I have always been a system lover, apart from playing with toy OSes and KV stores in the undergraduate days, I have now gained experience navigating millions lines of code, hacking and debugging linux kernel, as well as modern rust-based hypervisors through my last work.
That experience gives me great confidence to further my research on infrastuructural systems which aligns well with, and will benefit greatly from your team's effort in recent years.

I have noticed that you and your group have a great history on infrastructural software, including a series of works on all aspects of language runtime, including performance, scheduling, consistency and correctness, which dates back to your early PhD and industry days, and your recent works on kernel level support for disaggregated memory, as well as integrating other heterogeneous memory techonogies to the cloud including persistent memory.

During the visiting period, I aim to finish my next work and fomulate the results as one publication to the top system conference. I am also happy to participate in your on-going projects on disaggregated datacenters and/or emerging hardware.

I would be very grateful for the opportunity to discuss this possibility with you. I have attached my CV as the attachment for your reference. My home page is located at https://jlhu.io. Thank you for your time, thank you for considering this request, and I look forward to your response.

Best regards,
Junliang Hu
```

---

> 撰写访问学生（visiting student）套瓷信时，核心在于清晰展示你的学术背景、研究兴趣与目标教授/项目的契合度，同时体现专业性和诚意。以下是具体建议和模板参考：
>
> ------
>
> ### **1. 邮件标题：简洁明确**
>
> 标题需直接说明目的，例如：
> **"Inquiry about Visiting Student Opportunity in [研究方向/实验室名称]"**
> 或
> **"Request for Visiting Student Position in [具体领域]"**
>
> ------
>
> ### **2. 称呼与开头**
>
> - **称呼**：尽量确认对方姓名和职称（如Prof./Dr. XXX）。
> - **开头**：简短自我介绍（学校、专业、年级）并说明来意。
>   **示例**：
>
> > Dear Prof. [Last Name],
> > I am [Your Full Name], a [Master’s/PhD] student in [专业] at [学校名称]. I am writing to inquire about the possibility of joining your research group as a visiting student in [时间段, e.g., Fall 2024].
>
> ------
>
> ### **3. 正文内容（分段落，逻辑清晰）**
>
> #### **（1）学术背景与研究经历**
>
> - 简述你的研究领域、当前课题或论文方向。
> - 突出与目标教授相关的技能或成果（如论文、项目、技术方法）。
>   **示例**：
>
> > My current research focuses on [具体方向], particularly [细分方向]. I have developed expertise in [技术/方法], which I applied in my recent project [项目名称/成果]. This work resulted in [成果, e.g., a conference paper/experimental findings].
>
> #### **（2）对教授研究的兴趣**
>
> - **具体说明**：引用对方近期论文或项目，表明你深入了解过其工作。
> - **契合点**：说明你的背景如何补充其研究，或你想探索的具体问题。
>   **示例**：
>
> > I have closely followed your work on [具体论文/项目], especially your findings on [具体内容]. I am particularly interested in exploring [具体问题] using [方法], which aligns with your team’s recent efforts in [领域].
>
> #### **（3）访问目的与计划**
>
> - 明确访问目标（如学习技术、数据收集、合作发表）。
> - 提及你希望参与的具体项目或课题（如有）。
>   **示例**：
>
> > During the visiting period, I aim to [具体目标, e.g., investigate X phenomenon using Y method]. I believe this could contribute to your ongoing project on [项目名称], while also advancing my thesis work on [你的课题].
>
> #### **（4）资金与时间安排**
>
> - 说明是否自费或需对方资助（如需要资助，需委婉表达）。
> - 注明计划访问的时间段。
>   **示例**：
>
> > I am self-funded/currently applying for [奖学金名称] to support this visit, and I plan to stay for [时长] starting from [月份].
>
> ------
>
> ### **4. 结尾礼貌请求**
>
> - 表达进一步沟通的意愿，并附上材料（CV、成绩单等）。
>   **示例**：
>
> > I would be grateful for the opportunity to discuss this possibility with you. Please find my CV and a summary of my research attached. Thank you for your time, and I look forward to your response.
>
> ------
>
> ### **5. 模板参考**
>
> ```
> Subject: Inquiry about Visiting Student Opportunity in [研究方向]
> 
> Dear Prof. [Last Name],
> 
> I am [Your Name], a [PhD/Master’s] student in [专业] at [学校]. I am writing to express my interest in joining your group as a visiting student during [时间段], focusing on [研究方向].
> 
> My research centers on [你的研究方向], where I have developed skills in [技术/方法]. In my recent project [项目名称], I [简要成果]. I believe my experience in [具体技能] could support your team’s work on [教授的研究方向], particularly your publication on [论文标题] (年份), which inspired me to explore [具体问题].
> 
> During the visit, I aim to [目标, e.g., collaborate on X analysis or apply Y method to Z problem]. This aligns with my thesis goal of [你的研究目标], and I am confident it would benefit from your expertise in [领域].
> 
> I am currently [self-funded/seeking funding through X] and plan to visit for [时长]. Attached are my CV and a research summary. I would appreciate the chance to discuss this opportunity further.
> 
> Thank you for considering my request.
> 
> Best regards,  
> [Your Full Name]  
> [学校]  
> [联系方式]  
> [个人网页/学术链接（如有）]
> ```
>
> ------
>
> ### **注意事项**
>
> 1. **个性化**：每封信需针对教授研究方向调整内容，避免模板化。
> 2. **简洁**：控制在300字以内，重点突出。
> 3. **附件**：附上CV（PDF格式命名：CV_YourName）、成绩单或研究计划。
> 4. **跟进**：若2周无回复，可发一封简短提醒邮件。
> 5. **学术礼仪**：避免夸大或过度吹捧，保持专业语气。
>
> 通过清晰展示你的价值与匹配度，套瓷信的成功率会显著提升。

