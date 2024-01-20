<br />
<div align="center">
  <h1 align="center"><img src="https://i.ibb.co/BNkhQH3/pyvene-logo.png"></h1>
  <a href="https://nlp.stanford.edu/~wuzhengx/"><strong>Library Paper and Doc Are Forthcoming »</strong></a>
</div>     

<br />
<a href="https://pypi.org/project/pyvene/"><img src="https://img.shields.io/pepy/dt/pyvene?color=green"></img></a>
<a href="https://pypi.org/project/pyvene/"><img src="https://img.shields.io/pypi/v/pyvene?color=red"></img></a> 
<a href="https://pypi.org/project/pyvene/"><img src="https://img.shields.io/pypi/l/pyvene?color=blue"></img></a>

*This is a beta release (public testing).*

# A Library for _Understanding_ and _Improving_ PyTorch Models via Interventions
Interventions on model-internal states are fundamental operations in many areas of AI, including model editing, steering, robustness, and interpretability. To facilitate such research, we introduce **pyvene**, an open-source Python library that supports customizable interventions on a range of different PyTorch modules. **pyvene** supports complex intervention schemes with an intuitive configuration format, and its interventions can be static or include trainable parameters.

**Getting Started:** [<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/stanfordnlp/pyvene/blob/main/pyvene/pyvene_101.ipynb) [**Main _pyvene_ 101**]  

## Installation
```bash
pip install pyvene
```

## _Wrap_ , _Intervene_ and _Share_
You can intervene with supported models as,
```python
import torch
import pyvene as pv

_, tokenizer, gpt2 = pv.create_gpt2()

pv_gpt2 = pv.IntervenableModel({
    "layer": 0,
    "component": "mlp_output",
    "source_representation": torch.zeros(
        gpt2.config.n_embd)
}, model=gpt2)

orig_outputs, intervened_outputs = pv_gpt2(
    base = tokenizer(
        "The capital of Spain is", 
        return_tensors="pt"
    ), 
    unit_locations={"base": 3}
)
print(intervened_outputs.last_hidden_state - orig_outputs.last_hidden_state)
```
which returns,
```
tensor([[[ 0.0000,  0.0000,  0.0000,  ...,  0.0000,  0.0000,  0.0000],
         [ 0.0000,  0.0000,  0.0000,  ...,  0.0000,  0.0000,  0.0000],
         [ 0.0000,  0.0000,  0.0000,  ...,  0.0000,  0.0000,  0.0000],
         [ 0.0483, -0.1212, -0.2816,  ...,  0.1958,  0.0830,  0.0784],
         [ 0.0519,  0.2547, -0.1631,  ...,  0.0050, -0.0453, -0.1624]]])
```
You can share your interventions through Huggingface with others with a single call,
```python
pv_gpt2.save(
    save_directory="./your_gpt2_mounting_point/",
    save_to_hf_hub=True,
    hf_repo_name="your_gpt2_mounting_point",
)
```

You can also use the `pv_gpt2` just like a regular torch model component inside another model, or another pipeline as,
```py
import torch
import torch.nn as nn
from typing import List, Optional, Tuple, Union, Dict

class ModelWithIntervenables(nn.Module):
    def __init__(self):
        super(ModelWithIntervenables, self).__init__()
        self.pv_gpt2 = pv_gpt2
        self.relu = nn.ReLU()
        self.fc = nn.Linear(768, 1)
        # Your other downstream components go here

    def forward(
        self, 
        base,
        sources: Optional[List] = None,
        unit_locations: Optional[Dict] = None,
        activations_sources: Optional[Dict] = None,
        subspaces: Optional[List] = None,
    ):
        _, counterfactual_x = self.pv_gpt2(
            base,
            sources,
            unit_locations,
            activations_sources,
            subspaces
        )
        counterfactual_x = counterfactual_x.last_hidden_state
        
        counterfactual_x = self.relu(counterfactual_x)
        counterfactual_y = self.fc(counterfactual_x)
        return counterfactual_y
```


## Selected Tutorials

