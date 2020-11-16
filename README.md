# pytorch-profiler

> Measures the time, number of estimated flop and parameters of each module in a PyTorch Model.

[![PyPI version](https://badge.fury.io/py/pytorch-profiler.svg)](https://pypi.org/project/pytorch-profiler/)

The pytorch-profiler prints the model graph with the measured profile attached to each module, and shows how time, flops and parameters are spent in a PyTorch model and which modules or layers could be the bottleneck. It also outputs the names of the top k modules in terms of aggregated time, flops, and parameters at depth l with k and l specified by the user.

The flops estimation is partly inspired by [ptflops](https://github.com/sovrasov/flops-counter.pytorch) with the major difference being that pytorch-profiler captures ```torch.nn.functional``` invoked in a module to estimate the flops, thus allowing customized modules in the module (e.g. ```ParallelTransformerLayerworks, ParallelSelfAttention, RowParallelLinear, etc.``` in [Megatron-LM](https://github.com/NVIDIA/Megatron-LM)). pytorch-profiler also supports flops computation at module level (for RNNs).

Below is an example output for LeNet5:
<!-- ![](header.png) -->


```
LeNet5(
  61.71 k, 100.000% Params, 429.248 KMac, 100.000% MACs, 1.201 ms, 100.00% time, 0.001 TFLOPS,
  (feature_extractor): Sequential(
    50.69 k, 82.151% Params, 418.328 KMac, 97.456% MACs, 984.669 us, 81.98% time, 0.001 TFLOPS,
    (0): Conv2d(156, 0.253% Params, 122.304 KMac, 28.493% MACs, 273.466 us, 22.77% time, 0.001 TFLOPS, 1, 6, kernel_size=(5, 5), stride=(1, 1))
    (1): Tanh(0, 0.000% Params, 0 Mac, 0.000% MACs, 33.379 us, 2.78% time, 0.0 TFLOPS, )
    (2): AvgPool2d(0, 0.000% Params, 4.704 KMac, 1.096% MACs, 41.723 us, 3.47% time, 0.0 TFLOPS, kernel_size=2, stride=2, padding=0)
    (3): Conv2d(2.42 k, 3.915% Params, 241.6 KMac, 56.284% MACs, 218.63 us, 18.20% time, 0.002 TFLOPS, 6, 16, kernel_size=(5, 5), stride=(1, 1))
    (4): Tanh(0, 0.000% Params, 0 Mac, 0.000% MACs, 26.464 us, 2.20% time, 0.0 TFLOPS, )
    (5): AvgPool2d(0, 0.000% Params, 1.6 KMac, 0.373% MACs, 39.816 us, 3.31% time, 0.0 TFLOPS, kernel_size=2, stride=2, padding=0)
    (6): Conv2d(48.12 k, 77.983% Params, 48.12 KMac, 11.210% MACs, 241.041 us, 20.07% time, 0.0 TFLOPS, 16, 120, kernel_size=(5, 5), stride=(1, 1))
    (7): Tanh(0, 0.000% Params, 0 Mac, 0.000% MACs, 23.365 us, 1.95% time, 0.0 TFLOPS, )
  )
  (classifier): Sequential(
    11.01 k, 17.849% Params, 10.92 KMac, 2.544% MACs, 154.018 us, 12.82% time, 0.0 TFLOPS,
    (0): Linear(10.16 k, 16.472% Params, 10.08 KMac, 2.348% MACs, 59.128 us, 4.92% time, 0.0 TFLOPS, in_features=120, out_features=84, bias=True)
    (1): Tanh(0, 0.000% Params, 0 Mac, 0.000% MACs, 18.597 us, 1.55% time, 0.0 TFLOPS, )
    (2): Linear(850, 1.377% Params, 840 Mac, 0.196% MACs, 41.485 us, 3.45% time, 0.0 TFLOPS, in_features=84, out_features=10, bias=True)
  )
)
Top 3 modules in flops at depth 2 are {'Conv2d': '412.02 KMac', 'Linear': '10.92 KMac', 'AvgPool2d': '6.3 KMac'}
Top 3 modules in params at depth 2 are {'Conv2d': '50.69 k', 'Linear': '11.01 k', 'Tanh': '0'}
Top 3 modules in time at depth 2 are {'Conv2d': '733.14 us', 'Tanh': '101.8 us', 'Linear': '100.61 us'}
Number of multiply-adds:        429.25 KMac
Number of parameters:           61.71 k
```


For models running on multi-node or multi-gpu runs, only the model parallelism affects the number of flops and parameters (e.g. ```--model-parallel-size``` in [Megatron-LM](https://github.com/NVIDIA/Megatron-LM)), i.e., model_parallel_size * flops = total_flops, model_parallel_size * parameters = total_parameters. The number of gpus or nodes does not affect the output profile.

## Installation

From PyPI:
```bash
pip install pytorch-profiler
```

From this repository:
```bash
pip install --upgrade git+https://github.com/sovrasov/pytorch-profiler.git
```

## Usage

```python
import torchvision.models as models
import torch
from pytorch_profiler import get_model_profile

with torch.cuda.device(0):
    mod = models.alexnet()
    macs, params = get_model_profile(mod, # model
                                     (3, 224, 224), # input shape or input to the input_constructor
                                     input_constructor=None, # if specified, a constructor taking the the parameter before is used as input to the model
                                     print_profile=True, # print model graph with the profile annotated
                                     print_aggregated_profile=True, # print aggregated profile for top modules
                                     depth=-1, # depth into the nested modules with -1 being the inner most modules
                                     top_num=3, # the number of top modules to print aggregated profile
                                     warm_up=10, # the number of warm-ups before measuring the time of each module
                                     as_strings=True, # print raw numbers (e.g. 1000) or strings (e.g. 1k)
                                     ignore_modules=None) # the list of modules to ignore in the profiling
    print('{:<30}  {:<8}'.format('Number of multiply-adds: ', macs))
    print('{:<30}  {:<8}'.format('Number of parameters: ', params))
```

<!-- For more examples and usage, please refer to the [Wiki][wiki]._ -->


## Release History
<!--
* 0.1.1
    * CHANGE: Update docs (module code remains unchanged) -->
* 0.1.0
    * The first proper release

## License

Distributed under the MIT license. See ``LICENSE`` for more information.

## Contributing

1. Fork it (<https://github.com/cli99/pytorch-profiler>)
2. Create your feature branch (`git checkout -b feature/fooBar`)
3. Commit your changes (`git commit -am 'Add some fooBar'`)
4. Push to the branch (`git push origin feature/fooBar`)
5. Create a new Pull Request

<!-- Markdown link & img dfn's -->
[npm-image]: https://img.shields.io/npm/v/datadog-metrics.svg?style=flat-square
[npm-url]: https://npmjs.org/package/datadog-metrics
[npm-downloads]: https://img.shields.io/npm/dm/datadog-metrics.svg?style=flat-square
[travis-image]: https://img.shields.io/travis/dbader/node-datadog-metrics/master.svg?style=flat-square
[travis-url]: https://travis-ci.org/dbader/node-datadog-metrics
[wiki]: https://github.com/yourname/yourproject/wiki
