(prototype) PyTorch BackendConfig Tutorial
==========================================
**Author**: `Andrew Or <https://github.com/andrewor14>`_

The BackendConfig API enables developers to integrate their backends
with PyTorch quantization. It is currently only supported in FX graph
mode quantization, but support may be extended to other modes of
quantization in the future. In this tutorial, we will demonstrate how to
use this API to customize quantization support for specific backends.
For more information on the motivation and implementation details behind
BackendConfig, please refer to this
`README <https://github.com/pytorch/pytorch/tree/master/torch/ao/quantization/backend_config>`__.

BackendConfig API Specification
-------------------------------

On a high level, BackendConfig specifies the quantization behavior for
each supported operator pattern (e.g. linear, conv-bn-relu). The API is
broken down into the following class hierarchy:

- `BackendConfig <https://pytorch.org/docs/stable/generated/torch.ao.quantization.backend_config.BackendConfig.html>`__:
  The main class to be passed to prepare and convert functions.
- `BackendPatternConfig <https://pytorch.org/docs/stable/generated/torch.ao.quantization.backend_config.BackendPatternConfig.html>`__:
  Config object that specifies quantization behavior for a given
  operator pattern. Each BackendConfig consists of many of these.
- `DTypeConfig <https://pytorch.org/docs/stable/generated/torch.ao.quantization.backend_config.DTypeConfig.html>`__:
  Config object that specifies the supported data types passed as
  arguments to quantize ops in the reference model spec, for input
  and output activations, weights, and biases. This object also
  optionally specifies constraints associated with the data types.
  Each BackendPatternConfig consists of one or more of these.
- `DTypeWithConstraints <https://pytorch.org/docs/stable/generated/torch.ao.quantization.backend_config.DTypeWithConstraints.html>`__:
  Constraints imposed by the backend on the quantization parameters
  (scale and zero point) and ranges when quantizing to a given data
  type. Each DTypeConfig consists of many of these.

The pattern specified in BackendPatternConfig follows the format
described `here <https://github.com/pytorch/pytorch/blob/master/torch/ao/quantization/backend_config/README.md#pattern-specification>`__.

BackendPatternConfig Specification
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

set_observation_type
^^^^^^^^^^^^^^^^^^^^

Observation type here refers to how observers (or quant-dequant ops)
will be placed in the graph. This is used to produce the desired
reference patterns understood by the backend. Weighted ops such as
linear and conv require different observers (or quantization parameters
passed to quantize ops in the reference model) for the input and the
output (see `ObservationType <https://pytorch.org/docs/stable/generated/torch.ao.quantization.backend_config.ObservationType.html>`__).

Note: This will be renamed in the near future, since we will soon insert
QuantDeQuantStubs with observers (and fake quantizes) attached instead
of observers themselves.

set_dtype_configs / add_type_config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each operator pattern may support one or more sets of
input/output/weight/bias data types, and each set may have their own
constraints. These requirements are captured in DTypeConfigs, which will
be described in more detail in the next section.

set_root_module / set_reference_quantized_module
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When we construct the reference quantized model during the convert
phase, the root modules (e.g. ``torch.nn.Linear`` for
``torch.ao.nn.intrinsic.LinearReLU``) will be swapped to the
corresponding reference quantized modules (e.g.
``torch.ao.nn.reference.quantized.Linear``). This allows custom backends
to specify custom reference quantized module implementations to match
the numerics of their lowered operators. Since this is a one-to-one
mapping, both the root module and the reference quantized module must be
specified in the same BackendPatternConfig in order for the conversion
to take place.

set_fuser_method
^^^^^^^^^^^^^^^^

As an optimization, operator patterns such as (``torch.nn.Linear``,
``torch.nn.ReLU``) may be fused into ``nni.LinearReLU``.
``set_fuser_method`` specifies the function through which this is
performed. The first argument of this function is ``is_qat``, and the
rest of the arguments are the items in the tuple pattern, e.g. the fuser
method for the above pattern will have three arguments, ``is_qat``,
``linear``, and ``relu``. See `this
example <https://gist.github.com/jerryzh168/8bea7180a8ba3c279f2c9b050f2a69a6>`__
for a slightly more complicated usage.