| **Level** |  **Tutorial** |  **Run in Colab** |  **Description** |
| --- | -------------  |  -------------  |  -------------  | 
| Beginner |  [**pyvene 101**](pyvene_101.ipynb) | [<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/stanfordnlp/pyvene/blob/main/pyvene/pyvene_101.ipynb)  |  Introduce you to the basics of pyvene |
| Intermediate | [**ROME Causal Tracing**](tutorials/advanced_tutorials/Causal_Tracing.ipynb) | [<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/stanfordnlp/pyvene/blob/main/tutorials/advanced_tutorials/Causal_Tracing.ipynb) | Reproduce ROME's Results on Factual Associations with GPT2-XL |
| Intermediate | [**Intervention v.s. Probing**](tutorials/advanced_tutorials/Probing_Gender.ipynb) | [<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/stanfordnlp/pyvene/blob/main/tutorials/advanced_tutorials/Probing_Gender.ipynb) | Illustrates how to run trainable interventions and probing with pythia-6.9B |
| Advanced | [**Trainable Interventions for Causal Abstraction**](tutorials/advanced_tutorials/DAS_Main_Introduction.ipynb) | [<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/stanfordnlp/pyvene/blob/main/tutorials/advanced_tutorials/DAS_Main_Introduction.ipynb) | Illustrates how to train an intervention to discover causal mechanisms of a neural model |

## A Little Guide for Causal Abstraction: From Interventions to Gain Interpretability Insights
Basic interventions are fun but we cannot make any causal claim systematically. To gain actual interpretability insights, we want to measure the counterfactual behaviors of a model in a data-driven fashion. In other words, if the model responds systematically to your interventions, then you start to associate certain regions in the network with a high-level concept. We also call this alignment search process with model internals.

### Understanding Causal Mechanisms with Static Interventions
Here is a more concrete example,
```py
def add_three_numbers(a, b, c):
    var_x = a + b
    return var_x + c
```
The function solves a 3-digit sum problem. Let's say, we trained a neural network to solve this problem perfectly. "Can we find the representation of (a + b) in the neural network?". We can use this library to answer this question. Specifically, we can do the following,

- **Step 1:** Form Interpretability (Alignment) Hypothesis: We hypothesize that a set of neurons N aligns with (a + b).
- **Step 2:** Counterfactual Testings: If our hypothesis is correct, then swapping neurons N between examples would give us expected counterfactual behaviors. For instance, the values of N for (1+2)+3, when swapping with N for (2+3)+4, the output should be (2+3)+3 or (1+2)+4 depending on the direction of the swap.
- **Step 3:** Reject Sampling of Hypothesis: Running tests multiple times and aggregating statistics in terms of counterfactual behavior matching. Proposing a new hypothesis based on the results. 

To translate the above steps into API calls with the library, it will be a single call,
```py
intervenable.evaluate(
    train_dataloader=test_dataloader,
    compute_metrics=compute_metrics,
    inputs_collator=inputs_collator
)
```
where you provide testing data (basically interventional data and the counterfactual behavior you are looking for) along with your metrics functions. The library will try to evaluate the alignment with the intervention you specified in the config.

--- 

### Understanding Causal Mechanism with Trainable Interventions
The alignment searching process outlined above can be tedious when your neural network is large. For a single hypothesized alignment, you basically need to set up different intervention configs targeting different layers and positions to verify your hypothesis. Instead of doing this brute-force search process, you can turn it into an optimization problem which also has other benefits such as distributed alignments.

