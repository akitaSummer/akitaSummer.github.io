---
title: DeepSeek 论文浅析
date: 2025-01-20 00:21:35
tags:
  - DeepSeek
categories: 学习笔记
---

# 前言
本文是阅读 DeepSeek 论文后的个人理解沉淀。

# LLM 相关原理快速入门
在 LLM 中的核心关键点是 pre-training（预训练）和 post-training（后训练）。

在我们一般说的基座模型，例如 [DeepSeek-V3-Base](https://huggingface.co/deepseek-ai/DeepSeek-V3-Base) 他就是属于预训练模型，通过 超大神经网络参数(＞600B)＋超多数据，得到一个 token predator (Token 预测机) 而不是 GPT Bot。这是非常困难的工作，一般是由大公司来做，他们拥有大量的数据和 GPU。

而 [DeepSeek-V3](https://huggingface.co/deepseek-ai/DeepSeek-V3) 属于通过后训练得到的模型，他是在预训练模型基础上进行多次后训练来得到。其中一个比较重要的概念就是 Reinforcement learning from human feedback（基于人类反馈的强化学习）,GPT 3.5 就是使用 RLHF 进行后训练，使傻傻的 Token 预测机能和人类说话，让人觉得他像个真人。

当然后训练有很多种，这里介绍几种：
1. SFT（监督微调，Supervised Fine-Tuning）： 相对预训练少量优质打标数据，有 Q and A。
2. RL (强化学习，Reinforcement learning)：只有 Q 没有 A，但是有策略 + 评分器来打分，比如力扣+测试集。
3. RLHF: 是否是人类偏好，一般是少量优质人类偏好的打标数据训练出一个打分模型，用这个模型来进行 RL


# DeepSeek-R1 做了什么
![approach](/image/deepseek/approach.png)
在 DeepSeek 论文中的第二章中阐述了他们的核心三件事：通过 RL 得到 ZERO，进一步后训练得到R1和蒸馏优化小模型

# DeepSeek-R1-Zero
人们通过人类学习方式创造了一些机器学习：
 - 重复，模仿 -> 监督学习 SFT
 - 尝试，失败 -> 强化学习 RL

在 DeepSeek-V3-Base 进行不断地 RL 后得到了 [DeepSeek-R1-Zero](https://huggingface.co/deepseek-ai/DeepSeek-R1-Zero)。

他有两种核心策略：
1. 正确性检验：力扣，数学计算等有准确评价标准的测试
2. 格式检验：\<think>...\</think>\<answer>...\</answer>

在他们不断训练的时候发现了一个 aha-moment，这个时候认为他有了推理能力。

![aha](/image/deepseek/aha.png)

```python
import re
import torch
from datasets import load_dataset, Dataset
from transformers import AutoTokenizer, AutoModelForCausalLM
import trl
from trl import GRPOConfig, GRPOTrainer
from peft import LoraConfig, get_peft_model, TaskType

SYSTEM_PROMPT = """
按照如下格式生成：
<think>
...
</think>
<answer>
...
</answer>
"""
def process_data(data):
    data = data.map(lambda x: {
        'prompt': [
            {'role': 'system', 'content': SYSTEM_PROMPT},
            {'role': 'user', 'content': x['question_zh-cn']}
        ],
        'answer': x['answer_only']
    }) 
    return data
def extract_answer(text):
    answer = text.split("<answer>")[-1]
    answer = answer.split("</answer>")[0]
    return answer.strip()

def mark_num(text):
    reward = 0
    if text.count("<think>\n") == 1:
        reward += 0.125
        
    if text.count("</think>\n") == 1:
        reward += 0.125
        
    if text.count("<answer>\n") == 1:
        reward += 0.125
        
    if text.count("</answer>\n") == 1:
        reward += 0.125
    return reward

# 生成答案是否正确的奖励
def correctness_reward(prompts, completions, answer, **kwargs):
    responses = [completion[0]['content'] for completion in completions]
    extracted_responses = [extract_answer(r) for r in responses]
    print(f"问题:\n{prompts[0][-1]['content']}", f"\n答案:\n{answer[0]}", f"\n模型输出:\n{responses[0]}", f"\n提取后的答案:\n{extracted_responses[0]}")
    return [2.0 if response == str(ans) else 0.0 for response, ans in zip(extracted_responses, answer)]
# 生成答案是否是数字的奖励（单纯依赖结果是否正确进行奖励，条件很苛刻，会导致奖励比较稀疏，模型难以收敛，所以加上答案是否是数字的奖励，虽然答案错误，但是至少生成的是数字（对于数学问题），也要给予适当奖励）
def digit_reward(completions, **kwargs):
    responses = [completion[0]['content'] for completion in completions]
    extracted_responses = [extract_answer(r) for r in responses]
    return [0.5 if response.isdigit() else 0.0 for response in extracted_responses]

# 格式奖励
def hard_format_reward(completions, **kwargs):
    pattern = r"^<think>\n.*?n</think>\n<answer>\n.*?\n</answer>\n$"
    responses = [completion[0]["content"] for completion in completions]
    matches = [re.match(pattern, response) for response in responses]
    return [0.5 if match else 0.0 for match in matches]
# 格式奖励
def soft_format_reward(completions, **kwargs):
    pattern = r"<think>.*?</think>\s*<answer>.*?</answer>"
    responses = [completion[0]["content"] for completion in completions]
    matches = [re.match(pattern, response) for response in responses]
    return [0.5 if match else 0.0 for match in matches]
# 标记奖励（改善格式奖励稀疏问题）
def mark_reward(completions, **kwargs):
    responses = [completion[0]["content"] for completion in completions]
    return [mark_num(response) for response in responses]


if __name__ == '__main__':
    model_name = "/Users/liuyu/AkitaSummer/deepseek_learn/Qwen2.5-0.5B-Instruct"

    model = AutoModelForCausalLM.from_pretrained(model_name)
    model.cuda()
    
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    
    ds = load_dataset('/Users/liuyu/AkitaSummer/deepseek_learn/gsm8k_chinese')
    data = process_data(ds['train'])
    
    output_dir="output"

    training_args = GRPOConfig(
        output_dir=output_dir,
        learning_rate=5e-6,
        adam_beta1 = 0.9,
        adam_beta2 = 0.99,
        weight_decay = 0.1,
        warmup_ratio = 0.1,
        lr_scheduler_type='cosine',
        logging_steps=1,
        bf16=True,
        per_device_train_batch_size=1,
        gradient_accumulation_steps=4,
        num_generations=16,
        max_prompt_length=256,
        max_completion_length=200,
        num_train_epochs=1,
        save_steps=100,
        max_grad_norm=0.1,
        log_on_each_node=False,
        use_vllm=False,
        report_to="tensorboard"
    )
    
    trainer = GRPOTrainer(
    model=model,
    processing_class=tokenizer,
    reward_funcs=[
        mark_reward,
        soft_format_reward,
        hard_format_reward,
        digit_reward,
        correctness_reward
        ],
    args=training_args,
    train_dataset=data,

)
    trainer.train()
    trainer.save_model(output_dir)
```

# DeepSeek-R1
在前面训练出 DeepSeek-R1-Zero 后，仍存在一些可用性问题，比如性能不够强大，双语回答问题等。

为了解决这个问题，他们做了以下步骤：
1. 先用 DeepSeek-R1-Zero 生产了 60W 条优质的 \<question>...\</question>\<think>...\</think>\<answer>...\</answer> 格式的推理数据
2. 再用 DeepSeek-V3 生产了 20W 条优质的 \<question>...\</question>\<answer>...\</answer>  格式的知识数据
3. 基于 DeepSeek-V3-Base 进行了一轮 SFT，得到一个中间模型
4. 对中间模型进行一次类似 Zero 的 RL，最后产出 DeepSeek-R1

# 蒸馏
对于大模型的知识蒸馏，主要分为两种.

## 黑盒知识蒸馏
使用大模型生成数据，通过这些数据去微调更小的模型，来达到蒸馏的目的。缺点是蒸馏效率低，优点是实现简单。
![approach](/image/deepseek/black.png)
```python
# per_device_train_batch_size: 4 # 单次训练步骤中处理的样本数,
# gradient_accumulation_steps: 2 # 梯度累积步数，模拟更大的批量大小
# learning_rate: 1.0e-4 # 初始学习率，控制参数更新步长
# num_train_epochs: 1 # 训练遍历整个数据集的次数
# lr_scheduler_type: cosine
# warmup_ratio: 0.1 # 学习率预热的比例
# bf16: true # 启用bfloat16混合精度训练
# ddp_timeout: 180000000

from datasets from load_dataset

ds = load_dataset('/Users/liuyu/AkitaSummer/deepseek_learn/OpenThought-114k')

ds = ds['train']

data = []
for item in ds:
    tmp = {}
    system = item['system']
    conversations = item['conversations']
    tmp['instruction'] = system
    tmp['input'] = conversations[0]['value']
    tmp['output'] = conversations[1]['value']
    data.append(tmp)
```


## 白盒知识蒸馏
获取学生模型和教师模型的输出概率分布（或者中间隐藏层的概率分布），通过kl散度将学生模型的概率分布向教师模型对齐。 下面主要介绍和测试白盒知识蒸馏： 白盒知识蒸馏主要在于模型分布的对齐，模型分布对齐主要依赖kl散度，对于kl散度的使用又有如下几种方式：

1. 前向kl散度。
也就是我们经常说的kl散度。
$$\hat{\mathrm{q}}=\mathrm{argmin}_\mathrm{q}\int_\mathrm{x}\mathrm{p(x)log}\frac{\mathrm{p(x)}}{\mathrm{q(x)}}\mathrm{dx}$$
p 为教师模型的概率分布，q 为学生模型的概率分布，[minillm](https://arxiv.org/abs/2306.08543)论文中提到前向 kl 散度可能会使学生模型高估教师模型中概率比较低的位置，结合公式来看，当 p 增大时，为了使得 kl 散度小，则 q 也需要增大，但是当 p 趋于0时，无论 q 取任何值， kl 散度都比较小，因为此时 $\mathrm{p(x)log}\frac{\mathrm{p(x)}}{\mathrm{q(x)}}$ 的大小主要受 p(x) 控制，这样起不到优化 q 分布的效果，可能会使q分布高估p分布中概率低的位置。 下图展示了前向 kl 散度的拟合情况，前向 kl 散度是一种均值搜索，更倾向于拟合多峰
![approach](/image/deepseek/fkl.png)

2. 反向kl散度。
为了缓解前向kl散度的缺点，提出了反向kl散度。
$$\mathrm{\hat{q}~=argmin_q~\int_x~q(x)log\frac{q(x)}{p(x)}~dx}$$
p 为教师模型的概率分布，q 为学生模型的概率分布，当 p 趋于零时，为了使 kl 散度小  q 也需趋于0。 minillm论文中说对于大模型的知识蒸馏，反向kl散度优于前向kl散度，但是也有其他论文说反向kl散度不一定比前向kl散度更优，实际选择中，可能要基于实验驱动。反向kl散度是一种模式搜索，更倾向于拟合单个峰 rkl
![approach](/image/deepseek/rkl.png)

3. 偏向前kl散度。
对学生模型的分布和教师模型的分布进行加权作为学生模型的分布。

4. 偏向反kl散度。
对学生模型的分布和教师模型的分布进行加权作为教师模型的分布。

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, DefaultDataCollator
from peft import LoraConfig, get_peft_model, TaskType
from peft import PeftModel
import torch
import torch.nn as nn
import torch.nn.functional as F
from transformers import Trainer, TrainingArguments
from dataset import SFTDataset

# 计算前向kl散度
def compute_fkl(
        logits, 
        teacher_logits, 
        target, 
        padding_id,
        reduction="sum",
        temp = 1.0, 
        
    ):
        logits = logits / temp
        teacher_logits = teacher_logits / temp

        log_probs = torch.log_softmax(logits, -1, dtype=torch.float32)
        teacher_probs = torch.softmax(teacher_logits, -1, dtype=torch.float32)
        teacher_log_probs = torch.log_softmax(teacher_logits, -1, dtype=torch.float32)
        kl = (teacher_probs * (teacher_log_probs - log_probs)) 
        kl = kl.sum(-1)
        if reduction == "sum":
            pad_mask = target.eq(padding_id)
            kl = kl.masked_fill_(pad_mask, 0.0)
            kl = kl.sum()

        return kl
# 计算反向kl散度
def compute_rkl(
        logits, 
        teacher_logits, 
        target, 
        padding_id,
        reduction="sum", 
        temp = 1.0
    ):
        logits = logits / temp
        teacher_logits = teacher_logits / temp

        probs = torch.softmax(logits, -1, dtype=torch.float32)
        log_probs = torch.log_softmax(logits, -1, dtype=torch.float32)
        teacher_log_probs = torch.log_softmax(teacher_logits, -1, dtype=torch.float32)
        kl = (probs * (log_probs - teacher_log_probs))
        kl = kl.sum(-1)
        if reduction == "sum":
            pad_mask = target.eq(padding_id)
            kl = kl.masked_fill_(pad_mask, 0.0)
            kl = kl.sum()
        return kl

# 计算偏向前kl散度
def compute_skewed_fkl(
        logits, 
        teacher_logits, 
        target, 
        padding_id, 
        reduction="sum", 
        temp = 1.0,
        skew_lambda = 0.1
    ):
        logits = logits / temp
        teacher_logits = teacher_logits / temp

        probs = torch.softmax(logits, -1, dtype=torch.float32)
        teacher_probs = torch.softmax(teacher_logits, -1, dtype=torch.float32)
        mixed_probs = skew_lambda * teacher_probs + (1 - skew_lambda) * probs
        mixed_log_probs = torch.log(mixed_probs)
        teacher_log_probs = torch.log_softmax(teacher_logits, -1, dtype=torch.float32)
        kl = (teacher_probs * (teacher_log_probs - mixed_log_probs))
        kl = kl.sum(-1)
        if reduction == "sum":
            pad_mask = target.eq(padding_id)
            kl = kl.masked_fill_(pad_mask, 0.0)
            kl = kl.sum()

            
        return kl
# 计算偏向反kl散度    
def compute_skewed_rkl(
    logits, 
    teacher_logits, 
    target,
    padding_id,
    reduction="sum", 
    temp = 1.0,
    skew_lambda = 0.1
):
    logits = logits / temp
    teacher_logits = teacher_logits / temp
    
    probs = torch.softmax(logits, -1, dtype=torch.float32)
    teacher_probs = torch.softmax(teacher_logits, -1, dtype=torch.float32)
    mixed_probs = (1 - skew_lambda) * teacher_probs + skew_lambda * probs
    mixed_log_probs = torch.log(mixed_probs)
    log_probs = torch.log_softmax(logits, -1, dtype=torch.float32)
    kl = (probs * (log_probs - mixed_log_probs))
    kl = kl.sum(-1)
    
    if reduction == "sum":
        pad_mask = target.eq(padding_id)
        kl = kl.masked_fill_(pad_mask, 0.0)
        kl = kl.sum()


    return kl


class KGTrainer(Trainer):
    
    def __init__(
        self,
        model = None,
        teacher_model = None,
        if_use_entropy = False,
        args = None,
        data_collator = None, 
        train_dataset = None,
        eval_dataset = None,
        tokenizer = None,
        model_init = None, 
        compute_metrics = None, 
        callbacks = None,
        optimizers = (None, None), 
        preprocess_logits_for_metrics = None,
    ):
        super().__init__(
            model,
            args,
            data_collator,
            train_dataset,
            eval_dataset,
            tokenizer,
            model_init,
            compute_metrics,
            callbacks,
            optimizers,
            preprocess_logits_for_metrics,
        )
        self.teacher_model = teacher_model
        self.if_use_entropy = if_use_entropy
        
    
    def compute_loss(self, model, inputs, return_outputs=False):
        
        outputs = model(**inputs)
        with torch.no_grad():
            teacher_outputs = self.teacher_model(**inputs)
        
        loss = outputs.loss
        logits = outputs.logits
        teacher_logits = teacher_outputs.logits
        
        # 如果教师模型和学生模型输出形状不匹配，对学生模型进行padding或对教师模型进行截断
        if logits.shape[-1] != teacher_logits.shape[-1]:
            # gap = teacher_logits.shape[-1] - logits.shape[-1]
            # if gap > 0:
            #     pad_logits = torch.zeros((logits.shape[0], logits.shape[1], gap)).to(logits.device)
            #     logits = torch.cat([logits, pad_logits], dim=-1)
            
            teacher_logits = teacher_logits[:, :, :logits.shape[-1]]
        
        labels = inputs['labels']
        kl = compute_fkl(logits, teacher_logits, labels, padding_id=-100, temp=2.0)
        
        if self.if_use_entropy:
            loss_total = 0.5 * kl + 0.5 * loss
        else:
            loss_total = kl
        
        return (loss_total, outputs) if return_outputs else loss_total
        

if __name__ == '__main__':
    
    # 学生模型
    model = AutoModelForCausalLM.from_pretrained("Qwen2.5-0.5B-Instruct")
    
    lora_config = LoraConfig(
    r=8,  
    lora_alpha=256,  
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.1, 
    task_type=TaskType.CAUSAL_LM)
    # 使用lora方法训练
    model = get_peft_model(model, lora_config)
    model.cuda()
    print(model.print_trainable_parameters())
    
    tokenizer = AutoTokenizer.from_pretrained("Qwen2.5-0.5B-Instruct")
    
    # 教师模型，在给定数据上通过lora微调
    teacher_model = AutoModelForCausalLM.from_pretrained("Qwen2.5-7B-Instruct")
    # 是否加载lora模型
    lora_path = 'qwen2.5_7b/lora/sft'
    teacher_model = PeftModel.from_pretrained(teacher_model, lora_path)
    teacher_model.cuda()
    teacher_model.eval()
    
    args = TrainingArguments(output_dir='./results', 
                            num_train_epochs=10, 
                            do_train=True, 
                            per_device_train_batch_size=2,
                            gradient_accumulation_steps=16,
                            logging_steps=10,
                            report_to='tensorboard',
                            save_strategy='epoch',
                            save_total_limit=10,
                            bf16=True,
                            learning_rate=0.0005,
                            lr_scheduler_type='cosine',
                            dataloader_num_workers=8,
                            dataloader_pin_memory=True)
    data_collator = DefaultDataCollator()
    dataset = SFTDataset('data.json', tokenizer=tokenizer, max_seq_len=512)
    trainer = KGTrainer(model=model,
                        teacher_model=teacher_model, 
                        if_use_entropy = True,
                        args=args, 
                        train_dataset=dataset, 
                        tokenizer=tokenizer, 
                        data_collator=data_collator)
    # 如果是初次训练resume_from_checkpoint为false，接着checkpoint继续训练，为True
    trainer.train(resume_from_checkpoint=False)
    trainer.save_model('./saves')
    trainer.save_state()
```