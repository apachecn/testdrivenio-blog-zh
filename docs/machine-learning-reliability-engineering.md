# 机器学习可靠性工程导论

> 原文：<https://testdriven.io/blog/machine-learning-reliability-engineering/>

机器学习可靠性工程(MLRE)是一个即将到来的网站可靠性工程(SRE)的专业化。

在本文中，我将向您介绍为什么在 SRE 需要专业化，以及其他一些已经存在的专业化。我还将讨论 MLRE 的角色和职责，并简要介绍不同的工程职能部门将如何与这一新角色互动。

> 在整篇文章中，我将使用 MLRE 来指代机器学习可靠性*工程*领域，以及机器学习可靠性*工程师*。

## 背景

15 年前，谷歌通过将软件工程的 T2 原则应用于 DevOps，首次提出了 SRE 的想法。从那时起，这个新领域已经形成了自己的形态，与 DevOps 共存。虽然 DevOps 已经分支到几个专业化领域，如 DataOps、DevSecOps 和 MLOps，但 SRE 领域尚未完全大规模分支到专业化领域。

随着时间的推移，随着数据科学、机器学习、安全工程和人工智能等其他专业领域的成熟，专业基础设施、工具和流程也将存在——这将导致 SRE 领域内的专业化。目前，SRE 在过去几年里蓬勃发展的唯一分支是数据库可靠性工程。这是因为数据库、数据仓库、数据湖、ETL 和其他相关技术已经大规模应用了几十年。