In its crux, we basically want to train an intervention to have our desired counterfactual behaviors in mind. And if we can indeed train such interventions, we claim that causally informative information should live in the intervening representations! Below, we show one type of trainable intervention `models.interventions.RotatedSpaceIntervention` as,
```py
class RotatedSpaceIntervention(TrainableIntervention):
    
    """Intervention in the rotated space."""
    def forward(self, base, source):
        rotated_base = self.rotate_layer(base)
        rotated_source = self.rotate_layer(source)
        # interchange
        rotated_base[:self.interchange_dim] = rotated_source[:self.interchange_dim]
        # inverse base
        output = torch.matmul(rotated_base, self.rotate_layer.weight.T)
        return output
```
Instead of activation swapping in the original representation space, we first **rotate** them, and then do the swap followed by un-rotating the intervened representation. Additionally, we try to use SGD to **learn a rotation** that lets us produce expected counterfactual behavior. If we can find such rotation, we claim there is an alignment. `If the cost is between X and Y.ipynb` tutorial covers this with an advanced version of distributed alignment search, [Boundless DAS](https://arxiv.org/abs/2305.08809). There are [recent works](https://www.lesswrong.com/posts/RFtkRXHebkwxygDe2/an-interpretability-illusion-for-activation-patching-of) outlining potential limitations of doing a distributed alignment search as well.

You can now also make a single API call to train your intervention,
```py
intervenable.train(
    train_dataloader=train_dataloader,
    compute_loss=compute_loss,
    compute_metrics=compute_metrics,
    inputs_collator=inputs_collator
)
```
where you need to pass in a trainable dataset, and your customized loss and metrics function. The trainable interventions can later be saved on to your disk. You can also use `intervenable.evaluate()` your interventions in terms of customized objectives.

## Contributing to This Library
Please see [our guidelines](CONTRIBUTING.md) about how to contribute to this repository.         

*Pull requests, bug reports, and all other forms of contribution are welcomed and highly encouraged!* :octocat:  

### Other Ways of Installation

**Method 2: Install from the Repo**
```bash
pip install git+https://github.com/stanfordnlp/pyvene.git
```

**Method 3: Clone and Import**
```bash
git clone https://github.com/stanfordnlp/pyvene.git
```
and in parallel folder, import to your project as,
```python
from pyvene import pyvene
_, tokenizer, gpt2 = pyvene.create_gpt2()
```

## Related Works in Discovering Causal Mechanism of LLMs
If you would like to read more works on this area, here is a list of papers that try to align or discover the causal mechanisms of LLMs. 
- [Causal Abstractions of Neural Networks](https://arxiv.org/abs/2106.02997): This paper introduces interchange intervention (a.k.a. activation patching or causal scrubbing). It tries to align a causal model with the model's representations.
- [Inducing Causal Structure for Interpretable Neural Networks](https://arxiv.org/abs/2112.00826): Interchange intervention training (IIT) induces causal structures into the model's representations.
- [Localizing Model Behavior with Path Patching](https://arxiv.org/abs/2304.05969): Path patching (or causal scrubbing) to uncover causal paths in neural model.
- [Towards Automated Circuit Discovery for Mechanistic Interpretability](https://arxiv.org/abs/2304.14997): Scalable method to prune out a small set of connections in a neural network that can still complete a task.
- [Interpretability in the Wild: a Circuit for Indirect Object Identification in GPT-2 small](https://arxiv.org/abs/2211.00593): Path patching plus posthoc representation study to uncover a circuit that solves the indirect object identification (IOI) task.
- [Rigorously Assessing Natural Language Explanations of Neurons](https://arxiv.org/abs/2309.10312): Using causal abstraction to validate [neuron explanations released by OpenAI](https://openai.com/research/language-models-can-explain-neurons-in-language-models).

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=stanfordnlp/pyvene&type=Date)](https://star-history.com/#stanfordnlp/pyvene&Date)

## Citation
Library paper is forthcoming. For now, if you use this repository, please consider to cite relevant papers:
```stex
  @article{geiger-etal-2023-DAS,
        title={Finding Alignments Between Interpretable Causal Variables and Distributed Neural Representations}, 
        author={Geiger, Atticus and Wu, Zhengxuan and Potts, Christopher and Icard, Thomas  and Goodman, Noah},
        year={2023},
        booktitle={arXiv}
  }

  @article{wu-etal-2023-Boundless-DAS,
        title={Interpretability at Scale: Identifying Causal Mechanisms in Alpaca}, 
        author={Wu, Zhengxuan and Geiger, Atticus and Icard, Thomas and Potts, Christopher and Goodman, Noah},
        year={2023},
        booktitle={NeurIPS}
  }
```
