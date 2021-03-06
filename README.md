# ONNX_custom_layer
Guide on how to convert custom PyTorch layers when using ONNX.

##### _ADDING CUSTOM LAYERS_
For custom layers, basically, if we could trace the model/graph in pytorch using torch.jit, we could export the model to ONNX models.
Nodes in ONNX graphs do not need registering, they are simply dicts that store information. Refer to: https://pytorch.org/docs/stable/onnx.html and https://blog.csdn.net/u012436149/article/details/78829329 
for more information. Also, you can refer to custom_example.py, which implements a ReLU and should export a custom layer (MyReLU) as following.

Basically, first thing you need to do is, re-write your custom layer to turn it from a legacy function to non-legacy function (which
means your custom layer is now a nn.module object, so it could be traced, if you understand Chinese, https://blog.csdn.net/u012436149/article/details/78829329). Then, modify symbolic function, what inputs to feed and what parameters to pass. Here, we do not modify symbolic.py, but keep the symbolic function inside the class, to make it more trackable. When you modify symbolic.py, or the symbolic function within the custom layer, refer to the example.

##### TO-DO
Btw, symbolic.py is a hugely useful file when converting models. So I uploaded my own version of it. Actually, ONNX supports multiple
libraries, so they are trying to support as many parameters from as many libraries (tensorflow, PyTorch, Caffe and etc.) as possible. But
if you are doing a specific conversion (for me it is Pytorch2Caffe), you do not need that many generic support. ONNX is trying to add layers
in order to support more parameters (use dynamic constant layer to store the parameter values and a "empty" layer that denotes the parameter
name), which is good, but for a specific task, quite cumbersome. Try comment out some sections in symbolic.py for better support. Do not worry
about registry stuff, every converted op is simply a dict. Just store enough useful info in the dict would be good.
- [ ] Write a blog post about symbolic.py and ONNX custom layer conversion details

```python
class MyReLUFunction(Function):

    @staticmethod
    def symbolic(g, input):
        return g.op('MyReLU', input)

    @staticmethod
    def forward(ctx, input):
        ctx.input = ctx
        return input.clamp(0)

    @staticmethod
    def backward(ctx, grad_output):
        grad_input = grad_output.clone()
        grad_input.masked_fill_(ctx.input < 0, 0)
        return grad_input

class MyReLU(nn.Module):

    def forward(self, input):
        return MyReLUFunction.apply(input)
```

```shell
graph(%0 : Float(10, 1, 3, 3)                                                                           
      %1 : Float(1, 1, 3, 3)                                                                            
      %2 : Float(1)) {                                                                                  
  %3 : Float(10, 1, 1, 1) = onnx::Conv[dilations=[1, 1], group=1, kernel_shape=[3, 3], pads=[0, 0, 0, 0]
, strides=[1, 1]](%0, %1, %2), scope: Sequential/Conv2d[0]                                              
  %4 : Float(10, 1, 1, 1) = onnx::MyReLU(%3), scope: Sequential/MyReLU[1]                               
  return (%4);                                                                                          
}            
```
