+++
title = 'Classifier Guidance 和 Classifier-Free Guidance 的理解与代码实现'
date = 2024-09-01T10:56:31+08:00
categories = ["AIGC", "Diffusion model"]
tags = ["Classifier Guidance", "Classifier-Free Guidance", "CFG"] 
+++

首先给出结论：

|         | Classifier Guidance   |  Classifier-Free Guidance  |
| :--------  | :-----  | :----:  |
| 是否需要重训Diffusion model | 不需要，使用已训好的Diffusion model就可以使用 |需要，重训Diffusion model|
| 是否需要额外模型 | 需要，额外的针对加噪图像的分类模型 | 相当于不需要，有文生图的clip文本编码器就行 |
| 实现效果 | 可控制分类模型支持的类别数生成 | 任意条件即可控制 |


### Classifier Guidance

Classifier Guidance 只能是给定一个分类模型中存在的类别，让模型生成这个类别的东西。比如指定模型生成图像类别是“狗”，模型就生成一张狗的图。所以这种方式是条件生成，条件是 y，扩散过程中的生成图像是 $X_t$。

公式上，用贝叶斯定理将条件生成概率进行展开：
<div>
$$
\begin{aligned}
\nabla logp(x_t|y)
&= \nabla log(\frac{p(x_t)p(y|x_t)}{p(y)}) \\ 
&= \nabla logp(x_t) + \nabla logp(y|x_t) - \nabla logp(y) \\
&= \nabla logp(x_t) + \nabla logp(y|x_t) 
\end{aligned}
$$
</div>

其中第二个等号后面最后一项可以去掉，是因为当我们要求模型生成特定类别的图像时，扩散过程的条件 y 都不会发生改变，对应的梯度也就0可以去掉。

第三个等号的这两项中，第一项是扩散模型本身的梯度引导，新增的只有第二项，也就是说 Classifier Guidance 条件生成只需要额外添加一个 classifier 的梯度来引导对应类别图像的生成。

同时上式可进一步改写为噪声预测的形势(为了控制分类器引导的强弱，可以加入 $w$ 来控制)：
<div>
$$
\begin{aligned}
\nabla logp(x_t) + \nabla logp(y|x_t) 
&\approx -\frac{1}{\sqrt{1-\bar{\alpha}_t}} \epsilon_\theta(\mathbf{x}_t, t) + \nabla_{x_t} \log f_\phi(y|\mathbf{x}_t) \\
&= -\frac{1}{\sqrt{1-\bar{\alpha}_t}}  (\epsilon_\theta(\mathbf{x}_t, t) - \sqrt{1-\bar{\alpha}_t} w \nabla_{x_t} \log f_\phi(y|\mathbf{x}_t))
\end{aligned}
$$
</div>

通过上述公式，我们可以发现，我们可以通过classifier 的**梯度**作用在无条件引导的Diffusion模型预测的噪声上，来实现条件图像的生成。

伪代码实现：

```python
classifier_model = ...  # 加载图像分类模型
y = 1  # 生成类别为 1 的图像，假设类别 1 对应“狗”这个类
guidance_scale = 7.5  # 控制类别引导的强弱，越大越强
input = get_noise(...)  # 从高斯分布随机取一个跟输出图像一样 shape 的噪声图

for t in tqdm(scheduler.timesteps):

    # 模型推理，预测噪声
    with torch.no_grad():
        noise_pred = unet(input, t).sample

    # 用 input 和预测出的 noise_pred 和 x_t 计算得到 x_t-1
    input = scheduler.step(noise_pred, t, input).prev_sample

    # classifier guidance 步骤
    # 把当前生成的图 input 和我们想要的类别 y 一起喂给分类模型，计算input和 y 的 loss 和 梯度
    class_guidance = classifier_model.get_class_guidance(input, y)
    # 把计算出来的梯度根据 guidance_scale 加到图像上
    input += class_guidance * guidance_scals
```

### Classifier-Free Guidance

Classifier Guidance 只能用训好的分类模型引导，生成有限的类别。如果分类模型是分 80 类，那么 Classifier Guidance 也只能引导 diffusion 模型生成固定的 80 类，多一类都不好使。但Classifier Free Guidance 这个方法的引导无需引入特定的分类模型，因此也没有类别数量的限制。

由Classifier Guidance上述的结果进一步推导，我们可以将其中的分类器引导对应项的梯度表示为：

<div>
$$
\begin{aligned}
\nabla_{x_t} \log p(y|x_t) &= \nabla_{x_t} \log p(x_t|y) - \nabla_{x_t} \log p(x_t) \\
&= -\frac{1}{\sqrt{1-\bar{\alpha}_t}} \left( \epsilon_\theta(x_t, t, y) - \epsilon_\theta(x_t, t) \right)
\end{aligned}
$$
</div>

代入到Classifier Guidance中推导得到的分类起引导对应的梯度项中我们可以得到：

<div>
$$
\begin{aligned}
\bar{\epsilon}_\theta(x_t, t, y) &= \epsilon_\theta(x_t, t, y) - \sqrt{1-\bar{\alpha}_t} w \nabla_{x_t} \log p(y|x_t) \\
&= \epsilon_\theta(x_t, t, y) + w \left( \epsilon_\theta(x_t, t, y) - \epsilon_\theta(x_t, t) \right) \\
&= (w+1)\epsilon_\theta(x_t, t, y) - w\epsilon_\theta(x_t, t)
\end{aligned}
$$
</div>

在得到的表达式中，我们可以发现其中的针对条件的引导，没有依赖于特定的分类器实现。

伪代码实现（以文生图为例）：

```python
clip_model = ...  # 加载一个官方的 clip 模型

text = "一只狗"  # 输入文本
text_embeddings = clip_model.text_encode(text)  # 编码条件文本
empty_embeddings = clip_model.text_encode("")  # 编码空文本
text_embeddings = torch.cat(empty_embeddings, text_embeddings)  # 把它俩 concate 到一起作为条件

input = get_noise(...)  # 从高斯分布随机取一个跟输出图像一样 shape 的噪声图

for t in tqdm(scheduler.timesteps):

    # 用 unet 推理，预测噪声
    with torch.no_grad():
        # 这里同时预测出了有文本的和空文本的图像噪声
        noise_pred = unet(input, t, encoder_hidden_states=text_embeddings).sample

    # Classifier-Free Guidance 引导
    noise_pred_uncond, noise_pred_text = noise_pred.chunk(2)  # 拆成无条件和有条件的噪声
    # 把【“无条件噪声”指向“有条件噪声”】看做一个向量，根据 guidance_scale 的值放大这个向量
    # （当 guidance_scale = 1 时，下面这个式子退化成 noise_pred_text）
    noise_pred = noise_pred_uncond + guidance_scale * (noise_pred_text - noise_pred_uncond)

    # 用预测出的 noise_pred 和 x_t 计算得到 x_t-1
    input = scheduler.step(noise_pred, t, input).prev_sample
```

其中，unet 预测的 noise_pred_uncond为无条件噪声，noise_pred_text为条件为“一只狗”对应的条件噪声。(noise_pred_text - noise_pred_uncond)我们可以理解为“无条件”到”一只狗“条件的向量，乘上 guidance_scale后通过调节 guidance_scale 的数值大小，我们即可控制文本条件噪声贴近文本语义的程度。





