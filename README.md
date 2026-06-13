# Image Watermark & Hiding System

A blind watermarking system supporting multiple algorithms, attack simulation, and real-time quality evaluation. Hide images inside images.

## Overview

This project implements several digital watermarking and steganography algorithms in a unified Python framework. It provides both a Flask web interface for interactive use and a modular codebase that can be used as a library.

Key capabilities:

- Embed a watermark or secret image into a cover image with minimal visual distortion
- Extract the watermark blindly (without the original cover) for most methods
- Simulate 16 types of attacks to test watermark robustness
- Evaluate results with PSNR, SSIM, and NCC metrics

## Supported Methods

| Method | Type | PSNR | Blind | Best For |
|--------|------|------|-------|----------|
| QIM-LL | Wavelet spread-spectrum | ~32 dB | Yes | General purpose, strongest overall |
| DCT | Block DCT QIM | ~24 dB | Yes | JPEG compression |
| Chroma | Cb channel SS | ~33 dB | Yes | Lossless transmission, invisibility |
| SVD | Singular value QIM | ~31 dB | No | JPEG, scaling, filtering |
| FreqBlend | Wavelet SS | ~32 dB | Yes | Simple, no training needed |
| DFT | Wavelet SS + angle search | ~30 dB | Yes | Rotation resistance |
| HiNet | Invertible neural network | ~20 dB | Yes | Deep learning, end-to-end |

## How It Works

### QIM-LL (Recommended Default)

The image is transformed via Haar DWT. The LL subband (low-frequency coefficients) is divided into blocks. Each watermark bit is encoded by adjusting the correlation between block coefficients and a fixed spread-spectrum pattern:

```
correlation = sum(block_centered * pattern)
target = sign * strength * sqrt(N)
adjust coefficients so correlation reaches target
```

At extraction, the same correlation is computed. Positive correlation decodes as 1, negative as 0. Because correlation aggregates signal across all coefficients in a block, it is naturally resistant to noise and mild compression.

### DCT

Each 8x8 DCT block encodes one bit using QIM (Quantization Index Modulation) on 16 mid-frequency AC coefficients. Since JPEG also operates in the DCT domain, this method survives JPEG compression extremely well. The QIM step size is larger than typical JPEG quantization steps for the same coefficients.

### Chroma

Same spread-spectrum approach as QIM-LL, but applied to the Cb channel of YCrCb color space. The human eye is far less sensitive to chroma variations, enabling very high PSNR. However, JPEG aggressively quantizes chroma channels, so this method is only suitable for lossless formats.

### SVD

Each 16x16 image block is decomposed via SVD. The largest singular value is multiplied by (1 + alpha) for bit=1 or (1 - alpha) for bit=0. Extraction compares S[0] between the original and watermarked images. This method shows strong resistance to JPEG, scaling, and median filtering, but requires the original cover for extraction.

### DFT

Uses the same embedding as QIM-LL. At extraction, the image is rotated through 36 angles (0 to 350 degrees, 10-degree steps). For each angle, block correlations are computed and the angle with the strongest correlation signal is selected. This provides genuine rotation resistance without modifying the embedding process.

### HiNet

An invertible neural network based on Haar DWT, coupling layers, and 1x1 convolutions. The forward pass fuses cover and secret into a container; the reverse pass recovers the secret. Training injects Gaussian noise, JPEG quantization, median filtering, scaling, and dropout to build robustness.

## Quick Start

### Installation

```bash
git clone https://github.com/yourname/image-watermark.git
cd image-watermark
pip install -r requirements.txt
```

For GPU support with HiNet:

```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124
```

### Launch Web Interface

```bash
python app.py
```

Open http://127.0.0.1:5000 in your browser. The interface has three tabs:

1. **Image Hiding** — upload a cover image and a secret watermark, select an algorithm, adjust the strength parameter, and embed. Results show the container image, difference map, extracted watermark, and quality metrics (PSNR, SSIM, NCC).

2. **Attack Testing** — upload a watermarked image, select one or more attacks, and see how each attack affects the image. Useful for testing robustness.

3. **Full Pipeline** — combines hiding and attack testing in one step. Upload cover and secret, select attack scope (all 16, basic 9, or real-world 5), and get a complete report with NCC scores for every attack.

### Web Interface Workflow