set_fused_module
^^^^^^^^^^^^^^^^

This is used to identify fused weighted modules (e.g.
``torch.ao.nn.intrinsic.LinearReLU``) that need to be converted to
reference quantized modules.

Data Type Restrictions
~~~~~~~~~~~~~~~~~~~~~~

Each DTypeConfig attached to a BackendPatternConfig represents a set of
supported data types passed as arguments to quantize ops in the reference
model spec. For example, consider the following reference model::

  quant1 - [dequant1 - fp32_linear - quant2] - dequant2

The pattern in the square brackets refers to the reference pattern of
statically quantized linear. Setting the input dtype as `torch.quint8`
in the DTypeConfig means we pass in `torch.quint8` as the dtype argument
to the first quantize op (quant1). Similarly, setting the output dtype as
`torch.quint8` means we pass in `torch.quint8` as the dtype argument to
the second quantize op (quant2).

Note that the dtype here does not refer to the interface dtypes of the
op. For example, the "input dtype" here is not the dtype of the input
tensor passed to the quantized linear op. Though it can still be the
same as the interface dtype, this is not always the case, e.g. the
interface dtype is fp32 in dynamic quantization but the "input dtype"
specified in the DTypeConfig would still be quint8. The semantics of
dtypes here are the same as the semantics of the dtypes specified in
the observers.

These dtypes are matched against the ones specified in the user’s
QConfig. If there is a match, and the QConfig satisfies the constraints
specified in the DTypeConfig (if any), then we will quantize the given
pattern using this DTypeConfig. Otherwise, the QConfig is ignored and
the pattern will not be quantized.

There are two ways of specifying ``input_dtype``, ``output_dtype``, and
``weight_dtype``, as simple ``torch.dtype`` or as
``DTypeWithConstraints``. The constraints currently supported are:

- **quant_min_lower_bound** and **quant_max_upper_bound**: Lower and upper
  bounds for the minimum and maximum quantized values respectively. If the
  QConfig’s ``quant_min`` and ``quant_max`` fall outside this range, then
  the QConfig will be ignored.
- **scale_min_lower_bound** and **scale_max_upper_bound**: Lower and
  upper bounds for the minimum and  aximum scale values respectively. If
  the QConfig’s minimum scale value (currently exposed as ``eps``) falls
  below the lower bound, then the QConfig will be ignored. Note that the
  upper bound is currently not enforced.
- **scale_exact_match** and **zero_point_exact_match**: Exact match
  requirements for scale and zero point, to be used for operators with
  fixed quantization parameters such as sigmoid and tanh. If the observer
  specified in the QConfig is neither ``FixedQParamsObserver`` nor
  ``FixedQParamsFakeQuantize``, or if the quantization parameters don't
  match, then the QConfig will be ignored.

End-to-End Example
------------------

Suppose we are a backend developer and we wish to integrate our backend
with PyTorch's quantization APIs. Our backend consists of two ops only:
quantized linear and quantized conv-relu. In this section, we will walk
through how to achieve this by quantizing an example model using a custom
BackendConfig through `prepare_fx` and `convert_fx`.

.. code:: ipython3

    import torch
    from torch.ao.quantization import (
        default_weight_observer,
        get_default_qconfig_mapping,
        MinMaxObserver,
        QConfig,
        QConfigMapping,
    )
    from torch.ao.quantization.backend_config import (
        BackendConfig,
        BackendPatternConfig,
        DTypeConfig,
        DTypeWithConstraints,
        ObservationType,
    )
    from torch.ao.quantization.quantize_fx import prepare_fx, convert_fx

For quantized linear, suppose our backend expects the reference pattern
`[dequant - fp32_linear - quant]` and lowers it into a single quantized
linear op. The way to achieve this is to first insert quant-dequant ops
before and after the float linear op, such that we produce the following
reference model::

  quant1 - [dequant1 - fp32_linear - quant2] - dequant2

