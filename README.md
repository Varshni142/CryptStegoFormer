# CryptStegoFormer

**Encrypted Adversarial Image Steganography with Vision Transformers**

CryptStegoFormer is a research framework for hiding encrypted messages inside digital images. It combines modern cryptography, error-correction coding, and a Vision Transformer (ViT)–guided adversarial embedding strategy to produce stego images that are visually indistinguishable from their covers and resistant to state-of-the-art deep-learning steganalysis.

---

## How It Works

The framework implements a complete end-to-end pipeline:

```
Secret Message
      │
      ▼
┌─────────────────────┐
│  ECIES Encryption   │  Elliptic-curve encryption (ECDH + HKDF + AES-256-GCM)
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Reed-Solomon ECC   │  RS(255, 223) coding — recovers data after noise/compression
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  ViT Encoder        │  Cross-attention guidance for adaptive, zone-based embedding
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Adversarial        │  Embedding optimized against multiple steganalysis
│  Embedding          │  networks (SRNet, LWENet, SiaStegNet)
└─────────────────────┘
      │
      ▼
Stego Image + Extraction Metadata
```

Extraction reverses the pipeline: the embedded payload is recovered from the stego image, Reed-Solomon decoding corrects channel errors, and the recipient's ECC private key decrypts the original message.

### Key Features

- **🔐 ECIES encryption** — Elliptic Curve Integrated Encryption Scheme with ECDH key exchange, HKDF (SHA-256) key derivation, and AES-256-GCM authenticated encryption; keys are serialized in standard PEM format.
- **🛡️ Reed-Solomon error correction** — RS(255, 223) over GF(256) with 32 bytes of parity per block and adaptive configuration, so messages survive image compression and transmission noise.
- **🧠 Vision Transformer encoder** — multi-head cross-attention integrates the message with image features and identifies optimal embedding regions (configurable depth, heads, and patch size).
- **⚔️ Multi-model adversarial embedding** — Integrated Gradients attribution and Syndrome Trellis Coding (STC) optimize embedding locations against an ensemble of steganalysis CNNs for cross-model detection resistance.
- **📊 Comprehensive evaluation suite** — PSNR, SSIM, hiding capacity, entropy analysis, sparsity metrics, and multi-model security scoring.

---

## Project Structure

```
.
├── models/
│   └── vit_encoder.py            # Enhanced ViT encoder with cross-attention
├── embedding/
│   └── multi_model_adversarial.py  # Adversarial embedder, IG attribution, STC
├── encryption/
│   ├── ecies/                    # ECIES implementation
│   │   ├── impl.py               #   Core ECIES (ECDH, HKDF, AES-256-GCM)
│   │   ├── stegano.py            #   ECIES ↔ steganography integration
│   │   └── demo.py, test*.py     #   Demos and unit tests
│   └── reed_solomon.py           # Steganographic Reed-Solomon system
├── evaluation/
│   ├── metrics.py                # PSNR, SSIM, capacity, efficiency
│   ├── entropy.py                # Entropy-based detectability analysis
│   ├── sparsity.py               # Embedding sparsity metrics
│   └── security.py               # Steganalysis resistance scoring
├── training_setup/
│   ├── train.py / train_real.py  # Model training scripts
│   ├── evaluate.py               # Trained-model evaluation
│   ├── data_preparation.py       # Dataset preprocessing
│   └── check_gpu.py              # CUDA environment check
├── datasets/                     # Dataset handling (BOSSbase demo samples)
├── tests/                        # Unit and integration tests
├── config/                       # Configuration files
├── fixed_complete_demo.py        # ⭐ Main end-to-end interactive demo
├── requirements.txt
└── setup.py
```

Supporting directories (`paper/`, `final_Paper/`, `accuracy/`, `analysis/`, `report_document/`, `conference_paper/`) contain figure generators, performance analyses, and academic documentation for the accompanying research paper.

---

## Installation

### Prerequisites

- Python 3.8+
- CUDA-capable GPU recommended for training (CPU works for the demo)

### Setup

```bash
# Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # Linux / macOS

# Install dependencies
pip install -r requirements.txt
pip install -e .
```

Verify your GPU setup (optional):

