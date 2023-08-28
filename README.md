# fastfeedforward
A repository for fast feedforward (FFF) networks.
Fast feedforward layers can be used in place of vanilla feedforward and mixture-of-expert layers, offering inference time that grows only logarithmically in the training width of the layer.

More information can be found in the paper "Fast Feedforward Networks" by Belcak and Wattenhofer, 2023.

## Quickstart
1. Install the package.
```
pip install fastfeedforward
```

2. Import the `FFF` layer implementation.
```
from fastfeedforward import FFF
```

3. Use `FFF` in place of feedforward or mixture-of-experts layers, e.g. instead of
```
my_ff = torch.nn.Sequential(
    torch.nn.Linear(input_width, hidden_width, bias=True),
    torch.nn.ReLU(),
    torch.nn.Dropout(p=dropout),
    torch.nn.Linear(hidden_width, output_width, bias=True)
)
```
use
```
depth = ... # your choice of the FFF depth
leaf_width = math.ceil(hidden_width / 2**depth)
my_ff = FFF(
    input_width,
    leaf_width,
    output_width,
    depth,
    activation=torch.nn.ReLU(),
    dropout=dropout
)
```

Note that in order to get performance equal to that of a vanilla feedforward layer (FF) of width `hidden_width`, you might have to choose `leaf_width` and `depth` such that `2**depth * leaf_width > hidden_width`, i.e. such that the training width of the FFF will be larger than the training width of the FF.


## Documentation
Use `help(fastfeedforward.FFF)` to display the following documentation.

```
class FFF(torch.nn.modules.module.Module)
 |  FFF(input_width: int, hidden_width: int, output_width: int, depth: int, activation=ReLU(), dropout: float = 0.0, train_hardened: bool = False)
 |
 |  An implementation of fast feedforward networks from the paper "Fast Feedforward Networks".
 |
 |  Method resolution order:
 |      FFF
 |      torch.nn.modules.module.Module
 |      builtins.object
 |
 |  Methods defined here:
 |
 |  __init__(self, input_width: int, hidden_width: int, output_width: int, depth: int, activation=ReLU(), dropout: float = 0.0, train_hardened: bool = False)
 |      Initializes a fast feedforward network (FFF).
 |
 |      Parameters
 |      ----------
 |      input_width : int
 |              The width of the input, i.e. the size of the last dimension of the tensor passed into `forward()`.
 |      hidden_width : int
 |              The width of every leaf of this FFF.
 |      output_width : int
 |              The width of the output, i.e. the size of the last dimension of the tensor returned by `forward()`.
 |      depth : int
 |              The depth of the FFF tree. Will result to 2**depth leaves.
 |      activation : torch.nn.Module, optional
 |              The activation function to use. Defaults to `torch.nn.ReLU()`.
 |      dropout : float, optional
 |              The probability to use for the dropout at the leaves after the activations have been computed. Defaults to 0.0.
 |      train_hardened : bool, optional
 |              Whether to use hardened decisions during training. Defaults to False.
 |
 |      Raises
 |      ------
 |      ValueError
 |              - if `depth`, `input_width`, `hidden_width` or `output_width` are not positive integers
 |
 |      Notes
 |      -----
 |      - The number of leaves of the FFF will be 2**depth.
 |      - The number of nodes of the FFF will be 2**depth - 1.
 |
 |  eval_forward(self, x: torch.Tensor) -> torch.Tensor
 |      Computes the forward pass of this FFF during evaluation (i.e. making hard decisions at each node and traversing the FFF in logarithmic time).
 |
 |      Parameters
 |      ----------
 |      x : torch.Tensor
 |              The input tensor. Must have shape (..., input_width).
 |
 |      Returns
 |      -------
 |      torch.Tensor
 |              The output tensor. Will have shape (..., output_width).
 |
 |  forward(self, x: torch.Tensor, return_entropies: bool = False)
 |      Computes the forward pass of this FFF.
 |      If `self.training` is True, `training_forward()` will be called, otherwise `eval_forward()` will be called.
 |
 |      Parameters
 |      ----------
 |      x : torch.Tensor
 |              The input tensor. Must have shape (..., input_width).
 |      return_entropies : bool, optional
 |              Whether to return the entropies of the decisions made at each node. Defaults to False.
 |              If True, the mean batch entropies for each node will be returned as a tensor of shape (n_nodes,).
 |
 |      Returns
 |      -------
 |      torch.Tensor
 |              The output tensor. Will have shape (..., output_width).
 |      torch.Tensor, optional
 |              The mean batch entropies for each node. Will be returned with shape (n_nodes,) if `return_entropies` is True.
 |              Will not be returned if `return_entropies` is False.
 |
 |      Raises
 |      ------
 |      ValueError
 |              - if `x` does not have shape (..., input_width)
 |              - if `return_entropies` is True and `self.training` is False
 |
 |      See Also
 |      --------
 |      `training_forward()`
 |      `eval_forward()`
 |
 |  training_forward(self, x: torch.Tensor, return_entropies: bool = False, use_hard_decisions: bool = False)
 |      Computes the forward pass of this FFF during training.
 |
 |      Parameters
 |      ----------
 |      x : torch.Tensor
 |              The input tensor. Must have shape (..., input_width).
 |      return_entropies : bool, optional
 |              Whether to return the entropies of the decisions made at each node. Defaults to False.
 |              If True, the mean batch entropies for each node will be returned as a tensor of shape (n_nodes,).
 |      use_hard_decisions : bool, optional
 |              Whether to use hard decisions during the forward pass. Defaults to False.
 |              If True, the decisions will be rounded to the nearest integer. This will effectively make the FFF tree non-differentiable.
 |
 |      Returns
 |      -------
 |      torch.Tensor
 |              The output tensor. Will have shape (..., output_width).
 |      torch.Tensor, optional
 |              The mean batch entropies for each node. Will be returned with shape (n_nodes,) if `return_entropies` is True.
 |              Will not be returned if `return_entropies` is False.
 |
 |      Notes
 |      -----
 |      - The FFF tree is traversed from the root to the leaves.
 |              At each node, the input is multiplied by the node's weight matrix and added to the node's bias vector.
 |              The result is passed through a sigmoid function to obtain a probability.
 |              The probability is used to modify the mixture of the current batch of inputs.
 |              The modified mixture is passed to the next node.
 |              Finally, the outputs of all leaves are mixed together to obtain the final output.
 |      - If `use_hard_decisions` is True and `return_entropies` is True, the entropies will be computed before the decisions are rounded.
 |
 |      Raises
 |      ------
 |      ValueError
 |              - if `x` does not have shape (..., input_width)
 |
 |      See Also
 |      --------
 |      `eval_forward()`
 |
 |  ----------------------------------------------------------------------
 |  The rest of the methods are inherited from torch.nn.modules.module.Module.
```