Here we specify using different observers (will be renamed) for the input
and output for the linear op, so the quantization params passed to the two
quantize ops (quant1 and quant2) will be different. This is commonly the
case for weighted ops like linear and conv.

The input dtype specified in the DTypeConfig will be passed as the dtype
argument to quant1, while the output dtype will be passed as the dtype
argument to quant2. If the output dtype is fp32, as in the case of dynamic
quantization, then the output quant-dequant pair will not be inserted.
This example also shows how to specify restrictions on quantization and
scale ranges on a particular dtype.

.. code:: ipython3

    quint8_with_constraints = DTypeWithConstraints(
        dtype=torch.quint8,
        quant_min_lower_bound=0,
        quant_max_upper_bound=255,
        scale_min_lower_bound=2 ** -12,
    )
    
    # Specify the dtypes passed to the quantized ops in the reference model spec
    weighted_int8_dtype_config = DTypeConfig(
        input_dtype=quint8_with_constraints,
        output_dtype=quint8_with_constraints,
        weight_dtype=torch.qint8,
        bias_dtype=torch.float)

    linear_config = BackendPatternConfig() \
        .set_pattern(torch.nn.Linear) \
        .set_observation_type(ObservationType.OUTPUT_USE_DIFFERENT_OBSERVER_AS_INPUT) \
        .add_dtype_config(weighted_int8_dtype_config) \
        .set_root_module(torch.nn.Linear) \
        .set_qat_module(torch.nn.qat.Linear) \
        .set_reference_quantized_module(torch.ao.nn.quantized.reference.Linear)

For quantized conv-relu, the observation type and DTypeConfig settings
are the same, since we wish to produce the following reference model,
where the reference pattern in the square brackets will be lowered into
a single quantized conv-relu op::

  quant1 - [dequant1 - fp32_conv_relu - quant2] - dequant2

However, first we need to fuse the conv and relu ops into a single
conv-relu op (`fp32_conv_relu`), and then quantize this op similar to
how the linear op is quantized. Thus, we need two BackendPatternConfigs
to support this op, one for fusion and one for quantization:

.. code:: ipython3

   def fuse_conv2d_relu(is_qat, conv, relu):
       """Return a fused ConvReLU2d from individual conv and relu modules."""
       return torch.ao.nn.intrinsic.ConvReLU2d(conv, relu)
    
    # For fusing Conv2d + ReLU into ConvReLU2d
    # No need to set observation type and dtype config here, since we are not
    # inserting quant-dequant ops in this step yet
    conv_relu_config = BackendPatternConfig() \
        .set_pattern((torch.nn.Conv2d, torch.nn.ReLU)) \
        .set_fused_module(torch.ao.nn.intrinsic.ConvReLU2d) \
        .set_fuser_method(fuse_conv2d_relu)
    
    # For quantizing ConvReLU2d
    fused_conv_relu_config = BackendPatternConfig() \
        .set_pattern(torch.ao.nn.intrinsic.ConvReLU2d) \
        .set_observation_type(ObservationType.OUTPUT_USE_DIFFERENT_OBSERVER_AS_INPUT) \
        .add_dtype_config(weighted_int8_dtype_config) \
        .set_root_module(torch.nn.Conv2d) \
        .set_qat_module(torch.ao.nn.intrinsic.qat.ConvReLU2d) \
        .set_reference_quantized_module(torch.ao.nn.quantized.reference.Conv2d)

Now we have all the necessary pieces, so we go ahead and define our
BackendConfig and test it out on an example model. Here we see that
both linear and conv-relu are quantized.
    
.. code:: ipython3

    backend_config = BackendConfig("my_backend") \
        .set_backend_pattern_config(linear_config) \
        .set_backend_pattern_config(conv_relu_config) \
        .set_backend_pattern_config(fused_conv_relu_config)

