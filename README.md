# Unsupervised Singing Voice Separation

Source code of the paper [Zero-Shot Duet Singing Voices Separation with
Diffusion Models](https://sdx-workshop.github.io/papers/Yu.pdf) at the SDX workshop 2023.

## Setup

Install requirements

```bash
pip install -r requirements.txt
```

Add environment variables, rename `.env.tmp` to `.env` and replace with your own variables (example values are random)
```bash
DIR_LOGS=/logs
DIR_DATA=/data

# Required if using wandb logger
WANDB_PROJECT=audioproject
WANDB_ENTITY=johndoe
WANDB_API_KEY=a21dzbqlybbzccqla4txa21dzbqlybbzccqla4tx

# Required if using Common Voice dataset
HUGGINGFACE_TOKEN=hf_NUNySPyUNsmRIb9sUC4FKR2hIeacJOr4Rm
```

## Training

The config we used for the paper is [`exp/singing.yaml`](exp/singing.yaml), you can run it with
```bash
python train.py exp=singing
```
You'll need to download the relevant dataset and resample them to 24 kHz. 
Them, modified the `datamodule` section of the config to point to the right path.

Resume run from a checkpoint

```bash
python train.py exp=singing +ckpt=/logs/ckpts/2022-08-17-01-22-18/'last.ckpt'
```

## Evaluation

First, download the [MedleyVox](https://github.com/jeonchangbin49/MedleyVox?tab=readme-ov-file) dataset.
Then, run the following command to evaluate the model on the `duet` subset of the dataset.

```bash
python eval.py logs/runs/XXXX/.hydra/config.yaml logs/ckpts/XXXX/last.ckpt /your/path/to/MedleyVox -T 100 --cond --hop-length 32768 --self-cond --retry 2
```

Some important arguments:

1. `-T`: number of diffusion steps
2. `--cond`: use auto-regressive conditioning on the ground truth (teacher forcing). Without this flag, the model will generate the full lenght audio at once
3. `--self-cond`: perform auto-regressive conditioning on the generated audio if use together with `--cond`
4. `--hop-length`: the hop length of the moving window
5. `--window`: the size of the moving window. Default to the same length as training data
6. `--retry`: number of retries for each auto-regressive step. The algorithm with generate `retry + 1` candidates and pick the most similar one to the ground truth. Default to 0

For other arguments, please check out the code.

### NMF baseline

This baseline depends on `torchnmf`.

```bash
python eval_nmf.py /your/path/to/MedleyVox/ --thresh 0.08 --division 10 --kernel-size 7
```

### Checkpoint/Logs

Our pre-trained singing voice diffusion model can be downloaded [here](https://drive.google.com/drive/folders/1nAj0JDiG70ddr_7UnhszpIiVCh4SzqgW?usp=sharing).
You can find the training logs and unconditional singing samples generated during training on [wandb](https://api.wandb.ai/links/aimless/fqtcyjke).

## FAQ

<details>
<summary>How do I use the CommonVoice dataset?</summary>

Before running an experiment on commonvoice dataset you have to:
1. Create a Huggingface account if you don't already have one [here](https://huggingface.co/join)
2. Accept the terms of the version of [common voice dataset](https://huggingface.co/mozilla-foundation) you will be using by clicking on it and selecting "Access repository".
3. Add your [access token](https://huggingface.co/settings/tokens) to the `.env` file, for example `HUGGINGFACE_TOKEN=hf_NUNySPyUNsmRIb9sUC4FKR2hIeacJOr4Rm`.

</details>

<details>
<summary>How do I load the model once I'm done training?</summary>

If you want to load the checkpoint to restore training with the trainer you can do `python train.py exp=my_experiment +ckpt=/logs/ckpts/2022-08-17-01-22-18/'last.ckpt'`.

Otherwise if you want to instantiate a model from the checkpoint:
```py
from main.mymodule import Model
model = Model.load_from_checkpoint(
    checkpoint_path='my_checkpoint.ckpt',
    learning_rate=1e-4,
    beta1=0.9,
    beta2=0.99,
    in_channels=1,
    patch_size=16,
    all_other_paratemeters_here...
)
```
to get only the PyTorch `.pt` checkpoint you can save the internal model weights as `torch.save(model.model.state_dict(), 'torchckpt.pt')`.

</details>


<details>
<summary>Why no checkpoint is created at the end of the epoch?</summary>

If the epoch is shorter than `log_every_n_steps` it doesn't save the checkpoint at the end of the epoch, but after the provided number of steps. If you want to checkpoint more frequently you can add `every_n_train_steps` to the ModelCheckpoint e.g.:
```yaml
model_checkpoint:
    _target_: pytorch_lightning.callbacks.ModelCheckpoint
    monitor: "valid_loss"   # name of the logged metric which determines when model is improving
    save_top_k: 1           # save k best models (determined by above metric)
    save_last: True         # additionaly always save model from last epoch
    mode: "min"             # can be "max" or "min"
    verbose: False
    dirpath: ${logs_dir}/ckpts/${now:%Y-%m-%d-%H-%M-%S}
    filename: '{epoch:02d}-{valid_loss:.3f}'
    every_n_train_steps: 10
```
Note that logging the checkpoint so frequently is not recommended in general, since it takes a bit of time to store the file.

</details>
