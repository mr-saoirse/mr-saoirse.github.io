---
layout: post
title:  "BERT setups"
date:   2020-11-26 11:49:28 -0500
categories: AI NLP
---
# Fine tuning BERT from TF Hub
I want to check out different ways to get up and running fine tuning [attention][attention-paper]-based [BERT][bert-paper] starting with the tensorflow tutorial and later comparing it with other frameworks. The [TensorFlow documentation][fine-tune-bert-tf] provides a way to get up and running in a notebook or colab.

The notebook download is nicely self-contained with a pip install for the model garden package and links to stored checkpoints, hyperparameter configs and vocab file on a `gs` bucket and links to the model [hub][tf-hub] model for BERT. The data can also be downloaded via the tensorflow datasets package. The dataset is based on sentence matching based on rephrasing.

To fine tune a pre-trained model you need to be sure that you're using exactly the same tokenization, vocabulary, and index mapping as you used during training.

# Some TF components
We have a tokenizer that can map words based on embeddings. A tensorflow (constant)ragged tensor is used to track sentences and we can get ids for toeksn such as `CLS` and `SEP` and append them onto the vectors.

```python
def encode_sentence(s, tokenizer):
   tokens = list(tokenizer.tokenize(s))
   tokens.append('[SEP]')
   return tokenizer.convert_tokens_to_ids(tokens)

def bert_encode(glue_dict, tokenizer):
  num_examples = len(glue_dict["sentence1"])

  sentence1 = tf.ragged.constant([
      encode_sentence(s, tokenizer)
      for s in np.array(glue_dict["sentence1"])])
  sentence2 = tf.ragged.constant([
      encode_sentence(s, tokenizer)
       for s in np.array(glue_dict["sentence2"])])

  #projection
  cls = [tokenizer.convert_tokens_to_ids(['[CLS]'])]*sentence1.shape[0]
  input_word_ids = tf.concat([cls, sentence1, sentence2], axis=-1)

  #add problem semantics using mask and type
  input_mask = tf.ones_like(input_word_ids).to_tensor()

  type_cls = tf.zeros_like(cls)
  type_s1 = tf.zeros_like(sentence1)
  type_s2 = tf.ones_like(sentence2)
  input_type_ids = tf.concat(
      [type_cls, type_s1, type_s2], axis=-1).to_tensor()

  inputs = {
      'input_word_ids': input_word_ids.to_tensor(),
      'input_mask': input_mask,
      'input_type_ids': input_type_ids}

  return inputs
```

We can use this function to generate the training input from a mapping of the dataset and `X` and train using the datasets `y`

We can reconstitute the BERT model (and encoder) from config and plot it using keras utils. In the model is a transform encoder which is the complicated transformer stack.

We can fit the classifier

```python
metrics = [tf.keras.metrics.SparseCategoricalAccuracy('accuracy', dtype=tf.float32)]
loss = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)

#optimizer = nlp.optimization.create_optimizer with 3 epochs, and batch sizes of 32
#see the original post/notebooks for recommended learning rates and warm up steps etc.
bert_classifier.compile(
    optimizer=optimizer,
    loss=loss,
    metrics=metrics)

bert_classifier.fit(
      glue_train, glue_train_labels,
      validation_data=(glue_validation, glue_validation_labels),
      batch_size=32,
      epochs=epochs)

```
The model can then encode new data, be saved, loaded etc.

The model takes some time to train over each epoch i.e. 10s of minutes.

More generally, beyond a specific trained task, we can load models from the HUB to get sentence embeddings and then run functions for various text tasks like semantic similarity between sentences. For example we plot such below.

```python
def plot_similarity(features, labels):
  """Plot a similarity matrix of the embeddings."""
  cos_sim = pairwise.cosine_similarity(features)
  sns.set(font_scale=1.2)
  cbar_kws=dict(use_gridspec=False, location="left")
  g = sns.heatmap(
      cos_sim, xticklabels=labels, yticklabels=labels,
      vmin=0, vmax=1, cmap="Blues", cbar_kws=cbar_kws)
  g.tick_params(labelright=True, labelleft=False)
  g.set_yticklabels(labels, rotation=0)
  g.set_title("Semantic Textual Similarity")
```

We can use BERT for [classification][bert-class] and [GLUE tasks][glue-tasks]

# More Advanced things

The basics are described above and stuff is done in memory for this small dataset. Now we could consider some more realistic things.
A chunk of this relates to the TD datasets and TF record formats for managing data streams using the mapping over batches with prefetch recipe.

New heads can be added and the optimizer can be modified with decay rates for example.

# Huggingface and Pytorch
Now we do a similar thing with [Pytorch][pytorch-tutorials] and [Huggingface transformers][hface-tx] packaging. We are trying to get a feel for the most elegant programming model. The basic deep learning ideas are going to be the same but some abstractions may be more tasteful than others. There are some things I do not like such as the name of the so called `offical` module ([TF model garden lib][garden-lib]) and its `nlp` sub modules.

[bert-class]:https://www.tensorflow.org/tutorials/text/classify_text_with_bert
[glue-tasks]: https://www.tensorflow.org/tutorials/text/solve_glue_tasks_using_bert_on_tpu
[tf-hub]: https://tfhub.dev/
[garden-lib]: https://github.com/tensorflow/models
[attention-paper]: https://arxiv.org/abs/1706.03762
[bert-paper]: https://arxiv.org/abs/1810.04805
[fine-tune-bert-tf]: https://www.tensorflow.org/official_models/fine_tuning_bert
[pytorch-tutorials]: https://pytorch.org/tutorials/
[hface-tx]: https://github.com/huggingface/transformers