```bash
python training_setup/check_gpu.py
```

---

## Quick Start

### Run the End-to-End Demo

The interactive demo walks through the full encrypt → embed → extract workflow:

```bash
python fixed_complete_demo.py
```

**To hide a message** (option 1): provide a recipient public key (or use the included demo key `fixed_public_key.pem`), a cover image, and your message. The demo produces a stego image plus a metadata file containing the ephemeral public key and salt needed for extraction.

**To reveal a message** (option 2): provide the private key (`fixed_private_key.pem` for the demo keys), the stego image, and the metadata file. The pipeline decodes, error-corrects, and decrypts the original message.

> ⚠️ The bundled PEM keys are for demonstration only — generate your own keypair for any real use.

### Programmatic Usage

```python
from encryption.ecies.impl import ECIES
from encryption.reed_solomon import SteganographicReedSolomon, ReedSolomonConfig
from embedding.multi_model_adversarial import create_multi_model_adversarial_embedder

# 1. Generate keys and encrypt
ecies = ECIES()
private_key, public_key = ecies.generate_keypair()

# 2. Configure error correction
rs = SteganographicReedSolomon(ReedSolomonConfig(n=255, k=223, adaptive=True))

# 3. Create the adversarial embedder (trained against multiple steganalysis models)
embedder = create_multi_model_adversarial_embedder(
    {
        'SRNet':      {'type': 'srnet',      'pretrained': False},
        'LWENet':     {'type': 'lwenet',     'pretrained': False},
        'SiaStegNet': {'type': 'siastegnet', 'pretrained': False},
    },
    image_size=(224, 224),
    message_capacity=2176,
    embed_strength=0.1,
)
```

See [fixed_complete_demo.py](fixed_complete_demo.py) for the complete pipeline wiring.

### Training

Training scripts and configuration live in [training_setup/](training_setup/):

```bash
python training_setup/data_preparation.py   # Prepare dataset (e.g., BOSSbase 1.01)
python training_setup/train_real.py         # Train the embedding model
python training_setup/evaluate.py           # Evaluate a trained checkpoint
```

---

## Evaluation

The [evaluation/](evaluation/) package measures both imperceptibility and security:

| Category | Metrics |
|---|---|
| Image quality | PSNR, SSIM, hiding capacity, embedding efficiency |
| Statistical stealth | Image entropy, entropy ratio, distribution analysis |
| Embedding sparsity | Gradient-based sparsity score, L1/L2 norm ratios |
| Detection resistance | Security score against the steganalysis model ensemble |

Reported results on the BOSSbase 1.01 benchmark include **44.72 dB PSNR** stego quality and **~94% undetectability** against the evaluated steganalysis models, with sub-second embedding time. See [accuracy/](accuracy/) and [paper/](paper/) for the full analysis.

---

## Dataset

Experiments use the **BOSSbase 1.01** grayscale image dataset (included as `BOSSbase_1.01.zip`). A small demo subset of PGM images is provided under [datasets/](datasets/) for quick testing.

---

## Documentation

- [CryptStegoFormer_Quick_Start.md](CryptStegoFormer_Quick_Start.md) — 5-minute beginner walkthrough
- [CryptStegoFormer_Beginner_Guide.md](CryptStegoFormer_Beginner_Guide.md) — concepts explained simply
- [CryptStegoFormer_FAQ_Beginners.md](CryptStegoFormer_FAQ_Beginners.md) — frequently asked questions
- [complete_modules_implementation_list.md](complete_modules_implementation_list.md) — full module-by-module reference
- [training_setup/README.md](training_setup/README.md) — training details

---

## Research Context

This framework supports research in:

- Adversarial machine learning for information hiding
- Vision Transformer applications in multimedia security
- Coding-theoretic robustness (Reed-Solomon) in steganographic channels
- Hybrid cryptographic–steganographic system design

If you use CryptStegoFormer in your research, please cite the accompanying paper (see [conference_paper/](conference_paper/)).

---

## Responsible Use

This project is intended for **research, education, and legitimate privacy protection**. Do not use it for unlawful activity. The demo encryption keys included in this repository must never be used to protect real data.

## License

Released under the MIT License.