A typical workflow in the web UI:

1. Select the **Image Hiding** tab
2. Upload a cover image (any size, PNG/JPG supported)
3. Upload a secret watermark (will be resized to match cover)
4. Choose **QIM-LL** from the method selector (recommended)
5. Adjust the delta slider — higher values mean stronger watermarking
6. Click **Start Hiding** to embed
7. Review the container image, difference map, and extracted watermark
8. Switch to the **Full Pipeline** tab, upload the same images, and run attacks
9. Observe which attacks the watermark survives

### Programmatic Usage

You can also use the watermarking algorithms directly in Python without the web server:

```python
import cv2
from watermark import embed_qim_ll, extract_qim_ll, calculate_psnr, calculate_ncc

# Load images
cover = cv2.imread('cover.jpg')
secret = cv2.imread('secret.png')

# Convert secret to grayscale and resize
secret_gray = cv2.cvtColor(secret, cv2.COLOR_BGR2GRAY)
secret_gray = cv2.resize(secret_gray, (cover.shape[1], cover.shape[0]))

# Embed watermark
container = embed_qim_ll(cover, secret_gray, delta=32)

# Extract watermark (blind — no original needed)
extracted = extract_qim_ll(container, secret_gray.shape, delta=32)

# Evaluate quality
psnr = calculate_psnr(cover, container)
ncc = calculate_ncc(secret_gray, extracted)
print(f'PSNR: {psnr:.1f} dB, NCC: {ncc:.4f}')

# Save results
cv2.imwrite('container.png', container)
cv2.imwrite('extracted.png', extracted)
```

Using other algorithms follows the same pattern:

```python
from watermark import embed_dct_full, extract_dct_full
container = embed_dct_full(cover, secret_gray, strength=55, block_group=1)
extracted = extract_dct_full(container, secret_gray.shape, strength=55, block_group=1)

from watermark import embed_chroma_qim, extract_chroma_qim
container = embed_chroma_qim(cover, secret_gray, delta=28)
extracted = extract_chroma_qim(container, secret_gray.shape, delta=28)

from watermark import embed_svd, extract_svd
container = embed_svd(cover, secret_gray, alpha=0.08)
extracted = extract_svd(cover, container, secret_gray.shape, alpha=0.08)
```

Applying attacks:

```python
from attacks import ATTACKS

# Get attack function by key
name, attack_fn = ATTACKS['jpeg_50']
attacked = attack_fn(container)

# Extract from attacked image
extracted_after_attack = extract_qim_ll(attacked, secret_gray.shape, delta=32)
ncc_after = calculate_ncc(secret_gray, extracted_after_attack)
print(f'NCC after {name}: {ncc_after:.4f}')

# Available attack keys:
# gaussian_noise, salt_pepper, jpeg_50, jpeg_30, jpeg_10,
# rotation, rotation_30, scaling, scaling_50, cropping,
# brightness, median_filter,
# social_media, screenshot, print_scan, cascade_hard, cascade_real
```

Running a batch comparison across all methods:

```bash
python test_comparison.py
```

This script runs QIM-LL, FreqBlend, and HiNet side by side on the same test images and saves a comparison grid to `comparison.png`.

### HiNet Training

HiNet is a deep learning model that needs to be trained before use:

```bash
# Step 1: Prepare training data
python prepare_data.py --source synthetic --num_synthetic 200

# Step 2: Train (GPU recommended)
python train.py --data_dir ./data/images --epochs 50 --batch_size 4 --crop_size 128 --num_blocks 4

# Step 3: The web app auto-detects checkpoints/best_model.pt
python app.py
```

Key training parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| --data_dir | required | Directory containing training images |
| --epochs | 100 | Number of training epochs |
| --batch_size | 8 | Batch size (reduce if out of memory) |
| --crop_size | 256 | Random crop size for training patches |
| --num_blocks | 8 | Number of invertible blocks (more = better but slower) |
| --max_images | 5000 | Maximum number of images to use |
| --lr | 2e-4 | Learning rate |
| --noise_strength | 0.02 | Strength of attack noise during training |

Training progress is printed every 50 batches. The model with the best validation loss is saved to `checkpoints/best_model.pt`. Sample visualizations are saved periodically to `checkpoints/samples_epoch_*.png`.

