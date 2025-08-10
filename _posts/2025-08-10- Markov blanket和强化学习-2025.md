---
layout:     post                       
title:     Markov blanket和强化学习
subtitle:   
date:       2025-08-10       
author:     Closure                         
header-img: img/0727.jpg
catalog: true                         
tags:                               
    - 笔记
    - 大模型
---

### 把马尔科夫毯看作RL的核心假设，会发生什么？

核心预设就是把马尔科夫毯看作时间因果图中“**把过去和未来隔绝开来**”的最小变量集合，谁在该毯内和谁在毯外直接决定了能否把决策问题压缩为仅依赖当前表征的形式，是否能在有限维的状态上写出Bellman递归？是否能用平稳Markov策略而不损失最优性?

概念回顾一下，马尔科夫毯是在因果图或贝叶斯网络里，给定一个节点，这里是当前状态变量，马尔科夫毯由能屏蔽该节点与其他所有节点依赖关系的最小变量集合构成，也就是一旦条件化在毯上这个节点与外界其他变量条件独立。在时间序列的上下文中，对于当前状态 St​ ，其马尔科夫毯 MB(St​) 使得未来 (St+1​,Rt+1​,...) 与过去 (St−1​,At−1​,...) 条件独立：

(St+k​,Rt+k​)k>0​⊥(St−k​,At−k​)k>0​∣MB(St​)

因果中介指的是那些在因果链中承担传递作用的变量或机制，把因果中介看作马尔科夫毯的内容意味着这些中介既承载了影响未来的全部信息，又是我们需要保留以实现正确决策的最小集。

### 对RL基本假设和数学结构的影响

传统表述经常把存在马尔科夫状态当作黑箱假设：存在某个 St​​ 能使未来与过去条件独立。用马尔科夫毯的视角问题变成可操作的结构问题，哪些变量组成那个屏蔽过去与未来的信息集。这使得马尔科夫性由模糊的假设变成可验证优化的目标，如果你能指出或学习出一个变量集合使得条件独立成立那么你就把原始高维或部分可观测问题降低为一个MDP。

影响要点有两个，一个是把状态为形式化的统计对象，状态是关于历史的充分统计量，即 St​=f(Ht​)，其中历史 Ht​=(O0​,A0​,...,Ot​)，而毯就是该充分统计量中的最小代表或包含它的集合。一个是研究问题从是否能做 RL变为如何构造能屏蔽历史的表示，从判定问题变为估计问题。

很多核心理论，像是Bellman最优性方程和算子的收缩性这些都依赖于下一个状态的分布只由当前状态与动作决定这一结构性条件，马尔科夫毯把这个只由当前状态决定的条件替换为只要我们条件化在毯上就成立。因此若表示包含毯，Bellman算子可以在表示空间上定义且保持收缩。例如，最优动作价值函数的 Bellman 方程：
Q∗(s,a)=Es′∼P(⋅∣s,a)​[R(s,a)+γa′max​Q∗(s′,a′)]
在该视角下，状态 s 必须是或包含马尔科夫毯。若不包含，Bellman算子在观测空间不自闭合，递推更新会产生系统偏差。

Principle of Optimality在马尔科夫毯视角下变成了一个约束化命题：在不损失最优性的前提下策略类可以限制为仅依赖毯内变量的函数类，所以若毯已知或可得，最优策略 π∗(s) 的函数类的复杂度可以显著降低，这直接影响样本复杂度、泛化界与可学习性边界。若毯缺失，任何以观测为输入的平稳 Markov 策略都可能是次优，必须考虑历史依赖策略或信念状态策略，策略类别的复杂度随之爆炸。

许多RL算法的无偏估计都隐含或显式依赖马尔科夫性，我们用毯视角可以精确指出偏差来源：

**TD / one-step update 无偏性**：当且仅当更新时条件期望在表示空间闭合（即表示包含毯）时，单步引导目标 Rt+1​+γV(St+1​) 的条件期望等于 Bellman 操作，样本更新是无偏的。否则，单步目标包含系统性偏差。

**策略梯度定理与状态分布**：策略梯度的推导需要定义状态分布 dπ(s)（长期占据分布）。若表示不包含毯，所谓的“状态分布”不可定义或不具备 Markov 特性，从而使推导与方差/偏差分析失效。策略梯度定理的一般形式为：

∇θ​J(πθ​)=Eτ∼πθ​​[t=0∑T​∇θ​logπθ​(At​∣St​)Gt​]

其中 St​ 必须是马尔科夫的。

