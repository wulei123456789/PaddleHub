# PaddleHub 阅读理解

本示例将展示如何使用PaddleHub Finetune API以及BERT预训练模型完成阅读理解任务。

## 如何开始Finetune

在完成安装PaddlePaddle与PaddleHub后，通过执行脚本`sh run_finetune.sh`即可开始使用BERT对SQuAD数据集进行Finetune。

其中脚本参数说明如下：

```bash
# 模型相关
--batch_size: 批处理大小，请结合显存情况进行调整，若出现显存不足，请适当调低这一参数
--learning_rate: Finetune的最大学习率
--weight_decay: 控制正则项力度的参数，用于防止过拟合，默认为0.01
--warmup_proportion: 学习率warmup策略的比例，如果0.1，则学习率会在前10%训练step的过程中从0慢慢增长到learning_rate, 而后再缓慢衰减，默认为0
--num_epoch: Finetune迭代的轮数
--max_seq_len: BERT模型使用的最大序列长度，最大不能超过512, 若出现显存不足，请适当调低这一参数
--use_data_parallel: 是否使用并行计算，默认False。打开该功能依赖nccl库。
--use_pyreader: 是否使用pyreader，默认False。

# 任务相关
--checkpoint_dir: 模型保存路径，PaddleHub会自动保存验证集上表现最好的模型
--version_2_with_negative: 若version_2_with_negative=False，则使用SQuAD 1.1数据集；若version_2_with_negative=True，则使用SQuAD 2.0数据集；
```

## 代码步骤

使用PaddleHub Finetune API进行Finetune可以分为4个步骤

### Step1: 加载预训练模型

```python
module = hub.Module(name="bert_uncased_L-12_H-768_A-12")
inputs, outputs, program = module.context(trainable=True, max_seq_len=384)
```
其中最大序列长度`max_seq_len`是可以调整的参数，建议值384，根据任务文本长度不同可以调整该值，但最大不超过512。

### Step2: 准备数据集并使用ReadingComprehensionReader读取数据
```python
dataset = hub.dataset.SQUAD(
    version_2_with_negative=False)
reader = hub.reader.ReadingComprehensionReader(
    dataset=dataset,
    vocab_path=module.get_vocab_path(),
    max_seq_length=args.max_seq_len,
    doc_stride=128,
    max_query_length=64)
```

其中数据集的准备代码可以参考 [squad.py](https://github.com/PaddlePaddle/PaddleHub/blob/release/v1.2/paddlehub/dataset/squad.py)

`hub.dataset.SQUAD()` 会自动从网络下载数据集并解压到用户目录下`$HOME/.paddlehub/dataset`目录

`module.get_vocab_path()` 会返回预训练模型对应的词表

`max_seq_len` 需要与Step1中context接口传入的序列长度保持一致

ReadingComprehensionReader中的`data_generator`会自动按照模型对应词表对数据进行切词，以迭代器的方式返回BERT所需要的Tensor格式，包括`input_ids`，`position_ids`，`segment_id`与序列对应的mask `input_mask`.

**NOTE**: Reader返回tensor的顺序是固定的，默认按照input_ids, position_ids, segment_id, input_mask这一顺序返回。

### Step3：选择优化策略和运行配置

```python
strategy = hub.AdamWeightDecayStrategy(
    learning_rate=5e-5,
    weight_decay=0.01,
    warmup_proportion=0.0,
    lr_scheduler="linear_decay",
)

config = hub.RunConfig(use_cuda=True, num_epoch=2, batch_size=12, strategy=strategy)
```

#### 优化策略
针对ERNIE与BERT类任务，PaddleHub封装了适合这一任务的迁移学习优化策略`AdamWeightDecayStrategy`

`learning_rate`: Finetune过程中的最大学习率;

`weight_decay`: 模型的正则项参数，默认0.01，如果模型有过拟合倾向，可适当调高这一参数;

`warmup_proportion`: 如果warmup_proportion>0, 例如0.1, 则学习率会在前10%的steps中线性增长至最高值learning_rate;

`lr_scheduler`: 有两种策略可选(1) `linear_decay`策略学习率会在最高点后以线性方式衰减; `noam_decay`策略学习率会在最高点以多项式形式衰减;

#### 运行配置
`RunConfig` 主要控制Finetune的训练，包含以下可控制的参数:

* `log_interval`: 进度日志打印间隔，默认每10个step打印一次
* `eval_interval`: 模型评估的间隔，默认每100个step评估一次验证集
* `save_ckpt_interval`: 模型保存间隔，请根据任务大小配置，默认只保存验证集效果最好的模型和训练结束的模型
* `use_cuda`: 是否使用GPU训练，默认为False
* `checkpoint_dir`: 模型checkpoint保存路径, 若用户没有指定，程序会自动生成
* `num_epoch`: finetune的轮数
* `batch_size`: 训练的批大小，如果使用GPU，请根据实际情况调整batch_size
* `enable_memory_optim`: 是否使用内存优化， 默认为True
* `strategy`: Finetune优化策略

### Step4: 构建网络并创建阅读理解迁移任务进行Finetune
```python
seq_output = outputs["sequence_output"]

# feed_list的Tensor顺序不可以调整
feed_list = [
    inputs["input_ids"].name,
    inputs["position_ids"].name,
    inputs["segment_ids"].name,
    inputs["input_mask"].name,
]

reading_comprehension_task = hub.ReadingComprehensionTask(
    data_reader=reader,
    feature=seq_output,
    feed_list=feed_list,
    config=config)

reading_comprehension_task.finetune_and_eval()
```
**NOTE:**
1. `outputs["sequence_output"]`返回了BERT模型输入单词的对应输出,可以用于单词的特征表达。
2. `feed_list`中的inputs参数指名了BERT中的输入tensor的顺序，与ClassifyReader返回的结果一致。

## 可视化

Finetune API训练过程中会自动对关键训练指标进行打点，启动程序后执行下面命令
```bash
$ tensorboard --logdir $CKPT_DIR/visualization --host ${HOST_IP} --port ${PORT_NUM}
```
其中${HOST_IP}为本机IP地址，${PORT_NUM}为可用端口号，如本机IP地址为192.168.0.1，端口号8040，用浏览器打开192.168.0.1:8040，即可看到训练过程中指标的变化情况

## 模型预测

通过Finetune完成模型训练后，在对应的ckpt目录下，会自动保存验证集上效果最好的模型。
配置脚本参数

```bash
CKPT_DIR=".ckpt_rc/"
python predict.py --checkpoint_dir $CKPT_DIR --max_seq_len 384 --batch_size=12 --version_2_with_negative=False
```
其中CKPT_DIR为Finetune API保存最佳模型的路径, max_seq_len是ERNIE模型的最大序列长度，*请与训练时配置的参数保持一致*

参数配置正确后，请执行脚本`sh run_predict.sh`，预测时程序会自动调用官方评价脚本(version_2_with_negative=False调用evaluate_v1.py，version_2_with_negative=True调用evaluate_v2.py)即可看到SQuAD数据集的最终效果。
如需了解更多预测步骤，请参考`predict.py`

**NOTE:**
运行预测脚本时，建议用单卡预测。