Expected training time on RTX 4070: ~30 minutes for 50 epochs with 300 images at 128x128 crop size.

## Attack Simulations

The system includes 16 attacks organized into three categories:

**Basic (9):** Gaussian noise, salt-and-pepper noise, JPEG compression (Q=50/30/10), rotation (10/30 degrees), scaling (0.8x/0.5x), cropping (20%), median filter (3x3), brightness adjustment (+30)

**Real-world (5):** social media transmission chain, screenshot simulation, print-scan simulation, hard cascade (JPEG+scale+JPEG), real propagation (JPEG+screenshot+social media)

## Performance

Results below are NCC values (higher is better, >0.85 indicates reliable extraction). Tested with a 512x256 cover image using default parameters.

| Method | Clean | JPEG50 | JPEG30 | Scale 0.8x | Salt 2% | Median 3x3 | Crop 20% | Rot 10 |
|--------|-------|--------|--------|------------|---------|------------|----------|--------|
| QIM-LL | 0.99 | 0.98 | 0.97 | 0.95 | 0.75 | 0.98 | 0.92 | ~0 |
| DCT | 1.00 | 1.00 | 1.00 | 0.91 | 0.82 | 0.68 | 0.94 | ~0 |
| SVD | 1.00 | 1.00 | 1.00 | 1.00 | 0.90 | 1.00 | 0.92 | 0.17 |
| DFT | 0.99 | 1.00 | - | 0.99 | 0.80 | - | - | 0.88 |
| HiNet | 0.99 | 0.96 | 0.86 | 0.92 | 0.98 | -0.11 | 0.94 | ~0 |

Notes:
- QIM-LL is the best all-around choice
- DCT is optimal when JPEG compression is expected
- SVD requires the original cover image for extraction
- DFT is the only method with rotation resistance
- HiNet continues to improve with more training

## Parameter Tuning

**QIM-LL delta (strength):**

| Delta | PSNR | Robustness | Use Case |
|-------|------|------------|----------|
| 8-16 | >35 dB | Low | Maximum invisibility |
| 20-28 | 30-34 dB | Medium | Balanced use |
| 32-40 | 27-31 dB | High | General purpose (recommended) |
| 44-48 | <27 dB | Maximum | Extreme conditions |

Higher delta means stronger watermark embedding. This increases robustness against attacks but reduces image quality (lower PSNR). The recommended default of 32 provides a good balance for most scenarios.

**DCT strength:**

| Strength | PSNR | JPEG30 NCC |
|----------|------|------------|
| 30-40 | >30 dB | ~0.8 |
| 45-55 | 23-27 dB | ~1.0 |
| 60-80 | <23 dB | ~1.0 |

## API Endpoints

**POST /api/hide** — Embed watermark
- cover (file): cover image
- secret (file): watermark image
- method (string): algorithm name
- param (float): strength parameter
- Returns: container image, extracted watermark, diff map, PSNR/SSIM/NCC

**POST /api/full-pipeline** — Hide, attack, extract, evaluate
- Additional parameter: attack_mode (all/basic/real)
- Returns: all results with per-attack NCC scores

**POST /api/attack** — Apply attacks to an image

**POST /api/evaluate** — Compare two images (PSNR/SSIM/NCC)

**GET /api/methods** — List available methods and default parameters

## Project Structure

```
app.py                  Flask server and API routes
watermark.py            Core algorithms (QIM-LL, Chroma, DCT, SVD, DFT)
attacks.py              Attack simulation functions
deep_hiding.py          HiNet model and FreqBlend
train.py                HiNet training script
prepare_data.py         Dataset preparation
test_comparison.py      Method comparison script
templates/index.html    Web interface
static/style.css        Styles
static/script.js        Frontend logic
```

## Dependencies

```
flask >= 3.0
opencv-python >= 4.8
numpy >= 1.24
scipy >= 1.10
PyWavelets >= 1.4
Pillow >= 10.0
torch >= 2.0           (for HiNet)
torchvision >= 0.15     (for HiNet)
```

## References

- HiNet: Deep Image Hiding by Invertible Network (ICCV 2021)
- Chen & Wornell: Quantization Index Modulation Methods for Digital Watermarking
- Cox et al.: Secure Spread Spectrum Watermarking for Multimedia

## License

MIT
