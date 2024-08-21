# Low-rank Adaptation for Fast Text-to-Image Diffusion Fine-tuning

<!-- #region -->
<p align="center">
<img  src="contents/alpha_scale.gif">
</p>
<!-- #endregion -->

> Using LoRA to fine tune on illustration dataset : $W = W_0 + \alpha \Delta W$, where $\alpha$ is the merging ratio. Above gif is scaling alpha from 0 to 1. Setting alpha to 0 is same as using the original model, and setting alpha to 1 is same as using the fully fine-tuned model.


## Main Features

- Fine-tune Stable diffusion models twice as fast than dreambooth method, by Low-rank Adaptation
- Get insanely small end result (1MB ~ 6MB), easy to share and download.
- Compatible with `diffusers`
- Support for inpainting
- Sometimes _even better performance_ than full fine-tuning (but left as future work for extensive comparisons)
- Merge checkpoints + Build recipes by merging LoRAs together
- Pipeline to fine-tune CLIP + Unet + token to gain better results.
- Out-of-the box multi-vector pivotal tuning inversion


# Installation

```bash
pip install git+https://github.com/LiangXin1001/lora.git
```

# Getting Started

## 1. Fine-tuning Stable diffusion with LoRA CLI

If you have over 12 GB of memory, it is recommended to use Pivotal Tuning Inversion CLI provided with lora implementation. They have the best performance, and will be updated many times in the future as well. These are the parameters that worked for various dataset. _ALL OF THE EXAMPLE ABOVE WERE TRAINED WITH BELOW PARAMETERS_

```bash
export MODEL_NAME="runwayml/stable-diffusion-v1-5"
export INSTANCE_DIR="./data/data_disney"
export OUTPUT_DIR="./exps/output_dsn"

lora_pti \
  --pretrained_model_name_or_path=$MODEL_NAME  \
  --instance_data_dir=$INSTANCE_DIR \
  --output_dir=$OUTPUT_DIR \
  --train_text_encoder \
  --resolution=512 \
  --train_batch_size=1 \
  --gradient_accumulation_steps=4 \
  --scale_lr \
  --learning_rate_unet=1e-4 \
  --learning_rate_text=1e-5 \
  --learning_rate_ti=5e-4 \
  --color_jitter \
  --lr_scheduler="linear" \
  --lr_warmup_steps=0 \
  --placeholder_tokens="<s1>|<s2>" \
  --use_template="style"\
  --save_steps=100 \
  --max_train_steps_ti=1000 \
  --max_train_steps_tuning=1000 \
  --perform_inversion=True \
  --clip_ti_decay \
  --weight_decay_ti=0.000 \
  --weight_decay_lora=0.001\
  --continue_inversion \
  --continue_inversion_lr=1e-4 \
  --device="cuda:0" \
  --lora_rank=1 \
#  --use_face_segmentation_condition\
```

