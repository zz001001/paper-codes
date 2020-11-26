# Abstract #
- Transformer 架构早已在自然语言处理任务中得到广泛应用，但在计算机视觉领域中仍然受到限制。在计算机视觉领域，注意力要么与卷积网络结合使用，要么用来代替卷积网络的某些组件，同时保持其整体架构不变。
- 该研究表明，对 CNN 的依赖不是必需的，当直接应用于图像块序列时，transformer 也能很好地执行图像分类任务。该研究基于大量数据进行模型预训练，并迁移至多个图像识别基准数据集（ImageNet、CIFAR-100、VTAB 等），结果表明 Vision Transformer（ViT）模型可以获得与当前最优卷积网络相媲美的结果，而其训练所需的计算资源大大减少。
# 1 Introduction #
- 受到 NLP 领域中 Transformer 缩放成功的启发，这项研究尝试将标准 Transformer 直接应用于图像，并尽可能减少修改。为此，该研究将图像分割成多个图像块（patch），并将这些图像块的线性嵌入序列作为 Transformer 的输入。然后用 NLP 领域中处理 token 的方式处理图像块，并以监督的方式训练图像分类模型。
- 在中等规模的数据集（如 ImageNet）上训练时，这样的模型产生的结果并不理想，准确率比同等大小的 ResNet 低几个百分点。这个看似令人沮丧的结果是可以预料的：Transformer 缺少一些 CNN 固有的归纳偏置，例如平移同变性和局部性，因此在数据量不足的情况下进行训练后，Transformer 不能很好地泛化
- 但是，如果在大型数据集（14M-300M 张图像）上训练模型，则情况大为不同。该研究发现大规模训练胜过归纳偏置。在足够大的数据规模上进行预训练并迁移到数据点较少的任务时，Transformer 可以获得出色的结果
- 该研究提出的 Vision Transformer 在 JFT-300M 数据集上进行预训练，在多个图像识别基准上接近或超过了 SOTA 水平，在 ImageNet 上达到了 88.36% 的准确率，在 ImageNet ReaL 上达到了 90.77% 的准确率，在 CIFAR-100 上达到了 94.55% 的准确率，在 VTAB 基准 19 个任务中达到了 77.16% 的准确率。
# 3 方法 # 
- 研究者尽可能地遵循原始 Transformer 的设计。这种故意为之的简单设置具有以下优势，即可扩展 NLP Transformer 架构和相应的高效实现几乎可以实现开箱即用。研究者想要证明，当进行适当地扩展时，该方法足以超越当前最优的卷积神经网络。
## 3.1 Vision Transformer（ViT）##
- 图 1 为模型架构图 我们将图像分割成固定大小的块，线性地嵌入每个块，添加位置嵌入，并将得到的向量序列输入标准的转换器编码器。为了进行分类，我们使用标准的方法在序列中添加额外可学习的“分类标记”。
- 标准 Transformer 接收 1D 序列的 token 嵌入为输入。为了处理 2D 图像，研究者将图像 x ∈ R^H×W×C 变形为一系列的扁平化 2D patch x_p ∈ R^N×(P^2 ·C)，其中 (H, W) 表示原始图像的分辨率，(P, P) 表示每个图像 patch 的分辨率。然后，N = HW/P^2 成为 Vision Transformer 的有效序列长度。Vision Transformer 在所有层使用相同的宽度，将每个flatten的patch 映射到可训练的线性投影的D维上（公式 1），相应的输出被称为 patch 嵌入
- 与 BERT 的 [class] token 类似，研究者在一系列嵌入 patch （z_0^0 = x_class）之前预先添加了一个可学习嵌入，它在 Transformer 编码器（z_0^L ）输出中的状态可以作为图像表示 y（公式 4）。在预训练和微调阶段，分类头（head）依附于 z_L^0
- 位置嵌入被添加到 patch 嵌入中以保留位置信息。研究者尝试了位置嵌入的不同 2D 感知变体，但与标准 1D 位置嵌入相比并没有显著的增益。所以，编码器以联合嵌入为输入。
- Transformer 编码器由多个交互层的多头自注意力（MSA）和 MLP 块组成（公式 2、3）。每个块之前应用 Layernorm（LN），而残差连接在每个块之后应用。MLP 包含两个呈现 GELU 非线性的层。
- （1）每个flatten的patch 映射到可训练的线性投影的D维
- （2）MSA
- （3）MLP
- （4）Transformer 编码器（z_0^L ）输出中的状态可以作为图像表示 y
- 归纳偏置。我们注意到，Vision Transformer比CNN具有更少的图像特异性归纳偏置。在CNNs中，局部性、二维邻域结构和平移同变性被加入到整个模型的每一层。在ViT中，只有MLP层是局部的和平移同变的，而自我注意层是全局的。二维邻域结构的使用非常少：在模型开始时，通过将图像分割成小块，并在微调时间调整不同分辨率图像的位置嵌入（如下所述）。除此之外，初始化时的位置嵌入不包含关于patchs的2D位置的信息，必须从头开始学习patchs之间的所有空间关系
- 混合结构。作为原始图像patchs的替代方案，输入序列可以由CNN的特征图形成（LeCun等人，1989）。在这个混合模型中，patchs嵌入投影E（式1）被应用于从CNN特征映射中提取的patchs。作为一种特殊情况，patchs的空间尺寸可以是1x1，这意味着输入序列是通过简单地将特征映射的空间维度展平并投影到Transformer dimension来获得的。如上所述添加分类输入嵌入和位置嵌入。
## 3.2 微调和更高分辨率 ##
- 研究者在大型数据集上预训练 ViT 模型，并针对更小规模的下游任务对模型进行微调。为此，研究者移除了预训练预测头，并添加了一个零初始化的 D × K 前馈层，其中 K 表示下游类的数量。与预训练相比，在更高分辨率时进行微调通常更有益处。当馈入更高分辨率的图像时，研究者保持 patch 大小不变，从而得到更大的有效序列长度。ViT 模型可以处理任意序列长度（取决于内存约束），但预训练位置嵌入或许不再具有意义。所以，研究者根据预训练位置嵌入在原始图像中的位置，对它们进行 2D 插值操作。需要注意的是，只有在分辨率调整和 patch 提取中，才能将 2D 图像的归纳偏置手动注入到 ViT 模型中。
# 4 实验 #
- 评估了ResNet、Vision Transformer（ViT）和hybrid的表现学习能力。
## 4.1 SETUP ## 
- Datasets. ILSVRC-2012 ImageNet数据集，其中包含1k个类和1.3M图像（我们在下文中将其称为ImageNet），其超集ImageNet-21k具有21k个类和14M个图像（Deng等人，2009年），以及JFT（Sun等人，2017年），其18k类和303M高分辨率图像。
- 模型变体 “base”和“large”模型直接取自BERT，并添加了更大的“huge”模型。
## 4.2 COMPARISON TO STATE OF THE ART ##
- 图 2 将 VTAB 任务分解为多个组，并对比了 ViT 与 SOTA 方法的性能，这些方法包括 BiT、VIVI 和 S4L。在 Natural 任务中，ViT-H/14 的性能略低于 BiT-R152x4；在 Specialized 任务中，ViT 的性能超过 BiT 等方法；而在 Structured 任务中，ViT 显著优于其他方法。
- 研究者首先将最大的 ViT 模型（在 JFT-300M 数据集上预训练的 ViT-H/14 和 ViT-L/16）与 SOTA CNN 模型进行对比，结果参见下表 2。
- 从上表中可以看出，规模较小的 ViT-L/16 模型在所有数据集上的性能堪比或者超过 BiT-L，同时它需要的算力也少得多。较大的 ViTH-14 模型进一步提升了性能，尤其在更具挑战性的数据集上，如 ImageNet、CIFAR-100 和 VTAB。ViTH-14 模型在所有数据集上的性能匹配或超过 SOTA，甚至在某些情况下大幅超过 SOTA 模型（如在 CIFAR-100 数据集上的性能高出 1%）。在 ImageNet 数据集上，ViT 模型的性能比 Noisy Student 低了大约 0.1%，不过在具备更干净 ReaL 标签的 ImageNet 数据集上，ViT 的性能超过 SOTA 模型。
## 4.3 预训练数据要求 ## 
- Vision Transformer 在大型 JFT-300M 数据集上进行预训练后表现出了优秀的性能。在 ViT 的归纳偏置少于 ResNet 的情况下，数据集规模的重要性几何呢？该研究进行了一些实验。
- 首先，在规模逐渐增加的数据集（ImageNet、ImageNet-21k 和 JFT300M）上预训练 ViT 模型。下图 3 展示了模型在 ImageNet 数据集上的性能：
- 图 5 展示了模型在不同预训练计算成本情况下的迁移性能：
- 实验结果表明
Vision Transformer 在性能 / 算力权衡中显著优于 ResNet。
混合模型在较小计算成本的情况下略优于 ViT，但在计算成本较高时，这一现象消失。该结果令人吃惊。
Vision Transformer 在实验尝试的算力范围内似乎并未饱和，未来可以进行更多可扩展性研究。
## 4.5 ViT 如何处理图像数据？## 
- 为了了解 ViT 处理图像数据的过程，研究者分析了其内部表示。
- ViT 的第一层将扁平化后的图像块线性投影至低维空间（公式 1），下图（左）展示了学得嵌入滤波器的主要组件。投影后，将学得的位置嵌入添加至图像块表示。下图（中）展示了模型学习编码图像内的距离，表明距离越近的图像块更有可能具备更相似的位置嵌入。自注意力允许 ViT 集成整个图像的信息，即使最低层也不例外。研究者调查了 ViT 网络利用这一能力的程度。具体而言，该研究计算图像空间中的平均距离（基于注意力权重）参见下图右。「注意力距离」类似于 CNN 中的感受野大小。