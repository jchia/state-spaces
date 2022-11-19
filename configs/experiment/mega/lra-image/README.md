## Mega: SSM Ablations (EMA and S4D)

This branch contains reproductions of the Mega: Moving Average Equipped Gated Attention model. To see all changes, see the PR for this branch (https://github.com/HazyResearch/state-spaces/commit/e9ce652126cc773dcb6bb7d6f7270c425d4a36a2).

| Model                  | Params   | s/epoch   | Val Acc   |
| --------------------   | -------- | --------- | --------- |
| (large) Mega-EMA       | 2.73M    | 180       | 82.56     |
| (large) Mega-EMA-Repro | 2.65M    | 124       | 83.42     |
| (large) Mega-S4D-Real  | 2.65M    | 121       | 84.44     |
| (large) Mega-S4D       | 2.65M    | 122       | 86.22     |
|                        |          |           |
| (small) Mega-EMA       | 299K     | 51        | 81.16     |
| (small) Mega-EMA-Repro | 279K     | 51        | 80.76     |
| (small) Mega-S4D-Real  | 279K     | 54        | 81.20     |
| (small) Mega-S4D       | 279K     | 53        | 81.46     |
|                        |          |           |
| (large) EMA            | 4.35M    | 128       | 70.96     |
| (large) EMA-Repro      | 3.96M    | 119       | 71.52     |
| (large) S4D-Real       | 3.96M    | 105       | 74.30     |
| (large) S4D            | 3.96M    | 105       | 88.28     |
|                        |          |           |
| (small) EMA            | 333K     | 31        | 69.96     |
| (small) EMA-Repro      | 267K     | 30        | 69.38     |
| (small) S4D-Real       | 267K     | 32        | 70.88     |
| (small) S4D            | 267K     | 30        | 82.78     |

These runs correspond to the experiment files
`{large-mega,small-mega,small,large}-{ema,ema-with-s4,s4d-real,s4d}.yaml`
described below.

------------

### Large Mega Models

```
python -m train experiment=mega/lra-image/large-mega-ema
```
Attempted reproduction of full LRA-Image model described in Mega paper, using their official PyTorch module, with all hyperparameters reproduced.


```
python -m train experiment=mega/lra-image/large-mega-ema-with-s4
```
Version of this model with minor changes matching the parameter count of S4.
This does not affect the speed or accuracy of the model; same performance as above.

```
python -m train experiment=mega/lra-image/large-mega-s4d-real
```
Same model but replacing the EMA component with S4D-Real.
This is a version described in the S4D paper with real-valued A matrices,
which has the same "multi-head EMA" interpretation as Mega.

```
python -m train experiment=mega/lra-image/large-mega-s4d
```
Same model but replacing the EMA component with original (complex) S4D.

----------

### Small Mega Models

Small Mega models and additional details about these variants:

```
python -m train experiment=mega/lra-image/small-mega-ema
```
Small version of the same model (using the original Mega EMA layer), with the following changes:
- Model depth halved from 8 to 4
- Model width halved (`d_model`, `d_ffn`, `v` in Table 8)
- Weight decay halved from 0.02 to 0.01
- Training time halved from 200 to 100 epochs (and warmup steps halved)

**Details**: This calls the Mega block (https://github.com/HazyResearch/state-spaces/blob/mega/src/models/sequence/mega.py) which uses the MultiHeadEMA module (https://github.com/HazyResearch/state-spaces/blob/mega/src/models/sequence/ss/ema.py).
Both of these are transcribed from the official Mega code.

```
python -m train experiment=mega/lra-image/small-mega-ema-with-s4
```
Same as above, but computing the EMA module with a codepath that goes through the S4 convolution block (does not use the actual S4 layer). It should be almost exactly the same as the above model.
This config only ablates that the alternate codepath behaves as expected, so that we can confidently replace inner modules (the convolution kernel).

**Details:**
This breaks the MultiHeadEMA module into two parts, the kernel construction and the convolution. The kernel construction module is `EMAKernel` ([code](https://github.com/HazyResearch/state-spaces/blob/17663f26f7e91f88757e1d61318ed216dfb8a8a5/src/models/sequence/ss/kernel.py#L869)) which can be seen as a drop-in replacement of alternative convolution kernels such as S4D's kernel.

The convolution goes through the [S4 block](https://github.com/HazyResearch/state-spaces/blob/mega/src/models/sequence/ss/s4.py), which is just a generic block implementing FFT convolution. The kernel inside is controllable by the `mode=ema` [option](https://github.com/HazyResearch/state-spaces/blob/17663f26f7e91f88757e1d61318ed216dfb8a8a5/src/models/sequence/ss/kernel.py#L1012)) to use the `EMAKernel`.

The swap from `MultiHeadEMA` to `S4Block`+`EMAKernel` is performed [here](https://github.com/HazyResearch/state-spaces/blob/17663f26f7e91f88757e1d61318ed216dfb8a8a5/src/models/sequence/mega.py#L95).

```
python -m train experiment=mega/lra-image/small-mega-s4d-real
```
Same as above, but replaces EMA with an S4D layer that is forced to be real-valued instead of complex-valued.

**Details**: Exactly the same as above but replaces `EMAKernel` with `SSKernelDiag` with the option `force_real=True`. Note that the latter has more features, but the minimal version of it ([here](https://github.com/HazyResearch/state-spaces/blob/17663f26f7e91f88757e1d61318ed216dfb8a8a5/src/models/s4/s4d.py#L16)) is nearly identical to the EMA kernel.

```
python -m train experiment=mega/lra-image/small-mega-s4d
```
Same as above, but with the original (complex-valued) S4D layer.

----------

The `{small,large}-{<model>}.yaml` experiments use a block with only an SSM convolution.
This is a variant of the small ablation models from the S4D paper
(`configs/experiment/cifar/s4-cifar-ablation.yaml`).

```
python -m train experiment=mega/lra-image/small-s4d
```

Pure S4D module from the S4D paper (no Attention).

To make the models more directly comparable, some architecture flags were tweaked to match the Mega models (namely using pre-batch-norm rather than post-layer-norm),
which might lower these results slightly compared to the original S4D results for these model sizes.

```
python -m train experiment=mega/lra-image/small-s4d-real
```
Same S4D module with real constraint (i.e. can be interpreted as multi-head EMA).

```
python -m train experiment=mega/lra-image/small-ema
```
Same as above, but EMA (original Mega module) instead of S4D.

```
python -m train experiment=mega/lra-image/small-ema-with-s4d
```
Same as above, but use settings to match the parameter count of S4D.

-----------

### Earlier runs with different warmup steps

The above configs have been updated with more warmup steps (see [TODO]()).
Earlier versions of these experiments were run where everything was exactly the same except all runs had `scheduler.num_warm_steps=1000`. These are the earlier results.

| Model                  | Params   | s/epoch   | Val Acc   |
| --------------------   | -------- | --------- | --------- |
| (large) Mega-EMA-Repro | 2.65M    | 120       | 82.60     |
| (large) Mega-S4D-Real  | 2.65M    | 121       | 83.98     |
| (large) Mega-S4D       | 2.65M    | 120       | 84.68     |
|                        |          |           |
| (small) Mega-EMA-Repro | 279K     | 48        | 79.98     |
| (small) Mega-S4D-Real  | 279K     | 55        | 81.62     |
| (small) Mega-S4D       | 279K     | 55        | 80.80     |
|                        |          |           |
| (small) EMA-Repro      | 267K      | 30        | 70.74     |
| (small) S4D-Real       | 200K      | 32        | 70.34     |
| (small) S4D            | 200K      | 31        | 84.40     |
