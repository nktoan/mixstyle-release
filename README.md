# MixStyle

This repo contains the code of our ICLR'21 paper, "Domain Generalization with MixStyle".

The OpenReview link is https://openreview.net/forum?id=6xHJ37MVxxp.

**########## Updates ############**

*12-04-2021*: A variable `self._activated` is added to MixStyle to better control the computational flow. To deactivate MixStyle without modifying the model code, one can do
```python
def deactivate_mixstyle(m):
    if type(m) == MixStyle:
        m.set_activation_status(False)

model.apply(deactivate_mixstyle)
```
Similarly, to activate MixStyle, one can do
```python
def activate_mixstyle(m):
    if type(m) == MixStyle:
        m.set_activation_status(True)

model.apply(activate_mixstyle)
```
Note that `MixStyle` has been included in [Dassl.pytorch](https://github.com/KaiyangZhou/Dassl.pytorch). See [the code](https://github.com/KaiyangZhou/Dassl.pytorch/blob/master/dassl/modeling/backbone/resnet.py#L280) for details.

*05-03-2021*: You might also be interested in our recently released survey on domain generalization at https://arxiv.org/abs/2103.02503, which summarizes the ten-year development in domain generalization, with coverage on the history, datasets, related problems, methodologies, potential directions, and so on.

**##############################**

**A brief introduction**: The key idea of MixStyle is to probablistically mix instance-level feature statistics of training samples across source domains. This idea is mainly inspired by neural style transfer where the feature statistics of a content image are replaced by those of a style image for image style transfer. MixStyle implicitly increases the diversity of source domains and thus makes CNNs more robust to unseen domains with domain shift.

MixStyle is very easy to implement. Below we show the PyTorch code of MixStyle.

```python
import random
import torch
import torch.nn as nn


class MixStyle(nn.Module):
    """MixStyle.

    Reference:
      Zhou et al. Domain Generalization with MixStyle. ICLR 2021.
    """

    def __init__(self, p=0.5, alpha=0.3, eps=1e-6):
        """
        Args:
          p (float): probability of using MixStyle.
          alpha (float): parameter of the Beta distribution.
          eps (float): scaling parameter to avoid numerical issues.
        """
        super().__init__()
        self.p = p
        self.beta = torch.distributions.Beta(alpha, alpha)
        self.eps = eps
        self.alpha = alpha

        print('* MixStyle params')
        print(f'- p: {p}')
        print(f'- alpha: {alpha}')

    def __repr__(self):
        return f'MixStyle(p={self.p}, alpha={self.alpha}, eps={self.eps})'

    def forward(self, x):
        if not self.training:
            return x

        if random.random() > self.p:
            return x

        B = x.size(0)

        mu = x.mean(dim=[2, 3], keepdim=True)
        var = x.var(dim=[2, 3], keepdim=True)
        sig = (var + self.eps).sqrt()
        mu, sig = mu.detach(), sig.detach()
        x_normed = (x - mu) / sig

        lmda = self.beta.sample((B, 1, 1, 1))
        lmda = lmda.to(x.device)

        perm = torch.randperm(B)

        mu2, sig2 = mu[perm], sig[perm]
        mu_mix = mu * lmda + mu2 * (1 - lmda)
        sig_mix = sig * lmda + sig2 * (1 - lmda)

        return x_normed * sig_mix + mu_mix
```

How to apply MixStyle to your CNN models? Say you are using ResNet as the CNN architecture, and want to apply MixStyle after the 1st and 2nd residual blocks, you can first initialize the MixStyle module using
```python
self.mixstyle = MixStyle(p=0.5, alpha=0.1)
```
and then apply MixStyle in the forward pass like
```python
def forward(self, x):
    x = self.conv1(x) # 1st convolution layer
    x = self.res1(x) # 1st residual block
    x = self.mixstyle(x)
    x = self.res2(x) # 2nd residual block
    x = self.mixstyle(x)
    x = self.res3(x) # 3rd residual block
    x = self.res4(x) # 4th residual block
    ...
```

In our paper, we have demonstrated the effectiveness of MixStyle on three tasks: image classification, person re-identification and reinforcement learning. The code for reproducing the experiments on these three tasks can be found in `mixstyle-release/imcls`, `mixstyle-release/reid`, and `mixstyle-release/rl`, respectively.

To cite MixStyle in your publications, please use the following bibtex entry

```
@inproceedings{zhou2021mixstyle,
  title={Domain Generalization with MixStyle},
  author={Zhou, Kaiyang and Yang, Yongxin and Qiao, Yu and Xiang, Tao},
  booktitle={ICLR},
  year={2021}
}
```
