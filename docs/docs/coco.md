# COCO

![COCO Dataset Examples](img/coco.jpg)

You can view the COCO minival leaderboard [here](https://sotabench.com/benchmarks/object-detection-on-coco-minival).

!!! Warning
    Object detection APIs in PyTorch are not very standardised across repositories, meaning that
    it may require a lot of glue to get them working with this evaluation procedure (which is based on torchvision).
    
    **For easier COCO integration with sotabench it is recommended to use the more general API [sotabencheval](https://sotagents.github.io/sotabench-eval/).**

## Getting Started

You'll need the following in the root of your repository:

- `sotabench.py` file - contains benchmarking logic; the server will run this on each commit
- `requirements.txt` file - Python dependencies to be installed before running `sotabench.py`
- `sotabench_setup.sh` *(optional)* - any advanced dependencies or setup, e.g. compilation

Once you connect your repository to [sotabench.com](https://www.sotabench.com), the platform 
will run your `sotabench.py` file whenever you commit to master. 

We now show how to write the `sotabench.py` file to evaluate a PyTorch object model with
the torchbench library, and to allow your results to be recorded and reported for the community.
 
## The COCO Evaluation Class

You can import the evaluation class from the following module:

``` python
from torchbench.object_detection import COCO
```

The `COCO` class contains several components used in the evaluation, such as the `dataset`:

``` python
COCO.dataset
# torchbench.datasets.coco.CocoDetection
```

And some default arguments used for evaluation (which can be overridden):

``` python
COCO.transforms
# <torchbench.object_detection.transforms.Compose at 0x7f60e9ffd0b8>

COCO.send_data_to_device
# <function torchbench.object_detection.coco.coco_data_to_device>

COCO.collate_fn
# <function torchbench.object_detection.coco.coco_collate_fn>

COCO.model_output_transform
# <function torchbench.object_detection.coco.coco_output_transform>
```

We will explain these different options shortly and how you can manipulate them to get the
evaluation logic to play nicely with your model.

An evaluation call - which performs evaluation, and if on the sotabench.com server, saves the results - 
looks like the following through the `benchmark()` method:

``` python
import torchvision
model = torchvision.models.detection.__dict__['maskrcnn_resnet50_fpn'](num_classes=91, pretrained=True)

COCO.benchmark(
    model=model,
    paper_model_name='Mask R-CNN (ResNet-50-FPN)',
    paper_arxiv_id='1703.06870'
)
```

These are the key arguments: the `model` which is a usually a `nn.Module` type object, but more generally,
is any method with a `forward` method that takes in input data and outputs predictions.
`paper_model_name` refers to the name of the model and `paper_arxiv_id` (optionally) refers to 
the paper from which the model originated. If these two arguments match a record paper result,
then sotabench.com will match your model with the paper and compare your code's results with the
reported results in the paper.

## A full `sotabench.py` example

Below shows an example for the [torchvision](https://github.com/pytorch/vision/tree/master/torchvision) 
repository benchmarking a Mask R-CNN model:


``` python
from torchbench.object_detection import COCO
from torchbench.utils import send_model_to_device
from torchbench.object_detection.transforms import Compose, ConvertCocoPolysToMask, ToTensor
import torchvision
import PIL

def coco_data_to_device(input, target, device: str = "cuda", non_blocking: bool = True):
    input = list(inp.to(device=device, non_blocking=non_blocking) for inp in input)
    target = [{k: v.to(device=device, non_blocking=non_blocking) for k, v in t.items()} for t in target]
    return input, target

def coco_collate_fn(batch):
    return tuple(zip(*batch))

def coco_output_transform(output, target):
    output = [{k: v.to("cpu") for k, v in t.items()} for t in output]
    return output, target

transforms = Compose([ConvertCocoPolysToMask(), ToTensor()])

model = torchvision.models.detection.__dict__['maskrcnn_resnet50_fpn'](num_classes=91, pretrained=True)

# Run the benchmark
COCO.benchmark(
    model=model,
    paper_model_name='Mask R-CNN (ResNet-50-FPN)',
    paper_arxiv_id='1703.06870',
    transforms=transforms,
    model_output_transform=coco_output_transform,
    send_data_to_device=coco_data_to_device,
    collate_fn=coco_collate_fn,
    batch_size=8,
    num_gpu=1
)
```

## `COCO.benchmark()` Arguments

The source code for the COCO evaluation method can be found [here](https://github.com/sotagents/torchbench/blob/develop/torchbench/object_detection/coco.py).
We now explain each argument.

### model

**a PyTorch module, (e.g. a ``nn.Module`` object), that takes in COCO data and outputs detections.**

For example, from the torchvision repository:

``` python
import torchvision
model = torchvision.models.detection.__dict__['maskrcnn_resnet50_fpn'](num_classes=91, pretrained=True)
```

### model_description

**(str, optional): Optional model description.**

For example:

``` python
model_description = 'Using ported TensorFlow weights'
```

### input_transform

**Composing the transforms used to transform the input data (the images), e.g. 
resizing (e.g ``transforms.Resize``), center cropping, to tensor transformations and normalization.**

For example:

``` python
import torchvision.transforms as transforms
input_transform = transforms.Compose([
    transforms.Resize(512, PIL.Image.BICUBIC),
    transforms.ToTensor(),
])
```

### target_transform

**Composing the transforms used to transform the target data**

### transforms

**Composing the transforms used to transform the input data (the images) and the target data (the labels) 
in a dual fashion - for example resizing the pair of data jointly.** 

Below shows an example; note the
fact that the `__call__` takes in two arguments and returns two arguments (ordinary `torchvision` transforms
return one result).

``` python
from torchvision.transforms import functional as F

class Compose(object):
    def __init__(self, transforms):
        self.transforms = transforms

    def __call__(self, image, target):
        for t in self.transforms:
            image, target = t(image, target)
        return image, target

class ToTensor(object):
    def __call__(self, image, target):
        image = F.to_tensor(image)
        return image, target

class ImageResize(object):
    def __init__(self, resize_shape):
        self.resize_shape = resize_shape

    def __call__(self, image, target):
        image = F.resize(image, self.resize_shape)
        return image, target
        
transforms = Compose([ImageResize((512, 512)), ToTensor()])
```

Note that the default transforms are:

``` python
from torchbench.object_detection.utils import Compose, ConvertCocoPolysToMask, ToTensor
transforms = Compose([ConvertCocoPolysToMask(), ToTensor()])
```

Where `ConvertCocoPolysToMask` is from the torchvision reference implementation to transform
the inputs to the right format to be entered into the model. You can pass whatever transforms
you need to make the dataset work with your model.

### model_output_transform

**(callable, optional): An optional function
                that takes in model output (after being passed through your
                ``model`` forward pass) and transforms it. Afterwards, the
                output will be passed into an evaluation function.**
                
The model output transform is a function that you can pass in to transform the model output
after the data has been passed into the model. This is useful if you have to do further 
processing steps after inference to get the predictions in the right format for evaluation.

The model evaluation for each batch is as follows from [utils.py](https://github.com/sotagents/torchbench/blob/db9fbdf5567350b8316336ca4f3fd27a04999347/torchbench/object_detection/utils.py#L189) 
are:

``` python
with torch.no_grad():
    for i, (input, target) in enumerate(iterator):
        input, target = send_data_to_device(input, target, device=device)
        original_output = model(input)
        output, target = model_output_transform(original_output, target)
        result = {
            tar["image_id"].item(): out for tar, out in zip(target, output)
        }
        coco_evaluator.update(result)
```

We can see the `model_output_transform` in use, and the fact that the `output` is then 
transformed to be a dictionary with image_ids as keys and output as values. 

The expected output of `model_output_transform` is a list of dictionaries (length = batch_size),
where each dictionary contains keys for 'boxes', 'labels', 'scores', 'masks', and each value is 
of the `torch.tensor` type.

The expected output of `result` is  converted to a dictionary with keys as the image ids, and 
values as a dictionary with the predictions (boxes, labels, scores, ... as keys).

### collate_fn

**How the dataset is collated - an optional callable passed into the DataLoader**

As an example the default collate function is:

``` python
def coco_collate_fn(batch):
    return tuple(zip(*batch))
```

### send_data_to_device

**An optional function specifying how the model is sent to a device**

As an example the COCO default is:

``` python
def coco_data_to_device(input, target, device: str = "cuda", non_blocking: bool = True):
    input = list(inp.to(device=device, non_blocking=non_blocking) for inp in input)
    target = [{k: v.to(device=device, non_blocking=non_blocking) for k, v in t.items()} for t in target]
    return input, target
```


### data_root

**data_root (str): The location of the COCO dataset - change this
                parameter when evaluating locally if your COCO data is
                located in a different folder (or alternatively if you want to
                download to an alternative location).**
                
Note that this parameter will be overriden when the evaluation is performed on the server,
so it is solely for your local use.

### num_workers

**num_workers (int): The number of workers to use for the DataLoader.**

### batch_size

**batch_size (int) : The batch_size to use for evaluation; if you get
                memory errors, then reduce this (half each time) until your
                model fits onto the GPU.**
                
### paper_model_name

**paper_model_name (str, optional): The name of the model from the
    paper - if you want to link your build to a machine learning
    paper. See the COCO benchmark page for model names,
    https://www.sotabench.com/benchmark/coco-minival, e.g. on the paper
    leaderboard tab.**
    
### paper_arxiv_id
    
**paper_arxiv_id (str, optional): Optional linking to ArXiv if you
    want to link to papers on the leaderboard; put in the
    corresponding paper's ArXiv ID, e.g. '1611.05431'.**

### paper_pwc_id

**paper_pwc_id (str, optional): Optional linking to Papers With Code;
    put in the corresponding papers with code URL slug, e.g.
    'u-gat-it-unsupervised-generative-attentional'**
    
### paper_results

**paper_results (dict, optional) : If the paper you are reproducing
    does not have model results on sotabench.com, you can specify
    the paper results yourself through this argument, where keys
    are metric names, values are metric values. e.g::**

``` python
{'box AP': 0.349, 'AP50': 0.592, ...}.
```

Ensure that the metric names match those on the sotabench
leaderboard - for COCO it should be 'box AP', 'AP50',
'AP75', 'APS', 'APM', 'APL'

### pytorch_hub_url

**pytorch_hub_url (str, optional): Optional linking to PyTorch Hub
    url if your model is linked there; e.g:
    'nvidia_deeplearningexamples_waveglow'.**

## Need More Help?

Head on over to the [Computer Vision](https://forum.sotabench.com/c/cv) section of the sotabench
forums if you have any questions or difficulties.