[Check here to see what these parameters mean](https://github.com/cloneofsimo/lora/discussions/121).

## 2. Other Options

Basic usage is as follows: prepare sets of $A, B$ matrices in an unet model, and fine-tune them.

```python
from lora_diffusion import inject_trainable_lora, extract_lora_ups_down

...

unet = UNet2DConditionModel.from_pretrained(
    pretrained_model_name_or_path,
    subfolder="unet",
)
unet.requires_grad_(False)
unet_lora_params, train_names = inject_trainable_lora(unet)  # This will
# turn off all of the gradients of unet, except for the trainable LoRA params.
optimizer = optim.Adam(
    itertools.chain(*unet_lora_params, text_encoder.parameters()), lr=1e-4
)
```

Another example of this, applied on [Dreambooth](https://arxiv.org/abs/2208.12242) can be found in `training_scripts/train_lora_dreambooth.py`. Run this example with

```bash
training_scripts/run_lora_db.sh
```

Another dreambooth example, with text_encoder training on can be run with:

```bash
training_scripts/run_lora_db_w_text.sh
```

## Loading, merging, and interpolating trained LORAs with CLIs.

We've seen that people have been merging different checkpoints with different ratios, and this seems to be very useful to the community. LoRA is extremely easy to merge.

By the nature of LoRA, one can interpolate between different fine-tuned models by adding different $A, B$ matrices.

Currently, LoRA cli has three options : merge full model with LoRA, merge LoRA with LoRA, or merge full model with LoRA and changes to `ckpt` format (original format)

```
SYNOPSIS
    lora_add PATH_1 PATH_2 OUTPUT_PATH <flags>

POSITIONAL ARGUMENTS
    PATH_1
        Type: str
    PATH_2
        Type: str
    OUTPUT_PATH
        Type: str

FLAGS
    --alpha
        Type: float
        Default: 0.5
    --mode
        Type: Literal['upl', 'lpl', 'upl', 'upl-ckpt-v2']
        Default: 'lpl'
    --with_text_lora
        Type: bool
        Default: False
```

### Merging full model with LoRA

```bash
$ lora_add PATH_TO_DIFFUSER_FORMAT_MODEL PATH_TO_LORA.safetensors OUTPUT_PATH ALPHA --mode upl
```

`path_1` can be both local path or huggingface model name. When adding LoRA to unet, alpha is the constant as below:

$$
W' = W + \alpha \Delta W
$$

So, set alpha to 1.0 to fully add LoRA. If the LoRA seems to have too much effect (i.e., overfitted), set alpha to lower value. If the LoRA seems to have too little effect, set alpha to higher than 1.0. You can tune these values to your needs. This value can be even slightly greater than 1.0!

**Example**

```bash
$ lora_add runwayml/stable-diffusion-v1-5 ./example_loras/lora_krk.safetensors ./output_merged 0.8 --mode upl
```

### Mergigng Full model with LoRA and changing to original CKPT format

Everything same as above, but with mode `upl-ckpt-v2` instead of `upl`.

```bash
$ lora_add runwayml/stable-diffusion-v1-5 ./example_loras/lora_krk.safetensors ./output_merged.ckpt 0.7 --mode upl-ckpt-v2
```

### Merging LoRA with LoRA

```bash
$ lora_add PATH_TO_LORA1.safetensors PATH_TO_LORA2.safetensors OUTPUT_PATH.safetensors ALPHA_1 ALPHA_2
```

alpha is the ratio of the first model to the second model. i.e.,

$$
\Delta W = (\alpha_1 A_1 + \alpha_2 A_2) (\alpha_1 B_1 + \alpha_2 B_2)^T
$$

Set $\alpha_1 = \alpha_2 = 0.5$ to get the average of the two models. Set $\alpha_1$ close to 1.0 to get more effect of the first model, and set $\alpha_2$ close to 1.0 to get more effect of the second model.

**Example**

```bash
$ lora_add ./example_loras/analog_svd_rank4.safetensors ./example_loras/lora_krk.safetensors ./krk_analog.safetensors 2.0 0.7
```

### Making Text2Img Inference with trained LoRA

Checkout `scripts/run_inference.ipynb` for an example of how to make inference with LoRA.

### Making Img2Img Inference with LoRA

Checkout `scripts/run_img2img.ipynb` for an example of how to make inference with LoRA.

### Merging Lora with Lora, and making inference dynamically using `monkeypatch_add_lora`.

Checkout `scripts/merge_lora_with_lora.ipynb` for an example of how to merge Lora with Lora, and make inference dynamically using `monkeypatch_add_lora`.

<!-- #region -->
<p align="center">
<img  src="contents/lora_with_clip_and_illust.jpg">
</p>
<!-- #endregion -->

Above results are from merging `lora_illust.pt` with `lora_kiriko.pt` with both 1.0 as weights and 0.5 as $\alpha$.

$$
W_{unet} \leftarrow W_{unet} + 0.5 (A_{kiriko} + A_{illust})(B_{kiriko} + B_{illust})^T
$$

and

$$
W_{clip} \leftarrow W_{clip} + 0.5 A_{kiriko}B_{kiriko}^T
$$

---

# Tips and Discussions

## **Training tips in general**

I'm curating a list of tips and discussions here. Feel free to add your own tips and discussions with a PR!

- Discussion by @nitrosocke, can be found [here](https://github.com/cloneofsimo/lora/issues/19#issuecomment-1347149627)
- Configurations by @xsteenbrugge, Using Clip-interrogator to get a decent prompt seems to work well for him, https://twitter.com/xsteenbrugge/status/1602799180698763264
- Super easy [colab running example](https://colab.research.google.com/drive/1iSFDpRBKEWr2HLlz243rbym3J2X95kcy?usp=sharing) of Dreambooth by @pedrogengo
- [Amazing in-depth analysis](https://github.com/cloneofsimo/lora/discussions/37) on the effect of rank, $\alpha_{unet}$, $\alpha_{clip}$, and training configurations from brian6091!

### **How long should you train?**

Effect of fine tuning (both Unet + CLIP) can be seen in the following image, where each image is another 500 steps.
Trained with 9 images, with lr of `1e-4` for unet, and `5e-5` for CLIP. (You can adjust this with `--learning_rate=1e-4` and `--learning_rate_text=5e-5`)

<!-- #region -->
<p align="center">
<img  src="contents/lora_with_clip_4x4_training_progress.jpg">
</p>
<!-- #endregion -->

> "female game character bnha, in a steampunk city, 4K render, trending on artstation, masterpiece". Visualization notebook can be found at scripts/lora_training_process_visualized.ipynb

You can see that with 2500 steps, you already get somewhat good results.

### **What is a good learning rate for LoRA?**

People using dreambooth are used to using lr around `1e-6`, but this is way too small for training LoRAs. **I've tried using 1e-4, and it is OK**. I think these values should be more explored statistically.

### **What happens to Text Encoder LoRA and Unet LoRA?**

Let's see: the following is only using Unet LoRA:

<!-- #region -->
<p align="center">
<img  src="contents/lora_just_unet.jpg">
</p>
<!-- #endregion -->

And the following is only using Text Encoder LoRA:

<!-- #region -->
<p align="center">
<img  src="contents/lora_just_text_encoder.jpg">
</p>
<!-- #endregion -->

So they learnt different aspect of the dataset, but they are not mutually exclusive. You can use both of them to get better results, and tune them seperately to get even better results.

With LoRA Text Encoder, Unet, all the schedulers, guidance scale, negative prompt etc. etc., you have so much to play around with to get the best result you want. For example, with $\alpha_{unet} = 0.6$, $\alpha_{text} = 0.9$, you get a better result compared to $\alpha_{unet} = 1.0$, $\alpha_{text} = 1.0$ (default). Checkout below:

<!-- #region -->
<p align="center">
<img  src="contents/lora_some_tweaks.jpg">
</p>
<!-- #endregion -->

> Left with tuned $\alpha_{unet} = 0.6$, $\alpha_{text} = 0.9$, right with $\alpha_{unet} = 1.0$, $\alpha_{text} = 1.0$.

Here is an extensive visualization on the effect of $\alpha_{unet}$, $\alpha_{text}$, by @brian6091 from [his analysis
](https://github.com/cloneofsimo/lora/discussions/37)

<!-- #region -->
<p align="center">
<img  src="contents/comp_scale_clip_unet.jpg">
</p>
<!-- #endregion -->

> "a photo of (S\*)", trained with 21 images, with rank 16 LoRA. More details can be found [here](https://github.com/cloneofsimo/lora/discussions/37)

---

TODOS

- Make this more user friendly for non-programmers
- Make a better documentation
- Kronecker product, like LoRA [https://arxiv.org/abs/2106.04647]
- Adaptor-guidance
- Time-aware fine-tuning.

# References

This work was heavily influenced by, and originated from these awesome researches. I'm just applying them here.

```bibtex
@article{roich2022pivotal,
  title={Pivotal tuning for latent-based editing of real images},
  author={Roich, Daniel and Mokady, Ron and Bermano, Amit H and Cohen-Or, Daniel},
  journal={ACM Transactions on Graphics (TOG)},
  volume={42},
  number={1},
  pages={1--13},
  year={2022},
  publisher={ACM New York, NY}
}
```

```bibtex
@article{ruiz2022dreambooth,
  title={Dreambooth: Fine tuning text-to-image diffusion models for subject-driven generation},
  author={Ruiz, Nataniel and Li, Yuanzhen and Jampani, Varun and Pritch, Yael and Rubinstein, Michael and Aberman, Kfir},
  journal={arXiv preprint arXiv:2208.12242},
  year={2022}
}
```

```bibtex
@article{gal2022image,
  title={An image is worth one word: Personalizing text-to-image generation using textual inversion},
  author={Gal, Rinon and Alaluf, Yuval and Atzmon, Yuval and Patashnik, Or and Bermano, Amit H and Chechik, Gal and Cohen-Or, Daniel},
  journal={arXiv preprint arXiv:2208.01618},
  year={2022}
}
```

```bibtex
@article{hu2021lora,
  title={Lora: Low-rank adaptation of large language models},
  author={Hu, Edward J and Shen, Yelong and Wallis, Phillip and Allen-Zhu, Zeyuan and Li, Yuanzhi and Wang, Shean and Wang, Lu and Chen, Weizhu},
  journal={arXiv preprint arXiv:2106.09685},
  year={2021}
}
```