正如数据库可靠性工程师需要深入了解数据库的高可用性、复制拓扑、数据库迁移等。，MLRE 将需要拥有与机器学习相关的特定领域的知识。例如，针对[GPU](https://blogs.nvidia.com/blog/2009/12/16/whats-the-difference-between-a-cpu-and-a-gpu/)和 [TPUs](https://cloud.google.com/tpu/docs/tpus) 的监控和警报。MLRE 的作用和责任基于与 SRE 相同的理念。

现在，让我们来看看这些年来 SRE 出现的不同分支:

| 作用 | 涉及 | 构思 | 参与的团队 |
| --- | --- | --- | --- |
| **SRE** | 应用程序、整体基础架构 | 2010 | 工程开发 |
| **DBRE** | 数据库、湖泊和仓库 | 2015 | 数据操作，数据工程 |
| 小姐 | 机器学习 | 2020 | 机器学习工程 |

根据设计，可靠性工程需要多面手。可靠性工程师需要对整个系统有一个清晰的认识，并且能够在需要时理解和处理不同的组件。除了核心职责之外，MLREs 还与其他工程职能部门分担职责。

**核心职责**

*   确保机器学习基础设施高度可用、可靠，并符合服务水平协议(SLA)。
*   设置系统以主动监控计算、内存、网络延迟等。
*   通过优化设计和工作流程控制机器学习基础设施的成本。

**共同责任**:

*   机器学习工程师-通过减少特征漂移、偏差、欺诈等，确保模型尽可能准确。
*   与其他工程职能部门——就更大的目标达成一致，并确保机器学习团队所做工作的输出是有用的，并且与业务目标相关。

现在，让我们谈谈 MLRE 背后的原则，这些原则是上述角色和职责的基础。

## 原则

作为 SRE 的一个分支，MLRE 也遵循同样的原则，如-

1.  通过自动化减少辛劳
2.  遵循[服务水平目标](https://www.datadoghq.com/blog/establishing-service-level-objectives/) (SLOs)
3.  将成本控制在预算之内
4.  确保顺利发布
5.  朝着[不变的基础设施](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure)努力
6.  拥有运行机器学习项目、产品、实验等的完整基础设施的可靠性

在前面的列表中已经很好地确定了 MLRE 人本质上是基础设施的所有者。除此之外，他们还负责[控制成本](https://www.vmware.com/topics/glossary/content/cloud-cost-management)，在基础设施可能超出预算时发出警告，等等。所有这些都需要[对 MLRE 必须应对的底层基础设施](https://algorithmia.com/blog/best-practices-in-machine-learning-infrastructure)有深刻的理解。

随着每一个主要的云平台引入机器学习功能，随着[机器学习和人工智能专用硬件](https://timdettmers.com/2018/12/16/deep-learning-hardware-guide/)的出现，深入理解这一切都需要专门的努力。随着云平台引入越来越多的服务，知识缺口会越来越大。在数据工程领域，这已经催生了专门针对云的职位，如 GCP 数据工程师、Azure 数据工程师和 AWS 数据工程师。类似的进化在机器学习和可靠性工程中也是完全可能的。

另一个伟大的想法是让一切重复和自动化。从长远来看，这节省了大量的时间和挫折。谷歌 SRE 卡拉·盖瑟(Carla Geisser )说，“如果一个人类操作员在正常操作中需要接触你的系统，你就有一个漏洞。随着系统的增长，正常变化的定义。”

## 特定知识

如前一节所述，MLRE 需要很好地理解基础设施。他们还需要理解使机器学习工作成为可能的操作过程。虽然机器学习工程师必须具备数据收集、数据验证、特征工程、元数据管理、模型分析等方面的深入知识，但 MLOps 工程师需要了解 DevOps 方面的内容，包括身份管理、角色、授权、许可、源代码控制和 CI/CD 管道。

> MLOps 将 DevOps 的最佳实践(协作、版本控制、自动化测试、合规性、安全性和 CI/CD)应用于生产机器学习。

尽管 MLOps 工程师负责上述所有事情，但他们通常不能确保底层基础设施正常工作。这就是 MLRE 的用武之地。这就给我们带来了 MLRE 需要知道的事情，以做好他们的工作。

## 技能组合

所有新的和即将出现的领域在很大程度上都来源于现有的领域。我已经讲过 SRE 是如何从德文衍生而来，DBRE 和 MLRE 是如何从 SRE 衍生而来的。在这个过程中还有很多其他的影响。由于这种复杂的血统，在这些不同领域工作所需的技能有很大程度的重叠。所以，尽管我提到 MLRE 是一个专业，但这并不是从技能的角度。即使是看似专业的工作，也需要广泛的技能。[罗伯特·A·海因莱因](https://en.wikipedia.org/wiki/Competent_man)，一个现代文艺复兴时期的人，写下了拥有广泛生活技能的需要:

> 一个人应该能够换尿布，策划入侵，杀猪，造船，设计建筑，写十四行诗，结算账目，建墙，接骨，安慰垂死的人，接受命令，发号施令，合作，独自行动，解方程，分析新问题，扔粪肥，给计算机编程，做一顿美味的饭菜，高效地战斗，英勇地死去。特殊化是为了昆虫。

罗伯特·海因莱茵的想法适用于计算机工程、数据科学、机器学习和人工智能。这就给我们带来了 MLRE 高效完成工作并为团队贡献价值所需的各种技能。让我们开始吃吧。

### 机器学习

MLRE 应该对机器学习的基础和目的有很好的理解。他们必须知道组成机器学习系统的组件，尤其是最关键和最昂贵的组件。

例如，它有助于理解数据收集、争论和初始处理比模型训练和参数调整等更多计算和内存密集型步骤花费更少。很多机器学习项目都是实验性的，可能非常昂贵。只有理解了底层流程，才能理解成本。

### 脚本和编程

对基于 Unix 的系统有一个很好的理解总是很有用的，并且是这项工作所必需的。这要求 MLRE 人能够轻松地编写 shell 脚本。他们还需要熟悉软件工程。如前所述，SRE 背后的整个想法是在未知投入生产之前应用软件工程的原则来充实它们。MLRE 必须能够编写代码，以便建立管道，配置机器学习堆栈，启动和拆除基础设施，等等。

### DevOps 和 MLOps

这就把我们带到了 MLRE 最关键的领域之一:[开发商运营](https://cloud.google.com/solutions/machine-learning/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning)。该领域要求人们了解如何使用 CI/CD 工具构建机器学习管道，同时还要考虑机器学习中从数据收集到生产化预测模型的所有步骤，即模型验证、特征存储、元数据管理和源代码控制管理等。正如前面提到的，虽然不需要对每个步骤都有深入的了解，但理解机器学习管道中的数据流以及底层基础设施是至关重要的。

### 基础设施即代码

加快基础设施建设不一定是 MLRE 的工作，但它确实属于德沃普斯和 SRE 的职责范围。不严格地说，提供对资源的访问是 MLOps 工程师的工作，而为机器学习工程师提供启动基础设施的有效方法是 MLRE 的工作。出于这个原因，他们需要知道如何使用像 [Terraform](https://blog.scottlogic.com/2018/10/08/infrastructure-as-code-getting-started-with-terraform.html) 、 [Pulumi](https://www.pulumi.com/docs/intro/vs/terraform/) 或 [AWS CloudFormation](https://aws.amazon.com/cloudformation/) 这样的工具来编写基础设施代码。这样的工具使 MLREs 能够编写可重用的插件、模板和模块，供机器学习工程师使用。这触及了 SRE 诞生背后的核心思想。

### 数据工程

从某种意义上说，数据科学、机器学习和人工智能都源于数据工程，并严重依赖于数据工程。每当数据从一个地方移动到另一个地方时，它都必须被处理、清理和转换。这就是数据工程思想发挥作用的地方。由于许多角色与许多其他职能有大量重叠，责任界限变得模糊不清。因此，MLRE 需要对数据工程系统如何工作有一个很好的了解。SQL 工作知识是必不可少的。有关系数据库、数据仓库和/或 ETL(提取、转换、加载)框架经验者优先。

### 自动化测试

测试机器学习模型与测试典型软件产品的方式截然不同。根本区别在于，机器学习模型是非确定性的。

在基本层面上，可以有两种类型的测试来测试机器学习模型:

1.  测试模型的**编码逻辑**
2.  测试模型的**输出/精度**

有很多方法可以测试后者。如前所述，您可以通过创建与训练数据集相对应的测试数据集来测试准确性，训练数据集测试训练前后的数据。另一方面，您并不总是有有效的甚至是正确的方法来测试模型的编码逻辑。这又回到了模型的非确定性。当你不知道会发生什么时，你如何测试代码？

> 关于测试机器学习模型的深入阅读，请阅读杰瑞米·乔登的博客文章，[机器学习系统的有效测试](https://www.jeremyjordan.me/testing-ml/)。

测试宜早不宜迟。这种向左移动的基本原则已经流行了一段时间。这个想法是为了确保在开发周期中尽可能早地以自动化的方式完成集成测试。这减少了你以后必须处理的未知数的数量。因此，左移还将帮助您避免以后处理技术债务，因为您将尽早进行设计和基础结构更改。

## 合作

关于各种工程功能之间的共享技能集的对话是讨论这些不同的工程功能如何协作的一个很好的方式。所有不同的团队都需要某种程度的持续协作。除了技能上的重叠，当不同的公司对一个特定的角色如何运作有不同的想法时，另一个层次的复杂性就出现了。这通常是公司与公司之间(甚至内部团队与团队之间)协作过程不同的原因。下面是我们讨论过的各种角色职责的鸟瞰图:

| 作用 | 责任 |
| --- | --- |
| 开发工程师 | 身份管理，资源访问，开发人员需要的任何东西。 |
| **物流工程师** | 专门针对机器学习工程师的 DevOps，构建 CI/CD 管道。 |
| **SRE** | 基础设施、应用、服务、数据库等的可靠性。 |
| **DBRE** | 数据库、数据仓库和湖泊基础设施、服务等的可靠性。 |
| 小姐 | 机器学习相关基础设施、应用和服务的可靠性。 |
| **数据工程师** | 需要数据的所有其他团队的数据可用性。 |

一个团队的工作成果通常是另一个团队的工作成果。在一个理想的场景中，您将自动完成大部分工作。

## 结论

我们已经讨论了可靠性工程诞生背后的一些原因，以及它是如何与不同的工程功能融合在一起的。

机器学习可靠性工程是 SRE 的一个即将到来的分支，我们很快就会看到那些希望建立基于机器学习和人工智能的可扩展解决方案的公司。可靠性团队将专注于保持系统正常运行，而机器学习工程师将专注于改进模型和做出更好的预测。