---
layout: page
title: Lab 04 Using and visualising pre-trained models
permalink: /education/cuda-glasgow/dl/lab04/
---

# Lab 04: Using and visualising pre-trained models #

*by Twin Karmakharm (University of Sheffield)*

**Remember to be working from ~/DLIntro directory throughout all practicals.**

The ImageNet competition provides a dataset of tagged images collected from the internet. The whole set is around 55GB and can take days or weeks to train.

In this lab we will look at using a pre-trained models provided by Caffe and its community, in particular the Caffenet model that was based on Alexnet architecture that won the 2012 ImageNet competition. In addition we will explore how the model parameters can be visualised using the Python interface.


## Caffe model zoo ##

Caffe provides a directory of official and community trained models. They can be found at the [model zoo](http://caffe.berkeleyvision.org/model_zoo.html) page. Today we will be using the pre-trained Caffenet model.


## Getting the pre-trained Caffenet model ##

All necessary files are included in `code/lab04/caffenet` folder apart from the pre-traind binary file which is too large to store in the repository (over 240MB). To get the file use:

```
wget "https://www.dropbox.com/s/srmy7h9bqy25vaq/bvlc_reference_caffenet.caffemodel?dl=0"  -O code/lab04/caffenet/bvlc_reference_caffenet.caffemodel
```

The model is more complex than our previous LeNet MNIST model using five stages of convolution and uses Local Response Normalization `LRN` layers after pooling which introduces lateral inhibition allowing more excited neurons to subdue the activty of others around it. A diagram of the model is shown below:

![Alexnet diagram](/static/img/intro_dl_sharc_dgx1/alexnet_small.png)

For more information about normalization in Neural Networks, [see this blog page](http://yeephycho.github.io/2016/08/03/Normalizations-in-neural-networks/).

## Exercise 4: Deploying the Caffenet model for classification ##

Using the previous python deployment code for our LeNet model which is now located in `code/lab04/mnist_deploy_sample.py`. Copy it to the root directory using:

```
cp code/lab04/mnist_deploy_sample.py caffenet_deploy.py
```

Now open the `caffenet_deploy.py` and modify it so that it can classify the `data/cat.jpg` image then print out the text labels with associated probabilities. See guidance below on the necessary steps:

* Add the following code at the top of your file that prevents pyplot from using X11 window:
  ```
  import matplotlib
  # Force matplotlib to not use any Xwindows backend.
  matplotlib.use('Agg')
  import matplotlib.pyplot as plt
  ```
* Use **only** the **CPU mode** for inferencing `caffe.set_mode_cpu()`. The model uses a deprecated layer that causes it to crash on the GPU.
* The model file is located at `code/lab04/caffenet/deploy.prototxt`
* The weights file is located at `code/lab04/caffenet/bvlc_reference_caffenet.caffemodel`
* The model takes an image input of `227 width x 227 height x 3 channels (RGB)`
  * The channel must be swapped from RGB to BGR.
      ```
      transformer.set_channel_swap('data', (2,1,0))
      ```
* The mean value file is located at `code/lab04/caffenet/ilsvrc_2012_mean.npy`
  * The mean file can be loaded like so:

    ```
    mu = np.load('code/lab04/caffenet/ilsvrc_2012_mean.npy')
    mu = mu.mean(1).mean(1)  # average over pixels to obtain the mean (BGR) pixel values
    ```

  * The transformer can take in the mean value before the image is transformed

    ```
    transformer.set_mean('data', mu)
    ```

* Text labels are located at `code/lab04/caffenet/synset_words.txt`, numpy can be used to load the set:
  ```
  labels = np.loadtxt(`code/lab04/caffenet/synset_words.txt`, str, delimiter='\t')
  ```
  This returns 1D list with the correct index to label mapping.
* The output blob is called `prob` instead of `loss`


Save and run your `caffenet_deploy.py` script. In the output results you should get the following at the end:

```
predicted class is: 281
output label: b'n02123045 tabby, tabby cat'
probabilities and labels:
[(0.3124361, "b'n02123045 tabby, tabby cat'"), (0.23797192, "b'n02123159 tiger cat'"), (0.12387263, "b'n02124075 Egyptian cat'"), (0.10075711, "b'n02119022 red fox, Vulpes vulpes'"), (0.070957027, "b'n02127052 lynx, catamount'")]
```

## Examining Blobs ##


The `net.blobs` object is an `OrderedDict` containing Blob objects with its name as the key. Add the following code to your `caffenet_deploy.py` script to print out all blob dimensions:

```
# for each layer, show the output shape
for layer_name, blob in net.blobs.items():
    print(layer_name + '\t' + str(blob.data.shape))

```

Where you will get:

```
data	(50, 3, 227, 227)
conv1	(50, 96, 55, 55)
pool1	(50, 96, 27, 27)
norm1	(50, 96, 27, 27)
conv2	(50, 256, 27, 27)
pool2	(50, 256, 13, 13)
norm2	(50, 256, 13, 13)
conv3	(50, 384, 13, 13)
conv4	(50, 384, 13, 13)
conv5	(50, 256, 13, 13)
pool5	(50, 256, 6, 6)
fc6	(50, 4096)
fc7	(50, 4096)
fc8	(50, 1000)
prob	(50, 1000)
```

## Examining model parameters ##


The parmeters `net.params` contain the weights and biases of the layers. To print our their dimension use the following code:

```
for layer_name, param in net.params.items():
    print(layer_name + '\t' + str(param[0].data.shape), str(param[1].data.shape))
```

Where you will get:

```
conv1	(96, 3, 11, 11) (96,)
conv2	(256, 48, 5, 5) (256,)
conv3	(384, 256, 3, 3) (384,)
conv4	(384, 192, 3, 3) (384,)
conv5	(256, 192, 3, 3) (256,)
fc6	(4096, 9216) (4096,)
fc7	(4096, 4096) (4096,)
fc8	(1000, 4096) (1000,)
```

## Visulisation of model parameters ##

We'll be using `matplotlib.pyplot` for visualising our model. Include the `import` code first:

```
import matplotlib.pyplot as plt
```

Let's try to visualise these parameters. Add the following function in to your script:

```
def vis_square(data):
    """Take an array of shape (n, height, width) or (n, height, width, 3)
       and visualize each (height, width) thing in a grid of size approx. sqrt(n) by sqrt(n)"""

    # normalize data for display
    data = (data - data.min()) / (data.max() - data.min())

    # force the number of filters to be square
    n = int(np.ceil(np.sqrt(data.shape[0])))
    padding = (((0, n ** 2 - data.shape[0]),
               (0, 1), (0, 1))                 # add some space between filters
               + ((0, 0),) * (data.ndim - 3))  # don't pad the last dimension (if there is one)
    data = np.pad(data, padding, mode='constant', constant_values=1)  # pad with ones (white)

    # tile the filters into an image
    data = data.reshape((n, n) + data.shape[1:]).transpose((0, 2, 1, 3) + tuple(range(4, data.ndim + 1)))
    data = data.reshape((n * data.shape[1], n * data.shape[3]) + data.shape[4:])

    plt.imshow(data); plt.axis('off')
    plt.savefig(figname+".png")
```

For `conv1` layer's parameter:

```
# the parameters are a list of [weights, biases]
filters = net.params['conv1'][0].data
vis_square(filters.transpose(0, 2, 3, 1), "conv1_params")
```

![Caffenet conv1 params](/static/img/intro_dl_sharc_dgx1/caffenet_conv1_param.png)

The first layer output, `conv1` (rectified responses of the filters above, first 36 only):

```
feat = net.blobs['conv1'].data[0, :36]
vis_square(feat, "conv1_output")
```

![Caffenet conv1 blob](/static/img/intro_dl_sharc_dgx1/caffenet_conv1_blob.png)


The fifth layer after pooling, `pool5`:

```
feat = net.blobs['pool5'].data[0]
vis_square(feat, "pool5_output")
```

![Caffenet pool5 blob](/static/img/intro_dl_sharc_dgx1/caffenet_pool5_blob.png)

The first fully connected layer, `fc6` (rectified). We show the output values and the histogram of the positive values:

```
feat = net.blobs['fc6'].data[0]
plt.subplot(2, 1, 1)
plt.plot(feat.flat)
plt.subplot(2, 1, 2)
_ = plt.hist(feat.flat[feat.flat > 0], bins=100)
plt.savefig("fc6_output.png")
```

The top plot shows the activation values of the output, the bottom plot shows the histogram of activation values.

![Caffenet fc6](/static/img/intro_dl_sharc_dgx1/caffenet_fc6_histogram.png)

The final probability output, `prob`:

```
feat = net.blobs['prob'].data[0]
plt.figure(figsize=(15, 3))
plt.plot(feat.flat)
plt.savefig("prob_distribution.png")
```

![Caffenet output prob](/static/img/intro_dl_sharc_dgx1/caffenet_prob_distribution.png)

Note the cluster of strong predictions; the labels are sorted semantically. The top peaks correspond to the top predicted labels, as shown above.

You can check the final code at `code/lab04/caffenet_deploy_visualise.py`.

## Extra: Fine-tuning and rehaping pre-train models ##

Caffe has provided tutorials for using an network trained on ImageNet data and repurposing it for classifying flickr style. To use the tutorial you will want to clone Caffe from github:

```
git clone https://github.com/BVLC/caffe.git
```

And follow the tutorial book in iPython format here: [https://github.com/BVLC/caffe/blob/master/examples/02-fine-tuning.ipynb](https://github.com/BVLC/caffe/blob/master/examples/02-fine-tuning.ipynb)

Another tutorial is also provided for re-shaping an existing network:
[https://github.com/BVLC/caffe/blob/master/examples/net_surgery.ipynb](https://github.com/BVLC/caffe/blob/master/examples/net_surgery.ipynb)

---

&#124; [Home](../../) &#124; [Lab03](../lab03) &#124; [Lab05](../lab05) &#124;
