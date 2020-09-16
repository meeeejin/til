# How to use Slim

## Install

```bash
$ pip install --upgrade tf_slim
```

## Usage

```bash
import tf_slim as slim
```

## How to train a model using the Slim tool

1. Clone the [repository](https://github.com/tensorflow/models):

```bash
$ git clone https://github.com/tensorflow/models.git
$ cd models/research/slim
```

2. Download the [Visual Wake Words](https://arxiv.org/abs/1906.05721) dataset and covert it to the labeled data (`TFRecords`).

```bash
$ python3 download_and_convert_data.py \
    --dataset_name=visualwakewords \
    --dataset_dir=data/visualwakewords
```

3. Train the *mobilenet_v1_025* model using the below command:

```bash
$ python3 train_image_classifier.py \
    --train_dir=vww_96_grayscale \
    --dataset_name=visualwakewords \
    --dataset_split_name=train \
    --dataset_dir=data/visualwakewords \
    --model_name=mobilenet_v1_025 \
    --preprocessing_name=mobilenet_v1 \
    --train_image_size=96 \
    --use_grayscale=True \
    --save_summaries_secs=300 \
    --learning_rate=0.045 \
    --label_smoothing=0.1 \
    --learning_rate_decay_factor=0.98 \
    --num_epochs_per_decay=2.5 \
    --moving_average_decay=0.9999 \
    --batch_size=96 \
    --max_number_of_steps=100000
```

If you use CPU only, specify `--clone_on_cpu=True`:

```bash
$ python3 train_image_classifier.py --clone_on_cpu=True\
    --train_dir=vww_96_grayscale \
    --dataset_name=visualwakewords \
    --dataset_split_name=train \
    --dataset_dir=data/visualwakewords \
    --model_name=mobilenet_v1_025 \
    --preprocessing_name=mobilenet_v1 \
    --train_image_size=96 \
    --use_grayscale=True \
    --save_summaries_secs=300 \
    --learning_rate=0.045 \
    --label_smoothing=0.1 \
    --learning_rate_decay_factor=0.98 \
    --num_epochs_per_decay=2.5 \
    --moving_average_decay=0.9999 \
    --batch_size=96 \
    --max_number_of_steps=100000
```

4. Monitor the training process using *tensorboard*:

```bash
$ tensorboard --logdir=vww_96_grayscale
...
TensorBoard 1.15.0 at http://mijin:6006/ (Press CTRL+C to quit)
```

You can see the experiment metrics like loss and accuracy by accessing the below links:

```bash
# On the local machine
http://mijin:6006

# On the remote machine (e.g., mijin's IP: 123.123.123.123)
http://123.123.123.123:6006
```

5. Evaluate the model during the training process using the below command.

> Replace the `model.ckpt-xxx` value of `--checkpoint_path` with the correct value according to the prefix of the checkpoint files in your `--train_dir`. In my case, the prefix was `model.ckpt-7505`: 

```bash
$ python3 eval_image_classifier.py \
--alsologtostderr \
--checkpoint_path=vww_96_grayscale/model.ckpt-7505 \
--dataset_dir=data/visualwakewords \
--dataset_name=visualwakewords \
--dataset_split_name=val \
--model_name=mobilenet_v1_025 \
--preprocessing_name=mobilenet_v1 \
--use_grayscale=True \
--train_image_size=96

...
INFO:tensorflow:Evaluation [406/406]
I0916 21:46:30.437922 140510024853248 evaluation.py:167] Evaluation [406/406]
eval/Recall_5[1]eval/Accuracy[0.52832514]
```

In the above case, the accuaracy is about **52.83% (0.52832514)**.

## Reference

- [TensorFlow-Slim](https://github.com/google-research/tf-slim)
- [TensorFlow Model Garden](https://github.com/tensorflow/models)
- [TinyML Book](https://tinymlbook.com/)