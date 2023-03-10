---
layout: post
title: Transfer Learning
comments: true
---

Andrew Ng said during his widely popular NIPS 2016 tutorial that transfer learning, after supervised learning, will be the next driver of ML commercial success.

![NG](https://raw.githubusercontent.com/yashk2810/yashk2810.github.io/master/images/andrew_ng_drivers_ml_success-1.png "NG")

### What is Transfer Learning?

Transfer learning is the transferring of knowledge gained from one model(trained on a significantly larger dataset) to another dataset with similar characteristics. For example, <a href="http://www.image-net.org/">Imagenet</a> contains images for 1000 categories. It is a competition held every year and VGG-16, Resnet50, InceptionV3, etc models were invented in this competition. 

If we use these models on say, <a href="https://www.kaggle.com/c/dogs-vs-cats">Dogs vs Cats</a> we would be using transfer learning. Since, Imagenet already has images of dogs and cats we would just be using the features learned by the models trained on Imagenet for our purpose.

Transfer learning was used in detecting <a href="http://news.stanford.edu/2017/01/25/artificial-intelligence-used-identify-skin-cancer/">skin cancer</a>. This paper was in the Nature magazine.

### Why Transfer Learning?

In practice, very few people train their own convolutional net from scratch because they don't have sufficient data. **It is always recommended to use transfer learning in practice.**

The below images show the things that convolutional networks learn when trained on Imagenet and why it is effective to use transfer learning.

![deconvnet1](https://raw.githubusercontent.com/yashk2810/yashk2810.github.io/master/images/deconvnet1.png "deconvnet1")
![deconvnet2](https://raw.githubusercontent.com/yashk2810/yashk2810.github.io/master/images/deconvnet2.png "deconvnet2")

As you can see in layer 1 and layer 2 visualization, the conv net learned to recognize edges, circles, etc. But the layer 4 and layer 5 learn to recognize the entire objects like dogs face, umbrella, etc. 

So, if we use a model trained on imagenet for Dogs vs Cats prediction we can just change the last classifier(fully-connected) layer since the last convolutional layer already knows what a dog or a cat looks like.

What if the dataset is not similar to the pretrained model data? In such cases, we can train the convnet by finetuning the weights of the pretrained model by continuing the backpropogation. You can finetune the entire convnet or keep some early layers of the network fixed(non-trainable) and finetune the higher layers. This is because the early layers contain general information about the image but the later layers become more specific to the classes in the original dataset.

Alright, let's code!

We will do transfer learning on the Dogs vs Cats competition using VGG-16 model trained on Imagenet. The library used for this is Keras with Theano backend. You can convert it to Tensorflow backend if you want to. 

VGG16 with Theano weights :- <a href="https://drive.google.com/file/d/0Bz7KyqmuGsilT0J5dmRCM0ROVHc/view">https://drive.google.com/file/d/0Bz7KyqmuGsilT0J5dmRCM0ROVHc/view</a>

VGG16 with Tensorflow weights :- <a href="https://github.com/fchollet/deep-learning-models/releases/download/v0.1/vgg16_weights_tf_dim_ordering_tf_kernels.h5">https://github.com/fchollet/deep-learning-models/releases/download/v0.1/vgg16_weights_tf_dim_ordering_tf_kernels.h5</a>

VGG16 is a sequential model with a very simple architecture. This makes transfer learning using VGG16 very easy.

![vgg16_architecture](https://raw.githubusercontent.com/yashk2810/yashk2810.github.io/master/images/vgg-16-architecture.png "vgg16_architecture")

### Code

Let's import the necessary libraries. 

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
%matplotlib inline
from keras.datasets import mnist
from keras.models import Sequential
from keras.layers import Dense, Dropout, Activation, Flatten
from keras.layers.core import Lambda
from keras.optimizers import Adam
from keras.layers.normalization import BatchNormalization
from keras.utils import np_utils
from keras.layers import Convolution2D, MaxPooling2D, ZeroPadding2D
from keras.preprocessing.image import ImageDataGenerator
```

Now we need to preprocess the input as the VGG16 authors did in their implementation(mentioned in the paper).

```python
# Subtracting the mean calculated by the VGG16 authors and reversing the channels from RGB to BGR.
def vgg_preprocess(x):
    x[:, 0, :, :] = x - 103.939
    x[:, 1, :, :] = x - 116.779
    x[:, 2, :, :] = x - 123.68
    return x[:, ::-1, :, :]
```

Now, we will prepare our data.

```python
gen = ImageDataGenerator()

train_generator = gen.flow_from_directory('data/dogscats/train', 
                                               target_size=(image_width, image_height),
                                               class_mode='binary',
                                               batch_size=64)

validation_generator = gen.flow_from_directory('data/dogscats/valid', 
                                               target_size=(image_width, image_height),
                                               class_mode='binary',
                                               batch_size=64)

# Converting the labels to one-hot encoded matrix
train_labels = np_utils.to_categorical(train_generator.classes)
validation_labels = np_utils.to_categorical(validation_generator.classes)                                           
```

Let's define the VGG16 model.

```python
# VGG16 was trained on 224*224 images. So we also need to use 224*224 images.
image_width = 224
image_height = 224
val_samples = 2000
train_samples = 23000

model = Sequential()

model.add(Lambda(vgg_preprocess, input_shape=(3, image_width, image_height), 
                 output_shape=(3, image_width, image_height)))

model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(64, 3, 3, activation='relu', name='conv1_1'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(64, 3, 3, activation='relu', name='conv1_2'))
model.add(MaxPooling2D((2, 2), strides=(2, 2)))

model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(128, 3, 3, activation='relu', name='conv2_1'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(128, 3, 3, activation='relu', name='conv2_2'))
model.add(MaxPooling2D((2, 2), strides=(2, 2)))

model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(256, 3, 3, activation='relu', name='conv3_1'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(256, 3, 3, activation='relu', name='conv3_2'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(256, 3, 3, activation='relu', name='conv3_3'))
model.add(MaxPooling2D((2, 2), strides=(2, 2)))

model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(512, 3, 3, activation='relu', name='conv4_1'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(512, 3, 3, activation='relu', name='conv4_2'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(512, 3, 3, activation='relu', name='conv4_3'))
model.add(MaxPooling2D((2, 2), strides=(2, 2)))

model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(512, 3, 3, activation='relu', name='conv5_1'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(512, 3, 3, activation='relu', name='conv5_2'))
model.add(ZeroPadding2D((1, 1)))
model.add(Convolution2D(512, 3, 3, activation='relu', name='conv5_3'))
model.add(MaxPooling2D((2, 2), strides=(2, 2)))

model.add(Flatten())
model.add(Dense(4096, activation='relu'))
model.add(Dropout(0.5))
model.add(Dense(4096, activation='relu'))
model.add(Dropout(0.5))

model.add(Dense(1000, activation='softmax'))

model.load_weights('vgg16_th.h5')
```

Transfer learning begins! Since we just have to predict 2 classes, this becomes a binary classification problem. So, we will pop the last dense layer which defines 1000 filters(since there are 1000 categories in Imagenet) and replace it with 1 filter. Moreover, we have to set all the layers as non-trainable. If they are set as trainable, then all the convolutional layers will be trained from scratch and transfer learning will be moot.
```python
model.pop()

for layer in model.layers:
    layer.trainable = False

model.add(Dense(1, activation='sigmoid'))
```

Set the loss, the optimizer and start training.

```python
model.compile(loss='binary_crossentropy', optimizer=Adam(lr=0.001), metrics=['accuracy'])

# ORIGINAL METHOD
model.fit_generator(train_generator, samples_per_epoch=train_samples, nb_epoch=1, 
                    validation_data=validation_generator, nb_val_samples=val_samples)
```

The accuracy achieved by doing these simple steps is an astounding **98.30%**.

### Next steps

You can use this technique on other datasets as well. The convolutional layers calculates all the features and the Dense(Fully connected) layers are used to classify the image based on the features. So, you can use a model which comprises of all the convolutional layers. Then use that model to predict the features based on our data. Next, create your own model and use the predicted features as the training data on this new model.

This new model can contain more convolutional layers or only dense layers or a combination of both or anything else. It's entirely upto you. You can try doing transfer learning on the <a href="https://www.kaggle.com/c/state-farm-distracted-driver-detection">Statefarm</a> competition or the <a href="https://www.kaggle.com/c/the-nature-conservancy-fisheries-monitoring">Fisheries</a> competition hosted on <a href="https://www.kaggle.com/">Kaggle</a>.

The <a href="http://course.fast.ai/index.html">Deep learning</a> course offered by Jeremy Howard and Rachel Thomas is fantastic. I highly recommend you to complete this course.

*If you???re interested in collaborating, discussing or working with me on an exciting idea, contact me at yash DOT katariya10 AT gmail.com*

<br />
Until next time,

Ciao  

<a href="https://twitter.com/share" class="twitter-share-button" data-size="large" data-text="Check out this AWESOME article" data-lang="en" data-show-count="false">Tweet</a><script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>