当观测缺失毯内信息时，理论后果是明确的，自然模型由 MDP → POMDP，恢复Bellman 结构的正确方法是把历史映射到信念状态 b(ht​)=P(St​=s∣Ht​=ht​) 或找到其它充分统计量，但belief是连续分布带来维数与计算难题。

### 马尔科夫毯视角对核心技术 / 算法的直接、可操作影响

马尔科夫毯视角对强化学习核心技术or算法的影响体现在对状态表示学习的根本重塑，我们之前的传统强化学习依赖对完整环境状态的假设近似，但是在实际应用中环境的真实状态通常是部分可观测或者高维复杂的。

马尔科夫毯理论为表示学习提供了一个理想目标，就是编码器应提取历史信息中所有与未来决策相关的最小充分统计量，就是编码得到的表示变量必须能屏蔽掉对未来无关的所有信息。

为了理论落地RL算法通常通过设计多任务损失函数实现对马尔科夫毯性质的逼近，预测损失（Lpredict​）促使编码器学会从当前表示和动作中准确预测未来的表示和即时奖励，保证了表示的动态完备性，而信息瓶颈损失（LIB​）则强制表示压缩不必要的信息，表示具备良好的泛化性和鲁棒性。此外的bisimulation损失（Lbisim​）引入了状态行为等价的概念，通过测量不同状态表示在奖励和转移概率上的相似性推动编码器学习行为等价的状态聚合，来形成紧凑且对策略最优性友好的表示空间。

算法来看这些损失项通常联合优化，形成一个端到端的训练管线。编码器负责将历史轨迹编码成低维表示，动态模型和奖励模型在该表示空间进行训练，从而捕捉环境的转移规律和回报结构。训练过程中 编码器 模型 策略网络相互耦合一起进化，策略直接在符合马尔科夫毯条件的表示空间中学习决策。该管线提升了数据利用效率降低了样本复杂度，而且通过约束表示的因果屏蔽性质还提高了策略在不同环境变化下的稳健性。

从我们实验来看调节各个损失项的权重系数是关键的超参数搜索方向，过强的信息瓶颈约束导致表示不足以保留所有必要信息，但是弱化bisimulation损失可能导致状态空间划分模糊。

### 马尔科夫毯视角对理论证明与复杂度边界的影响

马尔科夫毯引入之后丰富了rl工具，特别是保证算法收敛性和复杂度分析，经典的强化学习理论通常基于MDP假设，状态完整且满足无记忆性质。然而就像上面说的，现实中环境状态常常是高维和部分观测，直接保证马尔科夫性难以成立。马尔科夫毯视角通过定义足够的因果状态 信息屏蔽集来恢复严格的马尔科夫性。

上述使得策略与价值函数的收敛性证明能够在压缩的表示空间中进行，不再依赖对原始高维状态的全信息访问，所以显著降低了分析的复杂度。马尔科夫毯定义了状态与未来的因果中介让许多传统上因维度灾难和部分可观测性带来的问题得以规避。具体点就是样本复杂度和计算复杂度可以通过马尔科夫毯的维度和信息量刻画，例如从依赖于原始状态空间维度 ∣S∣ 变为依赖于马尔科夫毯的维度 ∣MB∣，而不是原始状态空间的维度。

### 因果中介视角对迁移、鲁棒与离线强化学习的影响

因果中介强调学习到的表示不仅仅是统计相关而是能够反映环境中的真实因果机制。迁移学习中的因果中介状态作为环境内在机制的抽象表征具有更强的通用性，因为它们捕捉的是决定未来动态和奖励的核心因果因素而不是表面统计特征，所以当环境发生变化时只要核心因果机制不变，基于马尔科夫毯的表示和策略就可以较快适应新环境。

鲁棒强化学习，因果中介来看就是帮助算法区分本质变量 干扰变量，通过专注于因果中介算法能够屏蔽环境中的噪声和扰动，只针对对未来行为和回报有因果影响的变量做决策。

在离线r中，因果中介为解决分布偏移和泛化问题提供了理论工具，因为离线数据集往往来自旧策略or不同环境分布，直接利用容易导致策略在新环境下失效，通过学习因果中介变量离线算法能构建对环境机制的深层理解来对对未知环境的因果变化做出更准确的推断和调整。

### Markov blanket

这是个Markov blanket视角的表示学习框架，就是表示RL中从部分观测的历史数据中学习出满足马尔科夫性质的紧凑状态表，兼顾了对未来环境动态和奖励的预测能力和利用因果中介思想，通过bisimulation损失保证状态的因果等价性。

能看到代码由四个核心的神经网络模块组成：Encoder RepresentationLearner  DynamicsModel RewardPredictor