.. code:: ipython3

    # ====================
    #  Example user model
    # ====================
    
    class MyModel(torch.nn.Module):
        def __init__(self, use_bn: bool):
            super().__init__()
            self.linear = torch.nn.Linear(10, 3)
            self.conv = torch.nn.Conv2d(3, 3, 3)
            self.bn = torch.nn.BatchNorm2d(3)
            self.relu = torch.nn.ReLU()
            self.sigmoid = torch.nn.Sigmoid()
            self.use_bn = use_bn
    
        def forward(self, x):
            x = self.linear(x)
            x = self.conv(x)
            if self.use_bn:
                x = self.bn(x)
            x = self.relu(x)
            x = self.sigmoid(x)
            return x

.. code:: ipython3

    # =======================
    #  Custom QConfigMapping
    # =======================
    
    # Define a QConfig that satisfies the constraints specified in DTypeConfig
    # Note: Here we use a quant_max of 127, but this could be up to 255 (see `quint8_with_constraints`)
    activation_observer = MinMaxObserver.with_args(quant_min=0, quant_max=127, eps=2 ** -12)
    qconfig = QConfig(activation=activation_observer, weight=default_weight_observer)

    # Note: All individual items of a fused pattern, e.g. Conv2d and ReLU in
    # (Conv2d, ReLU), must have the same QConfig
    qconfig_mapping = QConfigMapping() \
        .set_object_type(torch.nn.Linear, qconfig) \
        .set_object_type(torch.nn.Conv2d, qconfig) \
        .set_object_type(torch.nn.BatchNorm2d, qconfig) \
        .set_object_type(torch.nn.ReLU, qconfig)

.. code:: ipython3

    # =====================
    #  Prepare and Convert
    # =====================
    
    example_inputs = (torch.rand(1, 3, 10, 10, dtype=torch.float),)
    model = MyModel(use_bn=False)
    prepared = prepare_fx(model, qconfig_mapping, example_inputs, backend_config=backend_config)
    prepared(*example_inputs)  # calibrate
    converted = convert_fx(prepared, backend_config=backend_config)

.. parsed-literal::

    >>> print(converted)

    GraphModule(
      (linear): QuantizedLinear(in_features=10, out_features=3, scale=0.012136868201196194, zero_point=67, qscheme=torch.per_tensor_affine)
      (conv): QuantizedConvReLU2d(3, 3, kernel_size=(3, 3), stride=(1, 1), scale=0.0029353597201406956, zero_point=0)
      (sigmoid): Sigmoid()
    )
    
    def forward(self, x):
        linear_input_scale_0 = self.linear_input_scale_0
        linear_input_zero_point_0 = self.linear_input_zero_point_0
        quantize_per_tensor = torch.quantize_per_tensor(x, linear_input_scale_0, linear_input_zero_point_0, torch.quint8);  x = linear_input_scale_0 = linear_input_zero_point_0 = None
        linear = self.linear(quantize_per_tensor);  quantize_per_tensor = None
        conv = self.conv(linear);  linear = None
        dequantize_2 = conv.dequantize();  conv = None
        sigmoid = self.sigmoid(dequantize_2);  dequantize_2 = None
        return sigmoid

As an experiment, here we modify the model to use conv-bn-relu
instead of conv-relu, but use the same BackendConfig, which doesn't
know how to quantize conv-bn-relu. As a result, only linear is
quantized, but conv-bn-relu is neither fused nor quantized.

.. code:: ipython3

    # ================================================
    #  Prepare and Convert (only linear is quantized)
    # ================================================
    
    example_inputs = (torch.rand(1, 3, 10, 10, dtype=torch.float),)
    model = MyModel(use_bn=True)
    prepared = prepare_fx(model, qconfig_mapping, example_inputs, backend_config=backend_config)
    prepared(*example_inputs)  # calibrate
    converted = convert_fx(prepared, backend_config=backend_config)

