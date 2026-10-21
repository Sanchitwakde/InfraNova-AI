# Training Pipeline Documentation — InfraNova AI

**Document Version:** 2.1 (Exp9 Production Specification)  
**Date:** 2026-09-13  
**Module**: `src/training/` & `kaggle_kernel/`  

---

## 1. Multi-Term Loss Objective

The GAN objective is minimized via two alternating Adam optimizers:
`L_total = λ_adv·L_adv + λ_L1·L_L1 + λ_perc·L_perc + λ_ssim·L_ssim + λ_chroma·L_chroma + λ_feat·L_feat + λ_sat·L_sat`

### Hyperparameter Configurations:

| Hyperparameter | Legacy Baseline (`configs/config.yaml`) | Active Production Exp9 (`configs/exp9_resnet_generator.yaml`) |
| :--- | :---: | :---: |
| **Generator Architecture** | `Pix2PixHDGenerator` (21.38M) | `Pix2PixHDGlobalResNetGenerator` (11.37M) |
| **Adversarial Objective** | BCE (`gan_mode: bce`) | LSGAN (`gan_mode: lsgan`, MSE) |
| λ_adv | 1.0 | 1.0 |
| λ_L1 | 10.0 | 10.0 |
| λ_perc (VGG-19) | 10.0 | **5.0** (Prevents over-smoothing) |
| λ_ssim | 5.0 | 5.0 |
| λ_chroma | 2.0 | 2.0 |
| λ_feat | 5.0 | **0.0** (Ablated in Exp2; omitted in Exp9) |
| λ_sat (CIE Lab Saturation) | 0.0 | **0.05** (ε = 10⁻⁶ for stable gradients) |

---

## 2. Optimizer & Learning Rate Schedule

- **Optimizers**: Adam for both Generator and Discriminators (β₁ = 0.5, β₂ = 0.999, initial lr = 2 × 10⁻⁴).
- **Linear Learning Rate Annealing**:
  - Epochs 1–50 (or 1–100): Constant lr = 2 × 10⁻⁴.
  - Subsequent epochs: Linear decay down to η_min = 1 × 10⁻⁶:
    `lr(e) = η_min + (lr_base - η_min) · (1 - (e - e_decay) / (e_total - e_decay))`

- **Early Stopping**: Early stopping monitor watches validation SSIM with patience (`patience: 25`). In Exp9, peak SSIM occurred at epoch 46, and early stopping cleanly terminated training at epoch 71.
- **Long-Training Study**: Expanding training budget to 250 epochs (patience 100) on Tesla T4 resulted in 84 non-improving epochs post-epoch 46, causing spectral overfitting and saturation inflation. Further epoch increases are exhausted; Exp9 epoch 46 is the finalized production model.

---

## 3. GPU Memory Stability & Checkpoint Management

For long-running training runs:
1. **CPU State Dict Serialization**: When saving checkpoints, model parameters are cloned to CPU memory (`{k: v.detach().cpu() for k, v in model.state_dict().items()}`), preventing GPU VRAM fragmentation.
2. **Rolling Checkpoints**: Maintains rolling checkpoints for `latest.pth`, `best_checkpoint.pth`, and metric-specific peaks (`best_ssim.pth`, `best_psnr.pth`, `best_lab.pth`, `best_sam.pth`, `best_sat_ratio.pth`).
3. **Explicit Garbage Collection**: Invokes `torch.cuda.empty_cache()` and `gc.collect()` at epoch boundaries.
4. **NaN/Inf Shielding**: Gradient clipping and `Trainer._is_finite()` validation protect both generator and discriminator from non-finite weight poisoning during training.