编码器设计解决的问题是在部分可观测POMDP中，单帧观测Ot不足以构成马尔科夫状态。比如我们要知道一个球的运动速度至少需要连续两帧的图像，所以编码器必须处理历史信息。这里选用的是门控循环单元，它按时间顺序处理一个观测-动作对的序列 (obs_history, action_history)，hidden state在每个时间步更新，自然地将历史信息整合。

最终GRU在最后一个时间步的隐状态被认为是整个历史的浓缩摘要，这个摘要随后通过两个线性层输出一个高斯分布的mean和logvar。

bisimulation损失实现因果等价约束是通过衡量两个状态在奖励和转移概率上的相似性，使具有相似因果行为的状态在潜在空间靠得更近，强化状态表示的因果一致性和抽象能力。

```
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

def gaussian_w2_distance(mean1, logvar1, mean2, logvar2):
    """
    Computes a simplified Wasserstein-2 distance between two diagonal Gaussian distributions.
    This implementation focuses on the distance between means, which is often sufficient.
    A full implementation would also consider the covariance matrices.
    """
    # The distance is primarily driven by the difference in means.
    return torch.sum((mean1 - mean2)**2, dim=-1)

class Encoder(nn.Module):
    """
    Encodes a history of observations into a latent state distribution (Markov Blanket).
    Uses a GRU to process sequences, suitable for POMDPs.
    The output is stochastic (mean and log_variance of a Gaussian distribution).
    """
    def __init__(self, obs_dim, action_dim, hidden_dim=256, latent_dim=32):
        super().__init__()
        self.latent_dim = latent_dim
        # We process observation-action pairs to better capture history.
        self.rnn = nn.GRU(obs_dim + action_dim, hidden_dim, batch_first=True)
        self.fc_mean = nn.Linear(hidden_dim, latent_dim)
        self.fc_logvar = nn.Linear(hidden_dim, latent_dim)

    def forward(self, obs_history, action_history):
        """
        Args:
            obs_history (Tensor): Shape (batch_size, seq_len, obs_dim)
            action_history (Tensor): Shape (batch_size, seq_len, action_dim)
        Returns:
            mean (Tensor): Shape (batch_size, latent_dim)
            logvar (Tensor): Shape (batch_size, latent_dim)
        """
        # Concatenate observations and actions along the feature dimension
        x = torch.cat([obs_history, action_history], dim=-1)
        # The second output of the GRU is the hidden state of the last time step
        _, h_n = self.rnn(x)
        h_n = h_n.squeeze(0) # Remove the num_layers dimension
        
        mean = self.fc_mean(h_n)
        logvar = self.fc_logvar(h_n)
        return mean, logvar

class DynamicsModel(nn.Module):
    """
    Predicts the next latent state distribution given the current latent state and action.
    p(z_{t+1} | z_t, a_t)
    """
    def __init__(self, latent_dim, action_dim, hidden_dim=256):
        super().__init__()
        self.model = nn.Sequential(
            nn.Linear(latent_dim + action_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        self.fc_mean = nn.Linear(hidden_dim, latent_dim)
        self.fc_logvar = nn.Linear(hidden_dim, latent_dim)

    def forward(self, z, a):
        x = torch.cat([z, a], dim=-1)
        hidden = self.model(x)
        mean = self.fc_mean(hidden)
        logvar = self.fc_logvar(hidden)
        return mean, logvar

class RewardPredictor(nn.Module):
    """
    Predicts the immediate reward given a latent state.
    r_t ~ p(r | z_t)
    """
    def __init__(self, latent_dim, hidden_dim=256):
        super().__init__()
        self.model = nn.Sequential(
            nn.Linear(latent_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 1)
        )

    def forward(self, z):
        return self.model(z)

class RepresentationLearner(nn.Module):
    """
    The main module that combines the encoder and predictive models.
    It computes the total loss for learning a Markovian representation.
    """
    def __init__(self, obs_dim, action_dim, hidden_dim=256, latent_dim=32, 
                 beta=1e-3, eta=0.1, gamma=0.99):
        super().__init__()
        self.latent_dim = latent_dim
        self.gamma = gamma # Discount factor for bisimulation
        
        # Loss weights
        self.beta = beta   # Weight for VIB (KL divergence) loss
        self.eta = eta     # Weight for bisimulation loss

        self.encoder = Encoder(obs_dim, action_dim, hidden_dim, latent_dim)
        self.dynamics_model = DynamicsModel(latent_dim, action_dim, hidden_dim)
        self.reward_predictor = RewardPredictor(latent_dim, hidden_dim)
        # A decoder to predict the next observation for reconstruction loss
        self.obs_decoder = nn.Sequential(
            nn.Linear(latent_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, obs_dim)
        )

    def reparameterize(self, mean, logvar):
        """
        Reparameterization trick to sample from N(mean, var) while allowing backpropagation.
        """
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mean + eps * std

    def forward(self, obs_history, action_history, current_action, next_obs, reward):
        """
        Computes the total loss for a batch of transitions.
        
        Args:
            obs_history (Tensor): (batch, seq_len, obs_dim)
            action_history (Tensor): (batch, seq_len, action_dim)
            current_action (Tensor): (batch, action_dim)
            next_obs (Tensor): (batch, obs_dim)
            reward (Tensor): (batch, 1)
        """
        
        mean_t, logvar_t = self.encoder(obs_history, action_history)
        z_t = self.reparameterize(mean_t, logvar_t)

       
        predicted_next_z_mean, _ = self.dynamics_model(z_t, current_action)
        predicted_next_obs = self.obs_decoder(predicted_next_z_mean)
        prediction_loss = F.mse_loss(predicted_next_obs, next_obs)

       
        vib_loss = -0.5 * torch.sum(1 + logvar_t - mean_t.pow(2) - logvar_t.exp(), dim=1).mean()

       
        indices = torch.randperm(z_t.size(0))
        z_j = z_t[indices]
        mean_j, logvar_j = mean_t[indices], logvar_t[indices]
        reward_j = reward[indices]

       
        pred_reward_i = self.reward_predictor(z_t)
        pred_reward_j = self.reward_predictor(z_j)
        
        
        reward_dist = F.mse_loss(pred_reward_i, pred_reward_j)

       
        next_z_mean_i, next_z_logvar_i = self.dynamics_model(z_t, current_action)
        next_z_mean_j, next_z_logvar_j = self.dynamics_model(z_j, current_action)
        
     
        transition_dist = gaussian_w2_distance(
            next_z_mean_i.detach(), next_z_logvar_i.detach(), # Detach to prevent collapse
            next_z_mean_j, next_z_logvar_j
        ).mean()

        bisimulation_loss = reward_dist + self.gamma * transition_dist

     
        total_loss = prediction_loss + self.beta * vib_loss + self.eta * bisimulation_loss
        
        return {
            "total_loss": total_loss,
            "prediction_loss": prediction_loss.item(),
            "vib_loss": vib_loss.item(),
            "bisimulation_loss": bisimulation_loss.item()
        }

if __name__ == '__main__':
    # Hyperparameters
    BATCH_SIZE = 64
    SEQ_LEN = 10
    OBS_DIM = 16
    ACTION_DIM = 4
    LATENT_DIM = 32
    HIDDEN_DIM = 256
    
    
    learner = RepresentationLearner(
        obs_dim=OBS_DIM,
        action_dim=ACTION_DIM,
        hidden_dim=HIDDEN_DIM,
        latent_dim=LATENT_DIM,
        beta=1e-3, # VIB loss weight
        eta=0.1,   # Bisimulation loss weight
        gamma=0.99 # Discount factor
    )
    
    optimizer = torch.optim.Adam(learner.parameters(), lr=1e-4)

   
    obs_history = torch.randn(BATCH_SIZE, SEQ_LEN, OBS_DIM)
    action_history = torch.randn(BATCH_SIZE, SEQ_LEN, ACTION_DIM)
    current_action = torch.randn(BATCH_SIZE, ACTION_DIM)
    next_obs = torch.randn(BATCH_SIZE, OBS_DIM)
    reward = torch.randn(BATCH_SIZE, 1)

   
    print("--- Running a single training step ---")
    learner.train()
    optimizer.zero_grad()
    
    losses = learner(obs_history, action_history, current_action, next_obs, reward)
    
    total_loss = losses["total_loss"]
    total_loss.backward()
    optimizer.step()
    
    print(f"Total Loss: {losses['total_loss']:.4f}")
    print(f"  Prediction Loss: {losses['prediction_loss']:.4f}")
    print(f"  VIB Loss (KL): {losses['vib_loss']:.4f}")
    print(f"  Bisimulation Loss: {losses['bisimulation_loss']:.4f}")

   
    print("\n--- Running inference to get state representation ---")
    learner.eval()
    with torch.no_grad():
        # In a real agent, you would get the mean of the distribution as the state
        mean_z, logvar_z = learner.encoder(obs_history, action_history)
        # The learned state `z` can now be fed into a policy or value network
        state_representation = mean_z 
    
    print(f"Encoded state representation shape: {state_representation.shape}")
    assert state_representation.shape == (BATCH_SIZE, LATENT_DIM)
    print("Code execution successful.")


```