.. parsed-literal::

    >>> print(converted)

    GraphModule(
      (linear): QuantizedLinear(in_features=10, out_features=3, scale=0.015307803638279438, zero_point=95, qscheme=torch.per_tensor_affine)
      (conv): Conv2d(3, 3, kernel_size=(3, 3), stride=(1, 1))
      (bn): BatchNorm2d(3, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU()
      (sigmoid): Sigmoid()
    )
    
    def forward(self, x):
        linear_input_scale_0 = self.linear_input_scale_0
        linear_input_zero_point_0 = self.linear_input_zero_point_0
        quantize_per_tensor = torch.quantize_per_tensor(x, linear_input_scale_0, linear_input_zero_point_0, torch.quint8);  x = linear_input_scale_0 = linear_input_zero_point_0 = None
        linear = self.linear(quantize_per_tensor);  quantize_per_tensor = None
        dequantize_1 = linear.dequantize();  linear = None
        conv = self.conv(dequantize_1);  dequantize_1 = None
        bn = self.bn(conv);  conv = None
        relu = self.relu(bn);  bn = None
        sigmoid = self.sigmoid(relu);  relu = None
        return sigmoid

As another experiment, here we use the default QConfigMapping that
doesn't satisfy the dtype constraints specified in the backend. As
a result, nothing is quantized since the QConfigs are simply ignored.

.. code:: ipython3

    # ============================================
    #  Prepare and Convert (nothing is quantized)
    # ============================================
    
    example_inputs = (torch.rand(1, 3, 10, 10, dtype=torch.float),)
    model = MyModel(use_bn=True)
    prepared = prepare_fx(model, get_default_qconfig_mapping(), example_inputs, backend_config=backend_config)
    prepared(*example_inputs)  # calibrate
    converted = convert_fx(prepared, backend_config=backend_config)

.. parsed-literal::

    >>> print(converted)

    GraphModule(
      (linear): Linear(in_features=10, out_features=3, bias=True)
      (conv): Conv2d(3, 3, kernel_size=(3, 3), stride=(1, 1))
      (bn): BatchNorm2d(3, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU()
      (sigmoid): Sigmoid()
    )
    
    def forward(self, x):
        linear = self.linear(x);  x = None
        conv = self.conv(linear);  linear = None
        bn = self.bn(conv);  conv = None
        relu = self.relu(bn);  bn = None
        sigmoid = self.sigmoid(relu);  relu = None
        return sigmoid


Built-in BackendConfigs
-----------------------

PyTorch quantization supports a few built-in native BackendConfigs under
the ``torch.ao.quantization.backend_config`` namespace:

- `get_fbgemm_backend_config <https://github.com/pytorch/pytorch/blob/master/torch/ao/quantization/backend_config/fbgemm.py>`__:
  for server target settings
- `get_qnnpack_backend_config <https://github.com/pytorch/pytorch/blob/master/torch/ao/quantization/backend_config/qnnpack.py>`__:
  for mobile and edge device target settings, also supports XNNPACK
  quantized ops
- `get_native_backend_config <https://github.com/pytorch/pytorch/blob/master/torch/ao/quantization/backend_config/native.py>`__
  (default): a BackendConfig that supports a union of the operator
  patterns supported in the FBGEMM and QNNPACK BackendConfigs

There are also other BackendConfigs under development (e.g. for
TensorRT and x86), but these are still mostly experimental at the
moment. If the user wishes to integrate a new, custom backend with
PyTorch’s quantization API, they may define their own BackendConfigs
using the same set of APIs used to define the natively supported
ones as in the example above.

Further Reading
---------------

How BackendConfig is used in FX graph mode quantization:
https://github.com/pytorch/pytorch/blob/master/torch/ao/quantization/fx/README.md

Motivation and implementation details behind BackendConfig:
https://github.com/pytorch/pytorch/blob/master/torch/ao/quantization/backend_config/README.md

Early design of BackendConfig:
https://github.com/pytorch/rfcs/blob/master/RFC-0019-Extending-PyTorch-Quantization-to-Custom-Backends.md
